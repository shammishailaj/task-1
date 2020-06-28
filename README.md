# Task Pool

Task Pool 是一个易于使用且高度可配置的 golang类库，专门用于任务的管理&执行，支持自定义次数的重发。

## 功能特点
- 线程安全 - task pool 内所有的方法以及暴露的接口都是线程安全的 
- 异步发送 - 调用 PushTask 方法后回立即返回，任务将会被传递到io线程中异步发送，不阻塞用户操作。
- 失败重试 - 用户可以通过设置初始化的参数来指定任务执行失败的次数，超过重试次数将被投递到失败队列。
- 优雅关闭 - 用户调用关闭方法进行关闭时，task pool 会将所有其缓存的数据进行发送，防止任务丢失。
- 结果可控 - 用户可以自己实现 Task 接口，里面包含任务执行成功和失败后调用的方法，以及任务具体执行的操作。

## 安装
```go
go get github.com/overtalk/task
```

## 使用步骤
### 1.配置Config
-  Config 是提供给用户的配置类，用于配制任务执行策略，您可以根据不同的需求设置不同的值，具体的参数含义如文章尾的配置详解所示。

### 启动 taskPool 进程
```go
c := task.GetDefaultConfig()
taskPool, err := task.NewTaskPool(c)
if err != nil {
    log.Fatal(err)
}
taskPool.Start() // 启动 task pool 实例
```

### 3.调用 PushTask 方法加入任务
- 任务需要自定义，实现一下接口即可
```go
type Task interface {
	Execute() error
	CallBack(result *Result)
}
```

- task pool 中提供了 PushTask 方法供用户将任务加入到任务池中
```go
// 执行任务
for i := 0; i < 10; i++ {
    t := &myTask{
        name:  fmt.Sprintf("task - %d", i),
        times: 0,
    }
    taskPool.PushTask(t)
}
```

### 4.关闭 taskPool
- taskPool 提供安全关闭的方式，会等待 taskPool 中缓存的所有的任务全部发送完成以后在关闭 taskPool
```go
taskPool.SafeClose()
```

### 5.获取任务执行结果
- taskPool 中的任务执行是异步的，所以需要用户实现 Task 接口，去获得每次执行的结果。

- 实现 Task 接口需要实现其中的Execute()方法和CallBack()方法，两个方法分别会在任务执行成功或失败的时候去调用，两个方法会都会接收一个Result 实例，用户可以根据Result实例在CallBack回调方法中去获得每次任务执行的结果。下面写了一个简单的使用样例。
```go
type myTask struct {
	name  string
	times int
}

func (myTask *myTask) Execute() error {
	fmt.Printf("*Execute* [%s], times = [%d]\n", myTask.name, myTask.times)

	if myTask.times == 0 {
		myTask.times++
		return errors.New("fail")
	}

	return nil
}

func (myTask *myTask) CallBack(result *task.Result) {
    if result.IsSuccessful() {
        fmt.Printf("*success* [%s]! times = [%d]\n", myTask.name, myTask.times)
    } else {
        fmt.Printf("*fail* [%s]! times = [%d]\n", myTask.name, myTask.times)
    }
}
```

- 用户可以根据自己的需求调用Result实例提供的方法来获取任务执行结果信息。

## taskPool 配置详解

| 参数                | 类型   | 描述                                                         |
| ------------------- | ------ | ------------------------------------------------------------ |
| MaxTaskNum          | Int    | 单个 taskPool 实例能缓存的任务数量上限，默认为 1000。  |
| MaxBlockSec         | Int    | 如果 taskPool 任务数量已达数量上限，调用者在 PushTask 方法上的最大阻塞时间，默认为 60 秒。<br/>如果超过这个时间后任务数量没有空余，PushTask 方法会抛出TimeoutException。如果将该值设为0，当任务数量无法得到满足时，PushTask 方法会立即抛出 TimeoutException。如果您希望 PushTask 方法一直阻塞直到任务数量得到满足，可将该值设为负数。 |
| MaxIoWorkerNum      | Int64  | 单个 taskPool 能并发的最多groutine的数量，默认为50，该参数用户可以根据自己实际服务器的性能去配置。 |
| MaxRetryTimes       | Int    | 如果某个 task 首次执行失败，能够对其重试的次数，默认为 10 次。<br/>如果 retries 小于等于 0，该 ProducerBatch 首次发送失败后将直接进入失败队列。 |
| BaseRetryBackOffMs  | Int64  | 首次重试的退避时间，默认为 100 毫秒。 taskPool 采样指数退避算法，第 N 次重试的计划等待时间为 baseRetryBackOffMs * 2^(N-1)。 |
| MaxRetryBackOffMs   | Int64  | 重试的最大退避时间，默认为 50 秒。                           |

## 问题反馈
如果您在使用过程中遇到了问题，可以创建 [GitHub Issue](<https://github.com/overtalk/task/issues>)。