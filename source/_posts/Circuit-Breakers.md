---
title: Circuit Breakers
date: 2019-03-24 14:13:36
tags: "CircuitBreakers"
categories: "Microservice"
---

![Status Tranlate](Circuit-Breakers/status.png)
<!-- more -->

# Circuit Breakers [断路器]

## Circuit Breaker 是什么

Circuit Breaker 的概念来源于电子工程(电流达到一定阙值时断开电路)，其目的是当下游服务因压力过大而响应变慢或者出错，上游服务为了保护系统整体的可用性暂时切断对下游服务的访问。

## Circuit Breaker 的作用

**牺牲局部保护整体**

如下有一个请求链：
ServiceA -> ServiceB -> ServiceC

如果ServiceC达到瓶颈获取故障。这时ServiceB就会有大量的请求超时而导致ServiceB阻塞大量的请求线程导致ServiceB变慢或不可用，同理进而对其上游服务产生同样的影响，最后整个系统完全瘫痪。

如果在ServiceC服务变慢或故障时，ServiceB切断对ServiceC的调用那么久不会出现上述的情景。这就是熔断器的作用。

## Circuit Breaker 原理

Circuit Breaker 的原理是维护一个状态机，总共有3个状态：
- Closed 关闭状态，服务可以正常使用，当请求在一定时间内失败数量达到一定的阙值则表示服务不可用，将状态切换到Open状态。
- Open 打开状态，来自应用的请求会立即返回失败。在熔断一定时间后切换到HalfOpen状态检查服务是否恢复。
- HalfOpen 半打开状态， 仅仅允许少量的服务请求通过，如果这些请求调用都成功，表示服务已经恢复，则将其切换到Closed状态，反之如果存在服务调用有一个失败则将其切换到Open状态。

状态转换如下图所示：（[图片来源](https://docs.microsoft.com/en-us/previous-versions/msp-n-p/dn589784(v%3dpandp.10))）
![Status Tranlate](Circuit-Breakers/status.png)


## Circuit Breaker 的golang实现
在go生态里有3个较好的实现Packages:
- github.com/sony/gobreaker
- github.com/afex/hystrix-go
- github.com/streadway/handy

下面阅读一下比较简洁的gobreaker代码

### 结构定义

```golang
// CircuitBreaker 是一个状态机用于防止发送极可能失败的请求。
type CircuitBreaker struct {
    // 熔断器名字
	name          string
    // HalfOpen状态下允许的最大请求数量
	maxRequests   uint32
    // Close状态下定期清除counts计数器的循环周期
	interval      time.Duration
    // timeout 是状态从Open切换到HalfOpen状态的周期时间。 默认60s
	timeout       time.Duration
    // 请求失败时调用，返回True切换到Open状态，默认为失败数量超过5个则切换到Open.
	readyToTrip   func(counts Counts) bool
    // 状态变化时调用
	onStateChange func(name string, from State, to State)
    // 状态并发控制锁
	mutex      sync.Mutex
    // 状态
	state      State
    // generation 状态变化一次加1，interval周期性加1(清空计数器)， 如果请求前后的generation不同则表示该请求无效。
	generation uint64
    // 各种计数器
	counts     Counts
    // 记录时间点，不同状态下游不同含义。Closed: 下一次刷新counts的时间点； open: 下一次切换到HalfOpen状态的时间点。
	expiry     time.Time
}

// 计数器
type Counts struct {
    // 请求数
    Requests             uint32
    // 成功次数
    TotalSuccesses       uint32
    // 失败次数
    TotalFailures        uint32
    // 连续成功数
    ConsecutiveSuccesses uint32
    // 连续失败数
	ConsecutiveFailures  uint32
}
```

### 核心代码

```golang
func (cb *CircuitBreaker) Execute(req func() (interface{}, error)) (interface{}, error) {
    // 检查是状态，是否可以通过请求，如果可以生成一个generation id, 否则返回错误信息
	generation, err := cb.beforeRequest()
	if err != nil {
		return nil, err
	}
    // 如果在执行请求时发生错误，则认为请求失败，记录相应的信息到状态机
	defer func() {
		e := recover()
		if e != nil {
			cb.afterRequest(generation, false)
			panic(e)
		}
	}()
    // 执行请求
    result, err := req()
    // 记录请求相应的状态的状态机
    cb.afterRequest(generation, err == nil)
    // 返回结果和错误信息
	return result, err
}

// 请求前处理
func (cb *CircuitBreaker) beforeRequest() (uint64, error) {
	cb.mutex.Lock()
	defer cb.mutex.Unlock()

	now := time.Now()
	// 生成generation
	state, generation := cb.currentState(now)

	// 检查状态，判断是否可以接受新的请求
	if state == StateOpen {
        return generation, ErrOpenState
        // HalfOpen状态只允许一定数量(maxRequests)的请求通过
	} else if state == StateHalfOpen && cb.counts.Requests >= cb.maxRequests {
		return generation, ErrTooManyRequests
	}
	// 如果可以接受新的请求，请求数加1
	cb.counts.onRequest()
	return generation, nil
}
// 请求后处理
func (cb *CircuitBreaker) afterRequest(before uint64, success bool) {
	cb.mutex.Lock()
	defer cb.mutex.Unlock()

	now := time.Now()
	// 获取状态和generation
	state, generation := cb.currentState(now)
	if generation != before {
		return
	}
	// 更新计数器，
	// 如果是hafOpen则要切换状态，如果连续maxRequests个请求都成功切换到Closed状态， 如果这次失败切换到Open状态
	if success {
		cb.onSuccess(state, now)
	} else {
		cb.onFailure(state, now)
	}
}
```

## 参考文献

- [Microsoft Circuit Breaker Pattern](https://docs.microsoft.com/zh-cn/azure/architecture/patterns/circuit-breaker)