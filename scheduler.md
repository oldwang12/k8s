#### Setup

通过 Setup 函数中 c, err := opts.Config() 去连接k8s，开启选主、生成 informerfactory
```go
// Config return a scheduler config object
func (o *Options) Config() (*schedulerappconfig.Config, error) 
```

通过 Setup 函数 中 scheduler.New 去得到sched，
```go
// Setup creates a completed config and a scheduler based on the command args and options
cc, sched, err := Setup(ctx, opts, registryOptions...)
```


#### Run
```go
// Run executes the scheduler based on the given configuration. It only returns on error or when context is done.
func Run(ctx context.Context, cc *schedulerserverconfig.CompletedConfig, sched *scheduler.Scheduler) error
```

```go
// 开启所有的informers
// Start all informers.
	cc.InformerFactory.Start(ctx.Done())

// 同步缓存
// Wait for all caches to sync before scheduling.
	cc.InformerFactory.WaitForCacheSync(ctx.Done())

// 进入 sched.Run() --> scheduleOne 函数
// Run begins watching and scheduling. It starts scheduling and blocked until the context is done.
func (sched *Scheduler) Run(ctx context.Context){
  sched.SchedulingQueue.Run()
  
  // 1. flushBackoffQCompleted 每秒执行一次， 检查所有在backoffQ 的pod backoff 时候结束，若结束移动到 activeQ
  // 2. flushUnschedulablePodsLeftover 函数用于将在 unschedulablePods 中的存放时间超过 
  // podMaxInUnschedulablePodsDuration 值的 pod 移动到 backoffQ 或 activeQ 中。
  
  // func (p *PriorityQueue) Run() {
	// go wait.Until(p.flushBackoffQCompleted, 1.0*time.Second, p.stop)
	// go wait.Until(p.flushUnschedulablePodsLeftover, 30*time.Second, p.stop)
  // }

  
  
  // scheduleOne 在执行 Run 之前已经配置好。
  go wait.UntilWithContext(ctx, sched.scheduleOne, 0)

	<-ctx.Done()
	sched.SchedulingQueue.Close()
}

```

#### scheduleOne
```go
  // 通过 queue.Pop() 从调度队列中删除并拿出一个 pod
	podInfo := sched.NextPod()

  // 获取到 pod 对应的 调度器名称，从而拿到 framework 插件
  fwk, err := sched.frameworkForPod(pod)

  // 1. 对正在删除的pod进行跳过
  // 2. 
  if sched.skipPodSchedule(fwk, pod) {
		return
	}
```
