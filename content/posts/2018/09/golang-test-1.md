---
title: Go语言测试(一)
date: 2018-09-30T16:57:12+08:00
lastmod: 2018-09-30T16:57:12+08:00
cover: "/img/go-logo.jpg"
draft: false
categories: ["blog"]
tags: ["go", "test"]
description: 如何编写简单测试
---

如何编写简单测试?

<!--more-->

# Go 语言测试

## 自带测试介绍

Go 语言自己有一个轻量级的测试框架,由`go test`命令以及`testing`包组成

通过创建以`_test.go`为结尾的文件来编写对应文件的测试用例,该文件中应该包含名字为`TestXXX`,并且参数为`(t *testing)`的函数,那么执行`go test`命令的时候就会执行这些函数,如果在测试函数中调用了`t.Error`或者`t.Fail`,则认为测试用例失败

我个人认为,Go 语言的测试可以大致分为内部测试和外部测试两类:

- 外部测试:指的是测试代码和源代码在不同的包,名名规范应该是 `_test.go`
- 内部测试:指的是测试代码和源代码在是同一个包,命名规范应该是 `_internal_test.go`

我觉得大部分的测试应该是属于外部测试,就是只需要测试这个包大写的函数即可,如果有测试 private 函数的必要的话,可以再创建一个文件来写内部测试

下面举个小栗子:

写一个进行加法运算的函数

```go
// Package integers 整数计算
package integers

// Sum 求和
func Sum(x, y int) int {
	return x + y
}

```

### 单元测试(Unit Test)

那么就可以编写一个下面这样的测试用例

```go
package integers_test

import (
	"demo/integers"
	"testing"
)

func TestSum(t *testing.T) {
	actual := integers.Sum(2, 2)
	expected := 4
	if actual != expected {
		t.Errorf("expected '%d' but got '%d'", expected, actual)
	}
}

```

接下来可以执行`go test -v`执行测试用例,看到这样的输出就表示测试通过

![sum test](/img/2018/09/integers_test.png)

Go 语言测试中还支持写一个`Example`来说明函数的作用,并且会被写到`godoc`中方便查看,就以上面这个函数为例子

`Example`的语法是,函数名以`Example`开头,不需要参数,最后注释中有一个`Output:`来表示预期输出

```go
func ExampleSum() {
	actual := integers.Sum(1, 2)
	fmt.Println(actual)
	// Output: 3
}
```

这样就可以了,之后再执行一次`go test -v`

![sum example](/img/2018/09/integers_example.png)

再之后,可以通过

```shell
godoc -http=":6060"
```

> godoc 命令可以扫描所有 GOPATH 以及 GOROOT 中的包,并且生成这样一个文档

