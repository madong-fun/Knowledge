### 从addSource 开始

```
	public <OUT> DataStreamSource<OUT> addSource(SourceFunction<OUT> function, String sourceName, TypeInformation<OUT> typeInfo) {

		if (typeInfo == null) {
			if (function instanceof ResultTypeQueryable) {
				typeInfo = ((ResultTypeQueryable<OUT>) function).getProducedType();
			} else {
				try {
					typeInfo = TypeExtractor.createTypeInfo(
							SourceFunction.class,
							function.getClass(), 0, null, null);
				} catch (final InvalidTypesException e) {
					typeInfo = (TypeInformation<OUT>) new MissingTypeInfo(sourceName, e);
				}
			}
		}

		boolean isParallel = function instanceof ParallelSourceFunction;

		clean(function);
		StreamSource<OUT, ?> sourceOperator;
		if (function instanceof StoppableFunction) {
			sourceOperator = new StoppableStreamSource<>(cast2StoppableSourceFunction(function));
		} else {
			sourceOperator = new StreamSource<>(function);
		}

		return new DataStreamSource<>(this, typeInfo, sourceOperator, isParallel, sourceName);
	}
	
```
在执行 env.addSource 后，这段代码会被调用，返回一个构造好的DataStream。总结一下上段代码逻辑：

 1. 获取数据源 source 的 output 信息 TypeInformation
 2. 生成 StreamSource<OUT, ?> sourceOperator
 
 ```
 	public DataStreamSource(StreamExecutionEnvironment environment,
			TypeInformation<T> outTypeInfo, StreamSource<T, ?> operator,
			boolean isParallel, String sourceName) {
		super(environment, new SourceTransformation<>(sourceName, operator, outTypeInfo, environment.getParallelism()));

		this.isParallel = isParallel;
		if (!isParallel) {
			setParallelism(1);
		}
	}
 ```
 3. 将 StreamTransformation 添加到算子列表 transformations 中
 4. 构造DataStreamSource （typeInfo, sourceOperator, isParallel, sourceName）对象，并返回
 
 返回DataStreamSource 具体的继承关系如下：

![未命名文件 (4).png](resources/0D5F6EC40A87FBC95A9EC3C425719AB6.png =830x731) 

 DataStreamSource 是一个 DataStream 数据流抽象，StreamSource 是一个 StreamOperator 算子抽象，在 flink 中一个 DataStream 封装了一次数据流转换，一个 StreamOperator 封装了一个函数接口，比如 map、reduce、keyBy等。
 
 ![未命名文件 (5).png](resources/A5A4C8B6001CEBEBEC28CF80CCDA34A4.png =896x858)
 
 
### 在map操作上继续

```
	public <R> SingleOutputStreamOperator<R> map(MapFunction<T, R> mapper) {

		TypeInformation<R> outType = TypeExtractor.getMapReturnTypes(clean(mapper), getType(),
				Utils.getCallLocationName(), true);

		return transform("Map", outType, new StreamMap<>(clean(mapper)));
	}
	
```

从代码上看，一次map操作会触发一次transform，transform 又做了什么？

```
	public <R> SingleOutputStreamOperator<R> transform(String operatorName, TypeInformation<R> outTypeInfo, OneInputStreamOperator<T, R> operator) {

		// read the output type of the input Transform to coax out errors about MissingTypeInfo
		transformation.getOutputType();

		OneInputTransformation<T, R> resultTransform = new OneInputTransformation<>(
				this.transformation,
				operatorName,
				operator,
				outTypeInfo,
				environment.getParallelism());

		@SuppressWarnings({ "unchecked", "rawtypes" })
		SingleOutputStreamOperator<R> returnStream = new SingleOutputStreamOperator(environment, resultTransform);

		getExecutionEnvironment().addOperator(resultTransform);

		return returnStream;
	}
	
```
 上面代码看，transform方法内构造了一个 StreamTransformation并以此作为成员变量封装成另一个 DataStream 返回，StreamTransformation是 flink关于数据流转换的核心抽象，只有需要 transform 的流才会生成新的DataStream 算子。getExecutionEnvironment().addOperator(resultTransform)flink会将transformation维护起来。
 
