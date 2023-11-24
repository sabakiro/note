# 分布式ID生成器
# snowflake算法
1. 首先确定数值是64位，int64类型，被划分为4部分，
2. 不含开头的第一位，因为这个位是符号位。
3. 随后用41位来表示收到请求时的时间戳，单位为毫秒，
4. 然后用5位来表示数据中心的ID，
5. 再用5位来表示机器的实例ID，
6. 最后是12位的循环自增ID（到达1111 1111 1111后会归零）。


# 分布式锁
# 进程内加锁
```go
// ... 省略之前的部分
var wg sync.WaitGroup
var l sync.Mutex
for i := 0; i < 1000; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        l.Lock()
        counter++
        l.Unlock()
    }()
}
wg.Wait()
println(counter)
// ... 省略之后的部分
```
## 尝试锁
只是希望一个任务有单一的执行者，而不像计数器场景那样，
所有Goroutine都执行成功。后来的Goroutine在抢锁失败后，
需要放弃其流程。这时候就需要尝试锁（try lock）了
尝试锁如果加锁成功执行后续流程，如果加锁失败也不会阻塞，而会直接返回加锁的结果
```go
package main
import (
    "sync"
)
/*
在Go语言中可以用大小为1的通道模拟尝试锁

Lock 尝试锁
*/
type Lock struct {
	c chan struct{}
}

// NewLock 生成一个尝试锁
func NewLock() Lock {
    var l Lock
    l.c = make(chan struct{}, 1)
    l.c <- struct{}{}
    return l
}
// Lock 锁住尝试锁返回加锁结果
func (l Lock) Lock() bool {
    lockResult := false
    select {
    case <-l.c:
        lockResult = true
    default:
    }
    return lockResult
}
// Unlock 解锁尝试锁
func (l Lock) Unlock() {
    l.c <- struct{}{}
}
var counter int
func main() {
    var l = NewLock()
    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            if !l.Lock() {
                // log error
                println("lock failed")
                return
            }
            counter++
            println("current counter", counter)
            l.Unlock()
        }()
    }
    wg.Wait()
}
```
逻辑限定每个Goroutine只有成功执行了Lock才会继续执行后续逻辑，
因此在Unlock时可以保证Lock结构体中的通道一定是空，
从而不会阻塞，也不会失败。
理论上还可以使用标准库中的CAS来实现相同的功能且成本更低

在单机系统中，尝试锁并不是一个好选择，
因为大量的Goroutine抢锁可能会导致CPU无意义的资源浪费。
有一个专有名词用来描述这种抢锁的场景——活锁。

活锁指的是程序看起来在正常执行，但实际上CPU周期被浪费在抢锁而非执行任务上，
从而导致程序整体的执行效率低下。
活锁的问题定位起来要麻烦很多，所以在单机场景下，不建议使用这种锁。

## 分布式锁
#### 基于Redis的setnx
```go
package main
import (
    "fmt"
    "sync"
    "time"
    "github.com/go-redis/redis"
)
func incr() {
    client := redis.NewClient(&redis.Options{
        Addr:     "localhost:6379",
        Password: "", // no password set
        DB:       0,  // use default DB
    })
    var lockKey = "counter_lock"
    var counterKey = "counter"
    // lock
    resp := client.SetNX(lockKey, 1, time.Second*5)
    lockSuccess, err := resp.Result()
    if err != nil || !lockSuccess {
        fmt.Println(err, "lock result: ", lockSuccess)
        return
    }
    // counter ++
    getResp := client.Get(counterKey)
    cntValue, err := getResp.Int64()
    if err == nil {
        cntValue++
        resp := client.Set(counterKey, cntValue, 0)
        _, err := resp.Result()
        if err != nil {
            // log err
            println("set value error!")
        }
    }
    println("current counter is ", cntValue)
    delResp := client.Del(lockKey)
    unlockSuccess, err := delResp.Result()
    if err == nil && unlockSuccess > 0 {
        println("unlock success!")
    } else {
        println("unlock failed", err)
    }
}
func main() {
    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            incr()
        }()
    }
    wg.Wait()
}
```
远程调用setnx实际上和单机的尝试锁非常相似，
如果获取锁失败，那么相关的任务逻辑就不应该继续向前执行。

setnx很适合在高并发场景下，用来争抢一些“唯一”的资源。
例如，交易撮合系统中卖家发起订单，而多个买家会对其进行并发争抢。
这种场景我们没有办法依赖具体的时间来判断先后，因为不管是用户设备的时间，
还是分布式场景下的各台机器的时间，都是没有办法在合并后保证正确的时序的。
哪怕是同一个机房的集群，不同的机器的系统时间可能也会有细微的差别。

所以，我们需要依赖这些请求到达Redis节点的顺序来做正确的抢锁操作

