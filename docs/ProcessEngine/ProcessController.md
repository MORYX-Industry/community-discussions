---
uid: ProcessEngine.ProcessController
---
# ProcessController

The ProcessController is a composition of components hosted inside the ProcessEngine. It does not have a public API on its own. Its main task is to control the processes on the production facility. Most processes are production processes but there also setup and maintenance processes.

## Referenced Facades

Plugin API | Start Dependency | Optional | Usage
-----------|------------------|----------|------
IProductManagement | Yes | No | The ProductManagement is used to create the Article data for each article produced by a production process.
IResourceManagement | Yes | No | The ResourceManagement is used the get the resources needed to execute an activity.

## Used Data Models

- [Moryx.ControlSystem.Model](xref:Moryx.ControlSystem.Model)

One specialty of the data model are the primary keys of `ProcessEntity` and `ActivityEntity`. Unlike most entities within MORYX applications the ids are not generated by database sequences, but instead calculated within the ProcessEngine. For performance gains the ProcessEngine avoids an initial database insert to generate the id. Instead the ProcessEngine generates unique ids for new instances though numerically combining the parents id with the childs index. The combination is achieved by bit-shifting the parent index to the left and use the created space of empty lower bits for the child index:

| Entity | Id-layout
|--------|-----------|
| JobEntity | Job id |
| ProcessEntity | Job id << x \| process index |
| ActivityEntity | Process id << x \| activity index |
|  | Job id << 2x \| process index << x \| activity index. |

One consequence of this approach is that we limit the number of children for each parent instance. Increasing x raises less concerns for each instance, but impacts the available ids for jobs in the long run. Therefore we agreed on 14-bits per generation as a good compromise. This results in the following maximum amounts:

| Entity | Bits | Available ids (per parent) |
|------------|------|----------------------------|
| Activities | 14 | 16.383 |
| Processes  | 14 | 16.383 |
| Jobs | 35 | 34.359.738.368 |

## Internal Components

The following diagram shows the structure of the ProcessController composition.

![Processes](images\Processes.png)

A quick overview of all components within the process execution and transport system.

Component name|Implementation|StartOrder|Description
--------------|--------------|----------|------------
ActivityPool | Internal | - | Holds a list of all current activities and raises events when a new activity is added or an existing activity changes its state.
ProcessController | Internal | -1 | Listens on events of the ActivityPool and provides process changed messages to the outer system. It is the single API to the JobManagement for starting, loading, resuming and aborting processes.
StorageTrigger | Internal | 0 | Listens on events of the ActivityPool and saves the changes of the process, activities or article
ActivityProvider | Internal | 1 | Creates a new Workflow when a process is added. Proceeds in the workflow when an activity changes to 'ResultProcessed'.
ResourceAssignment | Internal | 5 | Searches for resources providing the required capabilities of an activity. Publishes a notifiation, if no matching resources are found. It also listens to capability changes and updates targets if necessary.
ProcessRemoval | Internal | 30 | Makes sure to remove broken or interrupted processes from the physical machine.
FailurePredictor | Internal | 40 | Can predict scrap parts of production processes **before** the process is completed.
ActivityTimeoutObserver | Internal | 45 | Starts a time out for activities with `IActivityTimeoutParameters`. A notification will be thrown if the timeout was reached.
ActivityDispatcher | Internal | 100 | Starts the activity when its state changed to 'Configured' and a resource is found that is 'ReadyToWork'
ProcessInterruption | Internal | 112 |Sets the process to 'Interrupted' when it was stopped and has reached a stable state.
ProcessTargets | Internal | - | Sends an update to the routing system when a 'Configured' activity has been added or changed.
ProcessRouter | Internal | - | Finds the best target resource for the process group.

In addition to the overview table the matrix below illustrates the different state transitions and conditions.

