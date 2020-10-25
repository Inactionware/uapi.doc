Execution of Behavior
======

# Terminology

* UTM - Unified Thread Management

# Introduction
For any application, thread management should be a framework level task, since the thread count should be based on hardware limitation. 
The UAPI framework provides `Behavior` to describe task content and `Responsible` to execute its `Behavior`, but `Responsible` has no thread to execute `Behavior`, so the `UTM` to provide thread for `Responsible` to execute `Behavior`.

Each `Responsible` own its task list, the task contains `Behavior` which request the corresponding `Responsible` to execute, the `Responsible` has not thread to execute these tasks.

The `UTM` manages a bundle thread - a thread pool, there is a system level thread which has highest priority and is aways runing, the system leve thread scans all task list in the `Responsible` and execute task by `Responsible`'s priority. 

# Workflow
There are two ways to execute `Behavior`:
* Execute directly
* Send message

## Execute Directly
When a `Behavior` needs invoke other `Behavior` to get its result, the invocation is sync, since the `Behavior` is an `Action`, so `Responsible` can specify a `Behavior` when constructing `Behavior`.

* Using `this::Behavior Name` to specify a `Behavior` which is defined in this `Responsible`
* Using `Responsible Name::Behavior Name` to specify a `Behavior` which is defined in other `Behavior`
* `::` is reseved keywor, `Responsibile` and `Behavior` name is not allowed to contain the keyword. 

## Send Message
When a `Responsible` needs notify some data to other `Responsible`, it will construct a message and send it to one or more `Responsible`, the message will wrapped to a task and put in the `Responsible`'s task list, and then the `UTM` will scan all `Responsible`'s task list, and dispatch thread to execute task by `Responsible`'s priority.

To specify which `Responsible` needs to send message:
* Using `SendMessage` action to send message to one specified `Responsible`, like below:
```java
responsible.newBehavior(...)
	...
	.then(SendMessage.class, "<Responsible Name>", <message data>)
	...
	.build();
```
* Using `Broadcast` action to send message to more `Responsible` which has subscribed the message, like below:
```java
responsible.newBehavior(...)
	...
	.then(Broadcast.clas, "<Message Topic>", <message data>)
	...
	.build();
```

# Submodule features

## UTM
* A main thread is always running and is able to manager all thread in the pool.
* The main thread has a `Responsible` list which has new incoming taks.
* The main thread scan `Responsible` list and schedular thread to execute the task which has higher priority.
* The thread scheduling algorithm should be abstructed to a common interface that can be easy to extend.

See [[Unified Thread Management]] for more details.

## Behavior Framework Enhancement
* Add task list to each `Responsible`.
* `Responsible` has priority which define how task execution frequence by Thread Pool. 
* The task list should be a non-blocking cycle ring.
* The task is corresponding with `Behavior` execution.
* When new task is added in the task list, it should notify to Thread Pool that the `Responsible` has a incoming task.

See [[Behavior Framework]] for more details
