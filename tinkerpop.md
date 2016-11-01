#gremlin
    实现以tinkerGraph为主要参考依据
##结构
###graph结构
    所有数据的起点, 管理节点和边, 直接相关的结构为GraphTraversalSource，含有addVertex/vertices等方法，但是不由自身进行调用(由traversal进行调用)。
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
###GraphTraversalSource结构(g=graph.traversal())
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
###GraphTraversal<S, E>结构(执行所有traversal function(addV(),V(),out()等)后返回)
    构造时S表示初始输入类型，E表示输出类型。
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

    //get all the traverser of traversal, and return this traversal
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
###Step<S, E>
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

    Traverser.Admin<E> processNextStart()
        如果之前有traversal，final Traverser.Admin<S> traverser = this.starts.next();//调用上一个step的next函数，
        return traverser.split(this.map(traverser), this);将之前的数据经过处理(此处为map操作)后返回。
        如果之前没有traversal,则获取此traversal对应的数据放入head(traverser)中返回。
###Traverser<T>
    T t;
    唯一作用是保存数据，如V()，则保存Iterator<Vertex>。
###GraphComputer结构(graph.computer(),包括mapreduce等函数)
    ???
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

