---
title: Timer && Ticker
date: 2019-04-07 15:15:57
tags:
---

熟悉Golang的应该都用过Timer和Ticker，这里记录一下在使用中遇到的坑。


如下代码

```go
func Ticker() {
	for {
		select {
		case <-time.Tick(10 * time.Millisecond):
			fmt.Println("ttt")
		}
	}
}
func Ticker2() {
	ticker := time.NewTicker(time.Millisecond * 200)
	ticker.Stop()
	for {
		select {
		case <-ticker.C:
		}
	}
}
func AfterFunc() {
	for {
		select {
		case <-time.After(1 * time.Millisecond):
		}
	}
}
```