+++
authors = ["Gaurav Mathur"]
title = "Triggering a custom callback on excessive GC activity in Java"
date = "2023-10-23"
description = "Tiggering a custom callback on excessive GC activity"
tags = [
    "java",
    "java 17",
    "jvm",
    "gc"
]
categories = [
    "java",
    "gc",
]
series = ["Java"]
+++

We might want to invoke some business logic when there is excessive GC activity in out program. The code snippet below shows one way 
of doing that. It defines a `GCAnalyzer` class that takes the following key parameters - 

* `trackedCycles` - The number of consecutive GC cycles that we want to track continously
* `percentThreshold` - If the time take in GC activity exceeds the wall time by this percentage, call the callback
* `thresholdExceededCb` - User supplied callback invoked when the threshold is exceeded

This code tracks GC activity, regardless of either the GC algorithm or GC type for the algorithm. It could be easily tweaked to, say, only track time taken by 
consecutive STW (Stop-The-World) cycles, or consecutive full GC cycles.

It works by keeping track of the GC times in the last `trackedCycles` cycles, and the total time spent from the start of the first cycle, to the 
end of the last one.

<!--more-->
```java

import com.sun.management.GarbageCollectionNotificationInfo;

import javax.management.Notification;
import javax.management.NotificationEmitter;
import javax.management.openmbean.CompositeData;
import java.lang.management.GarbageCollectorMXBean;
import java.lang.management.ManagementFactory;
import java.util.LinkedList;
import java.util.List;
import java.util.function.Consumer;

/**
 * GCAnalyzer monitors garbage collection activities and invokes a callback function
 * when the percentage of time spent in garbage collection, over a specified number
 * of consecutive cycles exceeds a given threshold.
 */
public class GCAnalyzer {
    private List<GarbageCollectorMXBean> gcBeans;
    private LinkedList<Long> durations = new LinkedList<>();
    private LinkedList<Long> startTimes = new LinkedList<>();
    // Number of consecutive GC cycles to keep track of
    private int trackedCycles = 0;
    // Percentage of time spent in GC over the  cycles to trigger the callback function
    private double percentThreshold = 0.0;
    // Callback function to call when the percentage threshold is exceeded
    private Consumer<Double> thresholdExceededCb;

    private final class GCNotificationListener implements javax.management.NotificationListener {
        /**
         * Calculates the percentage of time spent in garbage collection relative to total elapsed time.
         *
         * @param currentEndTime The end time of the current GC cycle.
         * @return The percentage of time spent in GC.
         */
        private double calculateGcPercent(long currentEndTime) {
            long totalDuration = durations.stream().mapToLong(Long::longValue).sum();
            return (double) totalDuration / (currentEndTime - startTimes.getFirst()) * 100.0;
        }

        @Override
        public void handleNotification(Notification notification, Object handback) {
            if (notification.getType().equals(GarbageCollectionNotificationInfo.GARBAGE_COLLECTION_NOTIFICATION)) {
                GarbageCollectionNotificationInfo info = GarbageCollectionNotificationInfo.from((CompositeData) notification.getUserData());
                long duration = info.getGcInfo().getDuration();
                long startTime = info.getGcInfo().getStartTime();
                long endTime = info.getGcInfo().getEndTime();

                if (durations.size() == trackedCycles) {
                    double percent = calculateGcPercent(endTime);
                    if (percent > percentThreshold) {
                        thresholdExceededCb.accept(percent);
                    }

                    durations.removeFirst();
                    startTimes.removeFirst();
                }

                durations.add(duration);
                startTimes.add(startTime);
            }
        }
    }

    /**
     * Registers the GCNotificationListener to all available GarbageCollectorMXBeans.
     */
    private void registerGCNotificationListeners() {
        for (GarbageCollectorMXBean bean : gcBeans) {
            if (bean instanceof NotificationEmitter) {
                NotificationEmitter emitter = (NotificationEmitter) bean;
                emitter.addNotificationListener(new GCNotificationListener(), null, null);
            }
        }
    }

    /**
     * Constructs a GCAnalyzer instance.
     *
     * @param trackedCycles The number of consecutive GC cycles to monitor.
     * @param percentThreshold The threshold of GC activity as a percentage of total time.
     * @param thresholdExceededCb A callback function to invoke when the threshold is exceeded.
     */
    public GCAnalyzer(int trackedCycles, double percentThreshold, Consumer<Double> thresholdExceededCb) {
        this.gcBeans = ManagementFactory.getGarbageCollectorMXBeans();
        this.trackedCycles = trackedCycles;
        this.percentThreshold = percentThreshold;
        this.thresholdExceededCb = thresholdExceededCb;

        registerGCNotificationListeners();
    }
}
```

It could be called from main like this - 

```java
   public static void main(String[] args) {
       ...
        new GCAnalyzer(6,
            70,
            timeSpentInGcInPercent -> System.out.printf("GC Threshold exceeded: %.2f\n", timeSpentInGcInPercent)
        );
       ...
   }
```

In the example above, the registered callback, which prints out the threshold exceeded message, is called when the program spends more than 70% of clock
time in GC activity over the 6 previous GC cycles. Note, the the `GCAnalyzer` implementation is agnostic of the GC type. The program could be easily modified
to filter on specific GC type (e.g. only `G1 Young Gen` etc.)


