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
