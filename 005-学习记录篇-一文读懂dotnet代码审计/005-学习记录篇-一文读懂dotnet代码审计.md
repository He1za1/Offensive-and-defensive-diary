# 005-学习记录篇-一文读懂dotnet代码审计

[TOC]

## 开发模式

目前，ASP.NET中两种主流的开发方式是：ASP.NET Webform和ASP.NET MVC。

![](img/66748d6257aecd63bfc7cb43bcfbaa9.jpg)

### WebForm开发模式

WebForm模型：

![](img/3912765385a623c242d8431b30fa27d.jpg)

aspx负责显示，服务器端的动作就是在aspx.cs定义的，.cs是类文件，公共类神马的就是这个了，.ashx是一般处理程序，主要用于写web handler,可以理解成不会显示的aspx页面，不过效率更高，dll就是编译之后的cs文件。

.aspx文件作为显示页面，通常代码是一些html代码，第一行文件头会显示具体触发功能代码的位置，就可以通过这个位置进行漏洞跟踪"。

![](img/bf1a7d09ca311262c46475cfb43d5ea.jpg)

我们先打开一个aspx页面，其中：

1、Labguage 表示当前所使用的语言，为C#。

2、AutoEventWireup 指的是是否页面自动事件回传，自动关联处理函数。

3、Inherits 继承于ChargeLogin, App_Web_ozdw0ygm。

### MVC开发模式

MVC模型：

![](img/80f0a2357ec77cac6b223946f6dab5d.jpg)

![](img/51a1af9dc732398a3642dba718446c5.jpg)

M：Model 

> 主要是存储或者是处理数据的组件，Model其实是实现业务逻辑层对实体类相应数据库操作，如：CRUD(C:Create/R:Read/U:Update/D:Delete)。它包括数据、验证规则、数据访问和业务逻辑等应用程序信息。

V：View 

> 是用户接口层组件，主要是将Model中的数据展示给用户，ASPX和ASCX文件被用来处理视图的职责。

C：Controller 

> 用来处理用户交互，从model中获取数据并将数据传给指定的view。

## 拓展名介绍

### 常见后缀

%windir%\Microsoft.NET\Framework\v2.0.50727\CONFIG\web.config中有详细定义，这里提取部分简单介绍。

![](img/af29f4106f92b4609fbc228adff72f3.jpg)

aspx

> 应用程序根目录或子目录
>
> asp.net的文件后缀名，是微软的在服务器端运行的动态网页文件，包含web控件与其他，他是在服务器端靠服务器编译执行的程序代码，可理解为页面显示文件。

cs

> App_Code 子目录
>
> 类文件，业务逻辑处理层的代码。

aspx.cs

> web窗体后台程序代码文件。

ascx

> 应用程序根目录或子目录
>
> Web用户控件文件，是作为一种封装了特定功能和行为（这两者要被用在Web应用程序的各种页面上）的Web页面被开发的。一个用户控件包含了HTML、代码和其他Web或者用户控件的组合，并在Web服务器上以自己的文件格式保存，其扩展名是*.ascx。

asmx

> 应用程序根目录或子目录
>
> 是webservice服务程序的后缀名，ASP.NET 使用.asmx 文件来对Web Services的支持。

asax

> 应用程序根目录
>
> 通常是Global.asax。

**Global.asax**

> Global.asax是 ASP.NET 应用程序的中心点，提供全局可用的代码，从HttpApplication基类派生的类，响应的是应用程序级别会话级别事件，通常ASP.NET的全局过滤代码就是在这里面。

config

> 应用程序根目录或子目录
>
> 通常是web.config，Web.config 文件向它们所在的目录和所有子目录提供配置信息。

**web.config**

