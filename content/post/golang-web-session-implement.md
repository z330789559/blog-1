---
title: "Golang Web Session Implement"
date: 2021-09-16T22:41:38+08:00
---

## 概 述

大家都知道 `session` 是`web应用`在服务器端实现的一种用户和服务器之间认证的解决方案，目前 `Go` 标准包没有为 `session` 提供任何支持，本文我将讲解`session`的实现原理，和一些常见基于`session`安全产生的防御问题。

当然有人可能看了会抬杠，说现在大部分不是前后端分离架构吗？对，你可以使用`JWT`解决你的问题。但是也有一些一体化`web`应用需要`session`，所以我准备造个轮子。自己造的轮子哪里出问题了，比别人更熟悉，有`bug`了，还不用求着别人修`bug`,自己修就好了，呵呵哈哈哈，当然这几句话有点皮😜。

![嘤 嘤 嘤...](https://tva1.sinaimg.cn/large/008eGmZEgy1goyrvcs0n1j307q04zaa0.jpg)

## 需 求

我觉得一名好的程序员，在写程序之前应该列一下需求分析，整理一下思路，然后再去写代码。

- 支持内存存储会话数据
- 支持分布式`redis`会话存储
- 会话如果有心跳就自动续命`30`分钟（生命周期）
- 提供防御：`中间人`，`会话劫持`，`会话重放`等攻击



## 工作原理

首先必须了解工作原理才能写代码，这里我就稍微说一下，`session`是基于`cookie`实现的，一个`session`对应一个`uuid`也是`sessionid`，在服务器创建一个相关的数据结构，然后把这个`sessionid`通过`cookie`让浏览器保存着，下次浏览器请求过来了就会有`sessionid`，然后通过`sessionid`获取这个会话的数据。

![请求过程](https://tva1.sinaimg.cn/large/008eGmZEgy1goyr6owhr0j30gw0c23z5.jpg)

## 代码实现

都是说着容易，实际写起来就是各种坑，不过我还是实现了。

![linus 🖕](https://tva1.sinaimg.cn/large/008eGmZEgy1goyrnh2ebmj30dc07j3yk.jpg)

**少说废话，还是直接干代码吧。**


1. 依赖关系

![依赖关系](https://tva1.sinaimg.cn/large/008eGmZEgy1goysgg2puij30hz0dz3ys.jpg)

上面是设计的相关依赖关系图，`session`是一个独立的结构体，
`GlobalManager`是整体的会话管理器负责数据持久化，过期会话垃圾回收工作♻️，`storage`是存储器接口，因为我们要实现两种方式存储会话数据或者以后要增加其他持久化存储，所以必须需要接口抽象支持，`memory`和`redis`是存储的具体实现。


2. `storage`接口

```go
package sessionx

// session storage interface
type storage interface {
    Read(s *Session) error
    Create(s *Session) error
    Update(s *Session) error
    Remove(s *Session) error
}
```
`storage`就9行代码，是具体的会话数据操作动作的抽象，全部参数使用的是`session`这个结构的指针，如果处理异常了就`即错即返回`。

为什么把函数签名的`形参`使用指针类型的，这个我想看的懂人应该知道这是为什么了😁

3. `memoryStore`结构体

```go
type memoryStore struct {
    sync.Map
}
```

`memoryStore`结构体里面就嵌入`sync.Map`结构体，一开始是使用的`map`这种，但是后面发现在并发读写然后加`sync.Mutex`锁🔐，性能还不如直接使用`sync.Map`速度快。`sync.Map`用来做`K:V`存储的，也就是`sessionid`对应`session data`的。


实现`storage`具体方法如下:

```go
func (m *memoryStore) Read(s *Session) error {
    if ele, ok := m.Load(s.ID); ok {
      // bug 这个不能直接 s = ele 
      s.Data = ele.(*Session).Data
      return nil
    }
    // s = nil
    return fmt.Errorf("id `%s` not exist session data", s.ID)
}
```
读取数据的时候先将持久化的数据读出来然后赋值给本次会话的`session`。

**注意:** 在go的`map`中的`struct`中的字段不能够直接寻址，官方**issue** [https://github.com/golang/go/issues/3117](https://github.com/golang/go/issues/3117)

其他几个函数:

```go
func (m *memoryStore) Create(s *Session) error {
      m.Store(s.ID, s)
      return nil
}

func (m *memoryStore) Remove(s *Session) error {
      m.Delete(s.ID)
      return nil
}

func (m *memoryStore) Update(s *Session) error {
      if ele, ok := m.Load(s.ID); ok {
        // 为什么是交换data 因为我们不确定上层是否扩容换了地址
        ele.(*Session).Data = s.Data
        ele.(*Session).Expires = s.Expires
        //m.sessions[s.ID] = ele
        return nil
      }
      return fmt.Errorf("id `%s` updated session fail", s.ID)
}
```
这句话代码没有什么好说的，写过`go`都能看得懂。


垃圾回收:

```go
func (m *memoryStore) gc() {
    // recycle your trash every 10 minutes
    for {
        time.Sleep(time.Minute * 10)
        m.Range(func(key, value interface{}) bool {
            if time.Now().UnixNano() >= value.(*Session).Expires.UnixNano() {
              m.Delete(key)
            }
            return true
        })
        runtime.GC()
        // log.Println("gc running...")
    }

}
```

比较会话过期时间，过期就删除会话，以上就是内存存储的实现。

4. `redisStore`结构体

```go
type redisStore struct {
    sync.Mutex
    sessions *redis.Client
}

func (rs *redisStore) Read(s *Session) error {
    sid := fmt.Sprintf("%s:%s", mgr.cfg.RedisKeyPrefix, s.ID)
    bytes, err := rs.sessions.Get(ctx, sid).Bytes()
    if err != nil {
      return err
    }
    if err := rs.sessions.Expire(ctx, sid, mgr.cfg.TimeOut).Err(); err != nil {
      return err
    }
    if err := decoder(bytes, s); err != nil {
      return err
    }
    // log.Println("redis read:", s)
    return nil
}

func (rs *redisStore) Create(s *Session) error {
    return rs.setValue(s)
}

func (rs *redisStore) Update(s *Session) error {
    return rs.setValue(s)
}

func (rs *redisStore) Remove(s *Session) error {
    return rs.sessions.Del(ctx, fmt.Sprintf("%s:%s", mgr.cfg.RedisKeyPrefix, s.ID)).Err()
}

func (rs *redisStore) setValue(s *Session) error {
    bytes, err := encoder(s)
    if err != nil {
        return err
    }
    err = rs.sessions.Set(ctx, fmt.Sprintf("%s:%s", mgr.cfg.RedisKeyPrefix, s.ID), bytes, mgr.cfg.TimeOut).Err()
    return err
}
```
代码也就50行左右，很简单就是通过`redis`客户端对数据进行持久化操作，把本地的会话数据提供`encoding/gob`序列化成二进制写到`redis`服务器上存储，需要的时候再反序列化出来。

**那么问题来了，会有人问了，redis没有并发问题吗？**

👨‍💻‍： 那我肯定会回答，你在问这个问题之前我不知道你有没有了解过`redis`？？？  

`Redis` 并发竞争指的是多个 `Redis` 客户端同时 `set key `引起的并发问题，`Redis` 是一种单线程机制的 `NoSQL` 数据库，所以 `Redis` 本身并没有锁的概念。

但是多客户端同时并发写同一个 `key`，一个 `key` 的值是 `1`，本来按顺序修改为 `2,3,4` ，最后 `key` 值是 `4`，但是因为并发去写 `key`，顺序可能就变成了 `4,3,2`，最后 `key` 值就变成了 `2`。

我这个库当前也就一个客户端，如果你部署到多个机子，那就使用 `setnx(key, value)` 来实现分布式锁，我当前写的这个库没有提供分布式锁，具体请自行`google`。

5. `manager`结构体

```go
type storeType uint8

const (
    // memoryStore store type
    M storeType = iota
    // redis store type
    R
    SessionKey = "session-id"
)

// manager for session manager
type manager struct {
    cfg   *Configs
    store storage
}

func New(t storeType, cfg *Configs) {
    switch t {
    case M:
        // init memory storage
        m := new(memoryStore)
        go m.gc()
        mgr = &manager{cfg: cfg, store: m}
    case R:
        // parameter verify
        validate := validator.New()
        if err := validate.Struct(cfg); err != nil {
            panic(err.Error())
        }

        // init redis storage
        r := new(redisStore)
        r.sessions = redis.NewClient(&redis.Options{
            Addr:     cfg.RedisAddr,
            Password: cfg.RedisPassword, // no password set
            DB:       cfg.RedisDB,       // use default DB
            PoolSize: int(cfg.PoolSize), // connection pool size
        })

        // test connection
        timeout, cancelFunc := context.WithTimeout(context.Background(), 8*time.Second)
        defer cancelFunc()
        if err := r.sessions.Ping(timeout).Err(); err != nil {
            panic(err.Error())
        }
        mgr = &manager{cfg: cfg, store: r}

    default:
      panic("not implement store type")
    }
}
```

`manager`结构体也就两个字段，一个存放我们全局配置信息，一个我们实例化不同的持久化存储的存储器，其他代码就是辅助性的代码，不细说了。

6. `Session`结构体

这个结构体是对应着浏览器会话的结构体，设计原则是一个`id`对应一个`session`结构体。

```go
type Session struct {
    // 会话ID
    ID string
    // session超时时间
    Expires time.Time
    // 存储数据的map
    Data map[interface{}]interface{}
    _w   http.ResponseWriter
    // 每个session对应一个cookie
    Cookie *http.Cookie
}
```

具体操作函数:

```go
// Get Retrieves the stored element data from the session via the key
func (s *Session) Get(key interface{}) (interface{}, error) {
    err := mgr.store.Read(s)
    if err != nil {
        return nil, err
    }
    s.refreshCookie()
    if ele, ok := s.Data[key]; ok {
        return ele, nil
    }
    return nil, fmt.Errorf("key '%s' does not exist", key)
}

// Set Stores information in the session
func (s *Session) Set(key, v interface{}) error {

    lock["W"](func() {
        if s.Data == nil {
          s.Data = make(map[interface{}]interface{}, 8)
        }
        s.Data[key] = v
    })

	  s.refreshCookie()
	  return mgr.store.Update(s)
}

// Remove an element stored in the session
func (s *Session) Remove(key interface{}) error {
	  s.refreshCookie()

    lock["R"](func() {
        delete(s.Data, key)
    })

	  return mgr.store.Update(s)
}

// Clean up all data for this session
func (s *Session) Clean() error {
    s.refreshCookie()
    return mgr.store.Remove(s)
}
// 刷新cookie 会话只要有操作就重置会话生命周期
func (s *Session) refreshCookie() {
    s.Expires = time.Now().Add(mgr.cfg.TimeOut)
    s.Cookie.Expires = s.Expires
    // 这里不是使用指针
    // 因为这里我们支持redis 如果web服务器重启了
    // 那么session数据在内存里清空
    // 从redis读取的数据反序列化地址和重新启动的不一样
    // 所有直接数据拷贝
    http.SetCookie(s._w, s.Cookie)
}
```
上面是几个函数是，会话的数据操作函数，`refreshCookie()`是用来刷新浏览器`cookie`信息的，因为我在设计的时候只有浏览器有心跳也就是有操作数据的时候，管理器就默认为这个浏览器会话还是活着的，会自动同步更新`cookie`过期时间，这个更新过程可不是光刷新`cookie`就完事的了，持久化的话的数据过期时间也一样更新了。

**Handler方法**

```go
// Handler Get session data from the Request
func Handler(w http.ResponseWriter, req *http.Request) *Session {
    // 从请求里面取session
    var session Session
    session._w = w
    cookie, err := req.Cookie(mgr.cfg.Cookie.Name)
    if err != nil || cookie == nil || len(cookie.Value) <= 0 {
        return createSession(w, cookie, &session)
    }
    // ID通过编码之后长度是73位
    if len(cookie.Value) >= 73 {
        session.ID = cookie.Value
        if mgr.store.Read(&session) != nil {
            return createSession(w, cookie, &session)
        }

        // 防止web服务器重启之后redis会话数据还在
        // 但是浏览器cookie没有更新
        // 重新刷新cookie

        // 存在指针一致问题，这样操作还是一块内存，所有我们需要复制副本
        _ = session.copy(mgr.cfg.Cookie)
        session.Cookie.Value = session.ID
        session.Cookie.Expires = session.Expires
        http.SetCookie(w, session.Cookie)
	    }
      // 地址一样不行！！！
      // log.Printf("mgr.cfg.Cookie pointer:%p \n", mgr.cfg.Cookie)
      // log.Printf("session.cookie pointer:%p \n", session.Cookie)
      return &session
}

func createSession(w http.ResponseWriter, cookie *http.Cookie, session *Session) *Session {
      // init session parameter
      session.ID = generateUUID()
      session.Expires = time.Now().Add(mgr.cfg.TimeOut)
      _ = mgr.store.Create(session)

      // 重置配置cookie模板
      session.copy(mgr.cfg.Cookie)
      session.Cookie.Value = session.ID
      session.Cookie.Expires = session.Expires

      http.SetCookie(w, session.Cookie)
      return session
}
```

`Handler`函数是从`http`请求里面读取到`sessionid`然后从持久化层读取数据然后实例化一个`session`结构体的函数，没有啥好说的，注释写上面了。



## 安全防御问题

首先我还是那句话：`不懂攻击，怎么做防守`。
那我们先说说这个问题怎么产生的:
> `中间人攻击`（`Man-in-the-MiddleAttack`，简称`MITM攻击`）是一种`间接`的入侵攻击，这种攻击模式是通过各种技术手段将受入侵者控制的一台计算机虚拟放置在网络连接中的两台通信计算机之间，这台计算机就称为`中间人`。

这个过程，正常用户在通过浏览器访问我们编写的网站，但是这个时候有个`hack`通过`arp`欺骗，把路由器的流量劫持到他的电脑上，然后黑客通过一些特殊的软件抓包你的网络请求流量信息，在这个过程中如果你`sessionid`如果存放在`cookie`中，很有可能被黑客提取处理，如果你这个时候登录了网站，这是黑客就拿到你的登录凭证，然后在登录进行`重放`也就是使用你的`sessionid`，从而达到访问你账户相关的数据目的。


```go
func (s *Session) MigrateSession() error {
    // 迁移到新内存 防止会话一致引发安全问题
    // 这个问题的根源在 sessionid 不变，如果用户在未登录时拿到的是一个 sessionid，登录之后服务端给用户重新换一个 sessionid，就可以防止会话固定攻击了。
    s.ID = generateUUID()
    newSession, err := deepcopy.Anything(s)
    if err != nil {
        return errors.New("migrate session make a deep copy from src into dst failed")
    }
    newSession.(*Session).ID = s.ID
    newSession.(*Session).Cookie.Value = s.ID
    newSession.(*Session).Expires = time.Now().Add(mgr.cfg.TimeOut)
    newSession.(*Session)._w = s._w
    newSession.(*Session).refreshCookie()
    // 新内存开始持久化
    // log.Printf("old session pointer:%p \n", s)
    // log.Printf("new session pointer:%p \n", newSession.(*Session))
    //log.Println("MigrateSession:", newSession.(*Session))
    return mgr.store.Create(newSession.(*Session))
}
```

如果大家写过`Java`语言，都应该使用过`springboot`这个框架，如果你看过源代码，那就知道这个框架里面的`session`安全策略有一个`migrateSession`选项，表示在登录成功之后，创建一个新的会话，然后讲旧的 `session` 中的信息复制到新的 `session` 中。

我参照他的策略，也同样在我这个库里面实现了，在用户匿名访问的时候是一个 `sessionid`，当用户成功登录之后，又是另外一个 `sessionid`，这样就可以有效避免会话固定攻击。

使用的时候也可以`随时使用通过MigrateSession进行调用`，这个函数一但被调用，原始数据和`id`全部被刷新了，内存地址也换了，可以看我的源代码。

## 使用演示

```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "time"

    sessionx "github.com/higker/sesssionx"
)

var (
    // 配置信息
    cfg = &sessionx.Configs{
          TimeOut:        time.Minute * 30,
          RedisAddr:      "127.0.0.1:6379",
          RedisDB:        0,
          RedisPassword:  "redis.nosql",
          RedisKeyPrefix: sessionx.SessionKey,
          PoolSize:       100,
          Cookie: &http.Cookie{
            Name:     sessionx.SessionKey,
            Path:     "/",
            Expires:  time.Now().Add(time.Minute * 30), // TimeOut
            Secure:   false,
            HttpOnly: true,
        },
    }
)

func main() {
    // R表示redis存储 cfg是配置信息
	  sessionx.New(sessionx.R, cfg)

    http.HandleFunc("/set", func(writer http.ResponseWriter, request *http.Request) {
        session := sessionx.Handler(writer, request)
        session.Set("K", time.Now().Format("2006 01-02 15:04:05"))
        fmt.Fprintln(writer, "set time value succeed.")
    })

    http.HandleFunc("/get", func(writer http.ResponseWriter, request *http.Request) {
        session := sessionx.Handler(writer, request)
        v, err := session.Get("K")
        if err != nil {
            fmt.Fprintln(writer, err.Error())
            return
        }
        fmt.Fprintln(writer, fmt.Sprintf("The stored value is : %s", v))
    })

    http.HandleFunc("/migrate", func(writer http.ResponseWriter, request *http.Request) {
        session := sessionx.Handler(writer, request)
        err := session.MigrateSession()
        if err != nil {
            log.Println(err)
        }
        fmt.Fprintln(writer, session)
    })
    _ = http.ListenAndServe(":8080", nil)
}
```
![健康检测](https://tva1.sinaimg.cn/large/008eGmZEgy1goyv71f8qdj30a106sdg9.jpg)


![多项集成化测试](https://tva1.sinaimg.cn/large/008eGmZEgy1goyv89devcj312a0f6jt5.jpg)


## 小 结

推荐还是使用`JWT`这种方式做鉴权，不过也有一体化`web应用`的`session`也不会被这么早淘汰，如果上面有问题，欢迎大佬们`pr`，还有部分代码没有列出，可以去仓库看看。

## 相关链接
代码仓库：[https://github.com/auula/sessionx](https://github.com/auula/sessionx)

