<!--more-->
# Process of submitTask 
## 1. task.startTaskThread() -> executingThread.start() -> 调用Task的run方法in TaskManager - TaskManager

因为创建一个task中有一段

```java

//this指的是Task，Task implements Runnable
executingThread = new Thread(TASK_THREADS_GROUP, this, taskNameWithSubtask);

```

## 2. run() - Task


```
try {
	// ----------------------------
	//  Task Bootstrap - We periodically
	//  check for canceling as a shortcut
	// ----------------------------

	// activate safety net for task thread
	LOG.info("Creating FileSystem stream leak safety net for task {}", this);
	FileSystemSafetyNet.initializeSafetyNetForThread();

  // 处理job引用计数
	blobService.getPermanentBlobService().registerJob(jobId);

	// first of all, get a user-code classloader
	// this may involve downloading the job's JAR files and/or classes
	LOG.info("Loading JAR files for task {}.", this);

   // 包含去JobManager下载Jar包的逻辑
	userCodeClassLoader = createUserCodeClassloader();
	final ExecutionConfig executionConfig = serializedExecutionConfig.deserializeValue(userCodeClassLoader);

	if (executionConfig.getTaskCancellationInterval() >= 0) {
		// override task cancellation interval from Flink config if set in ExecutionConfig
		taskCancellationInterval = executionConfig.getTaskCancellationInterval();
	}

	if (executionConfig.getTaskCancellationTimeout() >= 0) {
		// override task cancellation timeout from Flink config if set in ExecutionConfig
		taskCancellationTimeout = executionConfig.getTaskCancellationTimeout();
	}

	// now load the task's invokable code
	// 反射创建实例
	invokable = loadAndInstantiateInvokable(userCodeClassLoader, nameOfInvokableClass);

	// ----------------------------------------------------------------
	// register the task with the network stack
	// this operation may fail if the system does not have enough
	// memory to run the necessary data exchanges
	// the registration must also strictly be undone
	// ----------------------------------------------------------------

	LOG.info("Registering task at network: {}.", this);

   // 为ResultPartition和inputGate分配内存
	network.registerTask(this);
	
	...

	// next, kick off the background copying of files for the distributed cache
	try {
		for (Map.Entry<String, DistributedCache.DistributedCacheEntry> entry :
				DistributedCache.readFileInfoFromConfig(jobConfiguration))
		{
			LOG.info("Obtaining local cache file for '{}'.", entry.getKey());
			Future<Path> cp = fileCache.createTmpFile(entry.getKey(), entry.getValue(), jobId);
			distributedCacheEntries.put(entry.getKey(), cp);
		}
	}
	catch (Exception e) {
		throw new Exception(
			String.format("Exception while adding files to distributed cache of task %s (%s).", taskNameWithSubtask, executionId),
			e);
	}

	if (isCanceledOrFailed()) {
		throw new CancelTaskException();
	}

	// ----------------------------------------------------------------
	//  call the user code initialization methods
	// ----------------------------------------------------------------

	TaskKvStateRegistry kvStateRegistry = network
			.createKvStateTaskRegistry(jobId, getJobVertexId());

	Environment env = new RuntimeEnvironment(
		jobId,
		vertexId,
		executionId,
		executionConfig,
		taskInfo,
		jobConfiguration,
		taskConfiguration,
		userCodeClassLoader,
		memoryManager,
		ioManager,
		broadcastVariableManager,
		accumulatorRegistry,
		kvStateRegistry,
		inputSplitProvider,
		distributedCacheEntries,
		producedPartitions,
		inputGates,
		network.getTaskEventDispatcher(),
		checkpointResponder,
		taskManagerConfig,
		metrics,
		this);

	// let the task code create its readers and writers
	invokable.setEnvironment(env);

	// the very last thing before the actual execution starts running is to inject
	// the state into the task. the state is non-empty if this is an execution
	// of a task that failed but had backuped state from a checkpoint

	if (null != taskStateHandles) {
		if (invokable instanceof StatefulTask) {
			StatefulTask op = (StatefulTask) invokable;
			op.setInitialState(taskStateHandles);
		} else {
			throw new IllegalStateException("Found operator state for a non-stateful task invokable");
		}
		// be memory and GC friendly - since the code stays in invoke() for a potentially long time,
		// we clear the reference to the state handle
		//noinspection UnusedAssignment
		taskStateHandles = null;
	}

	// ----------------------------------------------------------------
	//  actual task core work
	// ----------------------------------------------------------------

	// we must make strictly sure that the invokable is accessible to the cancel() call
	// by the time we switched to running.
	this.invokable = invokable;

	// switch to the RUNNING state, if that fails, we have been canceled/failed in the meantime
	if (!transitionState(ExecutionState.DEPLOYING, ExecutionState.RUNNING)) {
		throw new CancelTaskException();
	}

	// notify everyone that we switched to running
	notifyObservers(ExecutionState.RUNNING, null);

	// ----------------------- 调度: from DEPLOYING to RUNNING.-------------------------
	taskManagerActions.updateTaskExecutionState(new TaskExecutionState(jobId, executionId, ExecutionState.RUNNING));

	// make sure the user code classloader is accessible thread-locally
	executingThread.setContextClassLoader(userCodeClassLoader);

	// run the invokable
	invokable.invoke();

	// make sure, we enter the catch block if the task leaves the invoke() method due
	// to the fact that it has been canceled
	if (isCanceledOrFailed()) {
		throw new CancelTaskException();
	}

	// ----------------------------------------------------------------
	//  finalization of a successful execution
	// ----------------------------------------------------------------

	// finish the produced partitions. if this fails, we consider the execution failed.
	for (ResultPartition partition : producedPartitions) {
		if (partition != null) {
			partition.finish();
		}
	}

	// try to mark the task as finished
	// if that fails, the task was canceled/failed in the meantime
	if (transitionState(ExecutionState.RUNNING, ExecutionState.FINISHED)) {
		notifyObservers(ExecutionState.FINISHED, null);
	}
	else {
		throw new CancelTaskException();
	}
}

```