这个命令来启动文档,之后呢访问[http://localhost:6060/pkg/demo/integers/](http://localhost:6060/pkg/demo/integers/)

会看到这样一个页面

![integers doc](/img/2018/09/integers_doc.png)

### 压力测试(Benchmark Test)

还是以上面的 Sum 函数来编写一个压力测试,与普通的测试用例很像,只是函数名和参数需要替换一下

举个栗子:

```go
func BenchmarkSum(b *testing.B) {
	for i := 0; i < b.N; i++ {
		integers.Sum(1, 1)
	}
}
```

压力测试函数的前缀要变成`Benchmark`,并且参数变成`b *testing.B`,之后可以通过`b.N`来决定执行次数,至于具体会执行多少次,这个是`testing`包来决定,它会找到最合适的值,那接下来就可以使用`go test -v -bench=.`来执行压力测试

![integers benchmark](/img/2018/09/integers_benchmark.png)

这里显示最后的结果就是,执行一次`Sum`函数,需要 0.29 纳秒,一共执行了 2000000000 次

### 子测试(Sub Test)

在测试用例中,可以通过调用`t.Run`来执行一次子测试,有时候对于一个事情,会有不同的场景,那么就可以利用子测试来进行描述,并且可以复用代码

继续以上面的`Sum`函数来举栗子

```go
func TestSumMulti(t *testing.T) {

	assert := func(t *testing.T, actual, expected int) {
		// 这里调用Helper函数表示当前函数并不是一个真正的测试用例
		// 在输出错误行号时,会将调用该函数的函数的位置打印出,而不是打印在Helper函数中的行号
		t.Helper()
		if actual != expected {
			t.Errorf("expected '%d' but got '%d'", expected, actual)
		}
	}

	t.Run("2+2", func(t *testing.T) {
		actual := integers.Sum(2, 2)
		expected := 5 // 这里故意失败一下
		assert(t, actual, expected)
	})
	t.Run("3+3", func(t *testing.T) {
		actual := integers.Sum(3, 3)
		expected := 6
		assert(t, actual, expected)
	})
}
```

然后运行`go test -v`可以看到

![sub test](/img/2018/09/integers_sub.png)

### 表格驱动 [Table-Driven](https://github.com/golang/go/wiki/TableDrivenTests)

表格驱动是 Go 语言非常推荐的一种测试方式,这种方式可以很清晰的看到输入与输出,并且 Go 语言的 struct 很适合编写表格驱动的测试用例,此外,当发现了错误的时候,也可以很轻松的修改测试用例

继续以上面的`Sum`函数举栗子:

```go
func TestSumTableDriven(t *testing.T) {
	// 定义一个struct,用来表示参数以及期望输出
	tests := []struct {
		name     string
		x        int
		y        int
		expected int
	}{
		{"1+1", 1, 1, 2}, // 表格中的每一条,包含输入的参数以及期望的输出
		{"2+3", 2, 3, 5},
		{"4+5", 4, 5, 9},
	}
	for _, test := range tests {
		// 利用子测试
		t.Run(test.name, func(t *testing.T) {
			actual := integers.Sum(test.x, test.y)
			if actual != test.expected {
				t.Errorf("expected '%d' but got '%d'", test.expected, actual)
			}
		})

	}
}
```

这样写的好处就很明显了,当你需要添加删除或者修改测试用例的参数,都可以直接修改表格中的数据,而不需要修改代码,而且使测试用例的代码非常地简介明了

## Testify

上面的写法都是 Go 语言自身提供的写法,但是在编写测试的时候就会发现,它有很多不足,所以我安利一个第三方的测试框架,叫做[Testify](https://github.com/stretchr/testify)

我自己常用的是下面三个包

- assert 用来判断结果是否符合期望,如果不符合会标记测试用例为失败,但是会继续执行
- require 和 assert 功能一样,但是如果不符合的话会立即终止测试程序
- mock 用来模拟一个假的实现对象

### 重构

下面就用这个包来重构一下刚才的代码

最开始的`TestSum`可以写成这样

```go

func TestSum(t *testing.T) {
	actual := integers.Sum(2, 2)
    expected := 4
    // Equal 函数可以判断是否相等,如果不相等,就会标记为失败
    assert.Equal(t, expected, actual)
    // 替换成require就是这样,函数和参数完全一致
    // require.Equal(t, expected, actual)
}

```

`TableDriven`可以写成这样

```go
func TestSumTableDriven(t *testing.T) {
	// 定义一个struct,用来表示参数以及期望输出
	tests := []struct {
		name     string
		x        int
		y        int
		expected int
	}{
		{"1+1", 1, 1, 2},
		{"2+3", 2, 3, 5},
		{"4+5", 4, 5, 9},
	}
	for _, test := range tests {
		t.Run(test.name, func(t *testing.T) {
			assert.Equal(t, test.expected, integers.Sum(test.x, test.y))
		})

	}
}

```

代码简洁了很多

### Mock 测试

mock 测试就是模拟一个对象来实现功能,为什么需要这个对象,是因为有时候不可以直接调用真正的实现,所以需要一个假的对象来代替,testify 的 mock 包就可以做到,举个栗子:

假设现在有一段调用 redis 的代码

```go
package mock

import "github.com/mediocregopher/radix.v2/redis"

// Handler 调用redis
type Handler struct {
	DB *redis.Client
}

// Ping 给redis发送一个命令
func (h *Handler) Ping() (string, error) {
	// 这里是真的redis客户端
	res := h.DB.Cmd("INCR", "ping")
	if res.Err != nil {
		return "", res.Err
	}
	return "pong", nil
}

```

按照这种写法,如果直接对 Handler 的 Ping 函数进行单元测试,那么肯定就需要创建一个 redis 的 client 去连接 redis 了,这明显是不符合要求的,所以需要对代码进行一下改造,将调用抽象为接口

因为调用 redisClient 的`Cmd`函数,所以定义这么一个接口

```go
package mock

import "github.com/mediocregopher/radix.v2/redis"

// RedisCaller 抽象出一个接口,这个接口是我们需要调用的,并且已经被真正的redis client实现,
type RedisCaller interface {
	Cmd(cmd string, args ...interface{}) *redis.Resp
}

// Handler 调用redis
type Handler struct {
	DB RedisCaller
}

// Ping 给redis发送一个命令
func (h *Handler) Ping() (string, error) {
	// 这里是真的redis客户端
	res := h.DB.Cmd("INCR", "ping")
	if res.Err != nil {
		return "", res.Err
	}
	return "pong", nil
}


```

之后通过`mock`包,编写一个 mock 对象来实现对应的接口

```go
package mock_test

import (
	my "demo/mock"
	"errors"
	"testing"

	"github.com/stretchr/testify/assert"

	"github.com/mediocregopher/radix.v2/redis"
	"github.com/stretchr/testify/mock"
)

type mockRedisCaller struct {
	mock.Mock // 只需要组合testify的mock包
}

// Cmd 实现接口
func (m *mockRedisCaller) Cmd(cmd string, args ...interface{}) *redis.Resp {
	var _ca []interface{}
	_ca = append(_ca, cmd)
	_ca = append(_ca, args...)
	// 这里是假装接收所有参数
	_args := m.Called(_ca...)
	// 这里是假装返回一个结果
	// Get 函数的参数指的是传入结果的index
	return _args.Get(0).(*redis.Resp)
}

func TestPing(t *testing.T) {
	sampleErr := errors.New("错误")
	tests := []struct {
		name     string
		response string
		err      error
	}{
		{
			name:     "success",
			response: "pong",
			err:      nil,
		},
		{
			name:     "fail",
			response: "",
			err:      sampleErr,
		},
	}
	for _, test := range tests {
		t.Logf("Running test case: %s", test.name)
		caller := &mockRedisCaller{}
		// 这个On表示的就是当调用 Cmd这个函数,并且参数是 INCR ping:count的时候
		caller.On("Cmd", "INCR", "ping").
			// Return表示返回什么样的结果
			Return(&redis.Resp{
				Err: test.err,
			}).Once()
		// 这里再创建之前需要测试的Handler
		h := &my.Handler{
			DB: caller,
		}
		// 这样调用Ping的时候,就会调用上面刚刚声明的caller了
		response, err := h.Ping()
		assert.Equal(t, test.err, err)
		assert.Equal(t, test.response, response)
	}
}

```

接下来运行`go test -v`

![mock test](/img/2018/09/mock_test.png)

这样就实现了替换掉真正的实现,而是用 mock 对象代替调用

### Mock 生成器

每次编写这样一个对象其实是很浪费时间的,所以这里安利一个 mock 对象生成器[mockery](https://github.com/vektra/mockery)

安装方式:

```shell
go get -u github.com/vektra/mockery
cd $GOPATH/github.com/vektra/mockery/cmd/mockery
go install
```

```shell
mockery --help
Usage of mockery:
  -all
        generates mocks for all found interfaces in all sub-directories
  -case string
        name the mocked file using casing convention [camel, snake, underscore] (default "camel")
  -cpuprofile string
        write cpu profile to file
  -dir string
        directory to search for interfaces (default ".")
  -inpkg
        generate a mock that goes inside the original package
  -keeptree
        keep the tree structure of the original interface files into a different repository. Must be used with XX
  -name string
        name or matching regular expression of interface to generate mock for
  -note string
        comment to insert into prologue of each generated file
  -outpkg string
        name of generated package (default "mocks")
  -output string
        directory to write mocks to (default "./mocks")
  -print
        print the generated mock to stdout
  -quiet
        suppress output to stdout
  -recursive
        recurse search into sub-directories
  -tags string
        space-separated list of additional build tags to use
  -testonly
        generate a mock in a _test.go file
  -version
        prints the installed version of mockery
```

使用方式非常简单,只需要到需要生成的目录,执行

```shell
mockery -all -testonly
```

以上面的`mock`包为例,执行命令之后就会出现

![mockery](/img/2018/09/mock_generator.png)

生成的代码是这样的

```go
// Code generated by mockery v1.0.0. DO NOT EDIT.

package mocks

import mock "github.com/stretchr/testify/mock"
import redis "github.com/mediocregopher/radix.v2/redis"

// RedisCaller is an autogenerated mock type for the RedisCaller type
type RedisCaller struct {
	mock.Mock
}

// Cmd provides a mock function with given fields: cmd, args
func (_m *RedisCaller) Cmd(cmd string, args ...interface{}) *redis.Resp {
	var _ca []interface{}
	_ca = append(_ca, cmd)
	_ca = append(_ca, args...)
	ret := _m.Called(_ca...)

	var r0 *redis.Resp
	if rf, ok := ret.Get(0).(func(string, ...interface{}) *redis.Resp); ok {
		r0 = rf(cmd, args...)
	} else {
		if ret.Get(0) != nil {
			r0 = ret.Get(0).(*redis.Resp)
		}
	}

	return r0
}

```

这样就不需要自己编写 mock 对象了,省了很多时间

---

## 测试覆盖率

相信通过上面的方法已经可以写出很多测试用例了,接下来就可以对这些测试进行一些分析,例如生成测试率报告

就以刚才那个 mock 测试为例,首先,在`go test`命令执行时添加`-cover`参数就可以直接看到覆盖率了

执行命令

```shell
go test -v -cover
```

![mock cover](/img/2018/09/mock_cover.png)

可以看到测试覆盖率是 100%,证明了测试用例中,成功和失败的都覆盖到了

还可以通过`-coverprofile`生成一份报告

```shell
go test -covermode=count -coverprofile=cover.out
```

这样就可以在当前目录生成一份覆盖率报告了,之后可以通过`go tool cover`来进行分析

比如说,分析每个函数的覆盖率

```shell
go tool cover -func=cover.out
```

会看到下面的结果

![mock cover func](/img/2018/09/mock_cover_func.png)

还可以使用命令来生成一个 html 页面

```shell
go tool cover -html=cover.out
```

![mock cover web](/img/2018/09/mock_cover_html.png)

---

第一篇 Go 语言测试简单介绍了如何编写简单的测试用例,并且生成覆盖率报告,之后的内容会介绍如何编写可以测试的代码,通过抽象接口,拆分实现,来让你的代码尽可能的可以测试,保证自己代码的健壮性,还有就是利用 Docker 以及 docker-compose 进行级联测试等
