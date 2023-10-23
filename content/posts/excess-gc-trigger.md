+++
authors = ["Gaurav Mathur"]
title = "Triggering a custom callback on excessive GC activity"
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

We might want to invoke some business logic when there is excessive GC activity in your program. The code snippet below shows one way 
of doing that. It defines a `GCAnalyzer` class that takes the following key parameters - 

* `cycles` - The number of consecutive GC cycles that we want to track continously
* `percentThreshold` - If the time take in GC activity exceeds the wall time by this percentage, call the callback
* `callback` - User supplied callback invoked when the threshold is exceeded

This code track GC activity, regardless of either the GC type of GC generation type. It could be easily tweaked to, say, only track time taken by 
consequitive STW (Stop-The-World) cycles, or consecutive full GC cycles.

It works by keeping track of the GC times in the last `cycle` cycles, and also the total time spent from the start of the first cycle, to the 
end of the last one.

<!--more-->

```
import javax.management.Notification;
import javax.management.NotificationEmitter;
import javax.management.openmbean.CompositeData;
import java.lang.management.GarbageCollectorMXBean;
import java.lang.management.ManagementFactory;
import java.util.ArrayList;
import java.util.List;
import java.util.function.Consumer;

public class GCAnalyzer {
    private List<GarbageCollectorMXBean> gcBean;
    private List<Long> durations = new ArrayList<>();
    private List<Long> startTimes = new ArrayList<>();
    // Number of consecutive GC cycles to keep track of
    private int cycles = 0;
    // Percentage of time spent in GC over the  cycles to trigger the callback function
    private double percentThreshold = 0.0;
    // Callback function to call when the percentage threshold is exceeded
    private Consumer<Double> callback;

    private final class NotificationListener implements javax.management.NotificationListener {
        @Override
        public void handleNotification(Notification notification, Object handback) {
            if (notification.getType().equals(GarbageCollectionNotificationInfo.GARBAGE_COLLECTION_NOTIFICATION)) {
                GarbageCollectionNotificationInfo info = GarbageCollectionNotificationInfo.from((CompositeData) notification.getUserData());

                long duration = info.getGcInfo().getDuration();

                if (durations.size() == cycles) {
                    // Get the start time of the first GC in the list
                    long startTime = startTimes.get(0);
                    // Get the end time of the last GC in the list
                    long endTime = info.getGcInfo().getEndTime();
                    // Sum up the durations of all GCs in the list
                    long totalDuration = durations.stream().reduce(0L, Long::sum);
                    // Calculate the percentage of time spent in GC
                    double percent = (double) totalDuration / (endTime - startTime) * 100.0;
                    // If the percentage is greater than the threshold, call the callback
                    if (percent > percentThreshold) {
                        callback.accept(percent);
                    }

                    // remove the first element from the list
                    durations.remove(0);
                    startTimes.remove(0);
                }
                durations.add(duration);
                startTimes.add(info.getGcInfo().getStartTime());
            }

        }
    }

    public GCAnalyzer(int cycles, double percentThreshold, Consumer<Double> callback) {
        this.gcBean = ManagementFactory.getGarbageCollectorMXBeans();
        this.cycles = cycles;
        this.percentThreshold = percentThreshold;
        this.callback = callback;

        for (GarbageCollectorMXBean bean : gcBean) {
            if (bean instanceof NotificationEmitter) {
                NotificationEmitter emitter = (NotificationEmitter) bean;
                emitter.addNotificationListener(new NotificationListener(), null, null);
            }
        }
    }
}
```

It could be called from main like this - 

```
   public static void main(String[] args) {
        ...
        new GCAnalyzer(6,
                0.1,
                timeSpentInGcInPercent -> System.out.printf("GC Threshold exceeded: %.2f\n", timeSpentInGcInPercent)
        );
        ...
    }
```