> web.config是基于XML的文件，可以保存到Web应用程序中的任何目录中，用来储存数据库连接字符、身份安全验证等。
>
> 加载方式：
> 当前目录搜索 -> 上一级到根目录 -> %windir%/Microsoft.NET/Framework/v2.0.50727/CONFIG/web.config -> %windir%/Microsoft.NET/Framework/v2.0.50727/CONFIG/machine.config -> 都不存在返回null。

ashx

>应用程序根目录或子目录
>
>一般处理程序，该文件包含实现 IHttpHandler 接口以处理所有传入请求的代码。

soap

>应用程序根目录或子目录
>
>soap拓展文件，基于XML简单协议。

### **ASPX和ASP的区别**

asp

> Active Server Pages，是MicroSoft公司开发的服务器端脚本环境，可用来创建动态交互式网页并建立强大的web应用程序，即为动态服务器页面。

.asp是asp的文件后缀名

.aspx是asp.net的文件后缀名

asp.net又叫 asp+ 是动态网络编程的一种设计语言

**ASPX的webshell权限为什么比asp的大？**
大概分两个阶段去回答这个问题

阶段一：在IIS6.0 - IIS7.5

> .NET默认的账户是aspnet，隶属于Users，而此时的ASP基于IUSER账户，隶属于Guest组，对于组来说Users组的权限 > Guest组，所以能执行或者读取的资源更多。

阶段二：IIS7.5以后到现在IIS10

> .NET默认的账户是iis apppool\defaultapppool，默认身份为 ApplicationPoolIdentity，而此账户继承了Users组和IIS_IUSRS组的成员，而ASP同样也是继承于apppool\defaultapppool，返回用户组也和.NET一致。

结论：在老版本的IIS里的确ASPX的webshell权限 > ASP Webshell；新版本的IIS里只要开启了ASP的话，两者的权限一样的，但新版的IIS默认已经关闭了ASP。

## 审计工具

### 反编译

如果遇到的asp.net cms源码包是没有编译成dll的话，那么就方便很多了，可以直接拖入Visual Studioont查看，而更多的是会遇到编译成了dll，这样相对地说就麻烦很多了，不过也有方法。

一、Reflector

可以将dotnet程序集中的中间语言反编译成C#或者Visual Basic代码，除了能将IL转换为C#或Visual Basic以外，Reflector还能够提供程序集中类及其成员的概要信息、提供查看程序集中IL的能力以及提供对第三方插件的支持，比如reflexil。

二、ILSpy、DnSpy

自reflector就开始转收费软件，ILSpy也就应运而生了，它反编译出的代码和reflector差不多。

