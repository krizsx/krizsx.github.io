---
title: Go语言测试(二)
date: 2018-10-20T17:11:49+08:00
lastmod: 2018-10-20T17:11:49+08:00
cover: "/img/go-logo.jpg"
draft: false
categories: ["blog"]
tags: ["test", "go"]
description:
---

接口抽象,集成测试以及容器

<!--more-->

Go 语言要想写出可测试的代码,就需要对项目结构以及代码设计有一定要求了,基本上来说,只要灵活使用`interface`,就可以提高测试覆盖率,让代码质量更有保证

# Object-Oriented Programming (OOP) 面向对象编程

OOP 是经常使用的一种编程思想,在 Go 语言中,也有这种思想存在,虽然 Go 语言没有继承,但是可以通过给`struct`绑定`method`以及,`interface`和`struct`之间的两两组合实现 OOP

我个人认为 OOP 是非常重要的一种思想,在代码复杂度越来越高的时候,越能体会到这种思想带来的好处,接下来的测试基本都是基于这个思想

# 如何编写可测试的代码

很长一段时间里,我都觉得我写的代码好像没有办法写测试用例,它总是与各种各样的东西有关,单元测试看起来是不可能的,在经历几次重构之后,我发现其实所有的代码都是可以测试的,只要用合适的方式设计你的代码,就可以写出可测试的代码

## 1. 尽量避免全局变量

之前我的代码中,经常使用全局变量,因为用起来很方便,每个包都可以直接获取,但是这种做法会让我在写测试用例的时候,不得不初始化一遍这些全局变量,这让我感觉很难受,而且在你测试单个函数的时候,还需要查看源码,确认使用了哪些全局变量然后才能编写测试用例,不同的函数需要初始化的全局变量还不同,这无疑是不可取的,所以尽可能的不要使用全局变量,这里可以借鉴一点 FP(Functional Programming)的思想,函数需要的变量都应该是使用参数来传入,这样的话每一个函数都是独立的,不与其他变量耦合

## 2. 接口抽象

在调用函数的时候,如果直接使用包或者`struct`去调用,那么就会导致函数与对应的包或者`struct`产生耦合,这样就很难编写单元测试了.这时候可以使用`interface`来解耦,一是解除函数与固定`struct`的耦合,二是解除包与包之间的耦合,这样的话每个包,每个函数都是独立的,可以很方便的进行单元测试

# 举个例子 🌰

这个例子里一共有两个包,`model`包是数据结构,`api`包用来提供接口

```go
package model

// Message test struct
type Message struct {
	Name string
	Info string
}

// TableName 指定表名
func (Message) TableName() string {
	return "message"
}


```

```go
package api

import (
	"demo/message-demo/model"

	"github.com/jinzhu/gorm"
)

var (
	db *gorm.DB // 使用了全局变量
)

// GetMessage 获取信息
func GetMessage(name string) *model.Message {
	message := &model.Message{}
	if err := db.Where("name = ?",name).First(message).Error; err != nil { // 这里直接使用 gorm去查询数据库
		return nil
	}
	return message
}

```

如果我想要对`api`包进行接口测试,目前的写法很难进行单元测试,因为`GetMessage`这个函数与`db`这个变量耦合,所以想要测试这个函数的话,必须构建一个`*gorm.DB`对象,然而如果这样的话,函数执行时,就会真的去查数据库,这样就没办法编写单元测试了,接下来按照 OOP 的思想,以及上面两个建议来重构代码

## 1. 抽象接口

在`api`包中,调用了`gorm.DB`的`First`函数,这个函数是用来查询数据库获取第一条数据的,这样的话就可以抽象出一个接口来代替`gorm.DB`这个 struct,像这样

```go
// MessageDB 接口抽象
type MessageDB interface {
	FirstMessage(name string) *model.Message
}
```

然后在`GetMessage`函数中,就调用`MessageDB`接口而不是调用`gorm.DB`

## 2. 定义对象

定义好的`MessageDB`这个接口,应该是其他地方实现之后传给`api`这个包里面,实现依赖注入,刚才也建议不要使用全局变量,再加上 OOP 的思想,所以还需要定义一个`strcut`来绑定`GetMessage`这个函数,并且将`MessageDB`设置为这个`struct`的一个字段,这样的话,代码就会改造成这样

