对 SDWebImageDownloaderOperation的总结

SDWebImageDownloaderOperation是继承 NSOperation的类，按照官方文档 https://developer.apple.com/reference/foundation/nsoperation 里面描述，NSOperation为基础的抽象类，
如果实现非并发的操作，只需要实现main()方法。
For non-concurrent operations, you typically override only one method:
main()
Into this method, you place the code needed to perform the given task. Of course, you should also define a custom initialization method to make it easier to create instances of your custom class. You might also want to define getter and setter methods to access the data from the operation. However, if you do define custom getter and setter methods, you must make sure those methods can be called safely from multiple threads.
如果实现并发，至少需要实现start()，isAsynchronous(),isExecuting(),isFinished()。

If you are creating a concurrent operation, you need to override the following methods and properties at a minimum:
start()
isAsynchronous
isExecuting
isFinished

In a concurrent operation, your start() method is responsible for starting the operation in an asynchronous manner. Whether you spawn a thread or call an asynchronous function, you do it from this method. Upon starting the operation, your start() method should also update the execution state of the operation as reported by the isExecuting property. You do this by sending out KVO notifications for the isExecuting key path, which lets interested clients know that the operation is now running. Your isExecuting property must also provide the status in a thread-safe manner.

Upon completion or cancellation of its task, your concurrent operation object must generate KVO notifications for both the isExecuting and isFinished key paths to mark the final change of state for your operation. (In the case of cancellation, it is still important to update the isFinishedkey path, even if the operation did not completely finish its task. Queued operations must report that they are finished before they can be removed from a queue.) In addition to generating KVO notifications, your overrides of the isExecuting and isFinished properties should also continue to report accurate values based on the state of your operation.

关于Cancel的操作说明
Responding to the Cancel Command
Once you add an operation to a queue, the operation is out of your hands. The queue takes over and handles the scheduling of that task. However, if you decide later that you do not want to execute the operation after all—because the user pressed a cancel button in a progress panel or quit the application, for example—you can cancel the operation to prevent it from consuming CPU time needlessly. You do this by calling the cancel() method of the operation object itself or by calling the cancelAllOperations() method of the OperationQueue class.

Canceling an operation does not immediately force it to stop what it is doing. Although respecting the value in the isCancelled property is expected of all operations, your code must explicitly check the value in this property and abort as needed. The default implementation of NSOperationincludes checks for cancellation. For example, if you cancel an operation before its start() method is called, the start() method exits without starting the task.

意思是说，具体的执行取消操作需要你在cancel方法中实现。
后面的篇幅是关于 NSURLSession的总结。下节分析。
