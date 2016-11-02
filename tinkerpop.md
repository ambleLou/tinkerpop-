#gremlin
    实现以tinkerGraph为主要参考依据
##graph Structure
    所有数据的起点, 操作节点和边, 直接相关的结构为GraphTraversalSource，含有addVertex/vertices等方法，但是不由自身进行调用(由traversal进行调用)。
    properties:
        Map<Object, Vertex> vertices    //with all vertices
        Map<Object, Edge> edges         //with all edges
    functions:
    GraphTraversalSource traversal()
        返回一个GraphTraversalSource
    Vertex addVertex(final Object... keyValues)
        根据参数制作一个vertex，将vertex.id和vertex写如vertices中。
    Iterator<Vertex> vertices(final Object... vertexIds)
        V()操作具体的函数，从map中使用lambda表达式获取到。edge同理.
###GraphTraversalSource Structure(g=graph.traversal())
    traversal()返回的是GraphTraversalSource，有addV/addE/V/E等起始函数(因为所有操作必须从这些起始函数开始)，由返回的traversal进行之后的操作。
    functions:
    GraphTraversal<Vertex, Vertex> V(final Object... vertexIds) {
        //clone traversalSource
        final GraphTraversalSource clone = this.clone();
        //init new traversal by this source
        final GraphTraversal.Admin<Vertex, Vertex> traversal = new DefaultGraphTraversal<>(clone);
        //add the step into traversal and return
        return traversal.addStep(new GraphStep<>(traversal, Vertex.class, true, vertexIds));
    }
##GraphTraversal<S, E> Structure (return structure of traversal functions(addV(),V(),out() etc.))
    构造时S表示初始输入类型，E表示输出类型，用于控制起点和终点的类型。
    GraphTraversal包含多个函数，has/out/valueMap等，所有函数执行逻辑都是将对应的step加入到List<Step> steps中。
    同时它也会执行step并获取数据，执行主要是生成step的traverser。
    functions:
    //traversal example:V()
    GraphTraversal<S, Vertex> V(final Object... vertexIdsOrElements) {
        return this.asAdmin().addStep(new GraphStep<>(this.asAdmin(), Vertex.class, false, vertexIdsOrElements));
    }

    //add step into list<step> steps and as Two-way linked list
    Traversal.Admin<S2, E2> addStep(final int index, final Step<?, ?> step) {
        this.steps.add(index, step);
        final Step previousStep = steps.get(index - 1);
        final Step nextStep = steps.get(index + 1);
        step.setPreviousStep(previousStep);
        step.setNextStep(nextStep);
        previousStep.setNextStep(step);
        nextStep.setPreviousStep(step);
        step.setTraversal(this);
        return (Traversal.Admin<S2, E2>) this;
    }

    //get the traverser of traversal, and return this traversal
    Traversal<A, B> iterate() {
        if (!this.asAdmin().isLocked()) this.asAdmin().applyStrategies();
        // use the end step so the results are bulked
        final Step<?, E> endStep = this.asAdmin().getEndStep();
        while (true) {
            endStep.next();
        }
        return (Traversal<A, B>) this;
    }

    //get result into traverser and then into collection(like List<T>)
    C fill(final C collection) {
        // use the end step so the results are bulked
        final Step<?, E> endStep = this.asAdmin().getEndStep();
        while (true) {
            //get the result of end step and store in traverser
            final Traverser<E> traverser = endStep.next();
            //add the data of traverser into collection
            TraversalHelper.addToCollection(collection, traverser.get(), traverser.bulk());
        }
        return collection;
    }
###Step<S, E> Structure
    有很多子类，每个子类代表一种操作(如V/addV/has等)，每个子类会有自己的属性，所有step共用next函数，同时实现自己的processNextStart函数。
    functions:
    //generate traverser by processNextStart and end while(true) by interrupt or send the result to next step
    Traverser.Admin<E> next() {
        if (null != this.nextEnd) {
            return this.prepareTraversalForNextStep(this.nextEnd);
        } else {
            while (true) {
                final Traverser.Admin<E> traverser = this.processNextStart();
                if (null != traverser.get() && 0 != traverser.bulk())
                    return this.prepareTraversalForNextStep(traverser);
            }
        }
    }

    //generate traverser example: flatStep(vertex step like outV/outE)
    Traverser.Admin<E> processNextStart() {
        while (true) {
            if (this.iterator.hasNext()) {
                //return iterator to traverser
                return this.head.split(this.iterator.next(), this);
            } else {
                //get the previous step
                this.head = this.starts.next();
                //get the iterator
                this.iterator = this.flatMap(this.head);
            }
        }
    }
###Traverser<T> Structure
    唯一作用是保存数据，如V()，则保存Iterator<Vertex>。
    property:
    T t;
