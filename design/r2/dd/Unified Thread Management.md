Thread Pool
======

# Introduction
The main target for Thread Pool is provide threads to execte `Task` by priority, there is a manager thread has resposiblity to scan all `Task` in the `Task` repository.

# Feature List
## Manager Thread - F2.7.1
1. A manager thread is always running and is able to manager all thread in the pool.
2. A manager thread is able to get notification if new task is added.

## Scheduling Algorithm - F2.7.2
1. An interface for Schduling Algorithm.
2. Default scheduling is priority based.
# Class Diagram
## Exposed APIs
```mermaid
classDiagram

class ITask {
	<<interface>>
	+execute()
}

class ITaskSender {
	<<interface>>
	+sendTask(ITask task)
}

class ITaskRepository {
	<<interface>>
	+int DEFAULT_PRIORITY = 10
	+initTaskSender(String name) ITaskSender
	+initTaskSender(String name, int priority) ITaskSender
}
```

## Implementation

```mermaid
classDiagram

class ITask {
	<<interface>>
}

class ITaskSender {
	<<interface>>
}

class TaskSender {
	
}
ITaskSender <|.. TaskSender

class ITaskRepository {
	<<interface>>
}

class TaskRepository {
	<<@Service>>
	-Map~String, TaskSender~ taskSenders
	-PrioritiedTaskCache taskCache
	-newTask(int prority, ITask task)
	~getNewTaskPriorities() List~int~
	~getTask(int priority) ITask
}
ITaskRepository <|.. TaskRepository
TaskRepository "1" *-- "*" ITaskSender
TaskRepository "1" *-- "1" PrioritiedTaskCache
TaskRepository "1" *-- "1" NewTaskNotifier
TaskRepository "1" *-- "*" ITask

class PrioritiedTaskCache {
	-Map~int, ITask~ taskBundles
	+putTask(int priority, ITask task)
}

class ThreadManager {
	<<@Service>>
}
Runnable <|.. ThreadManager
ThreadManager "1" *-- "1" TaskRepository
ThreadManager "1" *-- "1" IScheduling
ThreadManager "1" *-- "*" TaskHandler

class IScheduling {
	<<interface>>
	+nextTask() ITask
}
IScheduling <|.. PriorityScheduling

class PriorityScheduling {
	<<@Service>>
}
PriorityScheduling "1" *-- "1" TaskRepository

class NewTaskNotifier {
	+notify()
}
PriorityScheduling "1" *-- "1" NewTaskNotifier

class TaskHandler {
	+setTask(ITask task): void
}
Runnable <|.. TaskHandler
```
# Key Workflows
## Register
```mermaid
sequenceDiagram
autonumber

Client->>+TaskRepository: initTaskSender
TaskRepository->>+TaskSender: new
TaskSender-->>-TaskRepository: sender
TaskRepository-->>-Client: sender
```

## Add New Incoming Task
```mermaid
sequenceDiagram
autonumber

Client->>+TaskSender: sendTask(task)
TaskSender->>+TaskRepository: newTask(priority, task)
TaskRepository->>+PrioritiedTaskCache: putTask(priority, task)
PrioritiedTaskCache-->>-TaskRepository: void
TaskRepository->>+NewTaskNotifier: notify()
NewTaskNotifier-->>-TaskRepository: void
TaskRepository-->>-TaskSender: void
TaskSender-->>-Client: void
```
## Execute Task by Priority
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

### Scan time based scheduling
* Same priority task will be put in same FIFO queue.
* All task queue has a value to record its real priority.
* Scheduling will scan All task queue to find out the queue which has most highest real priority and then take task from it.
* The real priorty is dynamic, its calculation is:
	* The real priority value will be decrease (-1) when the scheduling take a task from the queue.
	* The real priority value will be increase (priority * coeficient) when the scheduling does not take any task from the queue.

_BAD_: Scan all task queue and increase/decrease the real priority will consume a lot of time.

### Execution time based scheduling
* Same priority task will be put in same FIFO queue.
* Each task queue has a value to trace the execution time for the task queue.