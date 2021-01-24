Unified Thread Management
======

# Introduction
The main target for Thread Pool is provide threads to execute `Task` by priority, there is a manager thread has responsibility to scan all `Task` in the `Task` repository.

| Feature | Status |
| ------ | ------ |
| [[Thread Pool Based Behavior Execution#Basic Thread Management]] | Processing |

# Feature List
## Manager Thread
1. A manager thread is always running and is able to manager all thread in the pool.
2. A manager thread is able to get notification if new task is added.

## Scheduling Algorithm
1. An interface for Schduling Algorithm.
2. Default scheduling is priority based.

# Implementation

## Class Diagram
Refer [[Thread Pool Based Behavior Execution#UTM APIs]] to see the APIs:

```mermaid
classDiagram

class ITask {
	<<interface>>
}

class ITaskChannel {
	<<interface>>
}

class TaskChannel
ITaskChannel <|.. TaskChannel

class ITaskRepository {
	<<interface>>
}

class TaskRepository {
	<<@Service>>
	-Map~String, TaskChannel~ taskChannels
	-PrioritiedTaskCache taskCache
	-newTask(int prority, ITask task)
	~getNewTaskPriorities() List~int~
	~getTask(int priority) ITask
	~buildTaskCache(int type) ITaskCache
}
ITaskRepository <|.. TaskRepository
TaskRepository "1" *-- "*" TaskChannel
TaskRepository "1" *-- "1" PrioritiedTaskCache
TaskRepository "1" *-- "1" NewTaskNotifier

class ITaskCache {
	<<interface>>
}

class PrioritiedTaskCache {
	-Map~int, ITask~ taskBundles
	+putTask(int priority, ITask task)
}
ITaskCache <|.. PrioritiedTaskCache
PrioritiedTaskCache "1" *-- "*" ITask

class ThreadManager {
	<<@Service>>
}
Runnable <|.. ThreadManager
ThreadManager "1" *-- "1" TaskRepository
ThreadManager "1" *-- "1" IScheduling
ThreadManager "1" *-- "*" TaskHandler
ThreadManager "1" *-- "1" NewTaskNotifier

class IScheduling {
	<<interface>>
	+nextTask() ITask
}
IScheduling <|.. PriorityScheduling

class PriorityScheduling {
	+int TYPE = 1
}
PriorityScheduling "1" *-- "1" PrioritiedTaskCache

class NewTaskNotifier {
	+notify()
}

class TaskHandler {
	+setTask(ITask task): void
}
Runnable <|.. TaskHandler

class IQueue~T~ { }

class IFlexibleQueue~T~ { }
IQueue <|-- IFlexibleQueue

class FastQueue { }
IQueue <|.. FastQueue

class FlexibleFastQueue { }
IFlexibleQueue <|.. FlexibleFastQueue

```

## Key Workflows
### Register

```mermaid
sequenceDiagram
autonumber

Client->>+TaskRepository: initTaskSender
TaskRepository->>+TaskSender: new
TaskSender-->>-TaskRepository: sender
TaskRepository-->>-Client: sender
```

### Add New Incoming Task

```mermaid
sequenceDiagram
autonumber

Client->>+TaskChannel: putTask(task)
TaskChannel->>+TaskRepository: newTask(priority, task)
TaskRepository->>+PrioritiedTaskCache: putTask(priority, task)
PrioritiedTaskCache-->>-TaskRepository: void
TaskRepository->>+NewTaskNotifier: notify()
NewTaskNotifier->>+ThreadManager: notify()
ThreadManager-->>-NewTaskNotifier: void
NewTaskNotifier-->>-TaskRepository: void
TaskRepository-->>-TaskChannel: void
TaskChannel-->>-Client: void
```

### Execute Task by Priority

```mermaid
sequenceDiagram
autonumber

ThreadManager->>+PriorityScheduling: nextTask()
PriorityScheduling->>+TaskRepository: getNewTaskPriorities()
TaskRepository-->>-PriorityScheduling: List<int>
PriorityScheduling->>+TaskRepository: getTask(priority)
TaskRepository-->>-PriorityScheduling: task
PriorityScheduling-->>-ThreadManager: task
ThreadManager->>+TaskHandler: setTask(task)
TaskHandler->>+Thread: notify()
Thread-->>-TaskHandler: void
TaskHandler-->>-ThreadManager: void
```

## Priority based Scheduling Algorithm
### Pure priority based scheduling
* Same priority task will put in same FIFO queue.
* Scheduling get task first from higher prority queue until the queue goes to empty then get task from lower prority queue.

==BAD: It is not fair, lower priority task will be not executed if higher priority task always exists in the queue==

### Scan time based scheduling
* Same priority task will be put in same FIFO queue.
* All task queue has a value to record its real priority.
* Scheduling will scan All task queue to find out the queue which has most highest real priority and then take task from it.
* The real priorty is dynamic, its calculation is:
	* The real priority value will be decrease (-1) when the scheduling take a task from the queue.
	* The real priority value will be increase (priority * coeficient) when the scheduling does not take any task from the queue.

==BAD: Scan all task queue and increase/decrease the real priority will consume a lot of time.==

### Execution time based scheduling
* Same priority task will be put in same FIFO queue.
* Each task queue has a value to trace the execution time for the task queue.