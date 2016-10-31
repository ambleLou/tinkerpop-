#gremlin
    实现以tinkerGraph为主要参考依据
##结构
###graph结构
    所有数据的起点，包含一些方法，直接相关的结构为GraphTraversalSource，含有一些具体函数但是不由自身进行调用(由traversal进行调用)。
    Vertex addVertex(final Object... keyValues)
        根据参数制作一个vertex，将vertex.id和vertex写如vertices中。
    Iterator<Vertex> vertices(final Object... vertexIds)
        V()操作具体的函数，从map中使用lambda表达式获取到。edge同理.
###GraphTraversalSource结构(g=graph.traversal())
    traversal()返回的是GraphTraversalSource，有addV/addE/V/E等起始函数，即所有操作必须从这些起始函数开始，返回的traversal进行之后的操作，
    具体GraphTraversal.addStep来完成操作，并返回GraphTraversal。
###GraphTraversal<S, E>结构(执行所有traversal function(addV(),V(),out()等)后返回)
    GraphTraversal包含多个函数，has/out/valueMap等，所有函数都是将对应的step加入到List<Step> steps中。
###Step<S, E>
    有很多子类，每个子类代表一种操作，每个子类会有自己的属性。
    核心函数：
    Traverser.Admin<E> processNextStart()：用于生成Traverser，实现为：
        如果之前有traversal，final Traverser.Admin<S> traverser = this.starts.next();//调用上一个step的next函数，
        return traverser.split(this.map(traverser), this);将之前的数据经过处理(此处为map操作)后返回。
        如果之前没有traversal,则获取此traversal对应的数据放入head(traverser)中返回。
###Traverser<T>
    T t;唯一的属性，保存数据，如V()，则保存Iterator<Vertex>。
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

