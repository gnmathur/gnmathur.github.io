+++
authors = ["Gaurav Mathur"]
title = "Exporting metrics to Prometheus using JMX JavaAgent"
date = "2023-10-28"
description = "Exporting metrics to Prometheus using JMX JavaAgent"
tags = [
    "java",
    "prometheus",
    "JVM",
    "JMX",
    "MBean"
]
categories = [
    "java",
    "JMX",
    "JVM",
    "prometheus",
]
series = ["Java"]
+++

Java [JMX](https://www.oracle.com/technical-resources/articles/javase/jmx.html) is a powerful technology that's available in Java natively. JXM has many uses and can form the basis of a powerful application management and monitoring system. What we will focus on in this article is the capability of JMX to provide a rich set of JVM metrics out-of-the-box, including - 

* Garbage collection statistics
* Memory usage (heap and non-heap)
* Thread metrics
* Class loading stats
* And much more...

There are tools like [VisualVM](https://visualvm.github.io/), [YourKit](https://www.yourkit.com/) etc. that you can use to view these metrics by attaching these tools to a running JVM process. What if we want to _export_ these metrics and view them in a Grafana? If your Grafana installation supports Prometheus as a data source, then one way to do this is via a [JMX to Prometheus exporter](https://github.com/prometheus/jmx_exporter).

Next, we'll walk through the steps to use the JMX exporter JavaAgent to pull metrics from a Java application and make them available for Prometheus to scrape.

# Pre-requisites

* A Java application with JMX enabled
* Prometheus up and running (and optionally a Grafana installation with Prometheus as a Data Source)
* The JMX exporter jar, which can be [downloaded from GitHub](https://github.com/prometheus/jmx_exporter/releases)

# Configuring JMX Exporter

Before running your Java application, you'll need a configuration file for the JMX exporter. This file helps specify which JMX metrics you're interested in and how they should be exported.

A basic configuration (let's call it `config.yml`) might look like:

```yml
rules:
- pattern: '.*'
```

This configuration will export all JMX metrics. However, for most applications, you'd want more specific rules to filter or rename metrics. Refer to [this page](https://github.com/prometheus/jmx_exporter#configuration) for a reference of all the keywords and expressiveness that can be added to the configuration. Also, the [example_configs](https://github.com/prometheus/jmx_exporter/tree/main/example_configs) folder in the JMX exporter project has many examples for configuring popular software. The examples can be used to better understand the different configuration constructs and idioms.

# Running your Java application with jmx_exporter

Use the `-javaagent` JVM argument to attach the JMX exporter to your application:

```bash
java -javaagent:/path/to/jmx_prometheus_javaagent-<version>.jar=<port>:<path_to_config_file> -jar your-java-app.jar
```

Replace `version`, `port`, and `path_to_config_file` with appropriate values. Once your application is running, it will start serving metrics on the specified port under the `/metrics` endpoint.

Note: The JMX exporter JavaAgent supports built-in HTTP server

# Configuring Prometheus to scrape metrics

Now, update Prometheus's configuration to scrape the metrics:

```yml
scrape_configs:
  - job_name: 'jmx-exporter'
    static_configs:
    - targets: ['localhost:<port>']
```

Replace <port> with the port number you specified while starting the Java application.

# Viewing Metrics 

## In Prometheus for verification

Once everything is set up and running:

* Navigate to the Prometheus UI.
* Select the jmx-exporter job.
* Browse the available metrics, which will have names based on their JMX names.

# Exporting application metrics via JMX

While these built-in metrics offer valuable insights into the JVM's internals, there's another layer of observability you can tap into leveraging the capability of JMX:  your application's business logic and performance related metrics. These application-specific metrics will be exported over the same interface, over which the JVM metrics are being exported. So, in addition to the standard JMX metrics, you can instrument your application to expose custom JMX MBeans that reflect specific behaviors or characteristics important to your domain. Here's how you can do it:

## Define your MBean Interface

This interface will declare the methods you want to expose via JMX.

```java
public interface MyApplicationMetricsMBean {
    int getInterfaceTxBytes();
    int getInterfaceRxBytes();
    long getAuthSuccessCount();
    long getAuthRejectCount();
}
```

## Implement the MBean: This class provides the actual metric values.

```java

public class MyApplicationMetrics implements MyApplicationMetricsMBean {
    public int getInterfaceTxBytes() {
        // Your logic to return the number of Tx bytes goes here
    }
    public int getInterfaceRxBytes() {
        // Your logic to return the number of Rx bytes goes here
    }
    public long getAuthSuccessCount() {
        // Your logic to return the successful authentication attempts goes here
    }
    public long getAuthRejectCount() {
        // Your logic to return failed authentication attempts goes here

    }
}
```

## Register the MBean with the MBeanServer:

```java

MBeanServer mbs = ManagementFactory.getPlatformMBeanServer();
ObjectInstance objectInstance = mbs.registerMBean(new MyApplicationMetrics(), new ObjectName("com.myapp:type=MyApplicationMetrics"));
```

## Configure JMX Exporter to Export Your Custom Metrics

When setting up your JMX Exporter configuration file, ensure that your custom metrics' names or patterns are included, so they're picked up and exported to Prometheus.

With these steps, not only will you have a rich set of JVM metrics, but you'll also be able to monitor and alert on metrics that directly pertain to your application's business logic and performance. This combination makes for a powerful and uniform observability solution for your application, allowing you to understand both the health of your JVM and the behavior of your application.

# Conclusion

The JMX exporter offers a seamless way to export JMX metrics from Java applications to Prometheus. By transforming these metrics into a Prometheus-friendly format, you can leverage all the powerful querying, alerting, and dashboarding features that Prometheus provides. Whether you're running a single Java application or overseeing a fleet of microservices, integrating JMX metrics with Prometheus can provide a deeper understanding of your system's health and performance.


