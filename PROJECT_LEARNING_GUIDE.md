# TinyWebServer 项目逻辑梳理与学习路线

> 适合只有 C++ 基础、第一次接触 Linux 网络编程的同学。

## 1. 先建立全局印象
这个项目是一个 Linux 下的轻量级 Web 服务器，核心目标是：
- 接收浏览器请求
- 解析 HTTP 报文
- 按 URL 返回静态页面或处理登录/注册
- 使用线程池、epoll、定时器、日志系统、MySQL 连接池提升并发能力

### 启动主线
对应源码入口：
- `main.cpp`
- `config.h` / `config.cpp`
- `webserver.h` / `webserver.cpp`
- `http/http_conn.h` / `http/http_conn.cpp`

主流程可以理解为：
1. `main.cpp` 里创建 `Config`，解析命令行参数
2. 创建 `WebServer`
3. 依次初始化：日志、数据库连接池、线程池、触发模式、监听套接字
4. 进入 `eventLoop()`，开始用 `epoll` 处理连接、读写事件、信号和超时

---

## 2. 项目整体架构图（文字版）
### 2.1 分层理解
- **入口层**：`main.cpp`、`config.cpp`
  - 负责参数解析和服务器启动
- **网络事件层**：`webserver.cpp`
  - 负责 `socket`、`bind`、`listen`、`epoll`、信号处理、定时器
- **HTTP 处理层**：`http/http_conn.cpp`
  - 负责请求解析、路由判断、响应构造
- **并发层**：`threadpool/threadpool.h`
  - 负责把读/写任务分发到工作线程
- **资源管理层**：`timer/lst_timer.cpp`、`CGImysql/sql_connection_pool.cpp`、`log/log.cpp`、`lock/locker.h`
  - 负责超时连接、数据库连接、日志、锁与信号量封装

### 2.2 数据流
浏览器请求进入后：
`epoll` 监听到事件 → `WebServer` 分发给 `http_conn` → `http_conn` 解析报文 → 若是登录/注册则访问 MySQL → 构造响应文件或错误页 → 写回浏览器

---

## 3. 每个模块要学什么

### 3.1 `config.cpp`：命令行参数
重点知识点：
- `getopt` 解析命令行
- 默认值设计
- 参数与服务器运行模式的映射

你要重点理解这些参数：
- `-p` 端口
- `-l` 日志写入方式
- `-m` 触发模式（LT/ET 组合）
- `-o` 是否优雅关闭
- `-s` 数据库连接数
- `-t` 线程数
- `-c` 是否关闭日志
- `-a` Reactor / Proactor 模式

### 3.2 `webserver.cpp`：服务器核心控制器
重点知识点：
- `socket` / `bind` / `listen`
- `epoll_create`、`epoll_wait`
- 非阻塞 socket
- `socketpair` 传递信号
- `SIGALRM` 驱动定时器，`SIGTERM` 触发退出
- `SO_LINGER` 的优雅关闭

你要特别理解：
- `eventListen()`：初始化监听、epoll、信号、管道
- `eventLoop()`：主循环，按事件类型分发
- `dealclientdata()`：接受新连接
- `dealwithread()` / `dealwithwrite()`：提交读写任务给线程池
- `timer()` / `adjust_timer()` / `deal_timer()`：连接超时控制

### 3.3 `http/http_conn.cpp`：HTTP 请求处理
重点知识点：
- 状态机解析 HTTP
- 请求行、请求头、请求体解析
- `GET` / `POST`
- `mmap` 映射文件，提高静态文件返回效率
- 响应报文拼接

你要理解的处理链：
1. `read_once()` 读入数据
2. `process_read()` 按状态机拆行
3. `parse_request_line()` 识别方法和 URL
4. `parse_headers()` 处理请求头
5. `parse_content()` 处理 POST 请求体
6. `do_request()` 处理路由和文件定位
7. `process_write()` 生成响应
8. `write()` 通过 `writev` 发送

### 3.4 `threadpool/threadpool.h`：线程池
重点知识点：
- 生产者/消费者模型
- `pthread_create` / `pthread_detach`
- 互斥锁 + 信号量
- 队列任务分发
- Reactor / Proactor 两种处理方式

核心理解：
- `append()`：Reactor 下把“读/写任务”放入队列
- `append_p()`：Proactor 下把请求对象放入队列
- `run()`：工作线程循环取任务并执行

### 3.5 `CGImysql/sql_connection_pool.cpp`：数据库连接池
重点知识点：
- 单例模式
- 连接池初始化
- `sem` 控制空闲连接数量
- RAII 自动获取/释放连接
- MySQL C API 基础用法

必须掌握：
- `connection_pool::GetInstance()`
- `connection_pool::init()`
- `GetConnection()` / `ReleaseConnection()`
- `connectionRAII` 的作用：自动回收数据库连接

### 3.6 `log/log.cpp`：日志系统
重点知识点：
- 单例日志对象
- 同步日志 vs 异步日志
- 队列写日志
- 按天/按行分割日志文件
- 格式化输出

你需要理解：
- `Log::get_instance()`：局部静态单例
- `init()`：决定同步还是异步
- `write_log()`：组装时间、级别、内容
- `flush()`：刷新缓冲区

