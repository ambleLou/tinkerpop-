#gremlin
    实现以tinkerGraph为主要参考依据
##graph Class
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
###GraphTraversalSource Class(g=graph.traversal())
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
##GraphTraversal<S, E> Class (return structure of traversal functions(addV(),V(),out() etc.))
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
###Step<S, E> Class
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
###Traverser<T> Class
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
        //initialize the memory
        this.memory = new TinkerMemory(this.vertexProgram, this.mapReducers);
        return computerService.submit(() -> {
            final TinkerGraphComputerView view;
            //execute the vertex program
            while (true) {
                //get all the vertices from graph!
                final SynchronizedIterator<Vertex> vertices = new SynchronizedIterator<>(this.graph.vertices());
                while (true) {
                    final Vertex vertex = vertices.next();
                    if (null == vertex) break;
                    vertexProgram.execute(
                        //vertexProgram on one vertex
                        vertex,
                        //vertexProgram use message to send effect data
                        new TinkerMessenger<>(vertex, this.messageBoard),
                        //all the vertex in this memory
                        this.memory
                    );
                }
            }
            return new DefaultComputerResult(resultGraph, this.memory.asImmutable());
        });
    }
###VertexProgram<M>结构
    VertexProgam会生成一个property附到vertex上，M表示此property的类型。核心函数是execute，负责生成新property，以及将对其他节点影响的数据发送到共享队列中。

    functions:
    //execute example about PageRankVertexProgram, link is:
    http://tinkerpop.apache.org/docs/current/reference/#pagerankvertexprogram
###TinkerMemory structure
    用于存储computer获得的数据，主要结构为Map<String, Object> currentMap，key为VertexProgram的标识，value可以为任意类型，由vertexProgram和MapReduce返回的数据。最后获取数据的方式如下：
    Object result = results.memory().get("KeyName");
###MapReduceProgram
    作用是对数据进行分组，是compute操作的一种，核心函数同样为submit。

    functions:
    //execute MapReduce
    Future<ComputerResult> submit() {
        this.mapReducers.addAll(this.vertexProgram.getMapReducers());
        //initialize the memory
        this.memory = new TinkerMemory(this.vertexProgram, this.mapReducers);
        return computerService.submit(() -> {
            // execute mapreduce jobs
            for (final MapReduce mapReduce : mapReducers) {
                final TinkerMapEmitter<?, ?> mapEmitter = new TinkerMapEmitter<>();
                final SynchronizedIterator<Vertex> vertices = new SynchronizedIterator<>(this.graph.vertices());
                while (true) {
                    final Vertex vertex = vertices.next();
                    if (null == vertex) break;
                    //different mapReduce has different map function
                    mapReduce.map(ComputerGraph.mapReduce(vertex), mapEmitter);
                }
                // sort results
                mapEmitter.complete(mapReduce);

                final TinkerReduceEmitter<?, ?> reduceEmitter = new TinkerReduceEmitter<>();
                //get key/Values by map result
                final Map.Entry<?, Queue<?>> keyValues = new  mapEmitter.reduceMap.entrySet().iterator();
                workers.executeMapReduce(workerMapReduce -> {
                    while (true) {
                        final Map.Entry<?, Queue<?>> entry = keyValues.next();
                        if (null == entry) break;
                        workerMapReduce.reduce(entry.getKey(), entry.getValue().iterator(), reduceEmitter);
                    }
                });
                //sort results
                reduceEmitter.complete(mapReduce); 
                //store reduce result into memory
                mapReduce.addResultToMemory(this.memory, reduceEmitter.reduceQueue.iterator());
            }
        }
    }
###MapReduce<MK, MV, RK, RV, R> Class
    用于自定义Mapreduce函数，将数据根据key归类并对每一类执行reduce，最后通过generateFinalResult将最后结果放入memory中。MK和MV为Map函数生成的类型，RK和RV为reduce生成的类型，R为最后返回的类型
    functions：
    //void map(final Vertex vertex, final MapEmitter<MK, MV> emitter)
    void void map(final Vertex vertex, final MapEmitter<Integer, Integer> emitter) {
        //get iterator with same age
        vertex.<Integer>property("age").ifPresent(age -> emitter.emit(age, age));
    }
    //void reduce(final MK key, final Iterator<MV> values, final ReduceEmitter<RK, RV> emitter)
    void reduce(Integer key, Iterator<Integer> values, ReduceEmitter<Integer, Integer> emitter) {
        values.forEachRemaining(i -> emitter.emit(i, 1));
    }
    //get the 
    public Integer generateFinalResult(final Iterator<KeyValue<NullObject, Integer>> keyValues) {
        return keyValues.next().getValue();
    }
    //add the result into memory like K/V pair <programKey, R>, R is iterator.next()
    void addResultToMemory(final Memory.Admin memory, final Iterator<KeyValue<RK, RV>> keyValues) {
        memory.set(this.getMemoryKey(), this.generateFinalResult(keyValues));
    }
###TinkerMapEmitter<K,V>
    用于保存mapreduce中map生成的数据，以及负责将发送过来的K/V数据进行保存。

    property：
    Map<K, Queue<V>> reduceMap; //store map result with reduce
    Queue<KeyValue<K, V>> mapQueue; //store map result without reduce
    functions:
    //put the key and value into reduceMap/mapQueue
    emit(K key, V value)
    //sort the map result to reduce step 
    complete(final MapReduce<K, V, ?, ?, ?> mapReduce)
###TinkerReduceEmitter<OK, OV>
    同TinkerMapEmitter，用于保存reduce产生的数据

    property:
    Queue<KeyValue<OK, OV>> reduceQueue //store reduce result
    functions:
    //add result into queue
    emit(final OK key, final OV value)
    //sort reduce result
    complete(final MapReduce<?, ?, OK, OV, ?> mapReduce)
