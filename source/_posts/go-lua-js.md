---
title: Go 执行Lua脚本和JS脚本测试
tags:
  - Backend
  - Golang
categories:
  - Golang
date: 2018.08.30 17:08:12
updated: 2018.08.30 17:08:12
---


最近有个需求需要在Go项目里面执行动态脚本，github上有好几个lua执行解释器，但是有很多要不就很久没维护了，要不就没有什么文档，经过几个对比我最后用的是 https://github.com/yuin/gopher-lua。JS解析器用的github.com/robertkrimen/otto。

具体测试代码如下，给有需求的朋友参考。

[github地址](https://github.com/fanlv/runJsAndrLuaInGo)


	package main
	
	import (
		"fmt"
		"github.com/robertkrimen/otto"
		"github.com/yuin/gluamapper"
		"github.com/yuin/gopher-lua"
		"time"
	)
	
	//function add(a, b)
	//return a+b
	//end
	var luaCode = `
	function testFun(tab)
		result = {}
		result["key"] = "test"
		result["key1"] = "val2"
	
	    if(tab["user"]=="test")then
	        result["title"]="good"
	    end
	    if(tab["os"]=="ios")then
	        result["url"]="http://www.google.com"
	    else
	        result["url"]="http://www.baidu.com"
	    end
	    
		return result
	end
	`
	
	func main() {
		dic := make(map[string]string)
		dic["user"] = "test"
		dic["os"] = "ios"
		dic["version"] = "1.0"
	
		start0 := time.Now()
		count := 10000
		for i := 0; i < count; i++ {
			LuaTest(dic)
		}
		tmp1 := time.Since(start0).Nanoseconds() / 1000 / 1000
	
		start1 := time.Now()
		for i := 0; i < count; i++ {
			JsTest(dic)
		}
		tmp2 := time.Since(start1).Nanoseconds() / 1000 / 1000
		fmt.Printf("LuaTest : %d,JsTest : %d", tmp1, tmp2)
	
	}
	
	func LuaTest(dic map[string]string) {
		L := lua.NewState()
		defer L.Close()
		if err := L.DoString(luaCode); err != nil {
			panic(err)
		}
		table := L.NewTable()
		for k, v := range dic {
			L.SetTable(table, lua.LString(k), lua.LString(v))
		}
	
		if err := L.CallByParam(lua.P{
			Fn:      L.GetGlobal("testFun"),
			NRet:    1,
			Protect: true,
		}, table); err != nil {
			panic(err)
		}
		ret := L.Get(-1) // returned value
		L.Pop(1)         // remove received value
		obj := gluamapper.ToGoValue(ret, gluamapper.Option{NameFunc: printTest})
		fmt.Println(obj)
	}
	
	func printTest(s string) string {
		return s
	}
	
	func JsTest(dic map[string]string) {
		vm := otto.New()
		v, err := vm.Run(`
	function testFun(tab) {
		result = {}
		result["key"] = "test"
		result["key1"] = "val2"
	 	if(tab["user"]=="test"){
	       result["title"]="good"
	    }
	    if(tab["os"]=="ios"){
	        result["url"]="http://www.google.com"
		}else{
	        result["url"]="http://www.baidu.com"
	    }
		return result
	}
	`)
		if err == nil {
			fmt.Println(v)
		}
		jsa, err := vm.ToValue(dic)
		if err != nil {
			panic(err)
		}
		result, err := vm.Call("testFun", nil, jsa)
	
	
		tmpR, err := result.Export()
		fmt.Println("object: ", tmpR)
	
	}
