# Python 异步编程、线程与进程

## 一句话总览

在常见的 Python `asyncio` 程序中，可以先记住下面这组对应关系：

```text
原生异步 I/O
→ 直接 await

同步阻塞 I/O
→ asyncio.to_thread() 或线程池

纯 Python CPU 密集计算
→ 进程池
```

判断依据不是“这个函数是不是同步函数”，而是它的大部分时间究竟在**等待**还是在**计算**。

---

## 一、进程、线程和 asyncio Task 的层级关系

```text
操作系统
│
├── Python 主进程
│   │
│   ├── 主线程
│   │   └── event loop
│   │       ├── Task A
│   │       ├── Task B
│   │       └── Task C
│   │
│   ├── 工作线程 1
│   └── 工作线程 2
│
├── 工作进程 1
│   └── 自己的主线程
│       └── CPU 计算任务
│
└── 工作进程 2
    └── 自己的主线程
        └── CPU 计算任务
```

需要区分三种概念：

- **进程**：一个正在运行的程序实例，拥有独立的内存空间、Python 解释器和 GIL。
- **线程**：进程内部的一条执行路线。同一个进程里的线程共享进程内存。
- **asyncio Task**：event loop 管理的异步任务，它不是操作系统线程。

一个进程可以拥有多个线程；一个线程里的 event loop 又可以管理很多 Task。

---

## 二、async def、coroutine、Task 和 await

### 1. async def

`async def` 定义的是异步函数，也叫协程函数：

```python
async def download():
    ...
```

调用它时，通常不会立即执行完整的函数体，而是先产生一个 coroutine object：

```python
coro = download()
```

可以把 coroutine object 理解为“一份待执行的异步代码”。

### 2. create_task

```python
task = asyncio.create_task(download())
```

`create_task()` 的作用是把协程包装成 Task，并加入当前 event loop 的调度清单。

event loop 拿回控制权后，就有机会运行这个 Task。

### 3. await

```python
result = await task
```

它的意思是：

> 当前协程需要这个 Task 的结果。如果结果还没有准备好，就暂停当前协程，把控制权交还给 event loop；结果完成后，再从这一行恢复。

重点是：

- 暂停的是当前协程，不是整个程序。
- 当前协程会停在 `await` 这一行。
- event loop 可以趁机运行其他已经调度的 Task。
- 如果没有其他 Task，`await` 不会凭空制造并发。

---

## 三、event loop 的调度方式

event loop 使用的是**协作式调度**：代码需要主动在合适的位置让出控制权。

```text
运行 Task A
→ Task A 遇到暂时无法完成的 await
→ 暂停 Task A
→ 运行 Task B
→ Task B 也进入等待
→ 运行 Task C
→ A 等待的 I/O 完成
→ 恢复 Task A
```

event loop 不能强行把一段正在运行的普通 Python 代码抢下来。

下面这个函数虽然写成了 `async def`，依然会卡住 event loop：

```python
async def bad_task():
    while True:
        calculate_something()
```

因为它内部没有能够暂停并交还控制权的 `await`。

所以，`async def` 不代表函数自动获得了非阻塞能力。

---

## 四、直接 await 和 create_task 的区别

### 有先后依赖：直接 await

```python
async def main():
    user = await get_user()
    orders = await get_orders(user.id)
```

必须先拿到 `user.id`，才能查询订单，所以这里适合顺序执行。

### 互相独立：创建多个 Task

```python
async def main():
    user_task = asyncio.create_task(get_user())
    weather_task = asyncio.create_task(get_weather())

    user = await user_task
    weather = await weather_task
```

也可以使用 `asyncio.gather()`：

```python
async def main():
    user, weather = await asyncio.gather(
        get_user(),
        get_weather(),
    )
```

可以这样记：

```text
直接 await coroutine
→ 当前步骤完成后才继续，适合有依赖的流程

create_task / gather
→ 先安排多个独立任务，再等待它们的结果
```

---

## 五、为什么 requests.get() 会阻塞 event loop

下面的代码仍然是阻塞的：

```python
import requests

async def download(url):
    response = requests.get(url)
    return response.text
```

`requests.get()` 是同步阻塞函数。网络结果返回以前，调用它的线程不能继续执行其他代码。

把同步函数写进 `async def`，不会自动把它改造成异步函数。

### 方案一：使用原生异步库

```python
import aiohttp

async def download(session, url):
    async with session.get(url) as response:
        return await response.text()
```

原生异步库能够在等待 socket 数据时主动把控制权交还给 event loop。

### 方案二：使用 asyncio.to_thread()

如果只有同步版本的库，可以把阻塞操作放进工作线程：

```python
import asyncio
import requests

async def download(url):
    response = await asyncio.to_thread(requests.get, url)
    return response.text
```

执行关系是：

```text
主线程
└── event loop：继续调度其他 Task

工作线程
└── requests.get()：负责阻塞等待网络
```