|[ProcessState](xref:Moryx.ControlSystem.ProcessEngine.Processes.ProcessState)|[ActivityState](xref:Moryx.ControlSystem.ProcessEngine.Processes.ActivityState)|Condition|Listener|Action|ProcessState|ActivityState|
|------------------------------------------------------------------------|--------------------------------------------------------------------------|---------|--------|------|------------|-------------|
| Initial | - | - | FailurePredictor | Monitor the engine instance for a restored process | - | - |
| Restored | - | process is ProductionProcess | ArticleArchiver | Get article | RestoredReady | - |
| CleaningUp | | | | | | |
| Ready | - | - | ActivityProvider | Create workflow engine instance | EningeStarted | Initial |
| RestoredReady | - | - | ActivityProvider | Reload workflow engine instance | EningeStarted | Initial |
| Ready | - | - | FailurePredictor | Monitor the engine instance for a restored process | - | Initial |
| EngineStarted | - | - | ActivityDispatcher | Start open activities that were held back before the process was ready | - | - |
| - | Initial | - | ResourceAssignment | Find resources, or notify about unassigable resources | - | Configured |
| >EngineStarted | Configured | Push RTW | ActivityDispatcher | Start activity | Running | Running |
| Running | Running | Result received | ActivityDispatcher| Set result | Running | ResultReceived |
| Running | ResultReceived | - | StorageTrigger | Write to database | Running | ResultProcessed |
| Running | ResultProcessed | Not last transition | ActivityProvider | Resume workflow engine | - | Initial & EngineProceeded |
| Running | EngineProceeded | -| ActivityDispatcher | CompleteSequence | - | Completed |
| Running | ResultProcessed | Last transition | ActivityProvider | Complete workflow | Success or Failure | EngineProceeded |
| Stopping | - | Stable state | ProcessInterruption | Signal stable state | Interrupted or Discarded (no progress worth saving) | - |
| Interrupted | - | - | Storage trigger | Write process to database | - | - |
| Discarded | | | | | | |
| Success | - | - | Storage trigger | Write process to database | - | - |
| Failure | - | - | Storage trigger | Write process to database | - | - |

### The ProcessController

As its name anticipates, the `ProcessController` is the main component of the composition
described here. It provides the external API for other components hosted inside the [ProcessEngine](@ref ProcessEngine).
There is no API for components outside the ProcessEngine. It is started last to ensure all other listeners are already running and is stopped first.

### ActivityPool

The `ActivityPool` is a central component of the ProcessController composition. It holdes the running processes and fires events if a process or activity was added or changed. It does not change a process or activity on its own. Instead the fired event will trigger other components to do their work. Those components will give a feedback if an activity or process has been updated; this feedback will cause a new event.

The ActivityPool can provide the [ActivityData](xref:Moryx.ControlSystem.ProcessEngine.ActivityData) of any running process.

### ActivityProvider

The `ActivityProvider` adds a workflow to each added process
and uses that workflow to determine and create the next activity.
When an activity is 'ResultProcessed' the workflow will be updated with the result of the activity.
In case of a completed workflow the process state will be updated to the workflow result ([Success](xref:Moryx.ControlSystem.ProcessEngine.Processes.ProcessState) or [Failure](xref:Moryx.ControlSystem.ProcessEngine.Processes.ProcessState)).
If the workflow has a next step, it will create a new activity and add it to the ActivityPool.
It also restores the activities of a process, when it is resumed and sets the process to broken, if no snapshot was found.

### ResourceAssignment

The `ResourceAssignment` uses the ResourceManagement to determine all resources with matching capabilities. It passes those resources through the list of configured [ResourceSelectors](xref:Moryx.ControlSystem.ProcessEngine.Processes.IResourceSelector), which can sort and filter it. The resulting targets are assigned to the activity and its state incremented to 'Configured'. If there is no matching resource, a notification will be published. The assignment will be tried again, when the user acknowledges the notification or when a capability change was received.

The resource selector API can be used to influence the ProcessEngines distribution and execution of activities, especially in a manufacturing system with redundant cells for activities. Below is an example for a random based load balancer using the `ResourceSelectorBase` class:

````cs
internal class LoadBalancer : ResourceSelectorBase<LoadBalancerConfig>
{
    public override IReadOnlyList<ICell> SelectResources(IActivityData activityData, IReadOnlyList<ICell> availableCells)
    {
        // Shuffling the list every time should create load balancing
        var resultList = availableCells.ToList();
        resultList.Shuffle();
        return resultList;
    }
}
````

### StorageTrigger

The `StorageTrigger` is a small component that performs necessary database writes for process, activities and articles. Processes are created with the first completed activity. All follow-up activities are added to the job as they are completed.

