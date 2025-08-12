---
RFC ID: 0001
Title: Elasticache CPUUtilization alarm threshold calculation
Author: Pranav
Date: 2025/08/07
---

# ELASTICACHE *CPUUtilization* ALARM THRESHOLD CALCULATION

## Objective
* Accurately monitor elasticache instance's overall CPU utilization, including redis engine processes and other processes running on the instance. 
* Monitor redis workload from host-level perspective.
* Use cloudwatch to monitor the elasticache cluster performance by collecting the metric data from every node in our cluster. Understand what might be impacting the performance, set alarms based on threshold that correspond to their workload. 

## Summary
### Issue
`CPUUtilization` metric for elasticache tracks the average CPU utilization across all the CPU cores on the node. Since Redis OSS is single-threaded and most of the workload is tied directly to the single vCPU core used by redis, hardcoding the value of alarm threshold to about 80~90% of the average CPU usage is ineffective as redis engine can become bottleneck long before the overall CPU usage reaches a saturation level.

### Proposal
We propose adopting the AWS recommended best practice for setting the threshold, where we need to determine the threshold based on the number of cores in the cache node that we are using. We calculate the threshold value as a fraction of the node's total capacity. For example, suppose we want to set the threshold at 90% of the available CPU and we are using a node type that has two cores. In this case, the threshold for `CPUUtilization` would be 90/2, or 45%. 
By implementing this approach we can define the alarm threshold that adapts to the number of vCPU cores the underlying node has.

## Overview
In order to calculate the alarm threshold, we need to understand the architecture of the elasticache node that we are currently using. The 'helium-stg-redis' cluster has 3 nodes, each with `cache.t3.medium` as the node type, which comes with two vCPUs. As we know redis OSS is single-threaded which means it uses single thread(one vCPU core) to process client commands. Since we are using redis OSS v5.0.6, the networking processing is also handled on the same thread. Where as the other available vCPU is used for background tasks like replication, system-level operations etc,.

We need to understand that most of the application's performance is directly related to the health of the vCPU core that redis uses to process commands, because in a typical workload `EngineCPUUtilization`(percentage of CPU usage on the redis thread) is often the one contributing to the overall CPU usage across the node. The total CPU usage by the non-redis vCPU thread will be significantly lower in comparision to the main thread unless the background tasks are heavy. 

For example, lets assume that the `EngineCPUUtilization` is 75% and the CPU used by the non-redis thread is 5%. The average `CPUUtilization` reported will only be 40%.

$$
\text{Average CPU Utilization} = \frac{\text{CPU}_{\text{core1}} + \text{CPU}_{\text{core2}}} {\text{Total number of vCPU cores }}  = \frac{75 + 5}{2} = 40\% 
$$

It is important to note that, `CPUUtilization` will rarely be higher than the `EngineCPUUtilization` unless the second vCPU is doing significant work. From the above example, we can conclude that setting the `CPUUtilization` alarm threshold simply around 80~90% will make the alarm ineffective as the average CPU utilization will never reach these level in most cases and also it we be too high of a threshold since the redis's CPU core would already have reached maximum CPU saturation level by the time we catch the issue. To justify this, lets say that the redis core's CPU usage has maxed out(100%) and CPU usage on the second core is around 10%. In that case the average CPU utilization will be, ((100 + 10) / 2) = 55%. As we can see, the avg CPU utilization even after redis core completely exhausting the available CPU is significantly lower than the threshold that we were trying to set.

## Alarm threshold calculation
* AWS suggests that we should be concerned when the redis-engine thread reaches 90% CPU utilization. 
* To calculate the alarm threshold, we should distribute the maximum load that the redis core can handle(90%), across the total number of cores on the node.

$$
\text{Threshold} = \frac{\text{Target CPU usage threshold for redis core}} {\text{Total number of vCPU cores}} = \frac{90\%} {2}  = 45\% 
$$

So, setting the threshold to about 45% to trigger the alarm when the average CPU utilization is greater than 45% will be the most effective way to monitor `CPUUtilization` metric and catch the application performance bottleneck before its too late. 

## Considerations
### Do we need to consider adding expected CPU usage on non-redis cores while calculating the threshold?
Lets assume that the we want to get alerted when redis-engine thread hits 90% CPU usage. Lets also assume a baseline threshold for the second CPU thread as 10%. 

$$
\text{Proposed Threshold} = \frac{90\% \,+\, 10\%}{2} = 50\% 
$$

$$
\text{AWS recommended threshold} = \frac{90\% + 0\%} {2} = 45\%
$$

Lets imagine a scenario where redis engine hits 90% CPU usage but the other core is running at just 5% CPU usage.
The actual CPU utilization that cloudwatch will report would be,

$$
\text{Actual CPU} = \frac{90\% + 5\%}{2} = 47.5\% 
$$

As we can see, 47.5% is below the proposed threshold of 50%, which means alarm won't be triggered, and the critical redis bottleneck goes unnoticed when redis thread reaches 90% CPU usage. 

With the above example scenario we can conclude that by setting a higher threshold based on assumed baseline CPU usage on non-redis CPU cores, we make the alarm less sensitive and risk missing the earliest signs of a critical redis-engine bottleneck. So its safer to set the threshold on non-redis CPU cores to 0% as recommended by AWS's best practices.  

## Glossary
| **Keyword** | **Description** |
| ------ | ----------- |
| CloudWatch   | CloudWatch is AWS's monitoring and observability service. For elasticache, it collects metrics such as *CPUUtilization* and *EngineCPUUtilization*. These metrics can be used to set up alarms and take action when resource usage crosses a threshold. |
| Redis OSS    | In context to elasticache, *Redis OSS* refers to open source software version of redis, which is a compatable engine used for in-memory caching. It stores data entirely in-memory for fast read and writes.  Redis stopped being open source from v7.4 onwards.|
| *CPUUtilization* | A CloudWatch metric that measures the percentage of the total CPU capacity used by an entire elasticache host. For multi-core instances, this is averaged across all vCPUs. It includes CPU consumed by the redis engine, background tasks and other OS-level processes. |
| *EngineCPUUtilization*    | A CloudWatch metric that measures the percentage of CPU used only by the redis thread. Since redis is single threaded, this metric is a good indicator of how much pressure is being placed on the redis engine itself. |
| I/O    | In context to elasticache redis, I/O refers to input/output operations between clients and the redis server. *input* includes the commands sent by clients to redis and *output* includes the responses returned by redis.|
| bottleneck  | A situation where CPU component hits its performance limit and subsequently constraints the overall speed and the performance of the application. |
| Valkey  | Valkey is an open source high performance key-value datastore. It is designed as a drop-in replacement for Redis OSS. Valkey is a fork of Redis OSS 7.2. Valkey was created to continue the development of a fully open-source, redis compatible in-memory data store. |

## References
1. https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Best_Practice_Recommended_Alarms_AWS_Services.html#ElastiCache
2. https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/CacheMetrics.WhichShouldIMonitor.html
3. https://aws.amazon.com/blogs/database/boosting-application-performance-and-reducing-costs-with-amazon-elasticache-for-redis/
4. https://aws.amazon.com/blogs/database/monitoring-best-practices-with-amazon-elasticache-for-redis-using-amazon-cloudwatch/
5. https://aws.amazon.com/blogs/database/enhanced-io-multiplexing-for-amazon-elasticache-for-redis/
6. https://redis.io/blog/redis-adopts-dual-source-available-licensing/
7. https://aws.amazon.com/blogs/database/amazon-elasticache-and-amazon-memorydb-announce-support-for-valkey/
