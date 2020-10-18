每个`Responsible`有自己的任务列表，任务列表包含了该`Responsible`拥有的`Behavior`的请求，`Responsible`本身没有线程执行这些任务。
任务是由系统统一的线程池来执行的，线程池里面会有一个系统级别的线程，该线程优先级是最高的，它会扫描所有的`Responsible`，根据优先级策略来执行相应的`Responsible`下面的任务。

# Behavior Framework Enhancement
* Add task list to each `Responsible`.
* `Responsible` has priority which define how task execution frequence by Thread Pool. 
* The task list should be a non-blocking cycle ring.
* The task is corresponding with `Behavior` execution.
* When new task is added in the task list, it should notify to Thread Pool that the `Responsible` has a incoming task.

# Thread Pool
* A main thread is always running and is able to manager all thread in the pool.
* The main thread has a `Responsible` list which has new incoming taks.
* The main thread scan `Responsible` list and schedular thread to execute the task which has higher priority.
* The thread scheduling algorithm should be abstructed to a common interface that can be easy to extend.

See [[Thread Pool]] for more details.