```go
package api

import (
	"demo/message-demo/model"
)

// MessageDB 接口抽象
type MessageDB interface {
	FirstMessage(name string) *model.Message
}

// MessageAPI 用来保存DB,并且绑定方法
type MessageAPI struct {
	DB MessageDB
}

// NewMessageAPI 构造函数
func NewMessageAPI(db MessageDB) *MessageAPI {
	return &MessageAPI{DB: db}
}

// GetMessage 获取信息
func (m *MessageAPI) GetMessage(name string) *model.Message {
	return m.DB.FirstMessage(name)
}

```

至于对于接口的实现呢,可以创建一个`database`包,把之前的代码移过去,像这样

```go
package database

import (
	"demo/message-demo/model"

	"github.com/jinzhu/gorm"
)

// GormDB 真正查询数据库的接口实现
type GormDB struct {
	db *gorm.DB
}

// NewGormDB 构造器
func NewGormDB(url string) (*GormDB, error) {
	db, err := gorm.Open("mysql", url)
	if err != nil {
		return nil, err
	}
	return &GormDB{db: db}, nil
}

// Close 关闭数据库连接
func (g *GormDB) Close() error {
	return g.db.Close()
}

// FirstMessage 接口实现
func (g *GormDB) FirstMessage(name string) *model.Message {
	message := &model.Message{}
	if err := g.db.Where("name = ?",name).First(message).Error; err != nil {
		return nil
	}
	return message
}
```

## 3. 重构结果

写成这样的好处:

- `api`包移除了对于`gorm.DB`这个第三方包的依赖
- `MessageDB`这个接口可以被多种实现替换,可以使用 mock 来代替查询真正的数据库
- `MessageAPI`这个对象用来存储`MessageDB`,移除了全局变量,单元测试时只需要初始化这一个对象即可,不需要再去看源码查找相关的代码

## 4. 编写测试用例

目前这样的代码结构就可以很方便的编写测试用例,可以使用 mock 对象,和上一篇提到的`TableDriver`

首先用`mockery`生成接口的 mock 对象

```sh
$ mockery -all
Generating mock for: MessageDB in file: mocks/MessageDB.go
```

创建`api_test.go`文件,编写测试用例,代码如下:

```go
package api_test

import (
	"testing"

	"github.com/stretchr/testify/assert"

	"demo/message-demo/api"
	"demo/message-demo/api/mocks"
	"demo/message-demo/model"
)

func TestMessageAPI_GetMessage(t *testing.T) {
	db := &mocks.MessageDB{}
	db.
		On("FirstMessage", "aaa").Return(&model.Message{Name: "", Info: "bbb"}).
		On("FirstMessage", "bbb").Return(nil)

	m := api.NewMessageAPI(db)

	testCases := []struct {
		name  string
		isNil bool
		info  string
	}{
		{"aaa", false, "bbb"},
		{"bbb", true, ""},
	}

	for _, tc := range testCases {
		t.Run(tc.name, func(tt *testing.T) {
			msg := m.GetMessage(tc.name)
			isNil := msg == nil
			assert.Equal(tt, tc.isNil, isNil)
			if !isNil {
				assert.Equal(tt, tc.info, msg.Info)
			}
		})
	}
}
```

通过生成的 mock 对象,就可以模拟一个`MessageDB`接口的实现,然后就可以测试`api`包了!

# 集成测试

上面的这种写法只是利用了 mock 来进行模拟测试,在很多场景下,其实这个并不能满足需求,例如我想对`database`这个包进行测试,但是它与 mysql 是无法解耦的,这时候我认为使用 docker 就是最好的方式了,docker 可以提供给你一个稳定的测试环境,每一次的测试,都是相同的初始环境,这对于测试来说是最好的了,所以接下来就利用 docker 来编写集成测试,集成测试应该写在`integration_test.go`这个文件中,表示里面的测试用例是需要外部依赖的

## 1. 使用 docker 启动一个 mysql

官方已经提供了一个镜像,只需要把 sql 文件添加到镜像中固定的目录,在容器启动时就会自动执行了,一共需要 3 个文件:

![](/images/2019/03/golang-test-2-docker.png)

### `Dockerfile`

```
FROM mysql:5.7.23

# 设置密码
ENV MYSQL_ROOT_PASSWORD=111111
# 添加初始化脚本
ADD ./test.sql /docker-entrypoint-initdb.d/
# 修改mysql默认配置文件
ADD ./mysqld.cnf /etc/mysql/mysql.conf.d/
```

### mysql 配置文件`mysqld.cnf`

