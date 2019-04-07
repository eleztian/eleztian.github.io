---
title: Exponential Backoff
date: 2019-03-31 16:00:54
tags: network
categories: "MicroServices"
---
![backoff](Exponential-Backoff/equation.svg)
<!-- more -->
# Exponential Backoff（指数退避算法）
## 指数退避算法是什么呢？
### wiki中这样解释：
> Exponential backoff is an algorithm that uses feedback to multiplicatively decrease the rate of some 
process, in order to gradually find an acceptable rate.

是一种用于通过反馈来成倍地降低某一个过程的速率以逐渐找到一个可行的速率的一种算法。通俗点讲，就是通过网络发送数据后，需要等待一段时间后再发送下一个，从而避免冲突。这里的等待时间就是通过这个算法实现的，是一个指数增长的数据。

其算法过程如下：

1. 确定基本退避时间，一般为端到端的往返时间为2t，2t也成为冲突窗口或争用期。
2. 定义参数k，k与冲突次数有关，规定k不能超过10，k=Min[冲突次数，10]。在冲突次数大于10，小于16时，k不再增大，一直取值为10。
3. 从离散的整数集合[0,1,2，……，(2k-1)]中随机的取出一个数r，等待的时延为r倍的基本退避时间，等于r x 2t。r的取值范围与冲突次数k有关，r可选的随机取值为2k个、这也是称为二进制退避算法的起因。
4.当冲突次数大于10以后，都是从0—210-1个2t中随机选择一个作为等待时间。
5. 当冲突次数超过16次后，发送失败，丢弃传输的帧，发送错误报告。

公式如下， E(c) 第c次sleep时间，$N=2^c -1$

![公式](Exponential-Backoff/equation.svg)

## 为什么要用指数退避算法呢？

使用指数退避算法，可以防止连续的失败，减轻失败服务的压力。

## 指数退避算法的应用场景有哪些呢？

1. 错误重试
2. 轮询

## 代码如何实现呢？

```golang
func BackOff(c int) float64 {
	return (math.Pow(2, float64(c)) - 1) / 2
}
```

# Jitter(抖动)

[src](https://docs.aws.amazon.com/zh_cn/general/latest/gr/api-retries.html) 大多数指数退避算法会利用抖动（随机延迟）来防止连续的冲突。由于在这些情况下您并未尝试避免此类冲突，因此无需使用此随机数字。但是，如果使用并发客户端，抖动可帮助您更快地成功执行请求。

## 代码实现

通过添加一个随机数来实现抖动

在github上发现一个较好的实现. github.com/jpillora/backoff

```golang
// Package backoff provides an exponential-backoff implementation.
package backoff

import (
	"math"
	"math/rand"
	"time"
)

// Backoff is a time.Duration counter, starting at Min. After every call to
// the Duration method the current timing is multiplied by Factor, but it
// never exceeds Max.
//
// Backoff is not generally concurrent-safe, but the ForAttempt method can
// be used concurrently.
type Backoff struct {
	//Factor is the multiplying factor for each increment step
	attempt, Factor float64
	//Jitter eases contention by randomizing backoff steps
	Jitter bool
	//Min and Max are the minimum and maximum values of the counter
	Min, Max time.Duration
}

// Duration returns the duration for the current attempt before incrementing
// the attempt counter. See ForAttempt.
func (b *Backoff) Duration() time.Duration {
	d := b.ForAttempt(b.attempt)
	b.attempt++
	return d
}

const maxInt64 = float64(math.MaxInt64 - 512)

// ForAttempt returns the duration for a specific attempt. This is useful if
// you have a large number of independent Backoffs, but don't want use
// unnecessary memory storing the Backoff parameters per Backoff. The first
// attempt should be 0.
//
// ForAttempt is concurrent-safe.
func (b *Backoff) ForAttempt(attempt float64) time.Duration {
	// Zero-values are nonsensical, so we use
	// them to apply defaults
	min := b.Min
	if min <= 0 {
		min = 100 * time.Millisecond
	}
	max := b.Max
	if max <= 0 {
		max = 10 * time.Second
	}
	if min >= max {
		// short-circuit
		return max
	}
	factor := b.Factor
	if factor <= 0 {
		factor = 2
	}
	//calculate this duration
	minf := float64(min)
	durf := minf * math.Pow(factor, attempt)
	if b.Jitter {
		durf = rand.Float64()*(durf-minf) + minf
	}
	//ensure float64 wont overflow int64
	if durf > maxInt64 {
		return max
	}
	dur := time.Duration(durf)
	//keep within bounds
	if dur < min {
		return min
	}
	if dur > max {
		return max
	}
	return dur
}

// Reset restarts the current attempt counter at zero.
func (b *Backoff) Reset() {
	b.attempt = 0
}

// Attempt returns the current attempt counter value.
func (b *Backoff) Attempt() float64 {
	return b.attempt
}

// Copy returns a backoff with equals constraints as the original
func (b *Backoff) Copy() *Backoff {
	return &Backoff{
		Factor: b.Factor,
		Jitter: b.Jitter,
		Min:    b.Min,
		Max:    b.Max,
	}
}
```

# 参考:

- [Exponential Backoff And Jitter](https://amazonaws-china.com/cn/blogs/architecture/exponential-backoff-and-jitter/)