<!--more-->

# 1. Scheduler

| Value | type |
| --- | --- |
| **allInstances** | Set&lt;Instance&gt; |
| **allInstancesByHost** | HashMap&lt;String, Set<Instance&gt;&gt; |
| **instancesWithAvailableResources** | Map&lt;ResourceID, Instance&gt; |
| **taskQueue** | ArrayDeque&lt;QueuedTask&gt; |
| **newlyAvailableInstances** | LinkedBlockingQueue&lt;Instance&gt; |
| **unconstrainedAssignments** | int |
| **localizedAssignments** | int |
| **nonLocalizedAssignments** | int |
| **executor** | Executor |


b2072642f819916b3310e7ee7b407195
d03a77eccd34dd757412c0605cf70b99 @ 192.168.0.102 - 1 slots - URL: akka.tcp://flink@lmsdemacbook-pro-2.local:53155/user/taskmanager

b2072642f819916b3310e7ee7b407195
d03a77eccd34dd757412c0605cf70b99 @ 192.168.0.102 - 1 slots - URL: akka.tcp://flink@lmsdemacbook-pro-2.local:53155/user/taskmanager


