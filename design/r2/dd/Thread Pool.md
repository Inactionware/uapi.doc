Thread Pool
======

# Introduction
The main target for Thread Pool is provide threads to execte `Behavior` which is defined in `Responsible`, there is a manager thread has resposiblity to scan all `Responsible` and fetch all tasks and dispatch thread to execute `Behavior` which is task associated with it by `Responsible` priority.

# Feature List
## Manager Thread - F2.7.1
1. A manager thread is always running and is able to manager all thread in the pool.
2. A manager thread is able to get notification if new task is added.

## Scheduling Algorithm - F2.7.2
1. An interface for Schduling Algorithm.
2. Default scheduling is priority based.
# Class Diagram
```mermaid
classDiagram

class IThreadManager {
	<<interface>>	
	+register(IResponsible responsible)
}
Runnable <|.. IThreadManager
IThreadManager <|.. ThreadManager

class ThreadManager {
	<<@Service>>
	-IResponsible[] responsibles
}
ThreadManager "1" *-- "1" TaskMonitor : monitor
ThreadManager "1" *-- "1" IScheduling

class IScheduling {
	<<interface>>
	+nextTask() ITask
}
IScheduling <|.. DefaultScheduling

class DefaultScheduling {
	<<@Service>>
}

class ITaskMonitor {
	<<interface>>
	+newTaskAdded(IPrioritizedTask task)
}

class TaskMonitor {
	
}
ITaskMonitor <|.. TaskMonitor

class TaskHandler {
	+setTask(IPrioritizedTask task): void
}
Runnable <|.. TaskHandler

class IPrioritized {
	<<interface>>
	+priority() int
}

class ITask {
	<<interface>>
	+execute()
}

class IPrioritizedTask {
	<<interface>>
}
ITask <|-- IPrioritizedTask
IPrioritized <|-- IPrioritizedTask
```
# Key Workflows
## Add New Incoming Task
```mermaid
sequenceDiagram
autonumber

Client->>+IResponsible: sendMessage(msg)
IResponsible->>+ITaskMonitor: newTaskAdd(task)
ITaskMonitor->>+ThreadManager: newTaskAdd(task)
ThreadManager->>+IScheduling: newTaskAdd(task)
IScheduling-->>-ThreadManager: void
ThreadManager-->-ITaskMonitor: void
ITaskMonitor-->>-IResponsible: void
IResponsible-->>-Client: void
```
## Execute Task
```mermaid
sequenceDiagram
autonumber

ThreadManager->>+IScheduling: nextTask()
IScheduling-->>-ThreadManager: task
ThreadManager->>+TaskHandler: setTask(task)
TaskHandler-->>-ThreadManager: void
ThreadManager->>+TaskHandler: notify()
TaskHandler-->>-ThreadManager: void
TaskHandler->>+ITask: execute()
ITask-->>-TaskHandler: done()
TaskHandler->>+ThreadManager: update()
ThreadManager-->>TaskHandler: void
```
## Default Schduling Algorithm