##GraphComputer (graph.computer())
    用于执行各类compute函数，一般用法graph.compute().program(new VertexProgram()).submit().get()，核心函数为submit, 负责执行其中的vertexProgram/MapReduce, 最终返回DefaultComputerResult。
    注意：computer和traversal是完全隔离的两种形式，一般不能再traversal后加computer操作。

    functions:
    //add vertexProgram into this.vertexProgram
    GraphComputer program(final VertexProgram vertexProgram)

    //execute vertexProgram
    Future<ComputerResult> submit() {
        //get the result graph and persist state to use for the computation
        this.resultGraph = GraphComputerHelper.getResultGraphState(Optional.ofNullable(this.vertexProgram), Optional.ofNullable(this.resultGraph));
        this.persist = GraphComputerHelper.getPersistState(Optional.ofNullable(this.vertexProgram), Optional.ofNullable(this.persist));

        //initialize the memory
        this.memory = new TinkerMemory(this.vertexProgram, this.mapReducers);
        return computerService.submit(() -> {
            final TinkerGraphComputerView view;
            view = TinkerHelper.createGraphComputerView(this.graph, this.graphFilter, this.vertexProgram.getVertexComputeKeys());
            //execute the vertex program
            this.vertexProgram.setup(this.memory);
            while (true) {
                this.memory.completeSubRound();
                //get all the vertices from graph!
                final SynchronizedIterator<Vertex> vertices = new SynchronizedIterator<>(this.graph.vertices());
                vertexProgram.workerIterationStart(this.memory.asImmutable());
                while (true) {
                    final Vertex vertex = vertices.next();
                    if (null == vertex) break;
                    vertexProgram.execute(
                        ComputerGraph.vertexProgram(vertex, vertexProgram),
                        new TinkerMessenger<>(vertex, this.messageBoard, vertexProgram.getMessageCombiner()),
                        this.memory
                    );
                }
                this.memory.completeSubRound();
                this.memory.incrIteration();
            }
            view.complete(); // drop all transient vertex compute keys

            
            // update runtime and return the newly computed graph
            this.memory.setRuntime(System.currentTimeMillis() - time);
            this.memory.complete(); // drop all transient properties and set iteration
            // determine the resultant graph based on the result graph/persist state
            final Graph resultGraph = view.processResultGraphPersist(this.resultGraph, this.persist);
            TinkerHelper.dropGraphComputerView(this.graph); // drop the view from the original source graph
            return new DefaultComputerResult(resultGraph, this.memory.asImmutable());
        });
    }
###VertexProgram<M>结构
    VertexProgam会生成一个property附到vertex上，M表示此property的类型。
    functions:
    void setup(final Memory memory)
        ???

    //execute example about ???, execute on vertex and effect other vertex by message then store result in memory

    //generate Map<String, MemoryComputeKey> memoryKeys in TinkerMemory,  key is string "a" as column name
    Set<MemoryComputeKey> getMemoryComputeKeys() {
        return new HashSet<>(Arrays.asList(
                MemoryComputeKey.of("a", Operator.sum),
                MemoryComputeKey.of("b", Operator.sum)));
    }

    //
    boolean terminate(final Memory memory)
###TinkerGraphComputerView

###TinkerMemory structure
    用于存储computer获得的数据

