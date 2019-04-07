---
title:  Golang Packages
date: 2019-03-16 15:04:21
tags: golang
---
![golang](golang-packages/golang.jpg)
<!-- more -->
# Golang Package
## json

### github.com/bitly/go-simplejson

 - 直接操作json
 - Example

    ``` go
    package main

    import (
        "fmt"
        "github.com/bitly/go-simplejson"
    )

    func main() {
        json, err := simplejson.NewJson([]byte(`{"a":123}`), )
        if err != nil {
            panic(err)
        }
        a, err := json.Get("a").Int()
        if err != nil {
            panic(err)
        }
        fmt.Println(a) // out: 123
        nJson := simplejson.New()
        nJson.Set("a", 123)

        nJson.SetPath([]string{"a","b","m"}, 456)
        nJson.SetPath([]string{"a","b","c","d"}, map[string]interface{}{"a":123, "b":456, "c":"string"})
        b, err := nJson.MarshalJSON()
        if err != nil {
            panic(err)
        }
        fmt.Println(string(b)) // out: {"a":{"b":{"c":{"d":{"a":123,"b":456,"c":"string"}},"m":456}}}
    }

    ```

## Thirty platform SDK

### github.com/aws/aws-sdk-go

## Database

### Redis
- github.com/garyburd/redigo

### Mysql
- github.com/go-sql-driver/mysql

### MongoDB
- gopkg.in/mgo.v2

## Net

### GRPC
- github.com/golang/protobuf

## mux

### github.com/gorilla/mux

##  Log

### github.com/Sirupsen/logrus


## Pool

### Goroutine

- github.com/eleztian/ants

## Test

- github.com/eleztian/assert

## 全局ID
- github.com/satori/go.uuid


- gopkg.in/validator.v2
- gopkg.in/yaml.v2