It uses the ProductManagement to save the article. When a process has changed to 'Finished'-state the archiver checks the process' result to set the article state to 'Success' or 'Failure'. When an activity has changed to `ResultRecieved`-state the storage trigger will safe any changes to the article. It then sets the activity state to `ResultProcessed`

### FailurePredictor

With the use of the `FailurePredictor` the control system is able to predict scrap parts **before** their process is completed. This prediction is based on the `IPathPredictor` of the MORYX Workflow Engine, which can predict the outcome of any workplan by analyzing all paths. The information about predicted failures is published through the job facade and can be used by job creating modules like the _OrderManagement_ to replace scrap parts earlier and increase overall efficiency.

### ActivityTimeoutObserver

The `ActivityTimeoutObserver` starts timers for activities with `IActivityTimeoutParameters`. If the timeout of an activity was reached, a notification will be published.

### ActivityDispatcher

The `ActivityDispatcher` is an interface component to the outer system. Its the connection between the `ResourceManagement` and the ProcessController. It registers to the events of all the cells and reacts on `ReadyToWork`, `NotReadyToWork` and `ActivityCompleted`. In case of a ReadyToWork it differs between the `ReadyToWorkType` 'Push' and 'Pull'. In 'Pull' mode it will answer the message directly with a matching activity or a `SequenceCompleted` message. In 'Push' mode it will answer directly if there's a matching activity or it will store the ready to work and checks it on each activity state change. In both modes it trys to find an activity that must fit in following conditions:

- Resource is among the PossibleResources
- Matching `ActivityClassification` (e.g. 'Production')
- Matching ProcessId
- No process assigned (to the carrier) if the activity requires a empty carrier (e.g. mount activities)
- Process assigned (to the carrier) if the activity requires a filled carrier (e.g. production activities)

If there is a matching activity the Dispatcher calls `StartActivity` on the cell and sets the activity state to 'Running'.
In case of a NotReadyToWork and 'Push' mode, it will remove the ReadyTowork from the store.
In case of a ActivityCompleted the dispatcher will get the activity from the ActivityPool, updates its state to 'ResultReceived'
and sends a 'SequenceComplete' message to the ResourceManagement.
In case of an 'Aborting' process state all activities in the `ActivityPool` belonging to the given process will be set to [Aborted](xref:Moryx.ControlSystem.ProcessEngine.Processes.ActivityState) and will be removed from the `ActivityPool`. In case of running activities the currently executing resource is informed about the process abortion.

When the ControlSystem is shutdown or a process is stopping it stops dispatching activities to break the execution chain reaction.

### ProcessRemoval

The ProcessRemoval checks if a process in state `Aborting` or `RemoveBroken` needs to be removed from the machine. It does that
by comparing the `IMountingActivity` and amount of  `MountOperation.Mount` or `MountOperation.Unmount` in the processes history. If necessary it creates an `ProcessFixupActivity`, otherwise
it updates the state immediatly. The same update otherwise occurs as soon as the `ProcessFixupActivity` was completed.

It also listens to the machines `IProcessReporter` implementations for removed or broken processes. Broken processes are flagged as aborting and follow its behavior through the `ActivityDispater`. Removed processes are directly flagged as `Failure` and any open activities are aborted.

The `ProcessFixupActivity` must be handled by any resource known to the ControlSystem by their associated `ProcessFixupCapabilities`. The activity is extended with `ProcessFixupParameters` with a visual instruction where the text is configured within the `ModuleConfig` of the ControlSystem.

### ProcessInterruption

The `ProcessInterrupted` listens to process and activitiy changes and waits till a stopping process has reached a stable state. A stable state is defined as all activities having either the state `ConfiguredSaved` or `CompletedSaved`.

When the ControlSystem is shutdown it determines a timeout for running activities to complete and processes to reach a stable state. This timeout can be either set in the
config or be calculated by using the average and standard deviation of previous activities execution times. In the config the sigma range can be configured as one of three
values:

| Interval type | Explaination |
|---------------|--------------|
| OneSigma      | Average time plus a single standard deviation. This covers roughly 70% of all activities. |
| TwoSigma      | Average time plus double standard deviation. This covers >90% of activities. |
| ThreeSigma    | Average time plus trippe standard deviation. This will cover >99% of all activities. |