###MapReduce
    作用是对数据进行分组，包含在compute操作中，核心函数为submit。

    functions:
    //execute vertexProgram
    Future<ComputerResult> submit() {
        GraphComputerHelper.validateProgramOnComputer(this, this.vertexProgram);
        this.mapReducers.addAll(this.vertexProgram.getMapReducers());
        //initialize the memory
        this.memory = new TinkerMemory(this.vertexProgram, this.mapReducers);
        return computerService.submit(() -> {
            view = TinkerHelper.createGraphComputerView(this.graph, this.graphFilter, Collections.emptySet());

            // execute mapreduce jobs
            for (final MapReduce mapReduce : mapReducers) {
                final TinkerMapEmitter<?, ?> mapEmitter = new TinkerMapEmitter<>(mapReduce.doStage(MapReduce.Stage.REDUCE));
                final SynchronizedIterator<Vertex> vertices = new SynchronizedIterator<>(this.graph.vertices());
                workers.setMapReduce(mapReduce);
                workers.executeMapReduce(workerMapReduce -> {
                    workerMapReduce.workerStart(MapReduce.Stage.MAP);
                    while (true) {
                        if (Thread.interrupted()) throw new TraversalInterruptedException();
                        final Vertex vertex = vertices.next();
                        if (null == vertex) break;
                        workerMapReduce.map(ComputerGraph.mapReduce(vertex), mapEmitter);
                    }
                    workerMapReduce.workerEnd(MapReduce.Stage.MAP);
                });
                // sort results if a map output sort is defined
                mapEmitter.complete(mapReduce);

                // no need to run combiners as this is single machine
                if (mapReduce.doStage(MapReduce.Stage.REDUCE)) {
                    final TinkerReduceEmitter<?, ?> reduceEmitter = new TinkerReduceEmitter<>();
                    final SynchronizedIterator<Map.Entry<?, Queue<?>>> keyValues = new SynchronizedIterator((Iterator) mapEmitter.reduceMap.entrySet().iterator());
                    workers.executeMapReduce(workerMapReduce -> {
                        workerMapReduce.workerStart(MapReduce.Stage.REDUCE);
                        while (true) {
                            if (Thread.interrupted()) throw new TraversalInterruptedException();
                            final Map.Entry<?, Queue<?>> entry = keyValues.next();
                            if (null == entry) break;
                            workerMapReduce.reduce(entry.getKey(), entry.getValue().iterator(), reduceEmitter);
                        }
                        workerMapReduce.workerEnd(MapReduce.Stage.REDUCE);
                    });
                    reduceEmitter.complete(mapReduce); // sort results if a reduce output sort is defined
                    mapReduce.addResultToMemory(this.memory, reduceEmitter.reduceQueue.iterator());
                } else {
                    mapReduce.addResultToMemory(this.memory, mapEmitter.mapQueue.iterator());
                }
            }
        }
    }
##traversal流程(V()/out()等)
    GraphTraversal.Admin<Vertex, Vertex> traversal = new DefaultGraphTraversal<>(clone);
    生成一个traversal，可以执行addstep和iterate，包含属性List<Step> steps，记录所有需要执行的step，记录作为path的来源
    
    return traversal.addStep(new GraphStep<>(traversal, Vertex.class, true, vertexIds));
    添加GraphStep，并执行获取V或者E，使用的函数是graph.vertices(ids)/graph.edges(ids),获取的结果存放在iteratorSupplier中

    返回traversal，数据存放在调用者(graph)的Map<Object, Vertex> vertices中
    是一个hashmap，把id作为key，整个vertex作为value，如果不建立索引则会影响搜索速度。
###g.addV()流程(addE类推)
    GraphTraversal.Admin<Vertex, Vertex> traversal = new DefaultGraphTraversal<>(clone);
    同上。
    
    traversal.addStep(new AddVertexStartStep(traversal, label));
    添加AddVertexStartStep，数据存入parameters中
##Step结构(每一种traversal都有对应的step)
###GraphStep<>: 处理V()/E()
    GraphStep(final Traversal.Admin traversal, final Class<E> returnClass, final boolean isStart, final Object... ids)
    traversal：执行者 returnClass：返回类型 isStart：是否traversal中第一个step ids：查找的key

    Traverser.Admin<E> processNextStart() {
        while (true) {
            if (this.iterator.hasNext()) {
                return this.isStart ? this.getTraversal().getTraverserGenerator().generate(this.iterator.next(), (Step) this, 1l) : this.head.split(this.iterator.next(), this);
            } else {
                if (this.isStart) {
                    if (this.done)
                        throw FastNoSuchElementException.instance();
                    else {
                        this.done = true;
                        this.iterator = null == this.iteratorSupplier ? EmptyIterator.instance() : this.iteratorSupplier.get();
                    }
                } else {
                    this.head = this.starts.next();
                    this.iterator = null == this.iteratorSupplier ? EmptyIterator.instance() : this.iteratorSupplier.get();
                }
            }
        }
    }
###AddVertexStartStep：处理addV()/addE()
    this.parameters.set(keyValues);
    this.parameters.integrateTraversals(this);
    数据添加到parameters中

    Traverser.Admin<Vertex> processNextStart(){
        //获取key/value依次数据，通过graph.addVertex()进行添加
        this.getTraversal().getGraph().get().addVertex(this.parameters.getKeyValues(EmptyTraverser.instance()));
        //？？？
        getTraversal().getTraverserGenerator().generate(vertex, this, 1l);
    }
    最后执行traversal链时，执行具体操作。
    
###TraversalStrategy
    ？？？
##Tests
###get_g_V_outXknowsX_V_name
    //生成traversal，将step加入steps
    Traversal<Vertex, String> traversal = g.V().out("knows").V().values("name");
    //具体执行steps，结果写入list中
    List<T> results = traversal.toList();
    
    List<E> toList() {
        return this.fill(new ArrayList<>());
    }
    
    <C extends Collection<E>> C fill(final C collection) {
        try {
            if (!this.asAdmin().isLocked()) this.asAdmin().applyStrategies();
            // use the end step so the results are bulked
            final Step<?, E> endStep = this.asAdmin().getEndStep();
            while (true) {
                final Traverser<E> traverser = endStep.next();
                TraversalHelper.addToCollection(collection, traverser.get(), traverser.bulk());
            }
        } catch (final NoSuchElementException ignored) {
        }
        return collection;
    }