`to_thread()` 没有把 `requests` 变成真正的异步库，只是把阻塞等待搬到了其他线程。它通常使用线程池，不一定每次调用都创建一个新线程。

---

## 六、为什么 CPU 密集计算通常使用进程池

CPU 密集函数不是在等待，而是在持续计算：

```python
def calculate(n):
    return sum(i * i for i in range(n))
```

如果直接在 event loop 所在的线程执行，它会长时间占用线程，其他 Task 无法得到调度。

把它放进线程，可以避免它直接占住 event loop；但是对于普通的纯 Python 计算，通常不能明显提高计算速度。

原因是常见的 CPython 存在 GIL：同一个进程中，通常只有一个线程能够同时执行 Python 字节码。多个计算线程更接近轮流执行，而不是真正占用多个 CPU 核心并行计算。

不同进程拥有各自的 Python 解释器和 GIL，因此多个进程有机会被操作系统安排到不同 CPU 核心上真正并行计算。

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor

def calculate(n):
    return sum(i * i for i in range(n))

async def main():
    loop = asyncio.get_running_loop()

    with ProcessPoolExecutor() as pool:
        result = await loop.run_in_executor(
            pool,
            calculate,
            50_000_000,
        )

    print(result)

if __name__ == "__main__":
    asyncio.run(main())
```

从概念上看，进程池的工作进程会在它自己的主线程中执行提交的计算函数。主进程的 event loop 可以同时继续处理其他 Task。

操作系统不会保证每个进程永久独占一个 CPU 核心，但在存在多个核心时，不同进程可以真正同时执行。

---

## 七、线程和进程对比

| 对比项 | 线程 | 进程 |
| --- | --- | --- |
| 所属关系 | 位于进程内部 | 独立的程序实例 |
| 内存 | 同进程线程共享内存 | 默认拥有独立内存 |
| 创建成本 | 较低 | 较高 |
| 数据交换 | 方便，但要注意线程安全 | 通常需要序列化、队列、管道或共享内存 |
| 典型用途 | 阻塞型 I/O | 纯 Python CPU 密集计算 |
| 多核并行 | 普通 Python 计算通常受 GIL 限制 | 可以利用多个 CPU 核心 |
| 隔离性 | 一个线程出问题可能影响整个进程 | 进程之间隔离性更好 |

---

## 八、实际选择方法

先问一个问题：这个函数的大部分时间是在**等待**，还是在**计算**？

### 大部分时间在等待

例如：

- 网络请求
- 数据库查询
- 文件读写
- 调用外部服务
- 同步第三方 SDK

优先级：

```text
有成熟的原生异步库
→ 直接使用原生异步库

只有同步库
→ asyncio.to_thread() 或线程池
```

### 大部分时间在计算

例如：

- 大量纯 Python 循环
- 数学算法
- 文本计算
- 密码学计算
- 纯 Python 图像处理

通常选择进程池。

### 很短的普通同步函数

直接执行即可。开线程或进程本身也有成本，不应把所有同步函数都放进线程池或进程池。

### 进阶例外

NumPy、OpenCV 等部分底层 C/C++ 扩展可能会在计算时释放 GIL。这种情况下，线程也可能真正并行。面试时可以先回答“纯 Python CPU 密集任务通常使用多进程”，再补充这个例外。

---

## 九、面试快速回答

### asyncio 为什么能提高并发能力？

因为 I/O 等待期间，当前协程可以通过 `await` 暂停并把控制权交还给 event loop。event loop 可以继续运行其他 Task，从而避免线程在 I/O 上空等。

### create_task() 做了什么？

它把 coroutine 包装成 Task，并加入当前 event loop 的调度清单。event loop 拿回控制权后，可以调度该 Task 运行。

### await 会不会阻塞？

它会暂停当前协程的后续逻辑，但不会阻塞整个 event loop。event loop 可以运行其他已经调度的 Task。

### 为什么 requests.get() 放进 async def 还是会阻塞？

因为 `requests.get()` 本身是同步阻塞函数，不会在等待网络期间把控制权交还给 event loop。可以改用 `aiohttp`，或者通过 `asyncio.to_thread()` 放到工作线程。

### 线程和进程怎么选？

- 阻塞型 I/O：线程或原生异步库。
- 纯 Python CPU 密集计算：进程。
- 很短的普通函数：直接执行。

### CPU 密集任务为什么通常不用线程？

常见 CPython 存在 GIL，同一个进程里的多个线程通常不能同时执行纯 Python 字节码。多进程拥有独立解释器和 GIL，可以利用多个 CPU 核心并行计算。

---

## 十、最终记忆版

```text
async def
→ 定义可以暂停和恢复的协程函数

create_task
→ 把协程加入 event loop 的调度清单

await
→ 当前协程等待并让出控制权

event loop
→ 调度当前可以运行的 Task

同步阻塞 I/O
→ 放到线程池，避免卡住 event loop

纯 Python CPU 密集计算
→ 放到进程池，避免卡住 event loop并利用多个 CPU 核心
```