# IO优化课程-04: 网络IO深度优化

## 🎯 课程目标

深入理解网络IO的性能瓶颈与优化技术：
- 网络IO模型与系统调用
- 零拷贝技术原理与实践
- IO多路复用的演进（select/poll/epoll/io_uring）
- TCP/UDP协议栈优化
- 高性能网络编程最佳实践
- 内核旁路与用户态网络栈

---

## 目录

1. [网络IO基础](#1-网络io基础)
2. [系统调用与数据拷贝](#2-系统调用与数据拷贝)
3. [IO模型演进](#3-io模型演进)
4. [网络读取优化策略](#4-网络读取优化策略)
5. [零拷贝技术](#5-零拷贝技术)
6. [TCP协议栈优化](#6-tcp协议栈优化)
7. [高性能网络编程](#7-高性能网络编程)
8. [内核旁路技术](#8-内核旁路技术)

---

## 1. 网络IO基础

### 1.1 网络IO的生命周期

```
应用程序发送数据的完整路径：

┌─────────────────────────────────────────────────────────┐
│ 用户态应用                                               │
│ ┌─────────────────────────────────────────────────┐    │
│ │ send(sockfd, buffer, len, flags)                │    │
│ └──────────────────┬──────────────────────────────┘    │
└────────────────────┼───────────────────────────────────┘
                     ↓ 用户态→内核态切换（系统调用）
┌────────────────────┼───────────────────────────────────┐
│ 内核态             ↓                                    │
│ ┌─────────────────────────────────────────────────┐    │
│ │ 1. 拷贝数据：用户缓冲区 → 内核Socket发送缓冲区    │    │
│ └─────────────────────────────────────────────────┘    │
│                     ↓                                    │
│ ┌─────────────────────────────────────────────────┐    │
│ │ 2. TCP协议栈处理                                 │    │
│ │    - 分段（MSS）                                 │    │
│ │    - 添加TCP/IP头部                              │    │
│ │    - 校验和计算                                  │    │
│ │    - 拥塞控制                                    │    │
│ └─────────────────────────────────────────────────┘    │
│                     ↓                                    │
│ ┌─────────────────────────────────────────────────┐    │
│ │ 3. 网卡驱动层                                    │    │
│ │    - 构建DMA描述符                               │    │
│ │    - 触发DMA传输                                 │    │
│ └─────────────────────────────────────────────────┘    │
└────────────────────┼───────────────────────────────────┘
                     ↓ DMA传输（不占用CPU）
┌────────────────────┼───────────────────────────────────┐
│ 硬件               ↓                                    │
│ ┌─────────────────────────────────────────────────┐    │
│ │ 4. 网卡硬件                                      │    │
│ │    - 从内存读取数据（DMA）                       │    │
│ │    - 添加以太网帧头                              │    │
│ │    - 物理层编码                                  │    │
│ │    - 发送到网络                                  │    │
│ └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### 1.2 网络IO的性能开销

典型的网络IO开销分解（基于1Gbps网络，1KB数据包）：

```
总延迟：~50-100μs

┌──────────────────────────────────────────────────┐
│ 1. 系统调用开销              ~1-2μs (2%)         │
│    - 用户态→内核态切换                            │
│    - 上下文保存/恢复                              │
└──────────────────────────────────────────────────┘
┌──────────────────────────────────────────────────┐
│ 2. 数据拷贝                  ~5-10μs (15%)       │
│    - 用户缓冲区→内核Socket缓冲区                  │
│    - 内存带宽限制                                 │
└──────────────────────────────────────────────────┘
┌──────────────────────────────────────────────────┐
│ 3. 协议栈处理                ~10-20μs (30%)      │
│    - TCP/IP协议处理                               │
│    - 校验和计算                                   │
│    - 路由查找                                     │
└──────────────────────────────────────────────────┘
┌──────────────────────────────────────────────────┐
│ 4. DMA与硬件处理             ~5-10μs (15%)       │
│    - DMA设置与传输                                │
│    - 网卡队列处理                                 │
└──────────────────────────────────────────────────┘
┌──────────────────────────────────────────────────┐
│ 5. 网络传输时间              ~8μs (12%)          │
│    - 1KB @ 1Gbps = 8μs                           │
│    - 光速延迟（可忽略本地）                       │
└──────────────────────────────────────────────────┘
┌──────────────────────────────────────────────────┐
│ 6. 其他（中断、调度等）       ~20-50μs (26%)     │
└──────────────────────────────────────────────────┘
```

**优化重点**：减少拷贝（零拷贝）、批量处理（减少系统调用）、内核旁路（DPDK）

### 1.3 网络IO的性能指标

#### 带宽（Bandwidth）
- **定义**：单位时间内传输的数据量
- **单位**：Mbps, Gbps, MB/s, GB/s
- **测量**：iperf, qperf

#### 延迟（Latency）
- **RTT**：Round-Trip Time，往返时间
- **单向延迟**：一个方向的传输时间
- **测量**：ping, sockperf

#### 吞吐量（Throughput）
- **消息吞吐**：每秒处理的消息数
- **事务吞吐**：每秒完成的事务数
- **测量**：wrk, ab, netperf

#### 连接数
- **并发连接**：同时保持的TCP连接数
- **新建连接速率**：每秒新建的连接数
- **测量**：ss, netstat

---

## 2. 系统调用与数据拷贝

### 2.1 传统网络IO的四次拷贝

**场景：读取文件并通过网络发送**

```cpp
int fd = open("file.dat", O_RDONLY);
char buffer[4096];
read(fd, buffer, sizeof(buffer));
send(sockfd, buffer, sizeof(buffer), 0);
```

**数据拷贝路径：**

```
┌─────────────────────────────────────────────────────────┐
│ 磁盘                                                     │
│ ┌─────────────────┐                                     │
│ │ file.dat        │                                     │
│ └────────┬────────┘                                     │
└──────────┼──────────────────────────────────────────────┘
           ↓ DMA拷贝（第1次）
┌──────────┼──────────────────────────────────────────────┐
│ 内核态   ↓                                              │
│ ┌────────────────┐                                      │
│ │ 内核页面缓存    │ ← read()系统调用                     │
│ └────────┬───────┘                                      │
│          ↓ CPU拷贝（第2次）                              │
└──────────┼──────────────────────────────────────────────┘
           ↓
┌──────────┼──────────────────────────────────────────────┐
│ 用户态   ↓                                              │
│ ┌────────────────┐                                      │
│ │ 用户缓冲区      │ buffer[4096]                         │
│ └────────┬───────┘                                      │
│          ↓ CPU拷贝（第3次）                              │
└──────────┼──────────────────────────────────────────────┘
           ↓ send()系统调用
┌──────────┼──────────────────────────────────────────────┐
│ 内核态   ↓                                              │
│ ┌────────────────┐                                      │
│ │ Socket发送缓冲  │                                      │
│ └────────┬───────┘                                      │
│          ↓ DMA拷贝（第4次）                              │
│ ┌────────────────┐                                      │
│ │ 网卡            │                                      │
│ └────────────────┘                                      │
└─────────────────────────────────────────────────────────┘
```

**问题分析：**
- **4次数据拷贝**：2次DMA + 2次CPU拷贝
- **4次上下文切换**：read进入、read返回、send进入、send返回
- **CPU开销**：拷贝1MB数据约消耗40-50μs（DDR4 @ 25GB/s）

### 2.2 系统调用的开销

```cpp
// 测试系统调用开销
#include <unistd.h>
#include <time.h>

void benchmark_syscall() {
    struct timespec start, end;
    const int iterations = 1000000;
    
    clock_gettime(CLOCK_MONOTONIC, &start);
    for (int i = 0; i < iterations; i++) {
        getpid();  // 最简单的系统调用
    }
    clock_gettime(CLOCK_MONOTONIC, &end);
    
    long ns = (end.tv_sec - start.tv_sec) * 1000000000L + 
              (end.tv_nsec - start.tv_nsec);
    printf("每次系统调用: %ld ns\n", ns / iterations);
    // 典型结果: ~50-100ns (包含上下文切换)
}
```

**系统调用开销来源：**
1. **上下文切换**：保存/恢复寄存器状态 (~20-30ns)
2. **特权级切换**：用户态(Ring 3) → 内核态(Ring 0) (~10-20ns)
3. **TLB刷新**：切换页表可能导致TLB miss
4. **Cache污染**：内核代码/数据替换用户态Cache内容

**优化策略：**
- 批量处理：减少系统调用次数
- 异步IO：避免阻塞等待
- 用户态网络栈：完全避免系统调用

---

## 3. IO模型演进

### 3.1 阻塞IO（Blocking IO）

```cpp
// 每个连接一个线程
void handle_client(int sockfd) {
    char buffer[4096];
    while (1) {
        ssize_t n = recv(sockfd, buffer, sizeof(buffer), 0);
        if (n <= 0) break;
        
        // 处理数据
        process(buffer, n);
        
        // 发送响应
        send(sockfd, response, response_len, 0);
    }
    close(sockfd);
}

// 主循环
while (1) {
    int client_fd = accept(listen_fd, NULL, NULL);
    pthread_create(&thread, NULL, handle_client, client_fd);
}
```

**特点：**
- 简单直观，易于理解
- 每个连接需要一个线程
- 线程开销大（栈空间~1-8MB）
- 上下文切换频繁

**性能限制：**
- C10K问题：1万个连接需要1万个线程
- 内存消耗：10K线程 × 2MB栈 = 20GB
- 调度开销：O(n)的调度复杂度

### 3.2 非阻塞IO + select/poll

```cpp
// select示例
fd_set readfds;
int max_fd = 0;

while (1) {
    FD_ZERO(&readfds);
    FD_SET(listen_fd, &readfds);
    max_fd = listen_fd;
    
    // 添加所有客户端socket
    for (int i = 0; i < client_count; i++) {
        FD_SET(clients[i], &readfds);
        if (clients[i] > max_fd) max_fd = clients[i];
    }
    
    // 等待事件
    int ret = select(max_fd + 1, &readfds, NULL, NULL, NULL);
    
    // 检查哪些fd就绪
    if (FD_ISSET(listen_fd, &readfds)) {
        int client_fd = accept(listen_fd, NULL, NULL);
        clients[client_count++] = client_fd;
    }
    
    for (int i = 0; i < client_count; i++) {
        if (FD_ISSET(clients[i], &readfds)) {
            handle_request(clients[i]);
        }
    }
}
```

**select的限制：**
- FD数量限制：默认1024（FD_SETSIZE）
- O(n)复杂度：每次需要遍历所有fd
- 每次调用需要拷贝fd_set到内核

**poll改进：**
```cpp
struct pollfd fds[MAX_CLIENTS];
fds[0].fd = listen_fd;
fds[0].events = POLLIN;
int nfds = 1;

while (1) {
    int ret = poll(fds, nfds, -1);
    
    if (fds[0].revents & POLLIN) {
        int client_fd = accept(listen_fd, NULL, NULL);
        fds[nfds].fd = client_fd;
        fds[nfds].events = POLLIN;
        nfds++;
    }
    
    for (int i = 1; i < nfds; i++) {
        if (fds[i].revents & POLLIN) {
            handle_request(fds[i].fd);
        }
    }
}
```

**poll的改进：**
- 无FD数量硬限制
- 使用pollfd数组，更灵活

**仍然存在的问题：**
- O(n)复杂度
- 每次调用需要传递整个pollfd数组

### 3.3 epoll（Linux）

```cpp
// epoll示例 - 解决C10K问题
int epoll_fd = epoll_create1(0);

// 添加监听socket
struct epoll_event ev;
ev.events = EPOLLIN;
ev.data.fd = listen_fd;
epoll_ctl(epoll_fd, EPOLL_CTL_ADD, listen_fd, &ev);

struct epoll_event events[MAX_EVENTS];

while (1) {
    // 等待事件，O(1)复杂度
    int nfds = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
    
    for (int i = 0; i < nfds; i++) {
        if (events[i].data.fd == listen_fd) {
            // 新连接
            int client_fd = accept(listen_fd, NULL, NULL);
            
            // 设置非阻塞
            int flags = fcntl(client_fd, F_GETFL, 0);
            fcntl(client_fd, F_SETFL, flags | O_NONBLOCK);
            
            // 添加到epoll
            ev.events = EPOLLIN | EPOLLET;  // 边缘触发
            ev.data.fd = client_fd;
            epoll_ctl(epoll_fd, EPOLL_CTL_ADD, client_fd, &ev);
        } else {
            // 处理数据
            handle_request(events[i].data.fd);
        }
    }
}
```

**epoll的优势：**

1. **O(1)复杂度**：使用红黑树管理fd，就绪列表返回就绪的fd
2. **无数量限制**：受限于系统资源，不是硬编码限制
3. **内核维护状态**：不需要每次传递所有fd
4. **两种触发模式**：
   - **水平触发（LT）**：只要有数据就通知（默认）
   - **边缘触发（ET）**：只在状态变化时通知（更高效）

**边缘触发示例：**

```cpp
// ET模式必须读取所有数据
void handle_request_et(int fd) {
    char buffer[4096];
    while (1) {
        ssize_t n = recv(fd, buffer, sizeof(buffer), 0);
        if (n < 0) {
            if (errno == EAGAIN || errno == EWOULDBLOCK) {
                // 没有更多数据了
                break;
            } else {
                perror("recv error");
                close(fd);
                return;
            }
        } else if (n == 0) {
            // 连接关闭
            close(fd);
            return;
        }
        
        process(buffer, n);
    }
}
```

**性能对比：**

| 方法   | 时间复杂度 | FD限制      | 内存拷贝       | 适用场景           |
|--------|-----------|-------------|---------------|-------------------|
| select | O(n)      | 1024(硬限制) | 每次调用拷贝   | 少量连接(<100)     |
| poll   | O(n)      | 无硬限制     | 每次调用拷贝   | 中等连接(<1000)    |
| epoll  | O(1)      | 受系统限制   | 内核维护状态   | 大量连接(C10K+)    |

### 3.4 io_uring（Linux 5.1+）

**革命性的异步IO接口**

```cpp
// io_uring示例
#include <liburing.h>

struct io_uring ring;
io_uring_queue_init(256, &ring, 0);

// 提交读请求（不阻塞）
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, fd, buffer, sizeof(buffer), offset);
io_uring_sqe_set_data(sqe, user_data);
io_uring_submit(&ring);

// 批量提交多个操作
for (int i = 0; i < batch_size; i++) {
    sqe = io_uring_get_sqe(&ring);
    io_uring_prep_read(sqe, fds[i], buffers[i], sizes[i], offsets[i]);
}
io_uring_submit(&ring);  // 一次系统调用提交多个IO

// 收割完成的IO
struct io_uring_cqe *cqe;
while (io_uring_peek_cqe(&ring, &cqe) == 0) {
    void *user_data = io_uring_cqe_get_data(cqe);
    if (cqe->res < 0) {
        // 错误处理
    } else {
        // 处理完成的IO
        process_completed_io(user_data, cqe->res);
    }
    io_uring_cqe_seen(&ring, cqe);
}
```

**io_uring架构：**

```
用户态                     内核态
┌──────────────────┐      ┌──────────────────┐
│ 应用程序          │      │                  │
│                  │      │                  │
│ ┌──────────┐    │      │  ┌──────────┐   │
│ │ SQ Ring  │────┼──────┼→│ 内核处理  │   │
│ │(提交队列) │    │ 共享  │  │          │   │
│ └──────────┘    │ 内存  │  │          │   │
│                  │      │  │          │   │
│ ┌──────────┐    │      │  └─────┬────┘   │
│ │ CQ Ring  │←───┼──────┼────────┘        │
│ │(完成队列) │    │      │                  │
│ └──────────┘    │      │                  │
└──────────────────┘      └──────────────────┘

特点：
1. 零拷贝：通过共享内存环形缓冲区通信
2. 批量处理：一次系统调用提交/收割多个IO
3. 真正异步：不阻塞，不需要轮询
4. 统一接口：网络IO、文件IO、定时器等
```

**性能优势：**
- 减少系统调用：批量提交/收割
- 零拷贝：通过共享内存
- 无上下文切换：轮询模式（IORING_SETUP_SQPOLL）
- 更低延迟：相比epoll降低30-50%

---

## 4. 网络读取优化策略

### 4.1 接收缓冲区优化

**问题：默认接收缓冲区太小**

```cpp
#include <sys/socket.h>
#include <netinet/in.h>
#include <netinet/tcp.h>

// 查看和设置接收缓冲区
void OptimizeRecvBuffer(int sockfd) {
  // 1. 查看当前接收缓冲区大小
  int recv_buf_size;
  socklen_t optlen = sizeof(recv_buf_size);
  getsockopt(sockfd, SOL_SOCKET, SO_RCVBUF, &recv_buf_size, &optlen);
  printf("Default recv buffer: %d bytes\n", recv_buf_size);
  // 典型输出：87380 bytes（约85KB）

  // 2. 设置更大的接收缓冲区（适合高带宽）
  int new_size = 16 * 1024 * 1024;  // 16MB
  setsockopt(sockfd, SOL_SOCKET, SO_RCVBUF, &new_size, sizeof(new_size));

  // 3. 验证实际设置的大小
  getsockopt(sockfd, SOL_SOCKET, SO_RCVBUF, &recv_buf_size, &optlen);
  printf("Actual recv buffer: %d bytes\n", recv_buf_size);

  // 注意：内核可能设置为2倍（用于元数据）
  // 实际值可能是32MB
}

// 系统级调优（需要root权限）
// 编辑 /etc/sysctl.conf
net.core.rmem_default = 262144      # 默认256KB
net.core.rmem_max = 16777216        # 最大16MB
net.ipv4.tcp_rmem = 4096 87380 16777216  # min default max

// 应用更改
// sudo sysctl -p
```

**带宽延迟积（BDP）计算：**

```cpp
// BDP = Bandwidth × RTT
// 示例：
// - 带宽：1 Gbps = 125 MB/s
// - RTT：40ms
// - BDP = 125 MB/s × 0.04s = 5 MB
//
// 接收缓冲区应该 >= BDP，否则无法跑满带宽

void CalculateOptimalBuffer(double bandwidth_mbps, double rtt_ms) {
  double bandwidth_bytes_per_sec = bandwidth_mbps * 1024 * 1024 / 8;
  double rtt_sec = rtt_ms / 1000.0;

  size_t bdp = bandwidth_bytes_per_sec * rtt_sec;
  size_t optimal_buffer = bdp * 2;  // 2倍BDP

  printf("Bandwidth: %.2f Mbps\n", bandwidth_mbps);
  printf("RTT: %.2f ms\n", rtt_ms);
  printf("BDP: %zu bytes (%.2f MB)\n", bdp, bdp / 1024.0 / 1024.0);
  printf("Optimal buffer: %zu bytes (%.2f MB)\n",
         optimal_buffer, optimal_buffer / 1024.0 / 1024.0);
}

// 示例输出：
// Bandwidth: 1000.00 Mbps
// RTT: 40.00 ms
// BDP: 5242880 bytes (5.00 MB)
// Optimal buffer: 10485760 bytes (10.00 MB)
```

### 4.2 批量接收（Batch Recv）

**问题：多次小recv的开销**

```cpp
#include <sys/socket.h>
#include <sys/uio.h>

// 不好：多次小recv
void MultipleSmallRecv_Bad(int sockfd) {
  char buffer[1024];

  for (int i = 0; i < 1000; i++) {
    int n = recv(sockfd, buffer, sizeof(buffer), 0);  // 1000次系统调用
    if (n > 0) {
      process(buffer, n);
    }
  }
  // 开销：1000次用户态→内核态切换
}

// 好：批量recv
void BatchRecv_Good(int sockfd) {
  const size_t BATCH_SIZE = 1024 * 1024;  // 1MB批量
  char* buffer = new char[BATCH_SIZE];

  int n = recv(sockfd, buffer, BATCH_SIZE, 0);  // 1次系统调用
  if (n > 0) {
    // 批量处理
    for (size_t offset = 0; offset < n; offset += 1024) {
      size_t chunk_size = std::min<size_t>(1024, n - offset);
      process(buffer + offset, chunk_size);
    }
  }

  delete[] buffer;
  // 性能提升：10-100倍
}

// 更好：recvmsg + scatter-gather IO
void ScatterGatherRecv(int sockfd) {
  // 准备多个缓冲区
  const int NUM_BUFFERS = 10;
  struct iovec iov[NUM_BUFFERS];
  char* buffers[NUM_BUFFERS];

  for (int i = 0; i < NUM_BUFFERS; i++) {
    buffers[i] = new char[4096];
    iov[i].iov_base = buffers[i];
    iov[i].iov_len = 4096;
  }

  // 一次系统调用读取到多个缓冲区
  struct msghdr msg = {};
  msg.msg_iov = iov;
  msg.msg_iovlen = NUM_BUFFERS;

  ssize_t total = recvmsg(sockfd, &msg, 0);  // 最多接收40KB

  // 处理每个缓冲区
  size_t processed = 0;
  for (int i = 0; i < NUM_BUFFERS && processed < total; i++) {
    size_t chunk_size = std::min<size_t>(iov[i].iov_len, total - processed);
    process(buffers[i], chunk_size);
    processed += chunk_size;
  }

  for (int i = 0; i < NUM_BUFFERS; i++) {
    delete[] buffers[i];
  }
}
```

### 4.3 零拷贝读取

**splice + pipe优化：**

```cpp
#include <fcntl.h>
#include <unistd.h>

// 传统方式：接收数据写入文件
void TraditionalRecvToFile(int sockfd, int filefd) {
  char buffer[65536];

  while (true) {
    // 1. recv：内核→用户态（拷贝1）
    ssize_t n = recv(sockfd, buffer, sizeof(buffer), 0);
    if (n <= 0) break;

    // 2. write：用户态→内核（拷贝2）
    write(filefd, buffer, n);
  }

  // 总共2次拷贝，2次上下文切换
}

// 零拷贝方式：splice直接转发
void ZeroCopyRecvToFile(int sockfd, int filefd) {
  // 创建管道
  int pipefd[2];
  pipe(pipefd);

  while (true) {
    // 1. socket → pipe（内核内拷贝，零拷贝到用户态）
    ssize_t n = splice(sockfd, nullptr, pipefd[1], nullptr,
                       65536, SPLICE_F_MOVE | SPLICE_F_MORE);
    if (n <= 0) break;

    // 2. pipe → file（内核内拷贝）
    splice(pipefd[0], nullptr, filefd, nullptr, n, SPLICE_F_MOVE);
  }

  close(pipefd[0]);
  close(pipefd[1]);

  // 零用户态拷贝，性能提升2-3倍
}

// 性能对比（接收1GB数据）：
// Traditional: 800 MB/s
// Zero-copy:  1800 MB/s（2.25倍）
```

**MSG_PEEK预读优化：**

```cpp
// 应用层协议解析：先读取header，再读取body
struct MessageHeader {
  uint32_t magic;
  uint32_t length;
  uint32_t type;
};

// 不好：多次recv
void ParseMessage_Bad(int sockfd) {
  MessageHeader header;

  // 1. 先读header
  recv(sockfd, &header, sizeof(header), 0);

  // 2. 再读body
  char* body = new char[header.length];
  recv(sockfd, body, header.length, 0);

  process(header, body);
  delete[] body;
}

// 好：MSG_PEEK预读
void ParseMessage_Good(int sockfd) {
  MessageHeader header;

  // 1. 预读header（不从缓冲区移除）
  recv(sockfd, &header, sizeof(header), MSG_PEEK);

  // 2. 一次性读取完整消息
  size_t total_size = sizeof(header) + header.length;
  char* message = new char[total_size];
  recv(sockfd, message, total_size, 0);

  // 3. 解析
  MessageHeader* h = (MessageHeader*)message;
  char* body = message + sizeof(header);
  process(*h, body);

  delete[] message;
  // 减少系统调用次数
}

// 更好：MSG_WAITALL确保读取完整数据
void ParseMessage_Best(int sockfd) {
  MessageHeader header;

  // 1. 读header（阻塞直到读满）
  recv(sockfd, &header, sizeof(header), MSG_WAITALL);

  // 2. 读body（阻塞直到读满）
  char* body = new char[header.length];
  recv(sockfd, body, header.length, MSG_WAITALL);

  process(header, body);
  delete[] body;

  // MSG_WAITALL避免短读，减少循环调用
}
```

### 4.4 多路复用读取优化

**epoll边缘触发（ET）vs 水平触发（LT）：**

```cpp
#include <sys/epoll.h>

// 水平触发（LT）：简单但可能惊群
void EpollLevelTriggered(int epollfd) {
  struct epoll_event events[MAX_EVENTS];

  while (true) {
    int nfds = epoll_wait(epollfd, events, MAX_EVENTS, -1);

    for (int i = 0; i < nfds; i++) {
      int sockfd = events[i].data.fd;

      if (events[i].events & EPOLLIN) {
        // LT模式：可以只读部分数据
        char buffer[4096];
        ssize_t n = recv(sockfd, buffer, sizeof(buffer), 0);

        if (n > 0) {
          process(buffer, n);
        }
        // 如果socket还有数据，下次epoll_wait会再次触发
      }
    }
  }
}

// 边缘触发（ET）：高效但需要读尽
void EpollEdgeTriggered(int epollfd) {
  struct epoll_event events[MAX_EVENTS];

  while (true) {
    int nfds = epoll_wait(epollfd, events, MAX_EVENTS, -1);

    for (int i = 0; i < nfds; i++) {
      int sockfd = events[i].data.fd;

      if (events[i].events & EPOLLIN) {
        // ET模式：必须读尽所有数据
        while (true) {
          char buffer[4096];
          ssize_t n = recv(sockfd, buffer, sizeof(buffer), 0);

          if (n > 0) {
            process(buffer, n);
          } else if (n == 0) {
            // 连接关闭
            close(sockfd);
            break;
          } else {
            if (errno == EAGAIN || errno == EWOULDBLOCK) {
              // 数据读完了
              break;
            } else {
              // 真正的错误
              handle_error();
              break;
            }
          }
        }
      }
    }
  }
}

// ET模式优势：
// - 减少epoll_wait触发次数
// - 避免惊群问题
// - 更高的吞吐量
//
// ET模式要求：
// - socket必须设置为非阻塞
// - 必须读尽所有数据
```

**epoll + 线程池：**

```cpp
#include <thread>
#include <queue>
#include <mutex>
#include <condition_variable>

// 高性能网络服务器架构
class NetworkServer {
  int epollfd_;
  std::vector<std::thread> thread_pool_;

  // 任务队列
  std::queue<int> task_queue_;
  std::mutex queue_mutex_;
  std::condition_variable queue_cv_;

public:
  NetworkServer(int num_threads) {
    epollfd_ = epoll_create1(0);

    // 创建工作线程
    for (int i = 0; i < num_threads; i++) {
      thread_pool_.emplace_back(&NetworkServer::WorkerThread, this);
    }
  }

  void Run() {
    struct epoll_event events[MAX_EVENTS];

    while (true) {
      int nfds = epoll_wait(epollfd_, events, MAX_EVENTS, -1);

      for (int i = 0; i < nfds; i++) {
        int sockfd = events[i].data.fd;

        if (events[i].events & EPOLLIN) {
          // 将socket交给工作线程处理
          {
            std::lock_guard<std::mutex> lock(queue_mutex_);
            task_queue_.push(sockfd);
          }
          queue_cv_.notify_one();

          // 从epoll移除（避免重复触发）
          epoll_ctl(epollfd_, EPOLL_CTL_DEL, sockfd, nullptr);
        }
      }
    }
  }

  void WorkerThread() {
    while (true) {
      int sockfd;

      // 1. 从队列获取任务
      {
        std::unique_lock<std::mutex> lock(queue_mutex_);
        queue_cv_.wait(lock, [this] { return !task_queue_.empty(); });

        sockfd = task_queue_.front();
        task_queue_.pop();
      }

      // 2. 读取数据
      char buffer[65536];
      ssize_t n = recv(sockfd, buffer, sizeof(buffer), 0);

      if (n > 0) {
        // 3. 处理数据（CPU密集）
        process(buffer, n);

        // 4. 重新加入epoll
        struct epoll_event ev;
        ev.events = EPOLLIN | EPOLLET;
        ev.data.fd = sockfd;
        epoll_ctl(epollfd_, EPOLL_CTL_ADD, sockfd, &ev);
      } else {
        close(sockfd);
      }
    }
  }
};

// 架构优势：
// - 主线程：epoll_wait（IO密集）
// - 工作线程：处理数据（CPU密集）
// - 充分利用多核CPU
```

### 4.5 HTTP读取优化

**HTTP长连接（Keep-Alive）：**

```cpp
// 短连接：每个请求建立新连接
void HTTPShortConnection(const char* host, int port) {
  for (int i = 0; i < 100; i++) {
    // 1. 建立连接（3次握手，延迟高）
    int sockfd = connect_to_server(host, port);

    // 2. 发送请求
    send_http_request(sockfd, "/api/data");

    // 3. 接收响应
    recv_http_response(sockfd);

    // 4. 关闭连接（4次挥手）
    close(sockfd);
  }
  // 100次请求 = 700次RTT（3+4握手挥手）
}

// 长连接：复用连接
void HTTPKeepAlive(const char* host, int port) {
  // 1. 建立一次连接
  int sockfd = connect_to_server(host, port);

  for (int i = 0; i < 100; i++) {
    // 2. 发送请求（带Keep-Alive头）
    send_http_request_keepalive(sockfd, "/api/data");

    // 3. 接收响应
    recv_http_response(sockfd);
  }

  // 4. 最后关闭
  close(sockfd);
  // 100次请求 = 7次RTT（只有首次握手挥手）
  // 性能提升：100倍RTT节省
}

// 实现Keep-Alive
void send_http_request_keepalive(int sockfd, const char* path) {
  std::string request =
    "GET " + std::string(path) + " HTTP/1.1\r\n"
    "Host: example.com\r\n"
    "Connection: keep-alive\r\n"  // 关键：保持连接
    "\r\n";

  send(sockfd, request.c_str(), request.size(), 0);
}
```

**HTTP/2多路复用：**

```cpp
// HTTP/1.1：HOL阻塞问题
// 一个连接同时只能有一个请求/响应
void HTTP1_Pipeline() {
  int sockfd = connect_to_server("example.com", 80);

  // 请求1
  send_request(sockfd, "/api/slow");    // 慢请求（10秒）
  recv_response(sockfd);                 // 阻塞10秒

  // 请求2（被阻塞）
  send_request(sockfd, "/api/fast");    // 快请求（10ms）
  recv_response(sockfd);                 // 必须等请求1完成

  close(sockfd);
  // 总时间：10秒（即使请求2很快）
}

// HTTP/2：多路复用
void HTTP2_Multiplexing() {
  // HTTP/2在一个TCP连接上复用多个流（stream）
  int sockfd = connect_to_http2_server("example.com", 443);

  // 并发发送多个请求（不同stream ID）
  send_http2_request(sockfd, stream_id=1, "/api/slow");
  send_http2_request(sockfd, stream_id=3, "/api/fast");

  // 并发接收（先到先得）
  while (has_pending_requests()) {
    Http2Frame frame = recv_http2_frame(sockfd);

    if (frame.stream_id == 3) {
      // 快请求先返回（10ms）
      process_response(frame);
    } else if (frame.stream_id == 1) {
      // 慢请求后返回（10秒）
      process_response(frame);
    }
  }

  close(sockfd);
  // 快请求不被阻塞，用户体验提升！
}
```

### 4.6 UDP优化

**批量接收UDP：**

```cpp
#include <sys/socket.h>

// recvmmsg：批量接收多个UDP包
void BatchRecvUDP(int sockfd) {
  const int BATCH_SIZE = 64;

  // 准备多个消息缓冲区
  struct mmsghdr msgs[BATCH_SIZE];
  struct iovec iovecs[BATCH_SIZE];
  char buffers[BATCH_SIZE][2048];

  memset(msgs, 0, sizeof(msgs));

  for (int i = 0; i < BATCH_SIZE; i++) {
    iovecs[i].iov_base = buffers[i];
    iovecs[i].iov_len = sizeof(buffers[i]);

    msgs[i].msg_hdr.msg_iov = &iovecs[i];
    msgs[i].msg_hdr.msg_iovlen = 1;
  }

  // 一次系统调用接收多个UDP包
  int nrecv = recvmmsg(sockfd, msgs, BATCH_SIZE, 0, nullptr);

  // 处理接收到的包
  for (int i = 0; i < nrecv; i++) {
    process_udp_packet(buffers[i], msgs[i].msg_len);
  }

  // 性能：比逐个recv快10-20倍
}

// 传统方式对比
void TraditionalRecvUDP(int sockfd) {
  char buffer[2048];

  for (int i = 0; i < 64; i++) {
    ssize_t n = recvfrom(sockfd, buffer, sizeof(buffer), 0, nullptr, nullptr);
    if (n > 0) {
      process_udp_packet(buffer, n);
    }
  }
  // 64次系统调用
}

// 性能测试（接收100万个UDP包）：
// Traditional recvfrom: 5000 ms
// Batch recvmmsg:        400 ms（12.5倍提升）
```

**UDP零拷贝（GSO/GRO）：**

```bash
# 启用通用接收卸载（GRO）
ethtool -K eth0 gro on

# 启用通用分段卸载（GSO）
ethtool -K eth0 gso on

# 效果：
# - 硬件聚合多个小包
# - 减少协议栈处理开销
# - 性能提升：20-40%
```

### 4.7 读取缓冲区管理

**环形缓冲区（Ring Buffer）：**

```cpp
// 高效的网络读取缓冲区
class RingBuffer {
  char* buffer_;
  size_t capacity_;
  size_t read_pos_;
  size_t write_pos_;
  size_t size_;

public:
  RingBuffer(size_t capacity)
    : capacity_(capacity), read_pos_(0), write_pos_(0), size_(0) {
    buffer_ = new char[capacity];
  }

  ~RingBuffer() {
    delete[] buffer_;
  }

  // 接收数据到缓冲区
  ssize_t RecvToBuffer(int sockfd) {
    // 计算可写空间
    size_t writable = capacity_ - size_;
    if (writable == 0) {
      return 0;  // 缓冲区满
    }

    // 处理环形回绕
    size_t contiguous = capacity_ - write_pos_;
    size_t to_recv = std::min(writable, contiguous);

    // 直接接收到缓冲区
    ssize_t n = recv(sockfd, buffer_ + write_pos_, to_recv, 0);

    if (n > 0) {
      write_pos_ = (write_pos_ + n) % capacity_;
      size_ += n;
    }

    return n;
  }

  // 从缓冲区读取数据
  size_t Read(char* dest, size_t len) {
    size_t to_read = std::min(len, size_);
    if (to_read == 0) {
      return 0;
    }

    // 处理环形回绕
    size_t first_part = std::min(to_read, capacity_ - read_pos_);
    memcpy(dest, buffer_ + read_pos_, first_part);

    if (to_read > first_part) {
      memcpy(dest + first_part, buffer_, to_read - first_part);
    }

    read_pos_ = (read_pos_ + to_read) % capacity_;
    size_ -= to_read;

    return to_read;
  }

  // 零拷贝：获取连续可读区域指针
  std::pair<const char*, size_t> GetReadableRange() {
    if (size_ == 0) {
      return {nullptr, 0};
    }

    size_t readable = std::min(size_, capacity_ - read_pos_);
    return {buffer_ + read_pos_, readable};
  }

  void Consume(size_t n) {
    read_pos_ = (read_pos_ + n) % capacity_;
    size_ -= n;
  }
};

// 使用示例
void UseRingBuffer(int sockfd) {
  RingBuffer buffer(1024 * 1024);  // 1MB环形缓冲区

  while (true) {
    // 1. 接收数据到缓冲区
    buffer.RecvToBuffer(sockfd);

    // 2. 处理缓冲区中的数据（零拷贝）
    auto [data, len] = buffer.GetReadableRange();
    if (len > 0) {
      size_t processed = process_data(data, len);
      buffer.Consume(processed);
    }
  }
}
```

---

## 5. 零拷贝技术

### 5.1 sendfile

```cpp
#include <sys/sendfile.h>

// 零拷贝发送文件
int send_file(int sockfd, const char *filename) {
    int fd = open(filename, O_RDONLY);
    struct stat st;
    fstat(fd, &st);
    
    // 直接从文件发送到socket，无用户态拷贝
    off_t offset = 0;
    ssize_t sent = sendfile(sockfd, fd, &offset, st.st_size);
    
    close(fd);
    return sent;
}
```

**sendfile数据路径：**

```
传统方式：4次拷贝，4次上下文切换

┌─────┐     ┌──────┐     ┌──────┐     ┌─────┐
│磁盘│ →DMA→│内核  │ →CPU→│用户  │ →CPU→│内核 │ →DMA→ 网卡
│     │     │Page  │     │buffer│     │Socket│
│     │     │Cache │     │      │     │Buffer│
└─────┘     └──────┘     └──────┘     └─────┘

sendfile：2次拷贝，2次上下文切换

┌─────┐     ┌──────┐     ┌──────┐
│磁盘│ →DMA→│内核  │ →DMA→│网卡  │
│     │     │Page  │     │      │
│     │     │Cache │     │      │
└─────┘     └──────┘     └──────┘

性能提升：减少50%的拷贝，减少50%的上下文切换
```

**sendfile with DMA scatter-gather（更先进）：**

```
┌─────┐     ┌──────┐     ┌──────┐
│磁盘│ →DMA→│内核  │     │网卡  │
│     │     │Page  │─描述符→│      │
│     │     │Cache │     │ DMA  │
└─────┘     └──────┘     │gather│
                          └──────┘

零拷贝：仅传递文件描述符和偏移量给网卡
网卡直接从Page Cache读取数据
```

### 5.2 splice

```cpp
// splice在两个fd之间移动数据，无需用户态
ssize_t splice(int fd_in, loff_t *off_in,
               int fd_out, loff_t *off_out,
               size_t len, unsigned int flags);

// 示例：从文件pipe到socket
int pipefd[2];
pipe(pipefd);

// 文件 → pipe
splice(file_fd, NULL, pipefd[1], NULL, len, SPLICE_F_MOVE);

// pipe → socket
splice(pipefd[0], NULL, sock_fd, NULL, len, SPLICE_F_MOVE);
```

**splice vs sendfile：**
- sendfile：文件 → socket（特定场景）
- splice：任意fd之间，更通用
- 都是零拷贝，都需要pipe作为中介

### 5.3 mmap + write

```cpp
// 使用mmap映射文件，然后write到socket
void *addr = mmap(NULL, file_size, PROT_READ, MAP_PRIVATE, file_fd, 0);

// write直接从映射的内存发送
write(sock_fd, addr, file_size);

munmap(addr, file_size);
```

**mmap数据路径：**

```
┌─────┐     ┌──────┐     ┌──────┐
│磁盘│ →DMA→│内核  │     │网卡  │
│     │     │Page  │ →DMA→│      │
│     │     │Cache │     │      │
└─────┘     └──────┘     └──────┘
              ↑ mmap映射（虚拟内存）
              │ write时page fault加载
```

**优点：**
- 用户态可以直接访问文件内容
- 延迟加载（页面错误时才加载）

**缺点：**
- 可能导致页面错误（page fault）
- 不如sendfile高效

### 5.4 MSG_ZEROCOPY

```cpp
// Linux 4.14+支持send/sendmsg零拷贝
char buf[8192];
// ... 填充数据 ...

// 启用零拷贝
int ret = send(sockfd, buf, sizeof(buf), MSG_ZEROCOPY);
if (ret < 0 && errno == ENOBUFS) {
    // 发送队列满，回退到普通拷贝
    send(sockfd, buf, sizeof(buf), 0);
}

// 接收完成通知
struct msghdr msg = {0};
struct cmsghdr *cm;
uint32_t range;

recvmsg(sockfd, &msg, MSG_ERRQUEUE);
cm = CMSG_FIRSTHDR(&msg);
if (cm->cmsg_level == SOL_IP && cm->cmsg_type == IP_RECVERR) {
    // 数据已经被网卡发送，现在可以释放buffer
}
```

**注意事项：**
- 只有大数据包（>10KB）才有收益
- 需要等待完成通知才能复用buffer
- 小数据包反而慢（通知开销）

### 5.5 零拷贝技术对比

| 技术        | 场景          | 拷贝次数 | 上下文切换 | 适用大小    | 内核版本    |
|------------|---------------|---------|-----------|-----------|------------|
| 普通read/write| 通用         | 4次     | 4次       | 任意      | 所有       |
| sendfile   | 文件→socket   | 2次     | 2次       | 任意      | 2.1+       |
| splice     | fd→fd         | 2次     | 2次       | 任意      | 2.6.17+    |
| mmap+write | 文件→socket   | 3次     | 4次       | 大文件    | 所有       |
| MSG_ZEROCOPY| 内存→socket   | 1次     | 2次       | >10KB     | 4.14+      |
| io_uring   | 任意IO        | 0-1次   | 0-1次     | 任意      | 5.1+       |

---


## 扩展：网络数据布局与对齐

### 5.6 网络数据包对齐

**问题：未对齐的网络缓冲区导致额外拷贝**

```cpp
// 网络数据包对齐的重要性

// 不好：未对齐的缓冲区
void UnalignedNetworkBuffer() {
    char buffer[1500];  // 以太网MTU

    // 未对齐的接收
    int sockfd = socket(AF_INET, SOCK_DGRAM, 0);
    recvfrom(sockfd, buffer, sizeof(buffer), 0, nullptr, nullptr);

    // 问题：
    // - 数据可能跨cache line
    // - 无法使用零拷贝
    // - DMA传输效率低
}

// 好：对齐的网络缓冲区
void AlignedNetworkBuffer() {
    // 64字节对齐（cache line）
    void* buffer = nullptr;
    posix_memalign(&buffer, 64, 2048);

    int sockfd = socket(AF_INET, SOCK_DGRAM, 0);
    recvfrom(sockfd, buffer, 2048, 0, nullptr, nullptr);

    // 优势：
    // - 完整cache line访问
    // - 可使用零拷贝技术
    // - DMA友好

    free(buffer);
}
```

**网络缓冲区池设计：**

```cpp
#include <queue>
#include <mutex>

// 预分配对齐的网络缓冲区池
class AlignedBufferPool {
    static constexpr size_t BUFFER_SIZE = 2048;  // 2KB
    static constexpr size_t ALIGNMENT = 64;       // Cache line
    static constexpr size_t POOL_SIZE = 1024;     // 1024个缓冲区

    struct Buffer {
        void* data;
        size_t size;
        bool in_use;
    };

    std::vector<Buffer> buffers_;
    std::queue<Buffer*> free_list_;
    std::mutex mutex_;

public:
    AlignedBufferPool() {
        buffers_.reserve(POOL_SIZE);

        for (size_t i = 0; i < POOL_SIZE; i++) {
            Buffer buf;
            posix_memalign(&buf.data, ALIGNMENT, BUFFER_SIZE);
            buf.size = BUFFER_SIZE;
            buf.in_use = false;

            buffers_.push_back(buf);
            free_list_.push(&buffers_[i]);
        }
    }

    ~AlignedBufferPool() {
        for (auto& buf : buffers_) {
            free(buf.data);
        }
    }

    Buffer* Allocate() {
        std::lock_guard<std::mutex> lock(mutex_);

        if (free_list_.empty()) {
            return nullptr;  // 池耗尽
        }

        Buffer* buf = free_list_.front();
        free_list_.pop();
        buf->in_use = true;

        return buf;
    }

    void Free(Buffer* buf) {
        std::lock_guard<std::mutex> lock(mutex_);

        buf->in_use = false;
        free_list_.push(buf);
    }
};

// 使用示例
void NetworkReceiveWithPool() {
    AlignedBufferPool pool;
    int sockfd = socket(AF_INET, SOCK_DGRAM, 0);

    while (true) {
        auto* buf = pool.Allocate();
        if (!buf) {
            std::cerr << "Buffer pool exhausted!\n";
            break;
        }

        ssize_t n = recvfrom(sockfd, buf->data, buf->size, 0, nullptr, nullptr);

        if (n > 0) {
            ProcessPacket(buf->data, n);
        }

        pool.Free(buf);
    }

    close(sockfd);
}
```

### 5.7 协议头部布局优化

**以太网帧对齐：**

```cpp
// 以太网帧结构（未对齐版本）
struct EthernetFrame_Bad {
    uint8_t dest_mac[6];      // 6 bytes
    uint8_t src_mac[6];       // 6 bytes
    uint16_t ethertype;       // 2 bytes
    uint8_t payload[];        // 变长
};  // 14 bytes头部（未对齐到4字节）

// 问题：IP头部从字节14开始（未对齐）
// IP头部中的32位字段会跨字对齐边界

// 以太网帧结构（对齐版本）
struct EthernetFrame_Good {
    uint8_t dest_mac[6];      // 6 bytes
    uint8_t src_mac[6];       // 6 bytes
    uint16_t ethertype;       // 2 bytes
    uint8_t padding[2];       // 2 bytes填充（总共16字节）
    uint8_t payload[];        // 从偏移16开始（4字节对齐）
};

// 优势：IP头部从16字节开始（4字节对齐）
// 访问IP头部的32位字段无需非对齐访问
```

**自定义协议头部设计：**

```cpp
// 不好的协议头部设计
struct MessageHeader_Bad {
    uint8_t version;          // 1 byte
    uint32_t msg_id;          // 4 bytes
    uint8_t msg_type;         // 1 byte
    uint64_t timestamp;       // 8 bytes
    uint16_t payload_length;  // 2 bytes
};  // 16 bytes（但有padding）

// 实际内存布局：
// [version:1][pad:3][msg_id:4][msg_type:1][pad:7][timestamp:8][payload_length:2][pad:6]
// 总共：32 bytes（浪费16字节！）

// 好的协议头部设计：按大小降序排列
struct MessageHeader_Good {
    uint64_t timestamp;       // 8 bytes, offset 0
    uint32_t msg_id;          // 4 bytes, offset 8
    uint16_t payload_length;  // 2 bytes, offset 12
    uint8_t version;          // 1 byte, offset 14
    uint8_t msg_type;         // 1 byte, offset 15
};  // 16 bytes（无padding！）

// 进一步优化：使用位域
struct MessageHeader_Optimized {
    uint64_t timestamp;       // 8 bytes
    uint32_t msg_id;          // 4 bytes
    uint16_t payload_length;  // 2 bytes
    uint8_t version : 4;      // 4 bits
    uint8_t msg_type : 4;     // 4 bits
    uint8_t reserved;         // 1 byte
};  // 16 bytes

// 性能对比
void BenchmarkHeaderParsing() {
    constexpr int N = 10000000;

    // Bad layout
    std::vector<MessageHeader_Bad> bad_headers(N);
    auto start = std::chrono::high_resolution_clock::now();
    uint64_t sum = 0;
    for (const auto& h : bad_headers) {
        sum += h.timestamp + h.msg_id;
    }
    auto end = std::chrono::high_resolution_clock::now();
    auto bad_time = std::chrono::duration_cast<std::chrono::milliseconds>(
        end - start).count();

    // Good layout
    std::vector<MessageHeader_Good> good_headers(N);
    start = std::chrono::high_resolution_clock::now();
    sum = 0;
    for (const auto& h : good_headers) {
        sum += h.timestamp + h.msg_id;
    }
    end = std::chrono::high_resolution_clock::now();
    auto good_time = std::chrono::duration_cast<std::chrono::milliseconds>(
        end - start).count();

    std::cout << "Bad layout:  " << bad_time << " ms, "
              << "Memory: " << N * sizeof(MessageHeader_Bad) / 1024 / 1024 << " MB\n";
    std::cout << "Good layout: " << good_time << " ms, "
              << "Memory: " << N * sizeof(MessageHeader_Good) / 1024 / 1024 << " MB\n";
    std::cout << "Memory saved: " << (sizeof(MessageHeader_Bad) - sizeof(MessageHeader_Good))
              << " bytes per message\n";
}

// 输出示例：
// Bad layout:  450 ms, Memory: 305 MB
// Good layout: 280 ms, Memory: 152 MB
// Memory saved: 16 bytes per message
// 性能提升：1.6倍
// 内存节省：50%
```

### 5.8 零拷贝友好的数据布局

**iovec结构对齐：**

```cpp
#include <sys/uio.h>

// 零拷贝发送：scatter-gather IO
void ZeroCopySend(int sockfd) {
    // 消息由多部分组成：头部 + 负载
    struct MessageHeader {
        uint32_t length;
        uint32_t type;
    } __attribute__((aligned(8)));  // 8字节对齐

    MessageHeader header = {1024, 1};

    // 对齐的负载缓冲区
    void* payload = nullptr;
    posix_memalign(&payload, 4096, 1024);
    memset(payload, 'A', 1024);

    // 使用writev零拷贝发送
    struct iovec iov[2];
    iov[0].iov_base = &header;
    iov[0].iov_len = sizeof(header);
    iov[1].iov_base = payload;
    iov[1].iov_len = 1024;

    // 一次系统调用发送两个缓冲区（无内存拷贝）
    ssize_t sent = writev(sockfd, iov, 2);

    std::cout << "Sent " << sent << " bytes in one syscall\n";

    free(payload);
}

// 零拷贝接收：scatter IO
void ZeroCopyRecv(int sockfd) {
    // 接收到不同的对齐缓冲区
    struct MessageHeader {
        uint32_t length;
        uint32_t type;
    } __attribute__((aligned(8)));

    MessageHeader header;

    void* payload = nullptr;
    posix_memalign(&payload, 4096, 4096);

    struct iovec iov[2];
    iov[0].iov_base = &header;
    iov[0].iov_len = sizeof(header);
    iov[1].iov_base = payload;
    iov[1].iov_len = 4096;

    // 一次系统调用接收到多个缓冲区
    ssize_t received = readv(sockfd, iov, 2);

    std::cout << "Received " << received << " bytes\n";
    std::cout << "Header: length=" << header.length
              << ", type=" << header.type << "\n";

    free(payload);
}
```

**sendfile/splice零拷贝数据布局：**

```cpp
#include <sys/sendfile.h>
#include <fcntl.h>

// 文件到socket的零拷贝传输
void ZeroCopyFileTransfer(int sockfd, const char* filename) {
    int file_fd = open(filename, O_RDONLY);
    if (file_fd < 0) {
        perror("open");
        return;
    }

    // 获取文件大小
    struct stat st;
    fstat(file_fd, &st);
    off_t file_size = st.st_size;

    // 零拷贝传输：文件 → 内核缓冲区 → 网卡（无用户态拷贝）
    off_t offset = 0;
    ssize_t sent = sendfile(sockfd, file_fd, &offset, file_size);

    std::cout << "Sent " << sent << " bytes with zero-copy\n";

    close(file_fd);
}

// 性能对比：传统方式 vs 零拷贝
void CompareFileTransfer() {
    const char* filename = "large_file.dat";
    int sockfd = socket(AF_INET, SOCK_STREAM, 0);

    // 方法1：传统read/write（2次拷贝）
    auto start = std::chrono::high_resolution_clock::now();
    {
        int fd = open(filename, O_RDONLY);
        char buffer[8192];
        ssize_t n;

        while ((n = read(fd, buffer, sizeof(buffer))) > 0) {
            write(sockfd, buffer, n);
        }

        close(fd);
    }
    auto end = std::chrono::high_resolution_clock::now();
    auto traditional_time = std::chrono::duration_cast<std::chrono::milliseconds>(
        end - start).count();

    // 方法2：零拷贝sendfile（0次用户态拷贝）
    start = std::chrono::high_resolution_clock::now();
    ZeroCopyFileTransfer(sockfd, filename);
    end = std::chrono::high_resolution_clock::now();
    auto zerocopy_time = std::chrono::duration_cast<std::chrono::milliseconds>(
        end - start).count();

    std::cout << "Traditional: " << traditional_time << " ms\n";
    std::cout << "Zero-copy:   " << zerocopy_time << " ms\n";
    std::cout << "Speedup:     " << (double)traditional_time / zerocopy_time << "x\n";

    close(sockfd);
}

// 输出示例（传输1GB文件）：
// Traditional: 8500 ms
// Zero-copy:   1200 ms
// Speedup:     7.1x
```

### 5.9 消息批处理与合并

**批量发送优化：**

```cpp
// 不好：逐条发送小消息
void SendSmallMessages_Bad(int sockfd, const std::vector<std::string>& messages) {
    for (const auto& msg : messages) {
        send(sockfd, msg.data(), msg.size(), 0);  // 1000次系统调用
    }
    // 问题：
    // - 大量系统调用开销
    // - TCP小包问题（Nagle算法）
    // - 网络效率低
}

// 好：批量合并发送
void SendSmallMessages_Good(int sockfd, const std::vector<std::string>& messages) {
    // 合并到单个缓冲区
    std::string buffer;
    buffer.reserve(messages.size() * 100);  // 预分配

    for (const auto& msg : messages) {
        // 添加长度前缀
        uint32_t len = msg.size();
        buffer.append(reinterpret_cast<const char*>(&len), sizeof(len));
        buffer.append(msg);
    }

    // 一次发送
    send(sockfd, buffer.data(), buffer.size(), 0);

    // 优势：
    // - 1次系统调用（vs 1000次）
    // - 减少TCP头部开销
    // - 提高网络利用率
}

// 更好：使用writev批量发送（无需额外拷贝）
void SendSmallMessages_Best(int sockfd, const std::vector<std::string>& messages) {
    std::vector<struct iovec> iovs;
    iovs.reserve(messages.size());

    for (const auto& msg : messages) {
        iovs.push_back({const_cast<char*>(msg.data()), msg.size()});
    }

    // 一次系统调用，无内存拷贝
    struct msghdr msghdr = {};
    msghdr.msg_iov = iovs.data();
    msghdr.msg_iovlen = iovs.size();

    sendmsg(sockfd, &msghdr, 0);
}

// 性能对比
void BenchmarkMessageBatching() {
    int sockfd = socket(AF_INET, SOCK_STREAM, 0);
    // ... connect ...

    // 生成1000条小消息
    std::vector<std::string> messages;
    for (int i = 0; i < 1000; i++) {
        messages.push_back("Message " + std::to_string(i));
    }

    // 方法1：逐条发送
    auto start = std::chrono::high_resolution_clock::now();
    SendSmallMessages_Bad(sockfd, messages);
    auto end = std::chrono::high_resolution_clock::now();
    auto bad_time = std::chrono::duration_cast<std::chrono::microseconds>(
        end - start).count();

    // 方法2：批量合并
    start = std::chrono::high_resolution_clock::now();
    SendSmallMessages_Good(sockfd, messages);
    end = std::chrono::high_resolution_clock::now();
    auto good_time = std::chrono::duration_cast<std::chrono::microseconds>(
        end - start).count();

    // 方法3：writev批量
    start = std::chrono::high_resolution_clock::now();
    SendSmallMessages_Best(sockfd, messages);
    end = std::chrono::high_resolution_clock::now();
    auto best_time = std::chrono::duration_cast<std::chrono::microseconds>(
        end - start).count();

    std::cout << "Individual send: " << bad_time << " μs\n";
    std::cout << "Batched send:    " << good_time << " μs\n";
    std::cout << "Writev send:     " << best_time << " μs\n";

    close(sockfd);
}

// 输出示例：
// Individual send: 15000 μs
// Batched send:    800 μs
// Writev send:     500 μs
// 性能提升：30倍（writev）
```

### 5.10 Protocol Buffers优化

**ProtoBuf内存布局优化：**

```protobuf
// 不好的Proto定义：字段顺序混乱
message UserInfo_Bad {
  optional int32 age = 1;           // 4 bytes
  optional string name = 2;         // 变长
  optional int64 user_id = 3;       // 8 bytes
  optional bool is_active = 4;      // 1 byte
  optional double balance = 5;      // 8 bytes
  optional string email = 6;        // 变长
  optional int32 level = 7;         // 4 bytes
}

// 问题：
// - 序列化大小：字段编号也占空间
// - 解析性能：字段乱序导致跳跃访问
// - 内存布局：padding多

// 好的Proto定义：按访问频率和大小排序
message UserInfo_Good {
  // 1. 固定大小字段优先
  optional int64 user_id = 1;       // 最常访问，最大
  optional double balance = 2;      // 常访问
  optional int32 age = 3;           // 常访问
  optional int32 level = 4;         // 常访问
  optional bool is_active = 5;      // 常访问

  // 2. 变长字段放后面
  optional string name = 6;
  optional string email = 7;
}

// 优势：
// - 热字段连续存储（cache友好）
// - 减少padding
// - 解析性能提升
```

**ProtoBuf零拷贝优化：**

```cpp
#include <google/protobuf/message.h>
#include <google/protobuf/io/zero_copy_stream.h>
#include <google/protobuf/io/coded_stream.h>

// 传统方式：需要拷贝
void SerializeProto_Traditional(const MyMessage& msg, int sockfd) {
    std::string serialized;
    msg.SerializeToString(&serialized);  // 拷贝1

    send(sockfd, serialized.data(), serialized.size(), 0);  // 拷贝2
    // 总共2次拷贝
}

// 零拷贝方式：直接序列化到网络缓冲区
class SocketOutputStream : public google::protobuf::io::ZeroCopyOutputStream {
    int sockfd_;
    std::vector<char> buffer_;
    size_t position_;

public:
    SocketOutputStream(int sockfd, size_t buffer_size = 8192)
        : sockfd_(sockfd), buffer_(buffer_size), position_(0) {}

    bool Next(void** data, int* size) override {
        if (position_ >= buffer_.size()) {
            // 缓冲区满，发送
            Flush();
        }

        *data = buffer_.data() + position_;
        *size = buffer_.size() - position_;
        position_ = buffer_.size();

        return true;
    }

    void BackUp(int count) override {
        position_ -= count;
    }

    int64_t ByteCount() const override {
        return position_;
    }

    void Flush() {
        if (position_ > 0) {
            send(sockfd_, buffer_.data(), position_, 0);
            position_ = 0;
        }
    }
};

void SerializeProto_ZeroCopy(const MyMessage& msg, int sockfd) {
    SocketOutputStream stream(sockfd);
    google::protobuf::io::CodedOutputStream coded_stream(&stream);

    // 直接序列化到socket缓冲区（无额外拷贝）
    msg.SerializeToCodedStream(&coded_stream);
    stream.Flush();

    // 总共0次额外拷贝！
}
```

### 5.11 网络字节序优化

**字节序转换与对齐：**

```cpp
#include <arpa/inet.h>
#include <endian.h>

// 网络字节序（大端）vs 主机字节序（可能小端）

// 不好：逐字段转换
struct NetworkPacket_Bad {
    uint32_t id;
    uint16_t type;
    uint16_t length;
    uint64_t timestamp;

    void ToNetworkOrder() {
        id = htonl(id);
        type = htons(type);
        length = htons(length);
        timestamp = htobe64(timestamp);
    }

    void ToHostOrder() {
        id = ntohl(id);
        type = ntohs(type);
        length = ntohs(length);
        timestamp = be64toh(timestamp);
    }
};

// 好：批量转换（利用SIMD）
#include <x86intrin.h>

struct NetworkPacket_Good {
    uint32_t id;
    uint16_t type;
    uint16_t length;
    uint64_t timestamp;

    void ToNetworkOrder() {
        // 使用SIMD字节交换
        #if __BYTE_ORDER__ == __ORDER_LITTLE_ENDIAN__
        // 小端 → 大端
        __m128i data = _mm_loadu_si128((__m128i*)this);

        // 字节反转（简化示例）
        __m128i shuffled = _mm_shuffle_epi8(data,
            _mm_set_epi8(0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15));

        _mm_storeu_si128((__m128i*)this, shuffled);
        #endif
    }
};

// 性能对比
void BenchmarkByteOrder() {
    constexpr int N = 1000000;

    // 方法1：逐字段转换
    std::vector<NetworkPacket_Bad> bad_packets(N);
    auto start = std::chrono::high_resolution_clock::now();

    for (auto& pkt : bad_packets) {
        pkt.ToNetworkOrder();
    }

    auto end = std::chrono::high_resolution_clock::now();
    auto bad_time = std::chrono::duration_cast<std::chrono::milliseconds>(
        end - start).count();

    // 方法2：批量SIMD转换
    std::vector<NetworkPacket_Good> good_packets(N);
    start = std::chrono::high_resolution_clock::now();

    for (auto& pkt : good_packets) {
        pkt.ToNetworkOrder();
    }

    end = std::chrono::high_resolution_clock::now();
    auto good_time = std::chrono::duration_cast<std::chrono::milliseconds>(
        end - start).count();

    std::cout << "Field-by-field: " << bad_time << " ms\n";
    std::cout << "SIMD batch:     " << good_time << " ms\n";
    std::cout << "Speedup:        " << (double)bad_time / good_time << "x\n";
}

// 输出：
// Field-by-field: 45 ms
// SIMD batch:     12 ms
// Speedup:        3.75x
```

### 5.12 Ring Buffer网络应用

**无锁环形缓冲区：**

```cpp
#include <atomic>
#include <vector>

// 单生产者单消费者环形缓冲区（网络数据缓冲）
template <typename T>
class SPSCRingBuffer {
    std::vector<T> buffer_;
    size_t capacity_;

    alignas(64) std::atomic<size_t> write_pos_{0};  // Cache line分离
    alignas(64) std::atomic<size_t> read_pos_{0};   // 避免false sharing

public:
    SPSCRingBuffer(size_t capacity)
        : buffer_(capacity), capacity_(capacity) {}

    // 生产者：写入网络数据包
    bool Push(const T& item) {
        size_t current_write = write_pos_.load(std::memory_order_relaxed);
        size_t next_write = (current_write + 1) % capacity_;

        // 检查缓冲区是否满
        if (next_write == read_pos_.load(std::memory_order_acquire)) {
            return false;  // 满了
        }

        buffer_[current_write] = item;
        write_pos_.store(next_write, std::memory_order_release);

        return true;
    }

    // 消费者：读取网络数据包
    bool Pop(T& item) {
        size_t current_read = read_pos_.load(std::memory_order_relaxed);

        // 检查缓冲区是否空
        if (current_read == write_pos_.load(std::memory_order_acquire)) {
            return false;  // 空了
        }

        item = buffer_[current_read];
        read_pos_.store((current_read + 1) % capacity_, std::memory_order_release);

        return true;
    }
};

// 网络接收线程 + 处理线程示例
void NetworkReceiveThread(SPSCRingBuffer<std::vector<char>>& ring_buffer) {
    int sockfd = socket(AF_INET, SOCK_DGRAM, 0);
    // ... bind ...

    while (true) {
        std::vector<char> packet(2048);
        ssize_t n = recvfrom(sockfd, packet.data(), packet.size(),
                             0, nullptr, nullptr);

        if (n > 0) {
            packet.resize(n);

            // 无锁写入环形缓冲区
            while (!ring_buffer.Push(packet)) {
                std::this_thread::yield();  // 缓冲区满，等待
            }
        }
    }
}

void NetworkProcessThread(SPSCRingBuffer<std::vector<char>>& ring_buffer) {
    while (true) {
        std::vector<char> packet;

        if (ring_buffer.Pop(packet)) {
            // 处理数据包
            ProcessPacket(packet);
        } else {
            std::this_thread::sleep_for(std::chrono::microseconds(10));
        }
    }
}

// 性能优势：
// - 无锁设计：高并发
// - Cache line对齐：避免false sharing
// - 解耦接收和处理：提高吞吐量
```

---

## 网络数据布局优化检查清单

### ✅ 缓冲区对齐检查

- [ ] 网络缓冲区64字节对齐（cache line）
- [ ] 使用缓冲区池预分配对齐内存
- [ ] DMA传输使用页对齐缓冲区
- [ ] iovec结构对齐

### ✅ 协议设计检查

- [ ] 协议头部字段按大小降序排列
- [ ] 避免不必要的padding
- [ ] 关键字段对齐到自然边界
- [ ] 考虑网络字节序转换开销

### ✅ 零拷贝检查

- [ ] 使用writev/readv scatter-gather IO
- [ ] 文件传输使用sendfile/splice
- [ ] 避免不必要的内存拷贝
- [ ] 直接序列化到网络缓冲区

### ✅ 批处理检查

- [ ] 小消息批量合并发送
- [ ] 使用消息队列减少系统调用
- [ ] 考虑Nagle算法影响
- [ ] 权衡延迟与吞吐量

### ✅ 性能验证

- [ ] 测量系统调用次数
- [ ] 监控网络带宽利用率
- [ ] 测量CPU使用率
- [ ] 验证零拷贝生效

---


## 6. TCP协议栈优化

### 6.1 TCP Socket选项

```cpp
int sockfd = socket(AF_INET, SOCK_STREAM, 0);

// 1. 禁用Nagle算法（降低延迟）
int flag = 1;
setsockopt(sockfd, IPPROTO_TCP, TCP_NODELAY, &flag, sizeof(flag));

// 2. 启用TCP_CORK（批量发送，提高吞吐量）
int cork = 1;
setsockopt(sockfd, IPPROTO_TCP, TCP_CORK, &cork, sizeof(cork));
// ... 多次write ...
cork = 0;
setsockopt(sockfd, IPPROTO_TCP, TCP_CORK, &cork, sizeof(cork)); // 刷新

// 3. 调整发送/接收缓冲区
int sndbuf = 256 * 1024;  // 256KB
int rcvbuf = 256 * 1024;
setsockopt(sockfd, SOL_SOCKET, SO_SNDBUF, &sndbuf, sizeof(sndbuf));
setsockopt(sockfd, SOL_SOCKET, SO_RCVBUF, &rcvbuf, sizeof(rcvbuf));

// 4. 启用TCP_QUICKACK（快速确认）
int quickack = 1;
setsockopt(sockfd, IPPROTO_TCP, TCP_QUICKACK, &quickack, sizeof(quickack));

// 5. 调整TCP_KEEPALIVE
int keepalive = 1;
int keepidle = 60;      // 60秒后开始keepalive
int keepintvl = 5;      // 每5秒发送一次
int keepcnt = 3;        // 3次失败后断开
setsockopt(sockfd, SOL_SOCKET, SO_KEEPALIVE, &keepalive, sizeof(keepalive));
setsockopt(sockfd, IPPROTO_TCP, TCP_KEEPIDLE, &keepidle, sizeof(keepidle));
setsockopt(sockfd, IPPROTO_TCP, TCP_KEEPINTVL, &keepintvl, sizeof(keepintvl));
setsockopt(sockfd, IPPROTO_TCP, TCP_KEEPCNT, &keepcnt, sizeof(keepcnt));
```

**Nagle算法 vs TCP_NODELAY：**

```
Nagle算法（默认启用）：
- 目的：合并小包，减少网络拥塞
- 规则：如果有未确认数据，缓冲小包直到ACK或达到MSS
- 适用：批量传输，延迟不敏感

TCP_NODELAY（禁用Nagle）：
- 立即发送，不等待
- 适用：低延迟应用（游戏、交易系统）
- 代价：可能增加网络包数量

选择：
延迟敏感 → TCP_NODELAY
吞吐敏感 → 保持Nagle算法
```

### 6.2 系统级TCP参数调优

```bash
# /etc/sysctl.conf 或 sysctl命令

# TCP窗口大小（影响带宽）
net.core.rmem_max = 16777216           # 最大接收缓冲16MB
net.core.wmem_max = 16777216           # 最大发送缓冲16MB
net.ipv4.tcp_rmem = 4096 87380 16777216  # min default max
net.ipv4.tcp_wmem = 4096 65536 16777216

# 启用TCP窗口扩展（支持>64KB窗口）
net.ipv4.tcp_window_scaling = 1

# TCP拥塞控制算法
net.ipv4.tcp_congestion_control = bbr   # Google BBR（推荐）
# 其他选项: cubic(默认), reno, vegas

# SYN队列和Accept队列
net.core.netdev_max_backlog = 5000     # 网卡接收队列
net.core.somaxconn = 1024              # listen backlog最大值
net.ipv4.tcp_max_syn_backlog = 2048    # SYN队列大小

# TIME_WAIT优化
net.ipv4.tcp_tw_reuse = 1              # 复用TIME_WAIT连接
net.ipv4.tcp_fin_timeout = 30          # FIN_WAIT_2超时时间

# TCP快速打开（TFO）
net.ipv4.tcp_fastopen = 3              # 1=客户端 2=服务端 3=都启用

# 减少TCP重传
net.ipv4.tcp_retries2 = 5              # 默认15，减少可更快失败

# TCP keepalive
net.ipv4.tcp_keepalive_time = 600      # 10分钟
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 3

# 应用设置
sysctl -p  # 加载配置
```

### 6.3 拥塞控制算法

**BBR（Bottleneck Bandwidth and RTT）：**

```
传统拥塞控制（如CUBIC）：
- 基于丢包判断拥塞
- 慢启动、拥塞避免、快速重传
- 在高延迟或丢包网络表现不佳

Google BBR：
- 基于带宽和RTT建模
- 不依赖丢包
- 高延迟网络性能提升2-25倍

启用BBR：
modprobe tcp_bbr
echo "tcp_bbr" >> /etc/modules-load.d/bbr.conf
sysctl -w net.ipv4.tcp_congestion_control=bbr
```

---

## 7. 高性能网络编程

### 7.1 Reactor模式

```cpp
// Reactor模式：事件驱动，单线程处理多个连接
class Reactor {
    int epoll_fd;
    std::map<int, std::function<void()>> handlers;
    
public:
    Reactor() {
        epoll_fd = epoll_create1(0);
    }
    
    void register_handler(int fd, uint32_t events, std::function<void()> handler) {
        handlers[fd] = handler;
        struct epoll_event ev;
        ev.events = events;
        ev.data.fd = fd;
        epoll_ctl(epoll_fd, EPOLL_CTL_ADD, fd, &ev);
    }
    
    void run() {
        struct epoll_event events[MAX_EVENTS];
        while (true) {
            int n = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
            for (int i = 0; i < n; i++) {
                int fd = events[i].data.fd;
                if (handlers.count(fd)) {
                    handlers[fd]();  // 调用注册的处理函数
                }
            }
        }
    }
};

// 使用示例
Reactor reactor;

// 注册监听socket
reactor.register_handler(listen_fd, EPOLLIN, [&]() {
    int client_fd = accept(listen_fd, NULL, NULL);
    set_nonblocking(client_fd);
    
    // 注册客户端socket
    reactor.register_handler(client_fd, EPOLLIN, [=]() {
        handle_client(client_fd);
    });
});

reactor.run();
```

### 7.2 Proactor模式（异步IO）

```cpp
// Proactor模式：异步操作完成后通知
class Proactor {
    struct io_uring ring;
    std::map<void*, std::function<void(int)>> completions;
    
public:
    Proactor() {
        io_uring_queue_init(256, &ring, 0);
    }
    
    void async_read(int fd, char *buf, size_t len, 
                    std::function<void(int)> completion) {
        struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
        io_uring_prep_read(sqe, fd, buf, len, 0);
        
        // 保存完成回调
        void *key = buf;
        completions[key] = completion;
        io_uring_sqe_set_data(sqe, key);
        
        io_uring_submit(&ring);
    }
    
    void run() {
        while (true) {
            struct io_uring_cqe *cqe;
            io_uring_wait_cqe(&ring, &cqe);
            
            void *key = io_uring_cqe_get_data(cqe);
            int result = cqe->res;
            
            if (completions.count(key)) {
                completions[key](result);  // 调用完成回调
                completions.erase(key);
            }
            
            io_uring_cqe_seen(&ring, cqe);
        }
    }
};
```

### 7.3 多线程Reactor（Multi-Reactor）

```
主线程（Acceptor）：
┌─────────────────────────────────┐
│ Listen Socket                   │
│ - 接受新连接                     │
│ - 负载均衡分配给Worker线程       │
└─────────────────────────────────┘
         │         │         │
         ↓         ↓         ↓
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│ Worker 1    │ │ Worker 2    │ │ Worker N    │
│ Reactor     │ │ Reactor     │ │ Reactor     │
│ - epoll_wait│ │ - epoll_wait│ │ - epoll_wait│
│ - 处理IO    │ │ - 处理IO    │ │ - 处理IO    │
└─────────────┘ └─────────────┘ └─────────────┘
```

```cpp
class MultiReactor {
    std::vector<std::thread> workers;
    std::vector<Reactor*> reactors;
    std::atomic<int> next_worker{0};
    
public:
    MultiReactor(int num_workers) {
        for (int i = 0; i < num_workers; i++) {
            Reactor *reactor = new Reactor();
            reactors.push_back(reactor);
            
            workers.emplace_back([reactor]() {
                reactor->run();
            });
        }
    }
    
    void accept_new_connection(int client_fd) {
        // 轮询分配给Worker
        int idx = next_worker.fetch_add(1) % reactors.size();
        reactors[idx]->register_handler(client_fd, EPOLLIN, 
            [client_fd]() { handle_client(client_fd); });
    }
};
```

### 7.4 批量IO操作

```cpp
// sendmmsg: 批量发送多个消息
struct mmsghdr msgs[BATCH_SIZE];
struct iovec iovecs[BATCH_SIZE];

for (int i = 0; i < BATCH_SIZE; i++) {
    iovecs[i].iov_base = buffers[i];
    iovecs[i].iov_len = lens[i];
    
    msgs[i].msg_hdr.msg_iov = &iovecs[i];
    msgs[i].msg_hdr.msg_iovlen = 1;
}

// 一次系统调用发送多个消息
int sent = sendmmsg(sockfd, msgs, BATCH_SIZE, 0);

// recvmmsg: 批量接收
int received = recvmmsg(sockfd, msgs, BATCH_SIZE, 0, NULL);
```

**性能收益：**
- 减少系统调用次数：100次send → 1次sendmmsg
- 减少上下文切换：约50-100ns/次 × 100 = 5-10μs
- 提高吞吐量：小包场景提升50-200%

---

## 8. 内核旁路技术

### 8.1 DPDK（Data Plane Development Kit）

**架构：**

```
传统网络栈：
应用 → 系统调用 → 内核协议栈 → 驱动 → 网卡
问题：系统调用、上下文切换、中断处理

DPDK：
应用 ←→ DPDK库 ←→ PMD（轮询模式驱动）←→ 网卡
优势：零拷贝、无中断、用户态、批量处理
```

```c
// DPDK简化示例
#include <rte_eal.h>
#include <rte_ethdev.h>
#include <rte_mbuf.h>

int main(int argc, char *argv[]) {
    // 初始化EAL（Environment Abstraction Layer）
    rte_eal_init(argc, argv);
    
    // 创建内存池
    struct rte_mempool *mbuf_pool = rte_pktmbuf_pool_create(
        "MBUF_POOL", 8192, 256, 0, RTE_MBUF_DEFAULT_BUF_SIZE, rte_socket_id());
    
    // 配置网卡
    uint16_t port_id = 0;
    rte_eth_dev_configure(port_id, 1, 1, &port_conf);
    rte_eth_rx_queue_setup(port_id, 0, 128, rte_eth_dev_socket_id(port_id), NULL, mbuf_pool);
    rte_eth_tx_queue_setup(port_id, 0, 128, rte_eth_dev_socket_id(port_id), NULL);
    rte_eth_dev_start(port_id);
    
    // 接收循环（轮询，无中断）
    struct rte_mbuf *bufs[BURST_SIZE];
    while (1) {
        uint16_t nb_rx = rte_eth_rx_burst(port_id, 0, bufs, BURST_SIZE);
        
        for (int i = 0; i < nb_rx; i++) {
            // 处理数据包
            process_packet(bufs[i]);
        }
        
        // 批量发送
        uint16_t nb_tx = rte_eth_tx_burst(port_id, 0, bufs, nb_rx);
        
        // 释放未发送的包
        for (int i = nb_tx; i < nb_rx; i++) {
            rte_pktmbuf_free(bufs[i]);
        }
    }
}
```

**DPDK关键技术：**

1. **PMD（Poll Mode Driver）**：轮询代替中断
2. **Hugepages**：大页内存减少TLB miss
3. **UIO/VFIO**：用户态直接访问设备
4. **无锁队列**：高效的核心间通信
5. **CPU亲和性**：绑定核心避免迁移

**性能：**
- 吞吐量：14.88Mpps（10GbE线速）
- 延迟：<1μs（vs 内核网络栈 ~50μs）
- CPU利用率：单核处理10Gbps

**代价：**
- 独占CPU核心（100%轮询）
- 绕过内核，应用负责协议栈
- 学习曲线陡峭

### 8.2 XDP（eXpress Data Path）

**更灵活的内核旁路方案**

```c
// XDP程序（eBPF）运行在网卡驱动中
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

SEC("xdp")
int xdp_drop_tcp(struct xdp_md *ctx) {
    void *data_end = (void *)(long)ctx->data_end;
    void *data = (void *)(long)ctx->data;
    
    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end)
        return XDP_PASS;
    
    if (eth->h_proto != htons(ETH_P_IP))
        return XDP_PASS;
    
    struct iphdr *ip = (void *)(eth + 1);
    if ((void *)(ip + 1) > data_end)
        return XDP_PASS;
    
    // 丢弃所有TCP包
    if (ip->protocol == IPPROTO_TCP)
        return XDP_DROP;
    
    return XDP_PASS;
}

char _license[] SEC("license") = "GPL";
```

**XDP动作：**
- **XDP_DROP**：直接丢弃（DDoS防护）
- **XDP_PASS**：传递给内核网络栈
- **XDP_TX**：从同一网卡发回
- **XDP_REDIRECT**：重定向到其他网卡
- **XDP_ABORTED**：错误，丢弃并跟踪

**性能：**
- 24Mpps丢包性能（vs iptables 1Mpps）
- 运行在网卡驱动，比内核网络栈早
- 不需要独占CPU核心

### 8.3 RDMA（Remote Direct Memory Access）

```c
// RDMA单边操作：无需对端CPU参与
struct ibv_context *ctx = ibv_open_device(device);
struct ibv_pd *pd = ibv_alloc_pd(ctx);

// 注册内存区域
struct ibv_mr *mr = ibv_reg_mr(pd, buffer, size, 
                                IBV_ACCESS_LOCAL_WRITE | IBV_ACCESS_REMOTE_WRITE);

// RDMA Write：直接写入远程内存
struct ibv_send_wr wr = {
    .opcode = IBV_WR_RDMA_WRITE,
    .wr.rdma.remote_addr = remote_addr,
    .wr.rdma.rkey = remote_key,
    // ...
};
ibv_post_send(qp, &wr, &bad_wr);
```

**RDMA优势：**
- **零拷贝**：DMA直接访问应用内存
- **零CPU**：单边操作不占用对端CPU
- **低延迟**：<1μs（vs TCP ~50μs）
- **高带宽**：100Gbps+

**适用场景：**
- 高性能计算（HPC）
- 分布式存储（Ceph、GlusterFS）
- 数据库集群
- 机器学习训练

---

## 9. 实战案例

### 9.1 高性能HTTP服务器

```cpp
// 使用epoll + 非阻塞IO + 线程池
class HTTPServer {
    int listen_fd;
    int epoll_fd;
    ThreadPool pool;
    
public:
    HTTPServer(int port, int num_threads) : pool(num_threads) {
        listen_fd = socket(AF_INET, SOCK_STREAM, 0);
        
        // 设置SO_REUSEADDR和SO_REUSEPORT
        int opt = 1;
        setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
        setsockopt(listen_fd, SOL_SOCKET, SO_REUSEPORT, &opt, sizeof(opt));
        
        struct sockaddr_in addr;
        addr.sin_family = AF_INET;
        addr.sin_port = htons(port);
        addr.sin_addr.s_addr = INADDR_ANY;
        
        bind(listen_fd, (struct sockaddr*)&addr, sizeof(addr));
        listen(listen_fd, SOMAXCONN);
        
        epoll_fd = epoll_create1(0);
        
        struct epoll_event ev;
        ev.events = EPOLLIN;
        ev.data.fd = listen_fd;
        epoll_ctl(epoll_fd, EPOLL_CTL_ADD, listen_fd, &ev);
    }
    
    void run() {
        struct epoll_event events[MAX_EVENTS];
        
        while (true) {
            int n = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
            
            for (int i = 0; i < n; i++) {
                if (events[i].data.fd == listen_fd) {
                    accept_connections();
                } else {
                    int client_fd = events[i].data.fd;
                    
                    // 提交到线程池处理
                    pool.submit([this, client_fd]() {
                        handle_http_request(client_fd);
                    });
                }
            }
        }
    }
    
    void accept_connections() {
        while (true) {
            int client_fd = accept(listen_fd, NULL, NULL);
            if (client_fd < 0) {
                if (errno == EAGAIN || errno == EWOULDBLOCK) {
                    break;
                }
                continue;
            }
            
            set_nonblocking(client_fd);
            set_tcp_nodelay(client_fd);
            
            struct epoll_event ev;
            ev.events = EPOLLIN | EPOLLET;  // 边缘触发
            ev.data.fd = client_fd;
            epoll_ctl(epoll_fd, EPOLL_CTL_ADD, client_fd, &ev);
        }
    }
    
    void handle_http_request(int fd) {
        HTTPRequest req = parse_request(fd);
        HTTPResponse resp = process_request(req);
        
        // 使用sendfile发送静态文件
        if (resp.is_static_file) {
            int file_fd = open(resp.file_path.c_str(), O_RDONLY);
            sendfile(fd, file_fd, NULL, resp.file_size);
            close(file_fd);
        } else {
            send(fd, resp.body.data(), resp.body.size(), 0);
        }
        
        close(fd);
    }
};
```

### 9.2 性能优化检查清单

```
网络IO性能优化检查清单：

□ 应用层
  □ 使用连接池，复用TCP连接
  □ 批量处理请求，减少系统调用
  □ 使用二进制协议（Protobuf, MessagePack）
  □ 启用压缩（对大数据）
  □ 实现请求合并（批量API）

□ 系统调用层
  □ 使用epoll/io_uring而非select/poll
  □ 边缘触发模式（EPOLLET）
  □ 非阻塞IO
  □ sendfile/splice用于文件传输
  □ sendmmsg/recvmmsg批量操作

□ Socket选项
  □ TCP_NODELAY（低延迟）或TCP_CORK（高吞吐）
  □ SO_SNDBUF/SO_RCVBUF适当大小
  □ SO_REUSEADDR/SO_REUSEPORT
  □ TCP_QUICKACK

□ 系统参数
  □ net.core.rmem_max/wmem_max
  □ net.ipv4.tcp_rmem/tcp_wmem
  □ net.ipv4.tcp_congestion_control=bbr
  □ net.core.netdev_max_backlog
  □ net.ipv4.tcp_fastopen

□ 架构设计
  □ 异步IO模型
  □ 多线程/多进程Reactor
  □ 无锁数据结构
  □ CPU亲和性绑定

□ 高级技术（根据需求）
  □ DPDK（超高吞吐）
  □ XDP（包过滤/防护）
  □ RDMA（低延迟）
```

---

## 10. 总结

### 关键要点

1. **理解开销**：系统调用、拷贝、上下文切换是主要开销
2. **选择IO模型**：epoll适用大多数场景，io_uring是未来
3. **零拷贝技术**：sendfile、splice、MSG_ZEROCOPY
4. **TCP优化**：合理配置socket选项和系统参数
5. **批量处理**：减少系统调用次数
6. **内核旁路**：极致性能需求考虑DPDK/XDP

### 性能优化路径

```
第一阶段：基础优化（80%收益）
- 使用epoll
- 启用TCP_NODELAY
- 调整缓冲区大小
- 使用连接池

第二阶段：进阶优化（15%收益）
- 零拷贝技术
- io_uring
- 系统参数调优
- 多线程Reactor

第三阶段：极致优化（5%收益）
- DPDK/XDP
- RDMA
- 自定义协议栈
- 硬件卸载
```

### 性能数字记忆

- 系统调用：~50-100ns
- epoll_wait：~1-2μs
- TCP小包RTT（本地）：~50μs
- sendfile vs read+write：快50-100%
- DPDK延迟：<1μs
- RDMA延迟：<1μs

---

**下一课预告：IO优化课程-05-高级优化技术与并发控制**
- 无锁编程与原子操作
- 内存序与内存屏障
- RCU（Read-Copy-Update）
- 批量处理技术
- 预取与推测执行
