# 信号(signal)
用于给进程发送事件通知.

信息处理方式:
1. 忽略
1. 按照系统默认方式处理
1. 自定义处理

向另一进程发送signal的要求: 我们必需是该进程的所有者或root.

## example
1. 自定义处理信号
```go
package main

import (
	"log"
	"os"
	"os/signal"
	"syscall"
	"time"
)

func main() {
	c := make(chan os.Signal, 1)
	signal.Notify(c, syscall.SIGHUP, syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT)

	go func() {
		s := <-c
		log.Printf("--- signal(%v) ---\n", s)

		os.Exit(0)
	}()

	time.Sleep(5 * time.Second)
}
```