[ILSpy](https://github.com/icsharpcode/ILSpy)

[DnSpy](https://github.com/dnSpy/dnSpy)

找到后端代码，直接将对应的DLL文件拖进来即可。

![](IMG/df06e6004a42b89c25e79e4acbcd3d7.jpg)

**预编译的原理：**

​	简单的来说，就是在网站发布时将aspx文件进行编译，转换成dll文件。因为.NET程序在运行时会优先加载bin目录下的程序集。

​	即index.aspx -> /bin/index.dll

​	在用户访问index.aspx时，则直接由index.dll进行处理。而不是index.aspx。一旦进行预编译后，相关程序会被转换为dll存储在bin目录下。这时候，程序的访问路径和相关逻辑，都被封装成了dll。无论根目录下是否存在index.aspx，都可以正常处理特定路由下的功能。

​	可参考畅捷通T+任意文件上传(CNVD-2022-60632 )漏洞来理解。

### 小玩意

[ListFile](https://github.com/He1za1/ListFile)

填写文件后缀和文件路径，筛选出项目里面对应后缀的文件路径，并把结果保存到工具根目录下的result.txt中。

![](img/9d0b07fb7d23c460764ae7167395f35.jpg)

用命令也可以实现，dir /s /b *.aspx。

![](img/57f5f8ba20d50f0bd450c8dd764edb0.jpg)

[dirscan](https://github.com/safe6Sec/dirScan)

图形化的目录扫描工具之一，可结合ListFile的筛选结果进行目录爆破，从而快速的找出可未授权访问的页面。

![](img/39d41282d4fff232fb3a5fadd942e27.jpg)

![](img/1bc008c1718cd55969de8a8a1923c2b.jpg)

[sublime](http://www.sublimetext.com/)

[sublimetext](http://www.sublimetext.com/)

如果不想下visual studio，可以用这个，一款轻量级的代码编辑器。将反编译后的源码直接拖入进去即可。

### 反混淆

[de4dot](https://github.com/de4dot/de4dot)

反混淆的工具有很多，其中de4dot是目前最主流的反混淆工具。

检测混淆器类型

> de4dot.exe -d f:\bin\h1z1.dll 

批量反混淆

> de4dot.exe -r c:\input -ru -ro c:\output

## 漏洞案例

### 任意文件上传

WebForm，通过class定位到后端业务处理出进行审计。

![](img/d728dc13214e1b84287c66fcd819201.jpg)

无后缀校验，经典的路径+时间+文件名拼接。

![](img/df06e6004a42b89c25e79e4acbcd3d7.jpg)

漏洞验证。

![](img/5919c3f37ed458d5d0b317146c6339d.jpg)

代码审计关注点：

```c#
saveas()
```

### 任意文件下载

MVC，发现存在下载函数。

![](img/b86b032a9a975e1ab5baff813bb01de.jpg)

跟进，文件路径拼接处未做过滤，可造成任意下载。

![](img/c193b22f7574c607c14be8a0fe612ba.jpg)

漏洞验证。

![](img/1952a8e9c327cee60b1866551760acb.jpg)

![](img/827921de6f2af74e6397ff052b1b44c.jpg)

代码审计关注点：

```c#
1. File对象的 OpenText和OpenRead方法
2. FileStream对象的FileMode.Open和FileMode.Read
3. Response.WriteFile 常用于文件下载
```

### SQL注入漏洞

MVC，发现存在数据交互。

![](img/ce41f1380e61b8951ef3700c1a979ba.jpg)

跟进，存在SQL语句拼接，且不存在预编译SQL parameter和其他过滤，可造成SQL注入漏洞。

![](img/3b5086a79fba90c2326af550f057c36.jpg)

漏洞验证。

![](img/f86bbd25ea44a49c4dd519c58f5f200.jpg)

代码审计关注点：

SQL语句拼接处。

### XSS

在asp.net中我们插入XSS代码经常会遇到一个错误`A potentially dangerous Request.Form`这是因为在`aspx`文件头一般会定义一句`<%@ Page validateRequest="true" %>` 。

```c#
<%@ Page Language="C#" ValidateRequest="true" AutoEventWireup="true" CodeBehind="WebForm1.aspx.cs" Inherits="WebApplication13.WebForm1" %>
```

当然也可以在`web.config`中定义，值得注意的是`validateRequest`的值默认为`true` ,所以通常情况下asp.net很少存在`XSS`的,除非程序员把他的值改变。

```c#
<system.web>
    <pages validateRequest="true"></pages>
    <httpRuntime requestValidationMode="2.0"/>
</stytem.web>
```

### 未授权访问

AuthorizeAttribute

一、AuthorizeAttribute是asp.net MVC的几大过滤器之一，俗称认证和授权过滤器，也就是判断登录与否，授权与否。当为某一个Controller或Action附加该特性时，没有登录或授权的账户是不能访问这些Controller或Action的。在 Controller上添加了自定义的[MyAuthorize()]特性，控制器下方没有注明[AllowAnonymous]特性的Action均无法直接访问，必须要登录，例如访问/Home/About页面跳转至登录口需要核实身份信息。

二、MVC提供了AllowAnonymousAttribute表示在授权期间跳过AuthorizeAttribute的控制器和操作，具体审计时关注Controller或者Action上方是否有[AllowAnonymous]，有的话表示当前控制器或方法可以被匿名用户访问，也就是我们常见的未授权访问漏洞。

![](img/c6f240e07e62e43ece57b8dac519633.jpg)

### 命令执行

部分代码，使用了ProcessStartInfo对象操作phantomjs进行网页快照，截图功能，param作为phantomjs的参数一部分，可以通过传递恶意的参数给param，达到命令执行的目的。

![](img/f1288b4f4baf55524e255341d2e0cc2.jpg)

漏洞验证。

![](img/91559bdda9b60f7a0d11f541aef0ec9.jpg)

### 反序列化

推荐看Ivan1ee师傅的文章，或者关注他的[github](https://github.com/Ivan1ee) 、公众号【dotNet安全矩阵】。

![](img/3d401a548dd1ebbac05dc809ab3c3ff.jpg)

简单介绍下几个常见的反序列漏洞：

**SoapFormatter反序列化漏洞**

• 背景介绍：

​	• SoapFormatter用于将对象持久化存储为SOAP数据，SOAP是基于XML的简易协议，让应用在HTTP上进行信息交换。

• 漏洞触发：

​	• Deserialize

• 利 用 链： 

```c#
1、SurrogateSelector -> ISurrogateSelector -> GetObjectData
2、byte[] -> Assembly.Load -> Assembly -> Assembly.GetType -> Type[] -> Activator.CreateInstance
```

**BinaryFormatter反序列化漏洞**

• 背景介绍：

​	• BinaryFormatter 二进制方式把对象进行序列化，优点是速度较快。

• 漏洞触发：

​	• UnsafeDeserialize

​	• UnsafeDeserializeMethodResponse

​	• Deserialize

​	• DeserializeMethodResponse

• 利 用 链： 

```c#
SurrogateSelector -> ISurrogateSelector -> GetObjectData
```

**XmlSerializer序列化**

• 背景介绍：

​	• `System.Xml.Serialization.XmlSerializer`类他可以将对象序列化到XML文档中和从XML文档中反序列化对象，在这个过程中构造`XmlSerializer`对象期间需要指定它将处理的类型`XmlSerializer(Type)` 传入的Type也就是我们的重点关注对象，因为Type类，是用来包含类型的特性。类型信息包含数据，属性和方法等信息。如果我们传入一个特定的payload那么我们就可以调用他的方法了。

• 历史漏洞： 

​	• CVE-2019-0604 Microsoft SharePoint RCE 漏洞

​	• DotNetNuke RCE 漏洞

• 漏洞触发：

​	• Type类静态方法GetType解析全限定名：**Type.GetType(typeName)** 

• 利 用 链： 

```c#
ResourceDictionary -> XamlReader -> ObjectDataProvider
```

**JavaScriptSerializer反序列化漏洞**

• 背景介绍：

​	• 常见于处理Ajax数据的序列化，实现.Net中所有类型和Json数据之间的转换，提供的SimpleTypeResolver类型解析器，可序列化自定义的全限定名。

• 漏洞触发： 

​	• DeserializeObject

​	• Deserialize

• 利 用 链： 

```c#
ObjectDataProvider
```

**FastJson反序列化漏洞**

• 背景介绍：

​	• FastJson是一个开源库，下载地址 http://www.codeproject.com/Articles/159450/fastJSON，和Json.Net相比速度和性能优势非常明显，究其原因是利用反射生成了大量的IL代码，而IL代码是托管代码，可以直接给运行库编译所以性能就此大大提升。

• 漏洞触发： 

​	• ToObject

• 利 用 链： 

```c#
ObjectDataProvider
```

**Json.Net反序列化漏洞**

• 历史漏洞：

​	• Breeze RCE 漏洞

• 漏洞触发： 

​	• 序列化数据结构： **TypeNameHandling.Objects**、Arrays、All、Auto

​	• DeserializeObject

• 利 用 链： 

```c#
ObjectDataProvider
```

• 安全配置：TypeNameHandling.None