**Attention:**

1. 包含去JobManager下载Jar包的逻辑
 ```
	userCodeClassLoader = createUserCodeClassloader();
```
2. 反射创建实例

```
	invokable = loadAndInstantiateInvokable(userCodeClassLoader, nameOfInvokableClass);
```

## 3. invokable.invoke() - StreamTask
```
 *  -- invoke()
 *        |
 *        +----> Create basic utils (config, etc) and load the chain of operators
 *        +----> operators.setup()
 *        +----> task specific init()
 *        +----> initialize-operator-states()
 *        +----> open-operators()
 *        +----> run()
 *        +----> close-operators()
 *        +----> dispose-operators()
 *        +----> common cleanup
 *        +----> task specific cleanup()

```

## 4. run() - SourceStreamTask (StreamTask)


```
protected void run() throws Exception {
	headOperator.run(getCheckpointLock(), getStreamStatusMaintainer());
}
```

## 5. run() - StreamSource (operator)

```
this.ctx = StreamSourceContexts.getSourceContext(timeCharacteristic,
			getProcessingTimeService(),
			lockingObject,
			streamStatusMaintainer,
			collector,
			watermarkInterval,
			-1);

try {
	userFunction.run(ctx);
```

## 6. run() - FromElementsFunction (function)

```
while (isRunning && numElementsEmitted < numElements) {
	T next;
	try {
		next = serializer.deserialize(input);
	}
	catch (Exception e) {
		throw new IOException("Failed to deserialize an element from the source. " +
						"If you are using user-defined serialization (Value and Writable types), check the " +
						"serialization functions.\nSerializer is " + serializer);
		}

	synchronized (lock) {
		ctx.collect(next);
		numElementsEmitted++;
	}
}
```

## 7. collect() - StreamSourceContexts

```
public void collect(T element) {
	synchronized (lock) {
		// 跳到 OperatorChain.collect
		output.collect(reuse.replace(element));
	}
}
```
其中,output就是**OperatorChain**

## 8. collect() - OperatorChain

```
public void collect(StreamRecord<T> record) {
	if (this.outputTag != null) {
		// we are only responsible for emitting to the main input
		return;
	}
	//这个时候this.operator是flatmap
	pushToOperator(record);
}
```

其中record就是从source的结果反序列得到的结果

```
protected <X> void pushToOperator(StreamRecord<X> record) {
	try {
		// we know that the given outputTag matches our OutputTag so the record
		// must be of the type that our operator expects.
		@SuppressWarnings("unchecked")
		StreamRecord<T> castRecord = (StreamRecord<T>) record;

		numRecordsIn.inc();
		operator.setKeyContextElement1(castRecord);
		operator.processElement(castRecord);
	}
	catch (Exception e) {
		throw new ExceptionInChainedOperatorException(e);
	}
}
		
```
	
其中operator就是flatmap

## 9. processElement() - StreamFlatMap (operator)

```
public void processElement(StreamRecord<IN> element) throws Exception {
	collector.setTimestamp(element);
	userFunction.flatMap(element.getValue(), collector);
}
```

其中useFuntion就是用户自己定义的，比如

```
public static final class Tokenizer implements FlatMapFunction<String, Tuple2<String, Integer>> {

	@Override
	public void flatMap(String value, Collector<Tuple2<String, Integer>> out) {
		// normalize and split the line
		String[] tokens = value.toLowerCase().split("\\W+");

		// emit the pairs
		for (String token : tokens) {
			if (token.length() > 0) {
				out.collect(new Tuple2<String, Integer>(token, 1));
			}
		}
	}
}
```





