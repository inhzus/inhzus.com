---
title: "Berater 文档"
date: 2019-09-29T10:42:47+08:00
---

这篇博客讲讲服务号的设计时的考量、组织结构及如何上手等。

项目地址：[https://github.com/berater](https://github.com/berater)，如果遇到难以处理的无法 Google 到的问题，请联系 inhzus@gmail.com 孙同学。

注：本文为方便将所有参考文档都嵌入文中。

# 概览

## 名称

Berater 这个名称与项目内容没有什么关系，只是偶然看到的一个单词，就当来项目名了。

## 预期

设计上来说，项目只承担“从服务号到相关功能进行鉴权、提供用户信息”的功能，权限上分为学生、考生和各级管理员等，鉴权类型分为经过微信 openid 与帐号密码的直接登录两种。但在迭代过程中，承担了一部分 datacenter 与业务逻辑的任务，包括：存储鉴权过程中需要使用的信息，照片替换业务中用于鉴权的逻辑与社团报名的后端等等。按照规划，这些业务与数据将会逐步转移出该系统。

# 部署

本项目使用 docker-compose 进行部署，如未接触过 docker 且对英文较抵触，推荐看 [Docker 三剑客](https://yeasy.gitbooks.io/docker_practice/swarm/) 进行初步了解，理解基本概念，对相关使用会有所裨益。但需指出，这一文档只是对 docker 的初步了解，详细文档还是推荐阅读官方的非常详尽的 [Docker Documentation]([https://docs.docker.com](https://docs.docker.com/))。

## 配置

强调，在 docker-compose 配置文件中，需要使用 .env 环境变量文件，这一文件在服务器上有副本，请及时备份。在该文件中进行配置一些私密参数，如 API KEY 等，在 python 中通过 `os.getenv` 即可取值。项目在 [berater/config.py](https://github.com/inhzus/berater/blob/master/berater/config.py) 中导入。

**请注意：这一文件一定一定不能上传至 GitHub 等公开平台**，一旦发生误操作，请参考[删除 git 记录](http://www.hollischuang.com/archives/1708)**及时**进行覆盖，否则为了安全，请重置环境变量中的**全部**参数。

## 快速上手

在了解以下指令之前，强烈建议好好了解 docker, docker-compose, volume 等相关知识，否则可能带来灾难性的问题，如果遇到不确定的问题，请一定要邮件联系进行确定。一些具体的无关紧要的知识，可以边学习边使用。本项目常用的命令有：

```
docker-compose up -d --no-deps --build berater
```

`-d` 启动为 daemon；`--no-deps` 不重启关联容器，如不想重启关联的 redis；`--build` 强制重新构建容器。

总之就是在 berater 代码更新之后，需要重启容器即使用上述指令。

```
docker build -t auth ./
```

在之前，[吴](https://github.com/IgnoranceW)的 vue 项目容器托管在阿里云，但之后一直是将代码发给我，我手动生成容器，也就是上述代码，生成一个名为 `auth` 的镜像供我们部署使用。

```
docker-compose down
```

这一指令会将所有的容器停止服务，**请注意：一定一定不要使用 "-v" 参数**，目前我们的数据库数据使用 data-volume 同步在本地磁盘，一旦使用 "-v" 参数，会把所有的数据全部清空，且目前数据库没有任何备份。

```
docker-compose up -d
```

这一指令将所有的容器启动为 daemon。

## 服务介绍

以下介绍这些服务存在的缘由、目的和注意事项。在进行使用前，希望对 nginx、mysql 和 redis 的使用方法和相关原理都有一定了解。需要指出，为了数据的安全性，我们的所有存储服务除非特殊情况（如需指导老师添加大量数据等），绝不对外开放任何端口。

### nginx

为了能够更加便携地管理端口，将 nginx 安装在本地，通过 `proxy-pass` 来反代本地服务。

需要注意需要开启 `Access-Control-Allow` 来确保 DELETE PATCH 等 method 可以正常使用。

同时，为了保证 SSL，请与指导老师确认 SSL 证书的过期时间并及时更新证书，以免影响服务正常运行。

### berater

直接使用当前文件夹下的 Dockerfile 构建，具体使用 supervisord 守护进程，gunicorn 启动多进程服务。需要注意，如果不等待 mysql 初始化，由于 flask-sqlalchemy 的连接池不能自动重新连接，因此需要使用 wait-for-it.sh 脚本等待 mysql 初始化后才开始启动 supervisord 。

### mysql

为了方便备份，且防止因重新启动导致数据丢失，将 mysql 数据备份在本地 volume 中。这方面请多多 Google docker volume 相关知识。

同上需要强调一点，涉及到重启 mysql 等相关危险操作，如果不确定是否会带来致命问题，且不知道如何备份，请及时邮件联系。

### redis

如果只是一个进程，那么所有的本地缓存都可以通过自己手写一个键值对容器进行管理。但由于使用 gunicorn，因此使用 redis 管理一些缓存。如验证码等信息，在 berater 项目中有进行进一步封装。

### wx-auth

正如快速上手所述，这一容器中包括的是这一项目的前端部分。这些前端界面也就是鉴权过程中外界可见过程，包括的界面有：学生注册、考生注册和问答机器人。

整体逻辑是：这些界面调用我们后端项目的一些接口进行鉴权，若无权限，则需要进行身份注册，否则跳转至参数中指定的地址。同时，这些代码中还内嵌了问答机器人这一应用。

## 本地调试

有两种方法。

### docker-compose

直接在本地使用 docker-compose.yaml 构建即可，缺点是调试情况下每次更新的时间较长。

### docker-compose.dev

当然，在项目的早期版本中，曾将原 supervisord 守护进程替换为裸 fask debug server，方便进行调试。后因为认为笨重，废弃。

### Flask development server

了解 docker network 概念后，不难理解，需要配置 /etc/hosts/ 将 mysql，redis 指向 localhost。

```
127.0.0.1	localhost
127.0.0.1	mysql
127.0.0.1	redis
```

之后本地配置好 mysql 与 redis （如需要调试的业务不需要，也可省略此步），再根据 .env 环境变量文件修改，生成新的 .env.dev 文件，并建议使用 virtualenv 创建新的 python3.6 环境，按照依赖安装对应 packages。执行：

```
export $(cat .env.dev | xargs) && python run.py
```

即进行调试。

# 准备工作

## 服务号 SDK

完整的文档请参考微信服务号的[官方文档](https://developers.weixin.qq.com/doc/offiaccount/Getting_Started/Overview.html)，在项目中，我们将需要的函数封装为一个简单的 [package](https://github.com/inhzus/berater/tree/master/berater/utils/wechat_sdk)，这个 package 的代码量非常少，封装性也比较差，请在了解以下基本概念之后将代码通读一遍，避免黑盒带来的潜在问题。

### 接收/回复消息

后端需要构建的最基本的关于服务号的接口就是聊天。当用户发送一条消息，会相应的请求在后台设置的地址。需要进行的校验请查看[接入指南](https://developers.weixin.qq.com/doc/offiaccount/Basic_Information/Access_Overview.html)，相关材料请参考[接收消息](https://developers.weixin.qq.com/doc/offiaccount/Message_Management/Receiving_standard_messages.html)、[回复消息](https://developers.weixin.qq.com/doc/offiaccount/Message_Management/Passive_user_reply_message.html)。在 [chat/views.py](https://github.com/inhzus/berater/blob/master/berater/chat/views.py) 都已经进行了实现。

### ID

首先我们需要了解一下微信公众号或服务号的基本逻辑。如果曾经有了解过的话，那么会了解到它会赋予每个用户 openid。需要注意：每次重新关注 openid 都会改变。在我们的数据库中，可以看到与微信耦合程度较高的表会以 openid 作为 primary key。

### API 鉴权

首先服务号后台会提供开发者 API KEY 与 API SECRET，通过 `AccessToken` 类可以获取 `access_token`。在请求相关接口，比如获取 openid，需要使用相关 API 时都需要填充这一字段。

### 获取 openid

分为两种情况。一种是之前所说的接收消息，根据接收消息的形式，`FromUserName` 就是用户的 openid。

另一种情况需要具体说下。这种情况需要从头说起：对于服务号的应用，其实就是菜单上对应的链接，关于这点我们需要了解一下[微信网页授权](https://developers.weixin.qq.com/doc/offiaccount/OA_Web_Apps/Wechat_webpage_authorization.html)，进行网页授权的网页地址格式请参考 [config.menu](https://github.com/inhzus/berater/blob/351a6185db18178cad52761d903ba8afd825e2bd/berater/config.py#L73) 中的配置。简单描述来说，按照官方给定的格式填写可以得到授权信息的网址，当通过授权后，会跳转至用户指定的地址，带参数 code，code 是一次性的获取 openid 的凭证。

### 请求格式

我们将所有的接口地址都放在 Url 这一类中方便调用，在调用接口时，请直接使用这里的地址，并使用 `str.format` 进行格式化。

## 鉴权逻辑

### 准备

在上手鉴权逻辑之前，希望读者了解下”从用户输入网址到显示出界面分别经历了哪些步骤“、”HTTP 协议“，更详细的，包括 cookie 和 session 的区别、JWT、无状态鉴权等等相关知识都建议有一定理解，如果有接触过 Python requests，可以分别从 method, header, data 等参数了解。

同时，希望能够在阅读以下内容前粗浅的阅读一下[接入应用鉴权文档](https://github.com/inhzus/berater/issues/2)，从外界了解后端的作用。

### 考虑因素

- 微信官方一直推荐尽量不要暴露 openid；
- 需要兼容用户名 / 密码登录的模式；
- 避免 session，使用无状态鉴权。

首先考虑使用 JWT，但考虑 JWT 的用户信息为明文传送，会暴露 openid，因此打算使用类似方式，但 token 为使用加密算法生成的字串。

### 实现

用户首先通过微信提供的 code 请求接口获取后端生成的 token，这一 token 是将 openid 和用户对应权限进行加密生成的字符串，之后每次请求 API 都需要携带该 token，后端将这一 token 进行解密，直接取其中的权限进行鉴权，再取 openid 识别用户。

# 组件

在开始了解这一节之前，首先希望已经有 python flask 或 django 的使用经验，否则仅凭描写可能无法理解。

以下内容开始，和代码的关系较为密切，请先大致浏览代码的整体结构，以以下部分的内容作为参考进行理解。

这一节主要介绍 [menu.py](https://github.com/inhzus/berater/blob/master/menu.py), [berater/config.py](https://github.com/inhzus/berater/blob/master/berater/config.py), [berater/utils](https://github.com/inhzus/berater/tree/master/berater/utils), [berater/misc](https://github.com/inhzus/berater/tree/master/berater/misc) 的相关内容，以下二级标题均与文件名一一对应。utils 主要是与项目依赖度较小的模块，而 misc 则依赖度较强。

## Config

项目的配置形式基本与 flask 项目的通常方式一致，通过 `BaseConfig`  ，在开发环境下的 `DevelopmentConfig` 和 生产环境下的 `ProductionConfig` 进行配置。但是，由于 [Getting config variables outside of the application context](https://stackoverflow.com/questions/16071613/getting-config-variables-outside-of-the-application-context) 的问题，当时比较简单的创建了 `config[0]` 这一变量用于解决。其实是一种不合适的实现，但由于除了不优雅没什么影响，所以一直没有修改。除了 flask 配置外，这一文件中还有 `MENU` 变量为服务号菜单配置，下面说明。

## Menu

请参考[创建菜单](https://developers.weixin.qq.com/doc/offiaccount/Custom_Menus/Creating_Custom-Defined_Menu.html)了解相关逻辑。

## Models

包括 redis 与 mysql 客户端的初始化与表结构实体。

其中 `CandidateTable` 与 `StudentTable` 为最基本的考生与学生的用户信息表；`SourceStudentTable` 在用户绑定学生身份时用于验证（逻辑为核对身份证与学号或录取号）；`PrivilegeTable` 作为上述两张表权限的延伸，如各类管理权限等；`AuthUserTable` 为用户名密码保存表，用户名与 openid 并列；`NovaRegTable` 为临时的加入 Nova 的报名信息表。

同时，简单的封装了 `Transaction`，规定：所有的数据库操作都使用这一类。

## Crypto

为与项目耦合度较低的进行鉴权加密解密的插件。

这一模块的代码建议参考 flask 的大多数插件的风格进行理解，就会明白 `LocalProxy` 等概念。

`gen_token` 顾名思义根据 openid 和权限生成 token；`token_required` 装饰器对 `route`权限进行限制；`current_identity` 指向当前用户 token 中的数据，查看代码中的使用范例即可理解。

了解 models 中的表后，在 `gen_token` 生成 token 过程中，会分别在 `CandidateTable`, `StudentTable`, `PrivilegeTable` 中查找对应的权限。

需要指出，在 `_token_required` 除了之前提到的根据 token 中的权限进行鉴权外， 还有一种之前未提到的对形如 "Bearer face face_token" 的 Authorization Header 进行鉴权的方式，实际可以在 config 中看到，目前支持照片替换应用。具体逻辑是，随便生成一段 token 写入配置，并将该 token 告知需要使用鉴权的 app，建议，之后全部转型为使用用户名密码登录的形式，这样的兼容性能够更广。这种 app token 方式仅是临时使用。

## Cache

由于 redis 不支持多表，因此对 redis 操作 key 进行了简单的封装。

## SMS

为阿里云短信服务的 SDK，只需要了解如何使用即可。

## TF_IDF

为使用 TF-IDF 算法实现问答机器人的相关代码，除了了解如何使用外，需要明确，这一模块依赖于 data 文件夹，这一文件在服务器上同样也有副本，请备份，具体如何使用 data 文件夹，请参考 Dockerfile。

## Bert

为使用 Bert 算法实现问答机器人，这部分原理首先需要了解 [frp](https://github.com/fatedier/frp) 这一内网穿透工具的基本使用。以 weixinbak.njunova.com 作为 frp 跳板，使用 bert client 访问在校园网内的 bert server。具体数据格式需要参照实际情况。

这一模块暂时并没有使用，在之后 bert 成型后会进行使用。

## Response

之后的部分，首先请了解 RESTful 这一概念，入门可参考[RESTful API 设计指南](http://www.ruanyifeng.com/blog/2014/05/restful_api.html)。

为实现 RESTful API，所有的返回值统一格式为 `Response` 这一实体的 jsonify 形式。与之相关的，同时参考 berater/exceptions 的实现。

# 业务逻辑

如果之前没有接触过 flask，一定要先看一些示例项目来了解 f 项目的常见结构。

了解组件后，在明白 flask blueprint 蓝图与 app 的关系之后，之后的介绍除 [berater/exception](https://github.com/inhzus/berater/tree/master/berater/exception) 外，其他均为业务逻辑代码。

大部分 API 见，理解 API 请求方式基本就会明白功能：[postman API](https://web.postman.co/collections/5030181-e9c28dfa-0f1a-4e4a-9224-5c5a1e6d29fa?version=latest&workspace=457a1b93-8475-4d70-8a1f-c62fbd2acff6)。

nova, face 参考上边链接即可知道具体实现。

## Exception

这一 blueprint 负责捕获错误与记录请求日志。这一方面需要与上一节的 `Response` 共同理解。

对于后端代码中发现的异常，都使用 flask 底层中 werkzeug 中的通用 `HTTPException` 抛出（即 `raise`)，此时该模块的 `handle_http_exceptions` 会捕获，并将错误状态码和错误信息填充至 `Response` 返回（具体规则详见 RESTful，如 401 Unthorized 等）。

如果该错误没有被捕捉，如发生除法除 0，index out of range 等错误而后端代码没有进行检查，也即 Internal Server Error，会将错误信息与状态码 500 填充至 `Response`，同时输出堆栈信息。

当然为了方便调试，目前会在每次请求结束之前，输出 request & response。

可能会有疑问：为什么用 HTTP 200 状态码来返回错误响应？这个问题争议很多，去 Google 可以搜到很多相关讨论。

## API

路径 /api，其他部分抛出异常、使用数据库的代码规范都参考这一部分。

### Token

`get_token`, `refresh_token`, `check_token` 为与 token 相关的接口。其中，获取 token 主要有两种方式，一种是通过用户名密码，另一种是通过微信提供的 code 换取 openid。

`test_token` 为用于得到测试 openid 的接口，由于用户名和 openid 不会对外泄漏，因此可以保证安全性。

### SMS

`send_code`，`check_code` 为与短信验证码相关的接口。

### 信息绑定

`candidate_signup`, `candidate_update`, `student_signup`, `student_update` 为考生与学生信息进行绑定与更新的接口。其中，更新信息的接口目前没有应用使用。

### 问答

`qna`, `bert` 为问答系统相关接口。

## 扩展

目前项目基本完善，如果需要修改代码， 请参照 API 蓝图的代码风格进行修改。如果需要增加其他的功能，如果是关于鉴权相关的一些逻辑，直接添加在 API 蓝图中；而如果是一些独立的模块，则另外创建蓝图模块，并将其添加至 app.py。