## zooKeeper
```go
package main
import (
    "time"
    "github.com/samuel/go-zookeeper/zk"
)
func main() {
    c, _, err := zk.Connect([]string{"127.0.0.1"}, time.Second) //*10)
    if err != nil {
        panic(err)
    }
    l := zk.NewLock(c, "/lock", zk.WorldACL(zk.PermAll))
    err = l.Lock()
    if err != nil {
        panic(err)
    }
    println("lock succ, do your business logic")
    time.Sleep(time.Second * 10)
    // do some thing
    l.Unlock()
    println("unlock succ, finish business logic")
}

```

基于ZooKeeper的锁与基于Redis的锁的不同之处在于Lock成功之前会一直阻塞，
这与单机场景中的mutex.Lock很相似。
其原理也是基于临时Sequence节点和监视API（watch API），
例如这里使用的是/lock节点。
Lock会在该节点下的节点列表中插入自己的值，
只要节点下的子节点发生变化，就会通知所有监听节点的程序。
这时候程序会检查当前节点下最小的子节点的ID是否与自己的一致。
如果一致，说明加锁成功了。

这种分布式的阻塞锁比较适合分布式任务调度场景，
但不适合高频次持锁时间短的抢锁场景。
按照谷歌的Chubby论文里的阐述，
基于强一致协议的锁适用于粗粒度的加锁操作。
这里的粗粒度指锁占用时间较长

## 基于etcd
etcd是分布式系统中功能与ZooKeeper类似的组件

#### 基于etcd实现分布式阻塞锁
```go
package main
import (
    "log"
    "github.com/zieckey/etcdsync"
)
func main() {
    m, err := etcdsync.New("/lock", 10, []string{"http://127.0.0.1:2379"})
    if m == nil || err != nil {
        log.Printf("etcdsync.New failed")
        return
    }
    err = m.Lock()
    if err != nil {
        log.Printf("etcdsync.Lock failed")
        return
    }
    log.Printf("etcdsync.Lock OK")
    log.Printf("Get the lock. Do something here.")
    err = m.Unlock()
    if err != nil {
        log.Printf("etcdsync.Unlock failed")
    } else {
        log.Printf("etcdsync.Unlock OK")
    }
}
```

#### 使用的etcdsync的加锁流程如下
1. 先检查/lock路径下是否有值，如果有值，说明锁已经被别人抢了。
2. 如果没有值，那么写入自己的值。如果写入成功返回，说明加锁成功。如果写入时节点被其他节点写入过了，那么会导致加锁失败，这时候到第3步.
3. 监视/lock下的事件，此时陷入阻塞。
4. 当/lock路径下发生事件时，当前进程被唤醒。检查发生的事件是否是删除事件（说明锁持有者主动解锁）或者过期事件（说明锁过期失效）。如果是的话，那么回到1，走抢锁流程。

## 如何选择合适的锁
1. 如果业务属于还在单机就可以搞定的量级，那么按照需求使用任意的单机锁方案就可以。
2. 如果发展到了分布式服务阶段，但在业务规模不大、每秒查询数很小的情况下，使用哪种锁方案都差不多。如果公司内已有可以使用的ZooKeeper、etcd或者Redis集群，那么就尽量在不引入新的技术栈的情况下满足业务需求。
3. 业务发展到一定量级的话，就需要从多方面来考虑了。首先是你的锁是否在任何恶劣的条件下都不允许数据丢失，如果不允许丢失，那么就不要使用Redis的setnx的简单锁。
4. 如果对锁数据的可靠性要求极高，那就只能使用etcd或者ZooKeeper这种通过一致性协议保证数据可靠性的锁方案。但可靠的背后往往都是较低的吞吐量和较高的延迟。

# 延时任务系统

#### 非实时的任务解决思路
1. 实现一套类似crontab的分布式定时任务管理系统。
2. 实现一个支持定时发送消息的消息队列。
## 定时器的实现
#### 时间堆

最常见的时间堆一般用小顶堆实现，小顶堆其实就是一种特殊的二叉树

小顶堆的好处：
对定时器来说，如果堆顶元素比当前的时间还要大，那么说明堆内所有元素都比当前时间大。
进而说明这个时刻我们还没有必要对时间堆进行任何处理。定时检查的时间复杂度是O(1)。

当我们发现堆顶的元素小于当前时间时，那么说明可能已经有一批事件过期了，这时进行正常的弹出和堆调整操作就可以了。每一次堆调整的时间复杂度都是O(logn)。

Go自身的内置定时器就是用时间堆来实现的，不过并没有使用二叉堆，而是使用了扁平一些的四叉堆。
小顶堆的性质是，父结点比其4个子结点都小，子结点之间没有特别的大小关系要求。
四叉对结构
0
	3
		20
		4
		5
		13
	2
		12
		14
		15
		16
	2
		3
		10
		5
		4
	10
		99
		13
		11
		12
