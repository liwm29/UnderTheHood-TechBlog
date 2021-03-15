# 设计模式

聊聊我熟悉的设计模式

首先,推荐一下这门课: https://time.geekbang.org/column/intro/100039001

我看了目录,确实很有吸引力,可惜太贵了:(

## 创建型

用于创建类型

### 单例

单例模式常用于创建全局唯一变量,大多数时候都增加了耦合,降低了可测试性

直接使用`sync.Once`,只调用一次是由once变量保证的

```go
type Once struct {
	done uint32
	m    Mutex
}
```

使用示例:

```go
package singleton

var (
    once sync.Once
    GlobalStatus map[string]string
)

func New() singleton {
	once.Do(func() {
		GlobalStatus = make(map[string]string)
	})
	return GlobalStatus
}
```

实现:

```go
func (o *Once) Do(f func()) {
	// Note: Here is an incorrect implementation of Do:
	//	if atomic.CompareAndSwapUint32(&o.done, 0, 1) {
	//		f()
	//	}
	if atomic.LoadUint32(&o.done) == 0 {
		o.doSlow(f)
	}
}

func (o *Once) doSlow(f func()) {
	o.m.Lock()
	defer o.m.Unlock()
	if o.done == 0 {
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}
```

> 注释说的很清楚,一个简单的CAS是不行的!
>
> 我们必须在f()调用完后,再将o.done置位

> 

### 工厂

#### 简单工厂

根据不同的类型

