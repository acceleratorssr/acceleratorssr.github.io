---
layout: post
title: 调度系统
tags: 技术沉淀
excerpt: 分别考虑调度者、任务间的依赖执行关系等设计，包括定时任务、异步任务等场景，顺便聊聊 FSM 和 DAG；
---

## FSM & DAG
&emsp;&emsp;有限状态机 FSM，侧重于描述系统的 **状态** 和 行为，描述状态间的 **转换**关系，可有环；

状态机的元素：
- state 状态，任意时刻，状态机都只能处于某一个状态；
- event / transition condition 事件：即事件驱动的事件；
- transition 转移：状态转移的过程（状态转移方程）；
- action 动作：到某个状态需要执行的动作；

如 tcp 的状态机、线程的状态机；（这里就不展示了）

&emsp;&emsp;有向无环图 DAG（directed acyclic graph），侧重于描述节点（状态）间的 **依赖**关系 和 **层次**关系，用于描述**复杂流程**，经典如 任务调度、编译优化、文件过算子的流程等，灵活性较好（易于添加任务）；

> 注：文件过算子：风控场景下，一个文件通常可能命中多个算子策略，如广度上可能需要过色情、伦理算子，深度上 算子间也可能组成一个 DAG，如大分类到若干小分类，色情、儿童色情、低俗、性暗示等；（这里只是举例，非真实过检流程）


#### 拓扑排序
对于 DAG 中，每个有向边（u, v），在序列中 顶点 u 在顶点 v 之前；

&emsp;&emsp;`Kahn` 算法（bfs）：初始化一个队列，将所有入度为 0 的节点入队；从队列中获取一个节点，将其添加到拓扑排序的结果中，并移除其所有出边；对于每条被移除的变，如果对应目标节点的入度变为 0，则将节点入队；循环直到队列为空；（注意排除环）

&emsp;&emsp;`dfs`：递归返回时的顶点入栈，最后依次从栈中弹出的顺序即为序列顺序；

eg：kahn：
```go
package main

import (
	"fmt"
	"strconv"
	"strings"
)

func main() {
	var n, m int
	fmt.Scanln(&n, &m)

	inDegree := make([]int, n)
	adjList := make(map[int][]int)

	for i := 0; i < m; i++ {
		var s, t int
		fmt.Scanln(&s, &t)
		inDegree[t]++
		adjList[s] = append(adjList[s], t)
	}

	queue := make([]int, 0, n)
	for i := 0; i < n; i++ {
		if inDegree[i] == 0 {
			queue = append(queue, i)
		}
	}

	result := make([]int, 0, n)
	for len(queue) > 0 {
		cur := queue[0]
		queue = queue[1:]

		result = append(result, cur)

		for _, next := range adjList[cur] {
			inDegree[next]--
			if inDegree[next] == 0 {
				queue = append(queue, next)
			}
		}
	}

	if len(result) == n {
		strs := make([]string, n)
		for i, num := range result {
			strs[i] = strconv.Itoa(num)
		}
		fmt.Println(strings.Join(strs, " "))
	} else {
		fmt.Println(-1)
	}
}
```

<br>
<br>

如：[Airflow](https://github.com/apache/airflow)，通过 DAG 实现任务调度：

注意：任务需要**完成什么动作** && 任务需要**依赖什么任务**

步骤：
1. 将复杂任务拆分为多个小任务，即 将可独立的小任务拆出来；
2. 根据小任务间的依赖关系进行调度，拓扑排序 or 优先级队列（元素为 小任务 : 依赖关系的集合）实现；
3. 当所有小任务完成后，将结果合并为一个完整任务（如通过 hook 实现）；



## 调度系统
时间管理（time management），如考虑任务计划、优先级，是否需要实时调度，资源结合 LB 进行分配等；

范围（scope），如任务的类型，调度系统的架构，支持的功能（如日志、监控告警等），系统的可伸缩性和扩展性，也可能涉及权限管理；

**能力**
1. 支持用户提交任务（注册 handler，如脚本或者常驻的轻量服务）；
2. 由调度者主动启停任务（如 cron 触发）；
3. 任务产出：
   1. 持久化到 db，由可视化平台或者数仓再去拉取进一步展示 or 处理；
   2. 数据分析场景中，就比如 spark 等业务中，存在任务依赖关系，可转换为 DAG 关系；

**健壮性**
1. 可扩展性
2. 可用性 & 可靠性

即依赖于 db + FSM；

调度 model 可能的考虑：
- job id
- owner id
- 二进制可执行文件下载 url
- 任务元信息获取 url（或者是 json 格式的 task_info）
- 任务执行结果下载 url
- status
- 重试次数

对于 任务的 **FSM**：

`ready` -> `waiting` -> `running` -> `success`
              |             |
          `retry`   <-  `failed` -> `abort`/`delay queue`

<p><img src="https://acceleratorssr.github.io/image/job.png" alt="调度系统流程"></p>

1. mq 可考虑存内存型或磁盘型（即是否无状态，但磁盘型的要考虑数据一致性，一种简单的可能是独立任务仅允许通过 mq 进行消费执行，**todo**）（同理，mq 可考虑换为一个小 scheduler，发布任务给 worker，设计和 MR 的scheduler 类似）；

DAG 构建简单任务调度系统的组件考虑：Vertex（包括行为），Graph（流程管理），Context（上下文）、Config（生成 Graph）等；

进一步案例分析：[广告平台化的探索与实践](https://tech.meituan.com/2022/03/03/exploration-and-practice-of-advertising-engine-platforming.html)

施工中...