## 时间轮
用时间轮来实现定时器时，需要定义每一个格子的“刻度”，可以将时间轮想象成一个时钟，中心有秒针顺时针转动。每次转动到一个刻度时，就需要去查看该刻度挂载的任务列表是否有已经到期的任务

从结构上来讲，时间轮和散列表很相似，如果把散列算法定义为触发时间%时间轮元素大小，那么它就是一个简单的散列表。在散列冲突时，采用链表挂载散列冲突的定时器。

除了这种单层时间轮，业界也有一些时间轮采用多层实现

## 任务分发
定时任务被触发之后需要通知用户侧，有两种思路。
1. 将任务被触发的信息封装为一条消息，发往消息队列，由用户侧对消息队列进行监听。
2. 对用户预先配置的回调函数进行调用。

两种方案各有优缺点，
1. 首先，如果采用第一种思路，那么如果消息队列出故障就会导致整个系统不可用，当然，现在的消息队列一般也会有自身的高可用方案，大多数时候我们不用担心这个问题。
2. 其次，一般业务流程中间走消息队列的话会导致延时增加，定时任务若必须在触发后的几十毫秒到几百毫秒内完成，那么采用消息队列就会有一定的风险。
3. 如果采用第二种思路，会加重定时任务系统的负担。我们知道，单机的定时器执行时最害怕的就是回调函数执行时间过长，这样会阻塞后续的任务执行。在分布式场景下，这种顾虑依然是存在的。一个不负责任的业务回调可能会直接拖垮整个定时任务系统。
4. 所以我们还要考虑在回调的基础上增加经过测试的超时时间设置，并且对由用户填入的超时时间做慎重的审核。

## 数据再平衡和幂等考量
一个任务只会在持有主副本的节点上被执行。

当有机器故障时，任务数据需要进行数据再平衡的工作，
如节点1宕机，节点1的数据会被迁移到节点2和节点3上
大写字母为主副本，小写为副本
故障前：
	节点一： A b d E
	节点二： a C D
	节点三：B d e
节点一故障后：
	节点一：
	节点二： a  b C D E
	节点三：A B c d e

# 负载均衡
## 常见的负载均衡思路
1. 按顺序挑：例如上次选了第一台，那么这次就选第二台，下次选第三台，如果已经到了最后一台，那么下一次再从第一台开始。这种情况下我们可以把服务节点信息都存储在数组中，每次请求完成下游节点之后，将一个索引后移即可。在移到尽头时再移回数组开头处。
2. 随机挑一台：每次都随机挑，真随机伪随机均可。假设选择第x台机器，那么x可描述为rand.Intn()%n。

## 基于洗牌算法的负载均衡
#### 错误的洗牌导致的负载不均衡
1. 考虑到我们需要随机选取每次发送请求的节点，
2. 同时在遇到下游返回错误时换其他节点重试。
3. 所以我们设计一个和节点数组大小一致的索引数组，每次来新的请求，对索引数组做洗牌，
4. 然后取第一个元素作为选中的服务节点，
5. 如果请求失败，那么选择下一个节点重试，依此类推
```go
var endpoints = []string {
    "100.69.62.1:3232",
    "100.69.62.32:3232",
    "100.69.62.42:3232",
    "100.69.62.81:3232",
    "100.69.62.11:3232",
    "100.69.62.113:3232",
    "100.69.62.101:3232",
}
// 重点在这个shuffle
func shuffle(slice []int) {
    for i := 0; i < len(slice)/2; i++ {
        a := rand.Intn(len(slice))
        b := rand.Intn(len(slice))
        slice[a], slice[b] = slice[b], slice[a]
    }
}
func request(params map[string]interface{}) error {
    var indexes = []int {0,1,2,3,4,5,6}
    var err error
    shuffle(indexes)
    maxRetryTimes := 3
    idx := 0
    for i := 0; i < maxRetryTimes; i++ {
        err = apiRequest(params, indexes[idx])
        if err == nil {
            break
        }
        idx++
    }
    if err != nil {
        // logging
        return err
    }
    return nil
}
```

ps:第一个元素有更大的概率会被选中。在负载均衡的场景下，也就意味着节点数组中的第一台机器负载会比其他机器高不少（大约是3倍）

#### 修正洗牌算法
从数学上得到过证明的还是经典的Fisher-Yates算法，主要思路为每次随机挑选一个值，放在数组末尾。然后在n-1个元素的数组中再随机挑选一个值，放在数组末尾，依此类推。
```go
func shuffle(indexes []int) {
    for i:=len(indexes); i>0; i-- {
        lastIdx := i - 1
        idx := rand.Int(i)
        indexes[lastIdx], indexes[idx] = indexes[idx], indexes[lastIdx]
    }
}
```
Go的标准库中
```go
func shuffle(n int) []int {
    b := rand.Perm(n)
    return b
}
```