```
	public void addOperator(StreamTransformation<?> transformation) {
		Preconditions.checkNotNull(transformation, "transformation must not be null.");
		this.transformations.add(transformation);
	}
```
用户的一连串操作 map join 等实际上在 DataStream 上做了转换，并且flink将这些 StreamTransformation 维护起来，一直到最后，用户执行 env.execute()这样一段逻辑，StreamGraph 的构建才算真正开始

```

	public JobExecutionResult execute(String jobName) throws Exception {
		Preconditions.checkNotNull(jobName, "Streaming Job name should not be null.");

		StreamGraph streamGraph = this.getStreamGraph();
		streamGraph.setJobName(jobName);

		transformations.clear();

		// execute the programs
		if (ctx instanceof DetachedEnvironment) {
			LOG.warn("Job was executed in detached mode, the results will be available on completion.");
			((DetachedEnvironment) ctx).setDetachedPlan(streamGraph);
			return DetachedEnvironment.DetachedJobExecutionResult.INSTANCE;
		} else {
			return ctx
				.getClient()
				.run(streamGraph, ctx.getJars(), ctx.getClasspaths(), ctx.getUserCodeClassLoader(), ctx.getSavepointRestoreSettings())
				.getJobExecutionResult();
		}
	}
	
```
这段代码两件事：
1. 调用 StreamGraphGenerator.generate 生成 StreamGraph
2. 使用 Client 运行 stream graph

### 说说 StreamGraphGenerator

```

	public static StreamGraph generate(StreamExecutionEnvironment env, List<StreamTransformation<?>> transformations) {
		return new StreamGraphGenerator(env).generateInternal(transformations);
	}

	/**
	 * This starts the actual transformation, beginning from the sinks.
	 */
	private StreamGraph generateInternal(List<StreamTransformation<?>> transformations) {
		for (StreamTransformation<?> transformation: transformations) {
			transform(transformation);
		}
		return streamGraph;
	}

```
迭代遍历transformations，调用transform方法，依据添加算子时保存的 transformations 信息生成 job graph 中的节点，并创建节点连接。


```
		if (transform instanceof OneInputTransformation<?, ?>) {
			transformedIds = transformOneInputTransform((OneInputTransformation<?, ?>) transform);
		} else if (transform instanceof TwoInputTransformation<?, ?, ?>) {
			transformedIds = transformTwoInputTransform((TwoInputTransformation<?, ?, ?>) transform);
		} else if (transform instanceof SourceTransformation<?>) {
			transformedIds = transformSource((SourceTransformation<?>) transform);
		} else if (transform instanceof SinkTransformation<?>) {
			transformedIds = transformSink((SinkTransformation<?>) transform);
		} else if (transform instanceof UnionTransformation<?>) {
			transformedIds = transformUnion((UnionTransformation<?>) transform);
		} else if (transform instanceof SplitTransformation<?>) {
			transformedIds = transformSplit((SplitTransformation<?>) transform);
		} else if (transform instanceof SelectTransformation<?>) {
			transformedIds = transformSelect((SelectTransformation<?>) transform);
		} else if (transform instanceof FeedbackTransformation<?>) {
			transformedIds = transformFeedback((FeedbackTransformation<?>) transform);
		} else if (transform instanceof CoFeedbackTransformation<?>) {
			transformedIds = transformCoFeedback((CoFeedbackTransformation<?>) transform);
		} else if (transform instanceof PartitionTransformation<?>) {
			transformedIds = transformPartition((PartitionTransformation<?>) transform);
		} else if (transform instanceof SideOutputTransformation<?>) {
			transformedIds = transformSideOutput((SideOutputTransformation<?>) transform);
		} else {
			throw new IllegalStateException("Unknown transformation: " + transform);
		}
		
```
针对不同的StreamTransformation的子类实现，委托不同的方法进行转化（scala写起来要比这个简单许多）。拿第一个oneInputTransform 举个例子：

