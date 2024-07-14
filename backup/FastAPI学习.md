## FastAPI介绍
FastAPI两部分组成：

>starlette负责web部分(Asyncio)
>
>​	包含ASGI框架，能够实现构建异步服务器
>
>Pydantic负责数据部分(类型提示)如下面代码所示:
>
>```python
>from datetime import datetime
>from typing import Tuple
>
>from pydantic import BaseModel
>
>
>class Delivery(BaseModel):
>    timestamp: datetime
>    dimensions: Tuple[int, int]
>
>
>m = Delivery(timestamp='2020-01-02T03:04:05Z', dimensions=['10', '20'])
>print(repr(m.timestamp))
># > datetime.datetime(2020, 1, 2, 3, 4, 5, tzinfo=TzInfo(UTC))
>print(m.dimensions)
># > (10, 20)
>```
>
>可以更好的处理数据
## HTTP协议的结构详解

![image-20240322173443809](https://s2.loli.net/2024/03/22/fqPBxiajpsIAwTd.png)

并且使用的http协议发送Get和Post请求的时候，需要设置好数据格式

```python
# web应用程序  : 遵循http协议
import socket

sock = socket.socket()

sock.bind(("127.0.0.1", 1010))
sock.listen(5)

while 1:
    # 服务端要实现先收再发
    conn, addr = sock.accept()  # 阻塞等待客户端连接
    data = conn.recv(1024)
    print("客户端发送的请求信息\n", data)
    # 请求的时候不用管是因为浏览器已经帮我们已经封装好了

    # 如果是这样的话，不符合http协议，服务端没有按照http协议的格式来发送数据给客户端
    # conn.send(b"hello,world!")

    # 换成下面这样的带有响应头的数据，就符合http协议了
    conn.send(b"HTTP/1.1 200 OK\r\nhi\r\n\r\n<h1>hello,world!</h1>")
    conn.close()
   
```

一般会给直接设置好

##  API接口

分离代表的是 职责 分离

### 前后端分离

![image-20240322174043560](https://s2.loli.net/2024/03/22/8BiyszQ3eOktEph.png)

### 前后端不分离(主流)

![image-20240322174255437](https://s2.loli.net/2024/03/22/ghLRcox46yp9WO8.png)

## RESTful开发规范

RESTful是一种专门为了Web开发而定义的API接口的设计风格，尤其适用于前后端分离的模式中。

这种风格的理念认为后端开发任务是提供数据的，对外提供的是**数据资源**的访问接口。所以在定义接口的时候，客户端访问的URL路径就表示这种要操作的数据资源。

其目的主要是为了在请求的过程中对某个资源进行处理的时候，能够通过改变请求方式来实现在后端方面进行操作。

下图中定义了方式 👇

![image-20240322192335801](https://s2.loli.net/2024/03/22/cnFtQhCzliTXrjY.png)

RESTful规范是一种通用的规范，不限制语言和框架。

**get**一般是查看

**post**一般是提交

...

## 初步开始

```python
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
# 在def前面添加一个async就能实现异步
async def home():
    return {"user_id": 1001}

```

需要区分的是，我们是利用web框架快速搭建起来的一个web应用程序	

```python
from fastapi import FastAPI

app = FastAPI()

# 进行路由映射、构建装饰器
# 路径操作装饰器
@app.get("/")
# 在def前面添加一个async就能实现异步
# 路径操作的函数
async def home():
    return {"user_id": 10012}

@app.get("/shop")
async def shop():
    return {"shop_id": "10012"}
```

## 文档的使用以及各个参数设置

在使用中，根url后添加  **\docs** 来进入文档界面

例如下面的代码：

```python
from fastapi import FastAPI

app = FastAPI()

# 这里会有很多的参数，大部分都是应用在文档上的，/docs 可以实现在开发阶段测试的各个功能
@app.get("/",
         tags=["主页"],
         description="这是一个主页接口",
         summary="主页接口",
         deprecated=True)
def home():
    return {"user_id": 1001}

```

## 路由的设置

```python
users = APIRouter()
```

关键就在于对子路由进行一个实例化

实例化之后能够实现

```ptyhon
http://127.0.0.1:1010/shop/food
http://127.0.0.1:1010/shop/clothes
```

等各个路径的耦合

## 查询参数

**路径函数中声明不属于路径参数的其他函数参数时候，他们将被自动解释为”查询字符串“参数，就是url?之后要用&分割的key-value键值对**

例如下面代码中的

```python
@app02.get("/jobs")
async def job(jobs: str, a, ab):
    return {
        "msg": jobs
    }
```

在上面的代码中的jobs即为路径参数也是查询参数，a和ab就只是查询参数

![image-20240325162335406](https://s2.loli.net/2024/03/25/16X2YNtvEauURKg.png)

三个键值对是按照**&**作为分割符

当然是可以将键值对设置为不确认

例如下面的代码：

```python
@app02.get("/jobs/{a}")
async def job(jobs: str, a=None, ab=None):
    return {
        "jobs": jobs,
        "a": a,
        "ab": ab
    }
```

a因为是路径参数，所以即使设置为了**None**也是必须填写的

ab则为可填可不填

<img src="https://s2.loli.net/2024/03/25/psmqhJYufMyZ1g2.png" alt="image-20240325163420118" style="zoom: 50%;" />

同时也需要注意的地方在于url路径上的区别

```json
'http://127.0.0.1:1010/jobs/12?jobs=12'
```

## union类型

在python3.6之后，增添了一个新的类型为<u>*union*</u>

使用方法为`from typing import Union`

示例：

```python
from fastapi import APIRouter
from typing import Union

app02 = APIRouter()


@app02.get("/jobs/{a}")
async def job(jobs: Union[str, None], a=None, ab=None):
    return {
        "jobs": jobs,
        "a": a,
        "ab": ab
    }

```

上面中对于`Union`的使用为**jobs**既可以为**Str**，也可以为**None**

 两种类型的使用都是正确的

```python
Union[str, None] = None
```

在使用等于None 来让其默认值为 None

Optional是Union的一个简写，当数据结构中有可能为None和str的时候，可以用Optional来代替Union

```python
Optional[str] 相当于 Union(str,None)
```

## 请求体数据

FastAPI基于Pydantic，其主要用来做<u>类型强制检测</u>,(**校验数据**)，不符合类型要求就会抛出异常

