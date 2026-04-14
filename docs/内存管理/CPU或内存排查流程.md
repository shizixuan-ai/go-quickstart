## Go 应用 CPU 飙升或内存打满的排查流程

当 Go 应用出现 CPU 持续飙升或内存耗尽（OOM、内存占用不断增长）时，需要系统性地定位问题。以下是推荐的排查步骤，涵盖常用工具和典型根因。

### 一、准备工作

- **确保应用开启了 `net/http/pprof` 或运行时已注册 pprof 端点**：`import _ "net/http/pprof"` 并启动一个 HTTP 服务，或者通过 `runtime/pprof` 写入文件。
- **如果应用无法直接访问**，可使用 `curl` 或 `go tool pprof` 抓取 profile。
- **保留现场**：如果内存持续增长，先执行 `curl -s http://localhost:6060/debug/pprof/heap > heap.pprof`；CPU 飙升时抓取 `curl -s http://localhost:6060/debug/pprof/profile?seconds=30 > cpu.pprof`。

### 二、CPU 飙升排查流程

#### 1. 确认 CPU 使用情况
```bash
top -H -p <pid>                # 查看进程内线程 CPU 占用
perf top -p <pid>              # (Linux) 查看热点函数
```

#### 2. 采集 CPU profile
```bash
go tool pprof -http=:8080 cpu.pprof   # 交互式分析
# 或直接在线分析
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
```

#### 3. 常见 CPU 飙升原因与定位
| 现象 | 可能原因 | pprof 特征 |
|------|----------|------------|
| 单核打满，其他核空闲 | 死循环、自旋锁、runtime.LockOSThread 绑定 | 某个 goroutine 长时间占用同一 P |
| 多核全高 | 大量 goroutine 并行计算、频繁 GC | 火焰图显示大量时间在 `runtime.mallocgc` 或 GC 相关函数 |
| 周期性突增 | 定时任务、大量请求、GC 频繁 | 查看 `goroutine` profile 是否堆积，或 GC 触发时间点 |

#### 4. 针对性解决
- **死循环/自旋**：使用 `go tool pprof -list` 找到循环体，增加 `runtime.Gosched()` 或改用 channel/定时器。
- **频繁 GC**：`go tool pprof -alloc_objects` 查看分配热点，减少内存分配、使用 `sync.Pool`、调整 `GOGC`。
- **大量 goroutine**：`go tool pprof goroutine` 查看堆积的 goroutine 栈，检查是否有 channel 泄漏或未等待的 WaitGroup。

### 三、内存打满（持续增长/泄漏）排查流程

#### 1. 观察内存趋势
```bash
watch -n 1 'ps -p <pid> -o rss,vsz'   # 监控 RSS 是否持续上升
# 或使用 Prometheus + Grafana 监控
```

#### 2. 采集 heap profile
```bash
curl -s http://localhost:6060/debug/pprof/heap > heap1.pprof
# 等待一段时间（如 30 秒）
curl -s http://localhost:6060/debug/pprof/heap > heap2.pprof
go tool pprof -base heap1.pprof heap2.pprof   # 对比增长的内存
```

#### 3. 分析内存占用
- **inuse_space**：当前正在使用的内存。
- **alloc_space**：累计分配的内存（可发现频繁分配）。
```bash
go tool pprof -alloc_space heap.pprof   # 查看累计分配最多的函数
go tool pprof -inuse_space heap.pprof   # 查看当前占用最多的对象
```

#### 4. 常见内存问题与特征
| 问题类型 | 典型症状 | pprof 指标 |
|----------|----------|------------|
| **goroutine 泄漏** | RSS 持续增长，`goroutine` profile 数量不断增加 | `go tool pprof goroutine` 显示大量相同栈的 goroutine |
| **未释放的缓存** | 内存稳定在高位，但不再增长 | `inuse_space` 显示某个 map/slice 占用巨大 |
| **引用未切断** | 大对象被全局变量或长生命周期对象引用 | `inuse_space` 无法释放，但对象本身无外部引用？需要看 GC 日志 |
| **CGO 内存泄漏** | RSS 增长但 Go heap 不大 | 检查 C 代码，使用 `valgrind` 或 `asan` |
| **slice/map 扩容导致内存碎片** | 内存使用量远大于实际对象大小 | `go tool pprof -inuse_space` 显示大量 `runtime.growslice` |

#### 5. 辅助排查手段
- **GC 日志**：设置 `GODEBUG=gctrace=1` 观察 GC 频率和停顿，若 `scvg`（清扫后剩余内存）持续增长，可能存在内存泄漏。
- **查看系统内存统计**：`runtime.ReadMemStats` 打印 `HeapInuse`、`HeapIdle`、`HeapReleased`。
- **内核工具**：`/proc/<pid>/smaps` 查看内存映射，`pmap -x <pid>`。

### 四、通用排查工具清单

| 工具 | 用途 | 命令示例 |
|------|------|----------|
| **pprof** | CPU/内存/阻塞/锁 profile | `go tool pprof http://localhost:6060/debug/pprof/heap` |
| **trace** | 调度器、GC、网络事件 | `go tool trace trace.out` |
| **goroutine 泄露** | 查看当前所有 goroutine 栈 | `curl http://localhost:6060/debug/pprof/goroutine?debug=2` |
| **gc 日志** | 观察 GC 频率和内存回收 | `GODEBUG=gctrace=1 ./app` |
| **Linux 性能工具** | 系统级热点 | `perf top -p pid`, `strace -c -p pid` |
| **内存分析** | 堆外内存、CGO 内存 | `valgrind`, `heaptrack` |

### 五、应急处理建议

- **临时止血**：重启应用（仅治标）；调整 `GOMAXPROCS` 降低 CPU 使用；调高 `GOGC` 减少 GC 频率。
- **长期修复**：根据 pprof 报告优化代码，避免 goroutine 泄漏，减少不必要的内存分配，使用 `sync.Pool`，修复死循环。
- **监控与告警**：集成 Prometheus + Grafana，监控 `go_goroutines`、`go_memstats_alloc_bytes`、`process_cpu_seconds_total` 等指标。

### 六、案例演示（简例）

```bash
# 1. 获取 CPU profile
curl -s http://localhost:6060/debug/pprof/profile?seconds=10 > cpu.pprof

# 2. 查看 top 函数
go tool pprof -top cpu.pprof
# 显示 runtime.mallocgc 占用 60% -> 说明 GC 压力大

# 3. 查看内存分配热点
go tool pprof -alloc_space http://localhost:6060/debug/pprof/heap
# 发现某个函数每秒分配 500MB，优化该函数

# 4. 查看 goroutine 泄漏
curl http://localhost:6060/debug/pprof/goroutine?debug=2 > stacks.txt
# 发现大量 goroutine 阻塞在同一个 channel 上，检查该 channel 的消费者是否退出
```

通过以上流程，大部分 Go 应用的性能问题都能被定位并解决。关键在于**平时就开启 pprof 端点**，并熟悉 pprof 交互式的分析方式。