### 3.7 `timer/lst_timer.cpp`：定时器
重点知识点：
- 侵入式链表 / 排序链表
- 连接超时回收
- `SIGALRM` 定时触发
- 关闭空闲连接

你要理解：
- `sort_timer_lst` 维护按过期时间排序的 timer 链表
- `add_timer()` / `adjust_timer()` / `del_timer()` / `tick()`
- `cb_func()` 是连接超时的回调

### 3.8 `lock/locker.h`：锁封装
重点知识点：
- `pthread_mutex_t`
- `sem_t`
- 条件变量 `pthread_cond_t`
- 用 C++ 类包装 C 风格同步原语

这个模块的意义是：让线程池、连接池、日志系统共享统一的同步封装。

---

## 4. 这份项目最重要的“约定”
### 4.1 路由是写死的
`http/http_conn.cpp` 里对 URL 的处理不是框架式路由，而是直接映射到 `root/` 下的固定文件：
- `/` → `judge.html`
- `/0` → `register.html`
- `/1` → `log.html`
- `/2` → 登录校验
- `/3` → 注册校验
- `/5` → `picture.html`
- `/6` → `video.html`
- `/7` → `fans.html`

所以：**不要随便改 `root/` 下页面名字**，否则 URL 会对不上。

### 4.2 登录/注册依赖数据库表
项目默认使用 `user` 表：
- `username`
- `passwd`

如果你自己搭数据库，要保证表结构和 `main.cpp` 里的数据库名一致。

### 4.3 默认并发模型不是“单一写法”
这个项目同时支持：
- **Proactor**：主线程负责读写，工作线程处理业务
- **Reactor**：工作线程自己完成读写再处理业务

所以看到 `m_actormodel`、`m_state`、`append()`、`append_p()` 时，不要当成冗余代码，它们是在切换模型。

---

## 5. 推荐学习顺序
如果你是 C++ 基础用户，建议按这个顺序学：

1. **先看 `README.md`**
   - 先知道这个项目要做什么
2. **看 `main.cpp` + `config.cpp`**
   - 先搞懂程序怎么启动
3. **看 `webserver.cpp`**
   - 理解 epoll 主循环和连接生命周期
4. **看 `http/http_conn.cpp`**
   - 理解 HTTP 请求如何解析和响应如何生成
5. **看 `threadpool/threadpool.h`**
   - 理解请求如何被多线程处理
6. **看 `timer/lst_timer.cpp`**
   - 理解超时连接怎么回收
7. **看 `CGImysql/sql_connection_pool.cpp`**
   - 理解数据库怎么安全复用连接
8. **看 `log/log.cpp`**
   - 理解日志系统怎么记录运行状态
9. **最后回到 `lock/locker.h`**
   - 理解同步原语封装

---

## 6. 学习这套技术时的知识点清单

### C++ 基础补充
- 类、构造函数、析构函数
- 指针与动态内存
- `new[] / delete[]`
- `map`、`list`
- 模板 `template <typename T>`
- `static` 成员变量
- RAII 思想

### Linux 网络编程
- `socket`
- `bind`
- `listen`
- `accept`
- `recv` / `send`
- 非阻塞 I/O
- `epoll`
- `ET` / `LT`
- `EPOLLONESHOT`
- `socketpair`
- `signal` / `sigaction`

### 并发编程
- 线程池
- 互斥锁
- 信号量
- 条件变量
- 生产者/消费者模型
- Reactor / Proactor

### HTTP 协议
- 请求行
- 请求头
- 请求体
- `GET` / `POST`
- `Connection: keep-alive`
- 状态码与响应体

### 数据库
- MySQL 连接池
- `mysql_init`
- `mysql_real_connect`
- `mysql_query`
- `mysql_store_result`
- 用户注册/登录逻辑

### 文件与内存映射
- `open`
- `stat`
- `mmap`
- `munmap`
- 用内存映射返回静态文件

### 日志系统
- 文件按日期/行数切分
- 同步写日志
- 异步写日志队列
- `printf` 风格格式化输出

---

## 7. 你可以怎么跑起来
### 编译
```bash
make server
```

或者：
```bash
sh ./build.sh
```

### 运行
```bash
./server
```

### 常用参数
```bash
./server [-p port] [-l LOGWrite] [-m TRIGMode] [-o OPT_LINGER] [-s sql_num] [-t thread_num] [-c close_log] [-a actor_model]
```

---

## 8. 学习时最容易踩的坑
- 只看 `http_conn.cpp` 不看 `webserver.cpp`，会搞不懂事件是怎么进来的
- 只看 `threadpool` 不看 `m_actormodel`，会分不清 Reactor 和 Proactor
- 改了 `root/` 页面名，却没改 `do_request()` 的硬编码映射
- 忽略数据库连接池和 RAII，容易漏释放连接
- 不理解 `epoll + 非阻塞 + ET`，会看不懂读写为什么要循环读完

---

## 9. 建议你的学习目标
学完这份项目后，你至少应该能回答：
- 服务器是怎么启动的？
- 一个请求从浏览器到返回页面经历了什么？
- 为什么要用线程池、epoll、定时器、连接池？
- Reactor 和 Proactor 的区别是什么？
- 登录/注册是如何和 MySQL 交互的？
- 为什么要把日志、锁、连接池做成封装类？

如果你能把这 6 个问题讲清楚，说明你已经真正理解了这个项目。

