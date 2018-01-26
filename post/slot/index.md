<!--more-->

# 1. Slot
![](/img/Slot.jpg)

- state: slot自身的状态
    
    - **ALLOCATED_AND_ALIVE**
    - **CANCELLED**: State where the slot has been canceled and is in the process of being released
    - **RELEASED**: State where all tasks in this slot have been canceled and the slot been given back to the instance
    
- allocationID:
- [allocatedSlot](#AllocatedSlot)
- [parent](#SharedSlot): The parent of this slot in the hierarchy, or null, if this is the parent
- groupID: The id of the group that this slot is allocated to. May be null.

# 3. SlotID
SlotID

# 4. <span id="AllocatedSlot">AllocatedSlot</span>

![](/img/AllocatedSlot.jpg)



## 1. AllocationID
Slot allocation identifier, created by the JobManager when requesting a slot, constant across re-tries. Used to identify responses by the ResourceManager and to identify deployment calls towards the TaskManager that was allocated from.

## 2.test