```

	/**
	 * Transforms a {@code OneInputTransformation}.
	 *
	 * <p>This recursively transforms the inputs, creates a new {@code StreamNode} in the graph and
	 * wired the inputs to this new node.
	 */
	private <IN, OUT> Collection<Integer> transformOneInputTransform(OneInputTransformation<IN, OUT> transform) {
    // 递归调用transform
		Collection<Integer> inputIds = transform(transform.getInput());

		// the recursive call might have already transformed this
		if (alreadyTransformed.containsKey(transform)) {
			return alreadyTransformed.get(transform);
		}
    // 确定SlotSharingGroup的名称
		String slotSharingGroup = determineSlotSharingGroup(transform.getSlotSharingGroup(), inputIds);
    // 添加节点
		streamGraph.addOperator(transform.getId(),
				slotSharingGroup,
				transform.getCoLocationGroupKey(),
				transform.getOperator(),
				transform.getInputType(),
				transform.getOutputType(),
				transform.getName());
    //设置属性
		if (transform.getStateKeySelector() != null) {
			TypeSerializer<?> keySerializer = transform.getStateKeyType().createSerializer(env.getConfig());
			streamGraph.setOneInputStateKey(transform.getId(), transform.getStateKeySelector(), keySerializer);
		}

		streamGraph.setParallelism(transform.getId(), transform.getParallelism());
		streamGraph.setMaxParallelism(transform.getId(), transform.getMaxParallelism());
    // 构建edge
		for (Integer inputId: inputIds) {
			streamGraph.addEdge(inputId, transform.getId(), 0);
		}

		return Collections.singleton(transform.getId());
	}
	
```

OneInputTransformation 对应的操作算子有很多，比如FlatMap、Map、filter等。
transfrom方法有很多，就不一一列举了，这里可以总结一下：tranform方法将 StreamTransformation 转换为 StreamNode，StreamNode 保存了算子的信息，StreamNode 构成的 DAG 图 StreamGraph。这个图提交给client时，还会做进一步的优化：
StreamGraph 将进一步转换为 JobGraph，这一步工作由 StreamingJobGraphGenerator 来完成。

```

private JobGraph createJobGraph() {

		// make sure that all vertices start immediately
		jobGraph.setScheduleMode(ScheduleMode.EAGER);

		// Generate deterministic hashes for the nodes in order to identify them across
		// submission iff they didn't change.
		Map<Integer, byte[]> hashes = defaultStreamGraphHasher.traverseStreamGraphAndGenerateHashes(streamGraph);

		// Generate legacy version hashes for backwards compatibility
		List<Map<Integer, byte[]>> legacyHashes = new ArrayList<>(legacyStreamGraphHashers.size());
		for (StreamGraphHasher hasher : legacyStreamGraphHashers) {
			legacyHashes.add(hasher.traverseStreamGraphAndGenerateHashes(streamGraph));
		}

		Map<Integer, List<Tuple2<byte[], byte[]>>> chainedOperatorHashes = new HashMap<>();

		setChaining(hashes, legacyHashes, chainedOperatorHashes);

		setPhysicalEdges();

		setSlotSharingAndCoLocation();

		configureCheckpointing();

		JobGraphGenerator.addUserArtifactEntries(streamGraph.getEnvironment().getCachedFiles(), jobGraph);

		// set the ExecutionConfig last when it has been finalized
		try {
			jobGraph.setExecutionConfig(streamGraph.getExecutionConfig());
		}
		catch (IOException e) {
			throw new IllegalConfigurationException("Could not serialize the ExecutionConfig." +
					"This indicates that non-serializable types (like custom serializers) were registered");
		}

		return jobGraph;
	}

```
代码主要逻辑如下：

1. 遍历stream graph
2. 生成operatorChain
3. 设置物理边
4. 设置SlotSharing Group
5. 配置checpoint

### operator Chain

在StreamGraph中可以知道一个Operator对应一个StreamNode，一个DataStream.map().filter() 这个关系中map和filter Operator会组成不同的StreamNode，最后生成Task, 如果这两个Task不在同一个Slot或在不同一个TaskManager中，数据会经过网络从map传到filter，执行性能会很差，考虑到这一点，flink引入 operator chain的概念， 一个operator chain 代表一组可以在同一个Slot执行的Operator串。

