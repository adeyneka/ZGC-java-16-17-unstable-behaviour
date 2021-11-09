# ZGC java 16-17 unstable behaviour
We are running a GC intensive application with allocation rate up to 1.5GB/s(4k requests/sec) on 32v cores instance (-Xms20G -Xmx20G).  
openjdk 17 2021-09-14  
OpenJDK Runtime Environment (build 17+35-2724)  
OpenJDK 64-Bit Server VM (build 17+35-2724, mixed mode, sharing)  

![Number Of requests](Requests.png?raw=true "Requests")  

During migration from java 15 to java 16-17 we see unstable behavior when zgc goes to infinite loop and doesn't return back to the normal state.  
We see the following behavior:  
- it runs perfectly(~30% better then on java 15) - 5-7GB used memory with 16% CPU utilization  
- something happens and CPU utilization jumps to 40%, used memory jumps to 19GB, zgc cycles time is increased in 10times

![CPU utilization](IInstance%20CPU%20utilization.png?raw=true "CPU Utilization")  
![ZGC cycles time](ZGC%20cycles%20time.png?raw=true "ZGC cycles time")  
![Heap](heap.png?raw=true "Heap")  

- sometimes app works about 30min before the crash sometimes it's crashed instantly  
- we see allocation stalls  
- we see the following GC stats:   
[17606.140s][info][gc,phases   ] GC(719) Concurrent Process Non-Strong References 25781.928ms  
[17610.181s][info][gc,stats    ] Subphase: Concurrent Classes Unlink   14280.772 / 25769.511  1126.563 / 25769.511   217.882 / 68385.750   217.882 / 68385.750   ms  
- we see JVM starts massively unload classes (jvm_classes_unloaded_total counter keep increasing)

![JVM class unloaded](JVM%20class%20unloaded.png?raw=true "JVM class unloaded")  
![class unloading](class%20unloading.png?raw=true "class unloading")  

- we’ve managed to profile JVM in such ‘unstable’ state and see ZTask consuming a lot of cpu doing CompiledMethod::unload_nmethod_caches(bool) job  

![ZGC CPU profiling during the problem](ZGC%20CPU%20profiling%20during%20the%20problem.png?raw=true "ZGC CPU profiling during the problem")  

- we see ICBufferFull safepoint events(it's blue on the image)  

![ICBufferFull](ICBufferFull.png?raw=true "ICBufferFull")  

```
void ZNMethod::unlink(ZWorkers* workers, bool unloading_occurred) {
  for (;;) {
    ICRefillVerifier verifier;

    {
      ZNMethodUnlinkTask task(unloading_occurred, &verifier);
      workers->run(&task);
      if (task.success()) {
        return;
      }
    }

    // Cleaning failed because we ran out of transitional IC stubs,
    // so we have to refill and try again. Refilling requires taking
    // a safepoint, so we temporarily leave the suspendible thread set.
    SuspendibleThreadSetLeaver sts;
    InlineCacheBuffer::refill_ic_stubs();
  }
```

- we see big difference in metaspace allocation between java 15 and java 17(1.5kB/s avg. vs 40mb/s)  

![metaspace allocation java 15](metaspace%20allocation%20java%2015.png?raw=true "metaspace allocation java 15")  
![metaspace allocation java 17](metaspace%20allocation%20java%2017.png?raw=true "metaspace allocation java 17")  

# Some more description of the problem we faced.  

So baseline was jdk 15 with ZGC. Everything worked fine without any significant issues, sometimes we could see allocation stalls in gc logs but rarely. 
We tried originally to upgrade to jdk 16 without changing anything but runtime, right from the beginning got unstable behaviour with a high load of ZWorker threads performing class unlink, jvm afterwards always crashed with OutOfMemory: metaspace. This was the result of excessive class generation (by tuning jackson) that could hit more than 1M classes loaded. We removed this excessive class loading limiting loaded classes to ~50k. We stopped receiving any metaspace OOM errors since then but started observing strange behaviour when we still see high ZWorker with unlink which becomes a precursor for crash, but here jvm silently crashes just stops responding and processing anything, memory usage as we see prior to crash reaches maximum OU = OC (jcmd -gc stats are available when jvm stops responding). Prior to the crash we see lots of allocation stalls and massive class unloading. Behaviour is consistent across jvm 16 and jvm 17. We experimented with metaspace cleaning policies as well as with adjusting its sizes as well as with various different zgc parameters without any luck. It is actually possible to heal jvm before it crashes i.e. when we see a spike in ZWorker load and the load persists at high level if we remove the load from this instance (switch traffic to some other instances) some time afterwards ZWorkers can come to normal state, anyway this does not heal jvm but lets it not to crash.  
Here is parameters we tries to change:  
- -XX:ZCollectionInterval=5
- -XX:SoftMaxHeapSize=12G
- -XX:ZAllocationSpikeTolerance=4
- -XX:MetaspaceReclaimPolicy=aggressive

# Fixes we have done to improve situation
Most of our load is serialization with jackson.  
The most impact on generated classes was from module https://github.com/FasterXML/jackson-modules-base/tree/master/blackbird.  
We disabled it and we could survive more time till the problem.  
Also we have found problem when we used ObjectMapper in wrong way(we copied ObjectMapper for some requests and it generated a lot of classes starting from java 16):  
```
ObjectMapper objectMapper = OBJECT_MAPPER.copy()
                    .setFilterProvider(dynamicFilters);
```
But it just postpones the problem.
