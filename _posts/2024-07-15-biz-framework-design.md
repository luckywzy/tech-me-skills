---
title: "业务框架设计的考量"
date: 2024-07-15
---

根据实际业务实现为业务服务的框架。

# 目标

设计一个通用的业务框架，支持将 模块化的逻辑组合起来执行。
举个例子：
业务流程是：初始化，召回，过滤，排序，补全信息。
那就会有5个逻辑组件，InitComponent, RecallComponent, FilterComponent, rankComponent, fillComponent.
而将其串联起来执行，称为一个业务流程。
业务框架就是要把这5块逻辑组合起来，并执行。
这里再分解一下功能点：
1. 提供组件，业务能实现不同的组件
2. 提供编排，将组件串联起来。
3. 执行入口，提供执行的能力。

OK，这就是一个基础的流程框架所需要实现的功能集。
等等？那业务呢？
所有组件都需要一个上下文，才能完成其功能，所以还需要一个业务上下文。
# 业务上下文
1. 跟框架无关，业务定制。
2. 跟框架执行有关，框架需要提供接入点。

这里需要思考一下，业务上下文的归属？是包含在框架里，还是仅提供接口，业务自由定制。

# 实现优先
假如说现在就要支持业务，那先实现了再说。
实现有如下三部分
1. 组件接口。
2. 流程串联，简单点，就是一个数组，按照数组的顺序执行。
3. 提供一个exec方法，入参就是业务上下文，按照顺序执行组件，

第一版大功告成。
老板也很满意。
现在有新的需求来了。
# 扩展：支持不同的业务上下文
我还有另外一个业务流。我发现业务上下文不满足要求。于是在里面加了另外一个范型参数。
例如
```go
BizContext{
	BizA T //不满足另外一个业务诉求
	BizB R
}
```
现在我在业务流程A执行的时候也能看见BizB参数，但我从来不会用到它。
注：不同的业务流程可以定制自己的上下文，如果框架想要干净，那么就会用变成liteflow的实现模式。

# 扩展：支持跳转
我期望支持 跳转的功能，也就是跳转到另一个节点去执行，因为我发现在走到组件padding之后，返回的数据不够，所以我想要再从某个组件开始执行，达到循环的目的。
```go
handle(ctx){
	if (ok){
		ctx.nextPipeline = "recallpipeline"
    }
}
```
例如上面的例子，需要再recall一次。
所以我直接在上下文中添加一个变量，支持跳转。
```go
BizContext{
	BizA T //不满足另外一个业务诉求
	BizB R
    NextPipeline string //跳转
}
```
很快实现，开发表示很赞。
注：这并不是一个好的实践，因为业务参数和框架功能的参数耦合在一起了。可以分成两个来解决此问题。

# 扩展3：支持并行
我期望支持并行的能力。因为我一个业务场景里面会分为两个小场景，这两部分的逻辑不同，而且可以并行。

例如：我支持了算法召回，手工召回。还有指定位置的召回，这三种可以并行，但是指定位置召回不需要filter，只需要补全信息即可。
算法召回->filter -> rank -> filling
普通召回->filter -> rank -> filling
然后统一进入merge流程。
如果还在一个流程里，那么在filter的组件里面，就需要判断上下文中哪些是可以filter的，哪些是不可以的。
如果一个组件中关注的逻辑太多，就会导致组件臃肿，而开发框架的目的就是尽可能的复用组件的能力，而不是每次在一个组件中添加逻辑，这样也失去了复用组件去快速支持新的业务。
框架不满足新的业务场景了，如何升级框架。

通过上面的几个扩展，我们可以了解到需要支持的能力：
1. 多分支流程支持
2. 并行执行
3. 合并多个分支流程

是不是发现也没有啥问题，干就完了。
用go的话并发不是简简单单，就跟吃饭喝水一样。
那并发带来的数据竞争问题呢？用channel 传递！perfect！

