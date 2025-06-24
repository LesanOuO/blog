---
title: "go程序注册为windows服务"
date: "2023-07-18"
tags: ['Go']
categories: ['动手实践']
---

编写完go程序后编译成exe后，双击可正常执行，但是通过`sc`注册为服务后启动报错`错误1053：服务没有及时响应启动或控制请求`，百思不得其解，只能看看能不能有别的方式注册成服务。

通过使用开源库`github.com/kardianos/service`可以更方便的注册成windows服务，并不会出现上述情况，下面就是简单的测试代码。

```go
package main

import (
	"fmt"
	"github.com/kardianos/service"
	"os"
)


func main() {
	srvConfig := &service.Config{
		Name:        "WinServiceExample",
		DisplayName: "WinServiceExample",
		Description: "WinServiceExample",
	}
	prg := &program{}
	s, err := service.New(prg, srvConfig)
	if err != nil {
		fmt.Println(err)
	}
	if len(os.Args) > 1 {
		serviceAction := os.Args[1]
		switch serviceAction {
		case "install":
			err := s.Install()
			if err != nil {
				fmt.Println("安装服务失败: ", err.Error())
			} else {
				fmt.Println("安装服务成功")
			}
			return
		case "uninstall":
			err := s.Uninstall()
			if err != nil {
				fmt.Println("卸载服务失败: ", err.Error())
			} else {
				fmt.Println("卸载服务成功")
			}
			return
		case "start":
			err := s.Start()
			if err != nil {
				fmt.Println("运行服务失败: ", err.Error())
			} else {
				fmt.Println("运行服务成功")
			}
			return
		case "stop":
			err := s.Stop()
			if err != nil {
				fmt.Println("停止服务失败: ", err.Error())
			} else {
				fmt.Println("停止服务成功")
			}
			return
		}
	}

	err = s.Run()
	if err != nil {
		fmt.Println(err)
	}
}

type program struct{}

func (p *program) Start(s service.Service) error {
	fmt.Println("服务运行...")
	go p.run()
	return nil
}
func (p *program) run() {
	// 具体的服务实现
}
func (p *program) Stop(s service.Service) error {
	return nil
}
```

注意：启动go编译好的exe时，需要使用管理员权限运行才能够注册到服务中。