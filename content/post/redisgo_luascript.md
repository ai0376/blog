---
title: "redisgo 操作lua script"
date: 2019-11-06T17:47:20+08:00
draft: false
tags: ["redisgo", "lua"]
---

redisgo 操作lua脚本参数传递过程中需要注意

`script.Do()`方法的参数一定要搞对，否则很容易出错， 要么填arg[0], arg[1] 这种形式或者用redis.Args{}往里面填参数，其他方式容易出错. 要保证通过(s *Script) args(spec string, keysAndArgs []interface{}) 方法操作之后所有的参数都在一个slice 中。


* test.lua
```
local key=KEYS[1]
local val=ARGV[1]
return redis.call('SET', key, val)
```

* go code
```
package main

import (
	"fmt"
	"io/ioutil"
	"time"

	"github.com/gomodule/redigo/redis"
)

func CliPool() *redis.Pool {
	return &redis.Pool{
		MaxIdle:     10,
		MaxActive:   100,
		IdleTimeout: time.Duration(20) * time.Second,
		Dial: func() (redis.Conn, error) {
			c, err := redis.Dial("tcp", "127.0.0.1:6379")
			if err != nil {
				return nil, err
			}
			return c, nil
		},
	}
}

func main() {
	var data []byte
	var err error
	if data, err = ioutil.ReadFile("test.lua"); err != nil {
		fmt.Printf("read failed %v", err.Error())
		return
	}
	script := redis.NewScript(1, string(data))
	pool := CliPool()
	defer pool.Close()

	cli := pool.Get()
	script.Load(cli)

	arg := []string{"key", "value"}
	args := redis.Args{}.AddFlat(arg)
	var reply interface{}
	if reply, err = script.Do(cli, args...); err != nil {
		fmt.Printf("redis exec failed %v", err.Error())
		return
	}
	fmt.Printf("return %v", reply)
}
```