## 遇到的问题
### 1.上下文如何分发呢？
例如说一开始只有一个业务上下文，开始并发，是要进行上下文复制吗？用同一个上下文？数据竞争问题怎么办？
这里有几种方案：
1. 使用`sync.Map` 将不同流程的上下文放到不同的key里。并发问题解决了，但是上下文不清晰，因为放在val里面了，用起来没那么方便
2. 复制上下文，从何处开始复制？何时开始复制？是在有并发的地方就复制吗？
3. 初始化多份上下文，发现有并发的地方就用对应的上下文。
4. 加锁。

这是并发带来的上下文的问题。
敏锐的你可能发现了，是否可以让业务自己解决上下文的问题，而框架只负责流程编排呢？
好问题，做职责分离。
接下来我去调研了两个框架`liteflow` & `goflow`. 看看这些出名的框架是怎么解决这个问题的。
为什么是这两个框架？这两个框架更适合使用在单个服务里面。
- java：https://github.com/dromara/liteflow
- go:https://github.com/s8sg/goflow
## liteflow
liteflow中有个`DataBus`的模块，是一个全局静态变量，来处理所有的上下文，每个请求会有一个slotIndex,请求内的组件如果需要上下文就可以从这里面获取，实现为`concurrentHashMap`.
你可能已经发现，一个请求内的并发对应的slotIndex是一样的，那么拿到的是同一个对象！
你需要自己处理并发问题，见【[文档](https://liteflow.cc/pages/845dff/#q-%E4%B8%8A%E4%B8%8B%E6%96%87%E9%87%8C%E7%9A%84%E6%95%B0%E6%8D%AE%E6%98%AF%E7%BA%BF%E7%A8%8B%E5%AE%89%E5%85%A8%E7%9A%84%E5%90%97)】。
## goflow
这个框架中的每个组件都有个参数`data []byte`，我想你已经明白了，当你拿到`[]byte`时，代表数据已经是序列化过的了。
所以每个组件都会拿到一份上下文的copy，从而避免了数据竞争的问题。

## 何去何从？

首先不能使用liteflow的方案，因为它没解决这个问题；其次业务流量比较大，如果太频繁的序列化反序列化会把CPU资源吃光，这个问题已经在localcache中遇见过，所以goflow的方案也不予考虑。

这个框架该如何升级？

回头再看看原始的方案。复制上下文，支持并发的容器。
如果是用多份上下文，那就需要提供工厂，还要确定再哪个节点开始使用这个流程的上下文。
如果使用复制上下文，那就在有并发的地方进行复制，当然后面需要接merge的节点，否则多个上下文不知道如何往下执行了。

你会发现为什么还在纠结上下文的问题？因为这是业务框架，要解决业务问题。
如果只需要实现流程编排，那不难。看第一个版本。

### 2.节点间的执行顺序控制？
例如：flow1: A->B->C->F->G
flow2: D->E->F->G
你会发现这是一个有向无环图`DAG`,那是否需要基于DAG去实现呢？
答案是：看业务。
我这里的业务，流程在10步以内，但是会有一两个阶段明确需要并发执行来提高效率。
所以我用两个 `Map`，一个用来标记节点，一个用来标记次序。
按照上面的例子
1. 标记节点的map:
   flow1_step1: A->B->C
   flow2_step1: D->E
   flow_step2: F->G
2. 标记次序的map:
    - 0:[flow1_step1,flow2_step1]
    - 1:[flow_step2]
    - 次序中的val是数组，如果数量大于1，表明需要并发执行。

如果使用DAG，那么所有已有的业务都需要迁移到这新的一套上面，包括接口层都需要变，成本很大。
我是想基于第一个版本上面改造，让接入成本不会太高。同时能支持以上的功能。

### 3. 跳转到指定节点执行？
上面说过DAG图是一个有向无环图，所以不能有环。但业务需要，否则改造较大。
但在一个支持并发的环境中支持环，这恐怕不太妙。上下文问题又出现了，根据什么跳转？跳转到哪里？
如果刚好是并发执行的节点呢？
发现特殊case很多，怎么办？

是否能有限的支持跳转？
答案是可以，例如我只支持flow1_step1中进行跳转，那一直都会是在一个协程中，不会有并发问题。


### 2.merge多个流程的组件如何设计？
那就不能是一个通用组件了，有没有想起map reduce的流程，merge其实就是在做reduce的过程。
而且需要merge多个上下文，那并发执行的等待怎么处理。
上面这两点在go里面比较好处理，waitGroup+ channel 就能很好的处理了。

# 总结
框架设计最好是专注于某个点。

# 代码附录
### Pipeline
```go
type Pipeline[T any, R any] interface {
	Handle(engineCtx *model.EngineContext[T, R]) error
	GetPipelineName() string
}

type AggregatePipeline[T any, R any] interface {
	// Handle key : data from pipeline, value : data from engineCtx
	Handle(engineCtxCh <-chan *model.EngineContext[T, R]) (*model.EngineContext[T, R], error)
	GetPipelineName() string
}
```

一个pipeline表示一个步骤，如果步骤中有多个操作要完成怎么办？

答案是在Pipeline的实现中去持有多个CommonProcessor，那么按照顺序执行Processor就能完成多个操作。
### 一个pipeline实现的例子
```go
type CommonPostPipeline[T any, R any] struct {
	Processors []processor.CommonProcessor[T, R]
}

func (p *CommonPostPipeline[T, R]) GetPipelineName() string {
	return "CommonPostPipeline"
}

func (p *CommonPostPipeline[T, R]) Handle(engineCtx *model.EngineContext[T, R]) error {
	for _, pr := range p.Processors {
		err := pr.Process(engineCtx)
		if err != nil {
			ulog.DefaultLogger().Withs("err", err.Error(), "engineCtx", engineCtx, "processor_name", pr.GetName()).Error(" processor execute err")
			return err
		}
	}
	return nil
}
```

### Processor

```go
type Processor[T any, R any] interface {
	GetName() string
}

type CommonProcessor[T any, R any] interface {
	Processor[T, R]
	Process(ctx *model.EngineContext[T, R]) error
}

type AggregateProcessor[T any, R any] interface {
	Processor[T, R]
	Process(ctx []*model.EngineContext[T, R]) (*model.EngineContext[T, R], error)
}
```

### 支持多个pipeline 串行执行

```go
type PipelineManager[T any, R any] struct {
	Pipelines        []Pipeline[T, R]
	PipelineIndexMap map[string]int
}

func NewPipelineManager[T any, R any](pipelines []Pipeline[T, R]) *PipelineManager[T, R] {
	pipelineIndexMap := make(map[string]int)
	for i, pipeline := range pipelines {
		pipelineIndexMap[pipeline.GetPipelineName()] = i
	}

	return &PipelineManager[T, R]{
		Pipelines:        pipelines,
		PipelineIndexMap: pipelineIndexMap,
	}
}

func (d *PipelineManager[T, R]) Do(engineCtx *model.EngineContext[T, R]) error {
	pipeline := d.Pipelines[0]
	index := 0
	for {
		if index >= len(d.Pipelines) {
			break
		}

		pipeline = d.Pipelines[index]
		startTime := time.Now()
		err := pipeline.Handle(engineCtx)
		engineCtx.AddTraceInfo(fmt.Sprintf("pipeline_name: %s, cost: %d", pipeline.GetPipelineName(), time.Since(startTime).Milliseconds()))
		if err != nil {
			return err
		}

		if engineCtx.GetNextPipelineName() != "" {
			i, ok := d.PipelineIndexMap[engineCtx.GetNextPipelineName()]
			if ok {
				index = i
				engineCtx.NextPipelineName = nil
			} else {
				return fmt.Errorf("wrong next_pipeline_name:%s", engineCtx.GetNextPipelineName())
			}
		} else {
			index++
		}
	}
	return nil
}

```
上面也支持了跳转到指定pipeline，只需要在具体的process中设置 `engineCtx.NextPipelineName =“xxxPipeline”`就能支持跳转了。

### 支持并发
```go
type PipelineGroup[T any, R any] struct {
	Name string
	//key: pipelinesId, value: pipelines
	//pinFlow: 【initPipeline->recallPipeline->fillPipeline】
	//nonPinFlow: initPipeline->recallPipeline->fillPipeline->rankPipeline->deboostPipeline
	//mergeFlow: mergePipeline->paddingPipeline
	//reuse PipelineManager，limited supported for goto， just goto the pipeline in PipelineManager
	PipelinesManagerMap map[string]*PipelineManager[T, R]
	//key:position, value: pipelinesId
	//0:[pinFlow,nonPinFlow]
	//1:[mergeFlow]
	PositionMaps map[uint][]string

	//key: pipelinesId, value: ctx factory func
	PipeLineCtxMap map[string]func() *model.EngineContext[T, R]

	//key: pipelinesId, value: aggregator pipelines
	AggregatorPipelinesMap map[string][]AggregatePipeline[T, R]
}

func (pg *PipelineGroup[T, R]) Run(engineCtx *model.EngineContext[T, R], option ...Option) error {
	positions := lo.Keys(pg.PositionMaps)
	sort.Slice(positions, func(i, j int) bool { return positions[i] < positions[j] })
	options := &Options{}
	if len(option) > 0 {
		for _, opt := range option {
			opt(options)
		}
	}

	var resultChan chan *model.EngineContext[T, R]

	EngCtx := engineCtx

	for _, pos := range positions {
		pipelineIds := pg.PositionMaps[pos]
		if len(pipelineIds) == 0 {
			continue
		}

		// Sequential execution
		if len(pipelineIds) == 1 {
			aggrPipelines := pg.AggregatorPipelinesMap[pipelineIds[0]]
			pManager := pg.PipelinesManagerMap[pipelineIds[0]]
			if len(aggrPipelines) > 0 {
				close(resultChan)
				for _, p := range aggrPipelines {
					if mergedCtx, err := p.Handle(resultChan); err != nil {
						return err
					} else {
						EngCtx = mergedCtx
					}
				}
			} else if pManager != nil {
				ctxFunc := pg.PipeLineCtxMap[pipelineIds[0]]
				if ctxFunc != nil {
					EngCtx = ctxFunc()
				}
				err := pManager.Do(EngCtx)
				if err != nil {
					return err
				}
			}
		} else {
			// parallel execute
			resultChan = make(chan *model.EngineContext[T, R], len(pipelineIds))

			group := errgroup.Group{}
			for _, pId := range pipelineIds {
				tempId := pId

				taskFunc := func(engContext *model.EngineContext[T, R]) error {
					newContext := engContext
					var err error
					if options.DeepCopyCtxWhenParallel {
						newContext, err = DeepCopy(engContext)
						if err != nil {
							return err
						}
					} else if ctxFunc := pg.PipeLineCtxMap[tempId]; ctxFunc != nil {
						newContext = ctxFunc()
					}
					pManager := pg.PipelinesManagerMap[tempId]
					err = pManager.Do(newContext)
					if err != nil {
						return err
					}

					resultChan <- newContext
					return nil
				}

				group.Go(func() error {
					return taskFunc(EngCtx)
				})

			}
			err := group.Wait()
			if err != nil {
				return err
			}
		}
	}

	return nil
}
```
上面的实现还有一个问题，就是上下文并没有和流程分开，按照正交分解的原则来说，框架上下文和业务上下文要分开，关注点分离，这样框架才不会侵入业务，业务也不会侵入框架。
反之一旦侵入，想要改回来就更难了。
不过这是时间和成本的考量。
