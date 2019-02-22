+ sync.Mutex是Go标准库中常用的一个排外锁。当一个goroutine获得了这个锁的拥有权之后，其他请求锁的goroutine就会阻塞在Lock方法的调用上，直到锁被释放。

#### 初版的Mutex
+ Russ Cox在2008提交的第一版Mutex的实现
```golang
type Mutex struct {
	key  int32
	sema int32
}

func xadd(val *int32, delta int32) (new int32) {
	for {
		v := *val
		if cas(val, v, v+delta) { //利用cas对key进行加一，
			return v + delta
		}
	}
	panic("unreached")
}

func (m *Mutex) Lock() {
	if xadd(&m.key, 1) == 1 { //如果key的值从0加到1，则直接获得了锁
		//change from 0 to 1
		return
	}
	sys.semacquire(&m.sema) //否则，通过semacquire进行sleep，被唤醒的时候就获得了锁
}

func (m *Mutex) Unlock() {
	if xadd(&m.key, -1) == 0 {
		//chang from 1 to 0
		return
	}
	sys.semrelease(&m.sema)
}
```
#### 源代码分析
+ 互斥锁有两种状态：正常状态和饥饿状态
* 正常状态下：所有等待锁的goroutine按照FIFO顺序等待。唤醒的goroutine不会直接拥有锁，而是会和新请求锁的goroutine竞争锁的拥有。新请求锁的goroutin有优势：它正在CPU上执行，而且可能有好几个，所以刚刚唤醒的goroutine有很大可能在锁竞争中失败。在这种情况下，这个被唤醒的goroutine会加入到等待队列的前面。如果一个等待的goroutine超过1ms没有获取锁，那么它将会把锁变成饥饿模式。
* 饥饿模式下：锁的所有权将从unlock的goroutine直接交给等待队列中的第一个。新来的goroutine将不会尝试去获得锁，即使锁看起来是unlock状态，也不会去尝试自旋操作，而是放在等待队列的尾部。
* 如果一个等待的goroutine获取了锁，并且满足以下其中的任何一个条件：1）它是队列中的最后一个；2）它等候的时间小于1ms。它会将锁的状态转换为正常状态
* 正常状态有很好的性能表现，饥饿模式也是非常重要的，因为它能阻止尾部延迟的现象。

+ 当一个goroutine获取这个锁的时候， 有可能这个锁根本没有竞争者， 那么这个goroutine轻轻松松获取了这个锁。
+ 而如果这个锁已经被别的goroutine拥有， 就需要考虑怎么处理当前的期望获取锁的goroutine。
+ 同时， 当并发goroutine很多的时候，有可能会有多个竞争者， 而且还会有通过信号量唤醒的等待者。