```

	public static boolean isChainable(StreamEdge edge, StreamGraph streamGraph) {
		StreamNode upStreamVertex = streamGraph.getSourceVertex(edge);
		StreamNode downStreamVertex = streamGraph.getTargetVertex(edge);

		StreamOperator<?> headOperator = upStreamVertex.getOperator();
		StreamOperator<?> outOperator = downStreamVertex.getOperator();

		return downStreamVertex.getInEdges().size() == 1
				&& outOperator != null
				&& headOperator != null
				&& upStreamVertex.isSameSlotSharingGroup(downStreamVertex)
				&& outOperator.getChainingStrategy() == ChainingStrategy.ALWAYS
				&& (headOperator.getChainingStrategy() == ChainingStrategy.HEAD ||
					headOperator.getChainingStrategy() == ChainingStrategy.ALWAYS)
				&& (edge.getPartitioner() instanceof ForwardPartitioner)
				&& upStreamVertex.getParallelism() == downStreamVertex.getParallelism()
				&& streamGraph.isChainingEnabled();
	}

```
从上面的代码可以看出，以下几种情况可以chain：

* 下游的input 只有一个上游（downStreamVertex.getInEdges().size() == 1）
* 同属一个SlotSharingGroup （upStreamVertex.isSameSlotSharingGroup(downStreamVertex)）
* 允许chain 打开 （streamGraph.isChainingEnabled()）
* Partitioner 为ForwardPartitioner (edge.getPartitioner() instanceof ForwardPartitioner)
* 并行度一致 upStreamVertex.getParallelism() == downStreamVertex.getParallelism()
* ChainingStrategy允许chain

总结一下JobGraph 生成逻辑：

* 由DataStream上的操作生成StreamTransformation列表
* 从StreamTransformation的生成关系创建StreamNode和StreamEdge
* 做算子chain，合并成 JobVertex，并生成 JobEdge

一个JobVertex 代表一个逻辑计划的节点，就是 DAG 图上的顶点。JobGraph相对于StreamGraph的最主要区别是将一些StreamNode合并成一个JobVertex, 而JobVertex通过JobEdge(物理边)相连, 最大程度的优化了StreamGraph。

### 生成ExecutionGraph

StreamGraph和JobGraph都是在client生成的,然后通过 submitJob 提交任务到JobMaster。

```

	public JobMaster(
			RpcService rpcService,
			JobMasterConfiguration jobMasterConfiguration,
			ResourceID resourceId,
			JobGraph jobGraph,
			HighAvailabilityServices highAvailabilityService,
			SlotPoolFactory slotPoolFactory,
			SchedulerFactory schedulerFactory,
			JobManagerSharedServices jobManagerSharedServices,
			HeartbeatServices heartbeatServices,
			JobManagerJobMetricGroupFactory jobMetricGroupFactory,
			OnCompletionActions jobCompletionActions,
			FatalErrorHandler fatalErrorHandler,
			ClassLoader userCodeLoader) throws Exception {

		super(rpcService, AkkaRpcServiceUtils.createRandomName(JOB_MANAGER_NAME));

		final JobMasterGateway selfGateway = getSelfGateway(JobMasterGateway.class);

       ……


		resourceManagerLeaderRetriever = highAvailabilityServices.getResourceManagerLeaderRetriever();

		this.slotPool = checkNotNull(slotPoolFactory).createSlotPool(jobGraph.getJobID());

		this.scheduler = checkNotNull(schedulerFactory).createScheduler(slotPool);

		this.registeredTaskManagers = new HashMap<>(4);

		this.backPressureStatsTracker = checkNotNull(jobManagerSharedServices.getBackPressureStatsTracker());
		this.lastInternalSavepoint = null;

		this.jobManagerJobMetricGroup = jobMetricGroupFactory.create(jobGraph);
		this.executionGraph = createAndRestoreExecutionGraph(jobManagerJobMetricGroup);
		this.jobStatusListener = null;

		this.resourceManagerConnection = null;
		this.establishedResourceManagerConnection = null;

		this.accumulators = new HashMap<>();
	}

```