```
# Copyright (c) 2014, 2016, Oracle and/or its affiliates. All rights reserved.
#
# This program is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation; version 2 of the License.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program; if not, write to the Free Software
# Foundation, Inc., 51 Franklin St, Fifth Floor, Boston, MA  02110-1301 USA

#
# The MySQL  Server configuration file.
#
# For explanations see
# http://dev.mysql.com/doc/mysql/en/server-system-variables.html

[mysqld]
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock
datadir         = /var/lib/mysql
#log-error      = /var/log/mysql/error.log
# By default we only accept connections from localhost
#bind-address   = 127.0.0.1
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
# character_set
default-storage-engine=INNODB
character-set-server=utf8mb4
collation-server=utf8_general_ci

[client]
default-character-set = utf8mb4
```

### 用于测试的`test.sql`文件

```sql
CREATE DATABASE test character set utf8;

USE test;

CREATE TABLE `message`(
    `name` varchar(50) DEFAULT NULL COMMENT '名字',
    `info` varchar(50) DEFAULT NULL COMMENT '信息'
) ENGINE=InnoDB CHARSET=utf8 COMMENT='测试用';

INSERT INTO message (name,info) VALUES ('aaa','bbb');
INSERT INTO message (name,info) VALUES ('bbb','ccc');
```

### 构建镜像并启动

![](/images/2019/03/golang-test-2-docker-start.png)

这样的话就可以每次都有一个稳定的测试数据库来进行测试了

## 2. 利用环境变量跳过或者运行测试用例

我看到过很多写集成测试的方式,我认为使用环境变量来控制是最好的,在这个例子中,就可以在环境变量中传入 mysql 的地址,如果环境变量中无法获取这个地址,那么就可以跳过集成测试,代码如下

```go
package database_test

import (
	"demo/message-demo/database"
	"fmt"
	"os"
	"testing"

	_ "github.com/go-sql-driver/mysql"
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
)

const (
	envMySQL = "MYSQL_ADDR" //
	// 这里写的比较简单,用户名和密码就写死了
	urlStr = "root:111111@tcp(%s)/test?charset=utf8&parseTime=true&loc=Local&multiStatements=true"
)

func TestGormDB(t *testing.T) {
	addr, exists := os.LookupEnv(envMySQL) // 从环境变量中查找数据库地址
	if !exists {
		t.Skipf("%s not set, skip GormDB test", envMySQL)
	}

	db, err := database.NewGormDB(fmt.Sprintf(urlStr, addr))
	require.NoError(t, err)
	defer db.Close()

	testCases := []struct {
		name string
		info string
	}{
		{"aaa", "bbb"},
		{"bbb", "ccc"},
	}

	for _, tc := range testCases {
		t.Run(tc.name, func(tt *testing.T) {
			msg := db.FirstMessage(tc.name)
			if assert.NotNil(t, msg) {
				assert.Equal(t, tc.info, msg.Info)
			}
		})
	}
}

```

运行的结果就像这样

![](/images/2019/03/golang-test-2-docker-test.png)

利用 docker,就可以进行各种各样的集成测试了,每次测试之后只需要把容器删除就可以

![](/images/2019/03/golang-test-2-docker-stop.png)

# Docker 快速启动命令

## Redis

### 1. start

```sh
docker run -d -p 6379:6379 --name redis-quickstart redis
```

### 2. test

```sh
$ redis-cli
127.0.0.1:6379> set a b
OK
127.0.0.1:6379> get a
"b"
```

### 3. stop

```sh
docker stop redis-quickstart
docker rm redis-quickstart
docker system prune --volumes -f
```

## ElasticSearch & Kibana

### 1. 创建一个网络

```sh
docker network create es
```

### 2. 启动 ES

```sh
docker run -d --name elasticsearch --net es -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" elasticsearch
```

### 3. 启动 Kibana

```sh
docker run -d --name kibana --net es -p 5601:5601 kibana
```

> 如果想连接其他的 es 集群,可以使用`-e`,来指定
> docker run -d --name kibana -p 5601:5601 kibana -e http://xxxx:9200

### 4. 测试 ES

```sh
# 新建一个index
$ curl -X PUT 'localhost:9200/weather'
{"acknowledged":true,"shards_acknowledged":true,"index":"weather"}
# 查询
$ curl -X GET 'http://127.0.0.1:9200/_cat/indices'
yellow open weather _KNhQA1mR-eRH8mJIxxEIQ 5 1 0 0  324b  324b
yellow open .kibana P-U4OWMFS3iW5ZcOM6FgjQ 1 1 1 0 3.1kb 3.1kb
```

### 5. 测试 kibana

打开[http://localhost:5601](http://localhost:5601)

### 6. 关闭

```sh
docker stop kibana
docker rm kibana
docker stop elasticsearch
docker rm elasticsearch
docker system prune --volumes -f
```
