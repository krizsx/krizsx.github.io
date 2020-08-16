---
title: "Golang中使用consul watch机制"
date: 2018-04-21T11:16:25+08:00
draft: false
tags: [consul, go]
categories: [blog]
# cover: "images/consul.png"
---

最近使用了[consul](https://www.consul.io/)作为服务的注册中心,但是在使用时发现官方的 api 没有提供可以添加 watch 的接口,这样的话服务节点信息变化就不能及时获取,但是在查看 consul 源码中发现了官方其实还是给了方法,只不过没有暴露 api 而已,直接上代码

<!--more-->

# consul watch 相关源码解读

watch 包在[这里](https://github.com/hashicorp/consul/tree/master/watch)

## 相关的 struct

### watch.Plan

这个在[watch.go](https://github.com/hashicorp/consul/blob/master/watch/watch.go)文件中

```go
type Plan struct {
	Datacenter  string // consul配置
	Token       string // consul配置
	Type        string // 需要监听变化的类型,比如nodes,service
	HandlerType string // handler的类型,可以是http的endpoint或者一个script,当然也可以是自己的func
	Exempt      map[string]interface{} // 额外参数

	Watcher   WatcherFunc // 这个是需要监听的变化
	Handler   HandlerFunc // 这个是在监听到变化之后的处理函数
	LogOutput io.Writer // 日志的writer,默认是os.Stdout

	address    string // consul地址
	client     *consulapi.Client // consul client
	lastIndex  uint64 // blocking quires 的index
	lastResult interface{} // blocking quires 上一次的结果

	stop       bool // 状态标志
	stopCh     chan struct{} // channel,用来传递信号
	stopLock   sync.Mutex
	cancelFunc context.CancelFunc
}
```

其中 Watcher 是需要监听的变化,如果节点信息变化之后就会调用 Handler,所以我们需要自定义的就是这个 Handler

### Watcher 和 Handler

也是在[watch.go](https://github.com/hashicorp/consul/blob/master/watch/watch.go)文件中,
这两个 func 的定义,很简单清晰

```go
// WatcherFunc is used to watch for a diff
type WatcherFunc func(*Plan) (uint64, interface{}, error)

// HandlerFunc is used to handle new data
type HandlerFunc func(uint64, interface{})

```

---

## 如何创建 Plan 并启动(以监听单个服务变化为例)

### watch.Parse 创建 Plan

```golang

// Parse takes a watch query and compiles it into a WatchPlan or an error
// 通过这个来创建Plan,参数是用map来传递的
func Parse(params map[string]interface{}) (*Plan, error) {
	return ParseExempt(params, nil)
}

// ParseExempt takes a watch query and compiles it into a WatchPlan or an error
// Any exempt parameters are stored in the Exempt map
// 解析参数map并创建Plan
func ParseExempt(params map[string]interface{}, exempt []string) (*Plan, error) {
	plan := &Plan{
		stopCh: make(chan struct{}),
		Exempt: make(map[string]interface{}),
	}

	// 所有的参数都是通过assert来转换成对应的类型的
	if err := assignValue(params, "datacenter", &plan.Datacenter); err != nil {
		return nil, err
	}
	if err := assignValue(params, "token", &plan.Token); err != nil {
		return nil, err
	}
	if err := assignValue(params, "type", &plan.Type); err != nil {
		return nil, err
	}
    // Ensure there is a watch type
    // 要保证type这个参数的值是consul支持的
	if plan.Type == "" {
		return nil, fmt.Errorf("Watch type must be specified")
	}
    // Get the specific handler
    // 这个是handler的type,如果是要指定自定义的func,那么就不需要这个了
	if err := assignValue(params, "handler_type", &plan.HandlerType); err != nil {
		return nil, err
	}
	switch plan.HandlerType {
        case "http":
    	// 此处省略...
	case "script":
		// Let the caller check for configuration in exempt parameters
	}

    // Look for a factory function
    // 从factory中根据type获取watcher
	factory := watchFuncFactory[plan.Type]
	if factory == nil {
		return nil, fmt.Errorf("Unsupported watch type: %s", plan.Type)
	}
    // Get the watch func
    // 获取watcher
	fn, err := factory(params)
	if err != nil {
		return nil, err
	}
	plan.Watcher = fn

    // 此处省略 ...
	return plan, nil
}

```

### Parse 需要的参数 map[string]interface 相关

#### 1. type

> 这个是创建 Plan 必要的参数,指的是 watcher 的类型,可以在[funcs.go](https://github.com/hashicorp/consul/blob/master/watch/funcs.go)中看到

```go
// watchFactory is a function that can create a new WatchFunc
// from a parameter configuration
// 所有的type都在这里面,在Parse中就是通过factory获取watcher
type watchFactory func(params map[string]interface{}) (WatcherFunc, error)

// watchFuncFactory maps each type to a factory function
var watchFuncFactory map[string]watchFactory

func init() {
	watchFuncFactory = map[string]watchFactory{
        // 监听KV中的key
        "key":       keyWatch,
        // 监听KV中以某种prefix开头
        "keyprefix": keyPrefixWatch,
        // 监听所有服务
        "services":  servicesWatch,
        // 监听节点变化
        "nodes":     nodesWatch,
         // 监听单个服务的
        "service":   serviceWatch,
        // 监听健康检查状态
        "checks":    checksWatch,
        // 监听event
        "event":     eventWatch,
    }
}

```

---

下面是 watcher 需要的参数,也是放在 params 中

Parse 在从 factory 中获取到对应的 watcher 之后,会将参数传入

其实和直接调用[api](https://github.com/hashicorp/consul/blob/master/api/health.go)查询的 QueryOptions 是一样的

#### 2. service

> 必要参数 ,需要监听的单个服务的名字,用 string 表示就可以

#### 3. tag

> 非必要参数,表示查询时服务需要拥有的 tag

#### 4. passingonly

> 非必要参数,表示服务是否必须是通过 check 的

有了这些参数就可以创建出 WatcherFunc 了,下面的是 serviceWatch 创建的源码

```go
// serviceWatch is used to watch a specific service for changes
func serviceWatch(params map[string]interface{}) (WatcherFunc, error) {
	stale := false
	if err := assignValueBool(params, "stale", &stale); err != nil {
		return nil, err
	}
	var service, tag string
	if err := assignValue(params, "service", &service); err != nil {
		return nil, err
	}
	if service == "" {
		return nil, fmt.Errorf("Must specify a single service to watch")
	}
	if err := assignValue(params, "tag", &tag); err != nil {
		return nil, err
	}
	passingOnly := false
	if err := assignValueBool(params, "passingonly", &passingOnly); err != nil {
		return nil, err
	}

	fn := func(p *Plan) (uint64, interface{}, error) {
		health := p.client.Health()
		opts := makeQueryOptionsWithContext(p, stale)
        defer p.cancelFunc()
        	// 其实这里也是调用了api去查询的
		nodes, meta, err := health.Service(service, tag, passingOnly, &opts)
		if err != nil {
			return 0, nil, err
		}
		return meta.LastIndex, nodes, err
	}
	return fn, nil
}
```

---

### Plan 设置自己的 Handler

```go
// 自定义的handler
// 这里的这个result是根据watcher的type不同变化的
// 这个例子创建的是service的watcher,所以这个result是ServiceEntry的slice
handler := func (index uint64, result interface{}) {
	fmt.Println("service changed")
	if entries, ok = result.([]*consulApi.ServiceEntry); ok {
		// your code
	}
}

plan.Handler = handler
```

---

### plan.Run() 启动 plan

```go
// Run is used to run a watch plan
func (p *Plan) Run(address string) error {
	// Setup the client
	p.address = address
	conf := consulapi.DefaultConfig()
	conf.Address = address
	conf.Datacenter = p.Datacenter
	conf.Token = p.Token
	// 这里是创建consul的client
	client, err := consulapi.NewClient(conf)
	if err != nil {
		return fmt.Errorf("Failed to connect to agent: %v", err)
	}
	p.client = client
	// Create the logger
	output := p.LogOutput
	if output == nil {
		output = os.Stderr
	}
	logger := log.New(output, "", log.LstdFlags)

	// Loop until we are canceled
	failures := 0
OUTER:
	for !p.shouldStop() {
		// Invoke the handler
		// 调用watcher来获取查询的结果
		index, result, err := p.Watcher(p)
		// Check if we should terminate since the function
		// could have blocked for a while
		if p.shouldStop() {
			break
		}
		// Handle an error in the watch function
		// watcher的错误处理
		if err != nil {
			// Perform an exponential backoff
			// 失败次数增加
			failures++
			// 将index归零
			p.lastIndex = 0
			retry := retryInterval * time.Duration(failures*failures)
			if retry > maxBackoffTime {
				retry = maxBackoffTime
			}
			logger.Printf("consul.watch: Watch (type: %s) errored: %v, retry in %v",
				p.Type, err, retry)
			select {
			// 执行重试
			case <-time.After(retry):
				continue OUTER
			case <-p.stopCh:
				return nil
			}
		}
		// Clear the failures
		// 成功之后将失败记录次数归零
		failures = 0
		// If the index is unchanged do nothing
		// 判断blocking quries 的index
		if index == p.lastIndex {
			continue
		}
		// Update the index, look for change
		oldIndex := p.lastIndex
		p.lastIndex = index
		if oldIndex != 0 && reflect.DeepEqual(p.lastResult, result) {
			continue
		}
		if p.lastIndex < oldIndex {
			p.lastIndex = 0
		}
		// Handle the updated result
		p.lastResult = result
		if p.Handler != nil {
			// 这里会调用自定义的handler
			p.Handler(index, result)
		}
	}
	return nil
}
```

---

通过上面几步就可以创建 plan 来监听服务节点变化啦

# 完整 demo

这里是[地址](https://github.com/krizss/demo/blob/master/consul_watch/main.go)

```go
package main

import (
	"fmt"
	"net/http"

	consulApi "github.com/hashicorp/consul/api"
	"github.com/hashicorp/consul/watch"
)

func main() {
	var (
		err    error
		params map[string]interface{}
		plan   *watch.Plan
		ch     chan int
	)
	ch = make(chan int, 1)

	params = make(map[string]interface{})
	params["type"] = "service"
	params["service"] = "test"
	params["passingonly"] = false
	params["tag"] = "SERVER"
	plan, err = watch.Parse(params)
	if err != nil {
		panic(err)
	}
	plan.Handler = func(index uint64, result interface{}) {
		if entries, ok := result.([]*consulApi.ServiceEntry); ok {
			fmt.Printf("serviceEntries:%v", entries)
			// your code
			ch <- 1
		}
	}
	go func() {
		// your consul agent addr
		if err = plan.Run("127.0.0.1:7888"); err != nil {
			panic(err)
		}
	}()
	go http.ListenAndServe(":8080", nil)
	go register()
	for {
		<-ch
		fmt.Printf("get change")
	}
}

func register() {
	var (
		err    error
		client *consulApi.Client
	)
	client, err = consulApi.NewClient(&consulApi.Config{Address: "127.0.0.1:7888"})
	if err != nil {
		panic(err)
	}
	err = client.Agent().ServiceRegister(&consulApi.AgentServiceRegistration{
		ID:   "",
		Name: "test",
		Tags: []string{"SERVER"},
		Port: 8080,
		Check: &consulApi.AgentServiceCheck{
			HTTP: "",
		},
	})
	if err != nil {
		panic(err)
	}
}

```
