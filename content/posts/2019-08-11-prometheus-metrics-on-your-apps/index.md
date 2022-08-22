---
title:  "Enabling Prometheus Metrics on your Applications"
author: "Mario"
tags: [ "prometheus", "golang", "metrics", "development" ]
url: "/prometheus-metrics-on-your-applications/"
draft: false
date: 2019-08-11
#lastmod: 2019-08-11
ShowToc: true
ShowBreadCrumbs: true
ShowReadingTime: true
ShowCodeCopyButtons: true
robotsNoIndex: false
searchHidden: false
ShowWordCount: false
---

# Instrumenting your Applications

We usually see systems being monitored by Ops teams, in fact, there are lots of valuable metrics that help Ops teams understand how the infrastructure they are managing is doing, but when it comes to applications monitoring, we don't see those being monitored that carefully most of the time. Sometimes that ends up in application crashes that might be prevented with a proper monitoring strategy. 

In this blog post we are going to see how we can instrument our applications using **Prometheus metrics libraries**. Prometheus metrics libraries are widely adopted, the Prometheus metrics format has become an independent project, OpenMetrics. OpenMetrics is trying to take Prometheus Metrics Format to the next level making it an industry standard.


## Custom Metrics Example

In this example we are going to use our [Simple Go Application](https://github.com/mvazquezc/reverse-words) as a reference.

Our example application is capable of:

* Reverses a word sent via POST on `/` endpoint
* Returns a release version set via an env var on `/` endpoint
* Returns the hostname of the machine where it's running on `/hostname` endpoint
* Returns the app status on `/health` endpoint

We are going to add the following metrics to our application:

* Total number of words that have been reversed by our application
* Total number of times that a given endpoint has been accessed

> **NOTE**: This is an example application with example metrics, you should think carefully of which metrics do you want to include in your production applications.

## Prometheus Client

The [Prometheus Client](https://prometheus.io/docs/instrumenting/clientlibs/) is available for multiple programming languages, for today's blog post we will be using the [Go Client](https://github.com/prometheus/client_golang).

The `Prometheus Client` provides some metrics enabled by default, among those metrics we can find metrics related to memory consumption, cpu consumption, etc. 

## Enable Prometheus Metrics Endpoint

> **NOTE**: Make sure you're following [metrics name best practices](https://prometheus.io/docs/practices/naming/#metric-names) when defining your metrics.

1. First, we need to import some required modules:

    ```go
    "github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
    ```
2. Define `reversewords_reversed_words_total` metric. This metric is a simple counter metric.

    ```go
    var (
	    totalWordsReversed = prometheus.NewCounter(
		    prometheus.CounterOpts{
			    Name: "reversewords_reversed_words_total",
			    Help: "Total number of reversed words",
		    },
	    )
    )
    ```
3. Define `reversewords_endpoints_accessed_total` metric. This metric is a vector counter metric.

    ```go
    var (
	    endpointsAccessed = prometheus.NewCounterVec(
		    prometheus.CounterOpts{
			    Name: "reversewords_endpoints_accessed_total",
			    Help: "Total number of accessed to a given endpoint",
		    },
		    []string{"accessed_endpoint"},
	    )
    )
    ```
4. Our application has 4 different endpoints as we already seen before, these are the endpoints:

    ```go
	router.HandleFunc("/", ReverseWord).Methods("POST")
	router.HandleFunc("/", ReturnRelease).Methods("GET")
	router.HandleFunc("/hostname", ReturnHostname).Methods("GET")
	router.HandleFunc("/health", ReturnHealth).Methods("GET")
    ```
5. The `reversewords_reversed_words_total` metric will be increased every time the function `ReverseWord` is called:

    ```go
    func ReverseWord(w http.ResponseWriter, r *http.Request) {
	   <OUTPUT_OMITTED>
	    totalWordsReversed.Inc()
	   <OUTPUT_OMITTED>
    }
    ```
6. The `reversewords_endpoints_accessed_total` metric will be increased every time the functions `ReverseWord`, `ReturnRelease`, `ReturnHostname` or `ReturnHealth` are called:

    ```go
    func ReturnRelease(w http.ResponseWriter, r *http.Request) {
        <OUTPUT_OMITTED>
	    endpointsAccessed.WithLabelValues("release").Inc()
    }
    func ReturnHostname(w http.ResponseWriter, r *http.Request) {
        <OUTPUT_OMITTED>
	    endpointsAccessed.WithLabelValues("hostname").Inc()
    }
    func ReturnHealth(w http.ResponseWriter, r *http.Request) {
        <OUTPUT_OMITTED>
	    endpointsAccessed.WithLabelValues("health").Inc()
    }
    func ReverseWord(w http.ResponseWriter, r *http.Request) {
        <OUTPUT_OMITTED>
        endpointsAccessed.WithLabelValues("reverseword").Inc()
    }
    ```
7. Finally, we need to add the `/metrics` endpoint to our application:

    ```go
    router.Handle("/metrics", promhttp.Handler()).Methods("GET")
    ```

## Gathering Metrics

With our application running we can send a GET request to the `/metrics` endpoints to get the metrics:

```sh
$ curl http://127.0.0.1:8080/metrics

# HELP go_gc_duration_seconds A summary of the GC invocation durations.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 0
go_gc_duration_seconds{quantile="0.25"} 0
go_gc_duration_seconds{quantile="0.5"} 0
go_gc_duration_seconds{quantile="0.75"} 0
go_gc_duration_seconds{quantile="1"} 0
go_gc_duration_seconds_sum 0
go_gc_duration_seconds_count 0
# HELP go_goroutines Number of goroutines that currently exist.
# TYPE go_goroutines gauge
go_goroutines 8
# HELP go_info Information about the Go environment.
# TYPE go_info gauge
go_info{version="go1.12.6"} 1
# HELP go_memstats_alloc_bytes Number of bytes allocated and still in use.
# TYPE go_memstats_alloc_bytes gauge
go_memstats_alloc_bytes 457776
# HELP go_memstats_alloc_bytes_total Total number of bytes allocated, even if freed.
# TYPE go_memstats_alloc_bytes_total counter
go_memstats_alloc_bytes_total 457776
# HELP go_memstats_buck_hash_sys_bytes Number of bytes used by the profiling bucket hash table.
# TYPE go_memstats_buck_hash_sys_bytes gauge
go_memstats_buck_hash_sys_bytes 2684
# HELP go_memstats_frees_total Total number of frees.
# TYPE go_memstats_frees_total counter
go_memstats_frees_total 172
# HELP go_memstats_gc_cpu_fraction The fraction of this program's available CPU time used by the GC since the program started.
# TYPE go_memstats_gc_cpu_fraction gauge
go_memstats_gc_cpu_fraction 0
# HELP go_memstats_gc_sys_bytes Number of bytes used for garbage collection system metadata.
# TYPE go_memstats_gc_sys_bytes gauge
go_memstats_gc_sys_bytes 2.240512e+06
# HELP go_memstats_heap_alloc_bytes Number of heap bytes allocated and still in use.
# TYPE go_memstats_heap_alloc_bytes gauge
go_memstats_heap_alloc_bytes 457776
# HELP go_memstats_heap_idle_bytes Number of heap bytes waiting to be used.
# TYPE go_memstats_heap_idle_bytes gauge
go_memstats_heap_idle_bytes 6.5347584e+07
# HELP go_memstats_heap_inuse_bytes Number of heap bytes that are in use.
# TYPE go_memstats_heap_inuse_bytes gauge
go_memstats_heap_inuse_bytes 1.368064e+06
# HELP go_memstats_heap_objects Number of allocated objects.
# TYPE go_memstats_heap_objects gauge
go_memstats_heap_objects 2079
# HELP go_memstats_heap_released_bytes Number of heap bytes released to OS.
# TYPE go_memstats_heap_released_bytes gauge
go_memstats_heap_released_bytes 0
# HELP go_memstats_heap_sys_bytes Number of heap bytes obtained from system.
# TYPE go_memstats_heap_sys_bytes gauge
go_memstats_heap_sys_bytes 6.6715648e+07
# HELP go_memstats_last_gc_time_seconds Number of seconds since 1970 of last garbage collection.
# TYPE go_memstats_last_gc_time_seconds gauge
go_memstats_last_gc_time_seconds 0
# HELP go_memstats_lookups_total Total number of pointer lookups.
# TYPE go_memstats_lookups_total counter
go_memstats_lookups_total 0
# HELP go_memstats_mallocs_total Total number of mallocs.
# TYPE go_memstats_mallocs_total counter
go_memstats_mallocs_total 2251
# HELP go_memstats_mcache_inuse_bytes Number of bytes in use by mcache structures.
# TYPE go_memstats_mcache_inuse_bytes gauge
go_memstats_mcache_inuse_bytes 6944
# HELP go_memstats_mcache_sys_bytes Number of bytes used for mcache structures obtained from system.
# TYPE go_memstats_mcache_sys_bytes gauge
go_memstats_mcache_sys_bytes 16384
# HELP go_memstats_mspan_inuse_bytes Number of bytes in use by mspan structures.
# TYPE go_memstats_mspan_inuse_bytes gauge
go_memstats_mspan_inuse_bytes 19440
# HELP go_memstats_mspan_sys_bytes Number of bytes used for mspan structures obtained from system.
# TYPE go_memstats_mspan_sys_bytes gauge
go_memstats_mspan_sys_bytes 32768
# HELP go_memstats_next_gc_bytes Number of heap bytes when next garbage collection will take place.
# TYPE go_memstats_next_gc_bytes gauge
go_memstats_next_gc_bytes 4.473924e+06
# HELP go_memstats_other_sys_bytes Number of bytes used for other system allocations.
# TYPE go_memstats_other_sys_bytes gauge
go_memstats_other_sys_bytes 527748
# HELP go_memstats_stack_inuse_bytes Number of bytes in use by the stack allocator.
# TYPE go_memstats_stack_inuse_bytes gauge
go_memstats_stack_inuse_bytes 393216
# HELP go_memstats_stack_sys_bytes Number of bytes obtained from system for stack allocator.
# TYPE go_memstats_stack_sys_bytes gauge
go_memstats_stack_sys_bytes 393216
# HELP go_memstats_sys_bytes Number of bytes obtained from system.
# TYPE go_memstats_sys_bytes gauge
go_memstats_sys_bytes 6.992896e+07
# HELP go_threads Number of OS threads created.
# TYPE go_threads gauge
go_threads 8
# HELP process_cpu_seconds_total Total user and system CPU time spent in seconds.
# TYPE process_cpu_seconds_total counter
process_cpu_seconds_total 0
# HELP process_max_fds Maximum number of open file descriptors.
# TYPE process_max_fds gauge
process_max_fds 1024
# HELP process_open_fds Number of open file descriptors.
# TYPE process_open_fds gauge
process_open_fds 7
# HELP process_resident_memory_bytes Resident memory size in bytes.
# TYPE process_resident_memory_bytes gauge
process_resident_memory_bytes 7.548928e+06
# HELP process_start_time_seconds Start time of the process since unix epoch in seconds.
# TYPE process_start_time_seconds gauge
process_start_time_seconds 1.56553026534e+09
# HELP process_virtual_memory_bytes Virtual memory size in bytes.
# TYPE process_virtual_memory_bytes gauge
process_virtual_memory_bytes 5.00559872e+08
# HELP process_virtual_memory_max_bytes Maximum amount of virtual memory available in bytes.
# TYPE process_virtual_memory_max_bytes gauge
process_virtual_memory_max_bytes -1
# HELP promhttp_metric_handler_requests_in_flight Current number of scrapes being served.
# TYPE promhttp_metric_handler_requests_in_flight gauge
promhttp_metric_handler_requests_in_flight 1
# HELP promhttp_metric_handler_requests_total Total number of scrapes by HTTP status code.
# TYPE promhttp_metric_handler_requests_total counter
promhttp_metric_handler_requests_total{code="200"} 0
promhttp_metric_handler_requests_total{code="500"} 0
promhttp_metric_handler_requests_total{code="503"} 0
# HELP reversewords_reversed_words_total Total number of reversed words
# TYPE reversewords_reversed_words_total counter
reversewords_reversed_words_total 0
```

Now, let's see how our metrics increase as we use our application:

```sh
$ curl -s http://127.0.0.1:8080/ -X POST -d '{"word":"PALC"}'
{"reverse_word":"CLAP"}

$ curl -s http://127.0.0.1:8080/health
Healthy

$ curl -s http://127.0.0.1:8080/hostname
Hostname: reverse-words-22j33j

$ curl -s http://127.0.0.1:8080/metrics | grep "reversewords_"

# HELP reversewords_endpoints_accessed_total Total number of accessed to a given endpoint
# TYPE reversewords_endpoints_accessed_total counter
reversewords_endpoints_accessed_total{accessed_endpoint="health"} 1
reversewords_endpoints_accessed_total{accessed_endpoint="hostname"} 1
reversewords_endpoints_accessed_total{accessed_endpoint="reverseword"} 1
# HELP reversewords_reversed_words_total Total number of reversed words
# TYPE reversewords_reversed_words_total counter
reversewords_reversed_words_total 1
```

As you can see the `reversewords_reversed_words_total` has increased by 1 and the `reversewords_endpoints_accessed_total` metrics now show the total number of times a given endpoint has been accessed.

# Next Steps

In a future blog post we are going to show how we can configure Prometheus to scrape our metrics endpoint and how Grafana can help us to create graphs that can be consumed by monitoring teams.

# Useful Resources

If you want to learn more, feel free to take a look at the resources below.

* [https://sysdig.com/blog/prometheus-metrics/](https://sysdig.com/blog/prometheus-metrics/)
* [https://prometheus.io/docs/guides/go-application/](https://prometheus.io/docs/guides/go-application/)