从上面的源代码可以看出，在JobMaster的构造函数中，会生成ExecutionGraph。

```
// 流式作业中，schedule mode固定是EAGER的
    executionGraph.setScheduleMode(jobGraph.getScheduleMode());
		executionGraph.setQueuedSchedulingAllowed(jobGraph.getAllowQueuedScheduling());

		try {
			executionGraph.setJsonPlan(JsonPlanGenerator.generatePlan(jobGraph));
		}
		catch (Throwable t) {
			log.warn("Cannot create JSON plan for job", t);
			// give the graph an empty plan
			executionGraph.setJsonPlan("{}");
		}

		// initialize the vertices that have a master initialization hook
		// file output formats create directories here, input formats create splits

		final long initMasterStart = System.nanoTime();
		log.info("Running initialization on master for job {} ({}).", jobName, jobId);
    // 检查executableClass(即operator类)，设置最大并发
    // 
		for (JobVertex vertex : jobGraph.getVertices()) {
			String executableClass = vertex.getInvokableClassName();
			if (executableClass == null || executableClass.isEmpty()) {
				throw new JobSubmissionException(jobId,
						"The vertex " + vertex.getID() + " (" + vertex.getName() + ") has no invokable class.");
			}

			if (vertex.getParallelism() == ExecutionConfig.PARALLELISM_AUTO_MAX) {
				if (parallelismForAutoMax < 0) {
					throw new JobSubmissionException(
						jobId,
						PARALLELISM_AUTO_MAX_ERROR_MESSAGE);
				}
				else {
					vertex.setParallelism(parallelismForAutoMax);
				}
			}

			try {
				vertex.initializeOnMaster(classLoader);
			}
			catch (Throwable t) {
					throw new JobExecutionException(jobId,
							"Cannot initialize task '" + vertex.getName() + "': " + t.getMessage(), t);
			}
		}

		log.info("Successfully ran initialization on master in {} ms.",
				(System.nanoTime() - initMasterStart) / 1_000_000);

		// 按拓扑顺序，获取所有的JobVertex列表
		List<JobVertex> sortedTopology = jobGraph.getVerticesSortedTopologicallyFromSources();
		if (log.isDebugEnabled()) {
			log.debug("Adding {} vertices from job graph {} ({}).", sortedTopology.size(), jobName, jobId);
		}
		// 根据JobVertex列表，生成execution graph
		executionGraph.attachJobGraph(sortedTopology);

```

可以看到，生成execution graph的代码，主要是在ExecutionGraph.attachJobGraph,进入方法，往下看：

```

		for (JobVertex jobVertex : topologiallySorted) {

			if (jobVertex.isInputVertex() && !jobVertex.isStoppable()) {
				this.isStoppable = false;
			}

			// create the execution job vertex and attach it to the graph
			ExecutionJobVertex ejv = new ExecutionJobVertex(
				this,
				jobVertex,
				1,
				rpcTimeout,
				globalModVersion,
				createTimestamp);

			ejv.connectToPredecessors(this.intermediateResults);

			ExecutionJobVertex previousTask = this.tasks.putIfAbsent(jobVertex.getID(), ejv);
			if (previousTask != null) {
				throw new JobException(String.format("Encountered two job vertices with ID %s : previous=[%s] / new=[%s]",
					jobVertex.getID(), ejv, previousTask));
			}

			for (IntermediateResult res : ejv.getProducedDataSets()) {
				IntermediateResult previousDataSet = this.intermediateResults.putIfAbsent(res.getId(), res);
				if (previousDataSet != null) {
					throw new JobException(String.format("Encountered two intermediate data set with ID %s : previous=[%s] / new=[%s]",
						res.getId(), res, previousDataSet));
				}
			}

			this.verticesInCreationOrder.add(ejv);
			this.numVerticesTotal += ejv.getParallelism();
			newExecJobVertices.add(ejv);
		}

```
可以看到，创建ExecutionJobVertex的重点就在它的构造函数中