---
title: '腾讯云 Web前端实习 直推'
layout: post
tags:
    - interview 
---
直推，第二天联系电话面，虽然还是我第一次面试，但是准备时间不足+临时起意转投Web，所以结果自然是凉了。时长30分钟，难度大概在2-3面的程度，问题各方面都会问到并且会问的比较深入，在各个问题之后也有一些紧跟着的追问。  
让我意识到我的基础和经验真的太不足了，各个方面几乎被问懵了，而且Web也不是非常对口。下面把我觉得比较有意义的问题总结下，一些Web相关的就找资料贴了。

<!--more-->

**目录**

* TOC
{:toc}

## 问题
### JS 作用域
简单谈一下JS的作用域，是比较简单的问题，但是我答成了JS运行时的上下文，因为不是很熟悉，且本来就是比较容易混淆的概念。  
详细解释见blog
[JS 作用域/上下文/闭包](https://www.cnblogs.com/sxhlf/p/6726976.html)  
与之相关的还有变量提升
[JS变量提升](https://blog.csdn.net/qq_39712029/article/details/80951958)  
比如这个例子：

```js
console.log(v1);
var v1 = 100;
function foo() {
    console.log(v1);
    var v1 = 200;
    console.log(v1);
}
foo();
console.log(v1)
```

提升之后的结果：

```js
var v1;
console.log(v1);
v1 = 100;
function foo() {
    var v1
    console.log(v1);
    v1 = 200;
    console.log(v1);
}
foo();
console.log(v1)

//undefined
//undefined
//200
//100
```

### JS prototype原型
一下子愣是没想起来...说了一点点和Class相关的，后来才想起来是prototype。不过概念也很复杂，主要是`__proto__`, `constructor`, `prototype`，见
[prototype、__proto__与constructor](https://blog.csdn.net/cc18868876837/article/details/81211729)  
要点如下：
- `__proto__`和`constructor`属性是对象所独有的；`prototype`属性是函数所独有的，因为函数也是一种对象，所以函数也拥有`__proto__`和`constructor`属性。
- `__proto__`属性的作用就是当访问一个对象的属性时，如果该对象内部不存在这个属性，那么就会去它的`__proto__`属性所指向的那个对象（即父对象）里找，一直找，直到`__proto__`属性的终点`null`，然后返回`undefined`，形成一个原型链
- `constructor`指向该对象的构造函数，所有函数（此时看作对象）最终的构造函数都指向`Function`
- `prototype`属性是将函数和对象相连接的纽带`foo()->foo.prototype`，作用是让该函数所实例化的对象们都可以找到公用的属性和方法

其中关系为：
- child的`__proto__`指向father的`prototype`，即`child.__proto__ === father.prototype`
  - `__proto__`链顶端指向`Object.prototype`，`Object.prototype.__proto__ === null`
- 每个对象的`constructor`，或者是自身的构造函数，如`foo.prototype.constructor === foo`，或者是继承父对象的构造函数而来（通过`__proto__`查找得到），即`child.prototype.constructor === father`
  - `constructor`链顶端是`Function()`

顺便了解一下  
[JS 实例方法 类方法 原型方法](https://blog.csdn.net/weixin_42006492/article/details/83753847)  
[JS 继承](https://www.cnblogs.com/humin/p/4556820.html)
- 原型链继承
- 构造继承
- 实例继承
- 拷贝继承
- 组合继承
- 寄生组合继承

### JS Event事件传递
[Event冒泡与捕获](https://blog.csdn.net/lhjuejiang/article/details/79455801)

### React Reconciliation/Diff
主要考察React框架原理，diff算法在面试前看的还是比较细致，可能算是唯一回答得比较全的问题了吧。  
Reconciliation的具体细节就不阐述了，见
[官方文档 Reconciliation](https://react.docschina.org/docs/reconciliation.html)
以及
[react diff详解](https://segmentfault.com/a/1190000010686582)  

- **追问1：如何判断DOM中的Component前后是否发生了变化？何种情况下会触发重渲染？** 
    - 每当根元素有不同类型，React将拆除旧树并且从零开始重新构建新树
    - 当比较两个相同类型的React DOM元素时，React则会观察二者的属性(attributes)，保持相同的底层DOM节点，并仅更新变化的属性
    - 件更新时，实例保持相同，React通过更新底层组件实例的属性(props)来匹配新元素，并在底层实例上调用`componentWillReceiveProps()` 和 `componentWillUpdate()`
    - React支持一个key属性。当子节点有key时，React使用key来匹配原始树的子节点和随后树的子节点
    - 在某些情况下组件并不需要更新，可以通过重写`shouldComponentUpdate()` 返回false来跳过整个渲染进程
- **追问：如下图所示两颗DOM是否会发生重渲染？**  

```html
<ul>
  <li>A</li>
  <li>B</li>
  <li>C</li>
</ul>

<ul>
  <li>A</li>
  <li>C</li>
  <li>B</li>
</ul>
```
答案是会的，默认时，当递归DOM节点的子节点时，React就是迭代在同一时间点的两个子节点列表，并在不同时产生一个变更

```html
<!-- 只会添加C节点，AB节点没有影响 -->
<ul>
  <li>A</li>
  <li>B</li>
</ul>

<ul>
  <li>A</li>
  <li>B</li>
  <li>C</li>
</ul>

<!-- 会将每个子节点进行重渲染 -->
<ul>
  <li>A</li>
  <li>B</li>
</ul>

<ul>
  <li>C</li>
  <li>A</li>
  <li>B</li>
</ul>
```
使用key可以解决这个问题

### React 开发中有没有遇见过，子控件需要渲染在父控件之外的情况？（例如子控件弹窗） 
这题应该是考的Portals，当时完全不知道  
[官方文档 Portals](https://react.docschina.org/docs/portals.html)  
可见好好看官方文档真的很重要，出了好几个问题

### React 聊聊React Native的原理
因为我项目写了RN所以问了，但是我因为之前的误解其实答错了！不知道有没有被发现，但是真的太丢脸了...  
[React Native原理](https://www.jianshu.com/p/038975d7f22d)
1. JS通过C++层执行并与Java层通信
2. Java层进行UI渲染和功能调用，RN的react一样使用virtual DOM控制UI渲染，原理基本相同，只不过Web上的元素替换成了Android Widget *（所以RN的本质还是Android而不是WebView！）*
3. Java与Js之间的调用，两边存在同一份模块配置表，最终均是将调用转化为`{moduleID, methodID, callbackID, args}`，处理端在模块配置表里查找注册的模块与方法并调用

### Net HTTPS
[HTTPS的加密方式](https://www.2cto.com/net/201709/681347.html)
- **HTTPS证书的本质**  
  SSL 证书是数字证书的一种，是遵守 SSL协议，由受信任的数字证书颁发机构CA，在验证服务器身份后颁发，具有服务器身份验证和数据传输加密功能。
- **加密的方式**
  - 对称加密算法：同一个密钥可以同时用作信息的加密和解密
  - 非对称加密算法：包括公钥和私钥，公钥公开用于加密，私钥保密用于解密
  - HASH算法(摘要算法)：无法解密，同样的输入必定得到相同的输出，只会用于比较两个消息的密文是否一致，以判断这两个原本的数据是否一致
- **服务端是如何解决HTTPS加密带来的性能负荷的？**  
  对称加密算法的复杂度低，所以在传输数据的时候，使用的是对称加密算法  
  但是对称加密算法使用的是相同的秘钥，为了保证秘钥的安全性，单独对秘钥的传输采用非对称加密  
  同时，数据的传输过程中时候用HASH算法用于验证数据的完整性
  非对称加密算法用于在握手过程中加密生成的密码
  对称加密算法用于对真正传输的数据进行加密
  而HASH算法用于验证数据的完整性   
1. 浏览器将自己支持的一套加密规则发送给网站，如RSA加密算法，DES对称加密算法，SHA1摘要算法
2. 网站从中选出一组加密算法与HASH算法，并将自己的身份信息以证书的形式发回给浏览器。证书里面包含了网站地址，加密公钥，以及证书的颁发机构等信息
3. 获得网站证书之后浏览器要：
    - 验证证书的合法性，如果证书受信任，则浏览器栏里面会显示一个小锁头，否则会给出证书不受信的提示
    - 如果证书受信任，或者是用户接受了不受信的证书，浏览器会生成一串随机的密码，用于之后数据通信的对称加密算法，并用证书中提供的公钥加密(非对称算法)
    - 使用约定好的HASH算法计算握手消息，并使用密码对消息进行加密，最后将公钥加密的密码，密码加密的HASH摘要值一起发送给服务器
4. 网站接收浏览器发来的数据之后：
    - 使用自己的私钥将信息解密并取出浏览器发送给服务器的密码(对称加密算法的秘钥)
    - 使用密码解密浏览器发来的握手消息，并验证HASH是否与浏览器发来的一致
    - 使用随机密码加密一段握手消息，发送给浏览器
5. 浏览器解密并计算握手消息的HASH，如果与服务端发来的HASH一致，此时握手过程结束，之后所有的通信数据将由之前浏览器生成的密码并利用对称加密算法进行加密。

### Git 如何merge时只merge特定的一个文件  
两种方法：
[git合并单个文件/文件夹](https://segmentfault.com/a/1190000008360855?utm_source=tag-newest) 
1. 强制覆盖合并
```shell
$ git branch
  * A  
    B
$ git checkout B [files]
``` 
2. merge合并
```shell
$ git branch
  * A  
    B
$ git checkout -b A_temp  
$ git merge B
$ git checkout A
$ git checkout A_temp [files]
```

### Dev 有没有用Node搭建过后台？有没有用过webpack？  
没有用过（简直丢人）  
然而事实证明没有用过你编一下大概也是可以的

### Dev 聊聊你的React Native项目  
主要聊聊自己的项目，因为之前有过准备所以主要按照准备的讲了一下流程以及主要解决的问题之类的。  
感觉项目一定要提前准备好，怎么讲，讲哪些，重点是什么，不要讲了一堆人家也没有懂你到底讲了什么
- **项目客户端和服务端的身份认证**  
  客户端和服务端使用token放到传输头的Authentication里实现，token使用RN提供的本地持久化AsyncStorage接口保存在客户端本地  
  现在想想他可能想问有关Cookie方面的问题，但是我这么一讲他就没话问了...

## 总结
作为第一次面试感觉收获还是非常丰富的吧，最深刻的体会：
- 一定要注重基础知识的复习，在这之上最好要能够加以拓展，深入了解原理、应用场景以及相关知识
- 遇到不会的点时不能表示不会，还是要尽可能地扯下去（也可能是因为太难了真的没法扯），可以谈谈相关的问题，要想办法充分地展示自己的能力
- 千万不要慌，其实我真的有点慌，导致很多问题一下子短路没答上来