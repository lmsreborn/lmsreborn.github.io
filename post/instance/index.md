<!--more-->
To jobManager, a instance represents a taskManager.

# 1. Instance

| Value | type |
| --- | --- |
| **taskManagerGateway** | TaskManagerGateway |
| **location** | TaskManagerLocation |
| **resources** | HardwareDescription |
| **instanceId** | InstanceID |
| **numberOfSlots** | int |
| **availableSlots** | Queue&lt;Integer&gt; |
| **allocatedSlots** | Set&lt;Slot&gt; |
| **slotAvailabilityListener** | SlotAvailabilityListener |
| **lastReceivedHeartBeat** | System.currentTimeMillis() |

## - TaskManagerLocation
| Value | type |
| --- | --- |
| **resourceID** | ResourceID |
| **inetAddress** | InetAddress |
| **fqdnHostName** | String |
| **hostName** | String |
| **dataPort** | int |

**Attention**: The ID of the resource in which the TaskManager is started. This can be, for example, the YARN container ID, Mesos container ID, or any other unique identifier


## - HardwareDescription
HardwareDescription describe the resources like **CPU cores**, **physical memory**, **free memory**, and **managed Memory**.

## - ad
adasd

# 2. InstanceManager

| Value | type |
| --- | --- |
| **registeredHostsById** | Map&lt;InstanceID, Instance&gt; |
| **registeredHostsByResource** | Map&lt;ResourceID, Instance&gt; |
| **deadHosts** | Set&lt;ResourceID&gt; |
| **instanceListeners** | List&lt;InstanceListener&gt; |
| **totalNumberOfAliveTaskSlots** | int |

**Attention**: A resourceID here identifies a **FlinkResourceManager**.

-------------------

# The process of **registration**: 

## **1. JobManager:**

1. The instance manager receives **`RegisterTaskManager`** message
2. Construct a `taskManagerGateway`
3. Calls `registerTaskManager` funtion with the information about hardware and number of slots.(The information is in the message reported by the task manager) **--> InstanceManager**
4. Put the task manager into the `taskManagerMap`
5. Sucess, acknowledge registration, or refuse.

## **2. InstanceManager** (registerTaskManager)
1. Check `registeredHostsByResource` with the task manager's info. If existed, throw exception.
2. Remove form `deadHosts`.
3. Construct a new instance.
4. Put into `registeredHostsById`.
5. Put into `registeredHostsByResource`.
6. Add `registeredHostsByResource`.
7. Update the instance's `lastReceivedHeartBeat`.
8. Notify every `instanceListeners` that new instance is available now.(call function `newInstanceAvailable`)**-->Scheduler**(newInstanceAvailable)

## **3.Scheduler** (newInstanceAvailable)
1. Add into scheduler's `allInstances`
2. Set the instance's `slotAvailabilityListener` to this Scheduler

3. Add into `allInstancesByHost`. A host name holds a set of instances.

4. Put into `instancesWithAvailableResources` with &lt;TaskManagerID,instance&gt;
5. Add all slots as available **-->Scheduler**(newSlotAvailable) 
    
## **4.Scheduler** (newSlotAvailable)
1. Add into `newlyAvailableInstances`
2. **executor.execute** function `handleNewSlot`

<font color=red>**Attention**</font>:

```
// WARNING: The asynchrony here is necessary, because  we cannot guarantee the order
// of lock acquisition (global scheduler, instance) and otherwise lead to potential deadlocks:
// 
// -> The scheduler needs to grab them (1) global scheduler lock
//                                     (2) slot/instance lock
// -> The slot releasing grabs (1) slot/instance (for releasing) and
//                             (2) scheduler (to check whether to take a new task item
// 
// that leads with a high probability to deadlocks, when scheduling fast
```

## **5.Scheduler** (handleNewSlot - asynchrony)

1. Pop a instance from `newlyAvailableInstances`(This instance may not be the same one)
2. Get a QueuedTask from `taskQueue`.
3. If the queuedTask is null, which means no queuedTask, add the instance into `instancesWithAvailableResources`
4. If not null, instance allocate a simple slot



