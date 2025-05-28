---
title: Go实现 MapReduce 的 Map 阶段
date: 2024-05-14 17:12:04
tags:
- Golang
- MapReduce
categories:
- Golang
index_img: https://github.com/whitestorm0316/picx-images-hosting/raw/master/image.99tl7oumim.webp
---
# **实现 MapReduce 的 Map 阶段（Go 语言示例）**

以下是一个完整的 `go-reader` 实现，支持并发处理输入数据、进程间通信和协程池管理：

```go
package main

import (
	"bufio"
	"fmt"
	"io"
	"os"
	"os/exec"
	"sync"
	"time"
)

const (
	workerPoolSize = 4     // 协程池大小（根据 CPU 核心数调整）
	bufferSize     = 100   // 任务缓冲队列长度
	processTimeout = 10 * time.Second // 单个任务超时时间
)

func main() {
	// 1. 解析命令行参数
	if len(os.Args) < 3 || os.Args[1] != "-cmd" {
		fmt.Println("Usage: cat file.txt | ./go-reader -cmd \"${cmd}\" > result.txt")
		os.Exit(1)
	}
	cmdTemplate := os.Args[2]

	// 2. 准备协程池和通信管道
	taskChan := make(chan string, bufferSize)
	resultChan := make(chan string, bufferSize)
	var wg sync.WaitGroup

	// 3. 启动 Worker 协程池
	for i := 0; i < workerPoolSize; i++ {
		wg.Add(1)
		go worker(cmdTemplate, taskChan, resultChan, &wg)
	}

	// 4. 读取输入并分发任务
	go func() {
		scanner := bufio.NewScanner(os.Stdin)
		for scanner.Scan() {
			taskChan <- scanner.Text()
		}
		close(taskChan)
	}()

	// 5. 收集结果并输出
	go func() {
		wg.Wait()
		close(resultChan)
	}()

	for result := range resultChan {
		fmt.Println(result)
	}
}

// Worker 协程：处理单个任务
func worker(cmdTemplate string, tasks <-chan string, results chan<- string, wg *sync.WaitGroup) {
	defer wg.Done()

	for line := range tasks {
		// 6. 创建子进程
		cmd := exec.Command("sh", "-c", cmdTemplate)
		stdin, _ := cmd.StdinPipe()
		stdout, _ := cmd.StdoutPipe()

		// 7. 启动子进程并设置超时
		if err := cmd.Start(); err != nil {
			results <- fmt.Sprintf("ERROR: %v", err)
			continue
		}

		// 8. 写入输入数据
		go func() {
			defer stdin.Close()
			io.WriteString(stdin, line)
		}()

		// 9. 读取输出并处理超时
		done := make(chan error, 1)
		go func() {
			output, _ := io.ReadAll(stdout)
			results <- string(output)
			done <- cmd.Wait()
		}()

		select {
		case <-time.After(processTimeout):
			cmd.Process.Kill()
			results <- fmt.Sprintf("TIMEOUT: %s", line)
		case err := <-done:
			if err != nil {
				results <- fmt.Sprintf("ERROR: %v", err)
			}
		}
	}
}
```

### **关键设计解析**

#### **1. 进程间通信方式**

- **标准输入/输出管道（Pipe）**通过 `cmd.StdinPipe()` 和 `cmd.StdoutPipe()` 创建管道，实现 Go 程序与子进程的通信。
- **协程间 Channel 通信**
  - `taskChan`：缓冲输入任务队列
  - `resultChan`：缓冲处理结果队列

#### **2. 协程池的必要性**

- **资源控制**：防止无限制创建子进程导致 OOM
- **性能优化**：固定数量的 Worker 可复用，减少协程创建开销
- **负载均衡**：均匀分配任务到多个 Worker

#### **3. 超时处理机制**

- 通过 `select` + `time.After` 实现超时控制
- 超时后强制终止卡住的子进程

#### **4. 错误处理**

- 捕获子进程启动失败
- 收集子进程执行错误信息
- 超时任务特殊标记

---

### **使用示例**

```bash
# 编译程序
go build -o go-reader main.go

# 运行示例（统计每行单词数）
cat input.txt | ./go-reader -cmd "wc -w" > output.txt

# 输入文件示例（input.txt）
Hello world
MapReduce test
Go concurrency pattern

# 输出结果（output.txt）
2
2
3
```

---

### **性能优化建议**

1. **动态调整协程池大小**根据任务类型（CPU/IO 密集型）自动扩容缩容：

   ```go
   // 示例：根据 CPU 核心数设置
   workerPoolSize := runtime.NumCPU() * 2
   ```
2. **批量处理任务**合并多个输入行批量处理，减少进程创建开销：

   ```go
   // 在 Worker 中处理批量任务
   const batchSize = 10
   var batch []string
   for line := range tasks {
       batch = append(batch, line)
       if len(batch) >= batchSize {
           processBatch(batch)
           batch = nil
       }
   }
   ```
3. **结果排序（如需保持顺序）**
   为每个任务添加序号，在结果收集时排序：

   ```go
   type Task struct {
       ID   int
       Line string
   }

   type Result struct {
       TaskID int
       Output string
   }
   ```

---

### **总结**

通过协程池 + 管道通信 + 超时控制的组合设计，该实现能够：

- 高效利用系统资源
- 安全处理进程间通信
- 提供基本的容错能力
- 灵活扩展批量处理等优化策略

实际生产环境中可在此基础上增加：

- 重试机制
- 更精细的资源监控
- 分布式任务调度（如需跨节点）
