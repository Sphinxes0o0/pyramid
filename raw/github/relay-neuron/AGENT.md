# Research Agent 启动指南

## 一句话启动

```
你: "研究 topic: 肌肥大与肌肉增长"
```

Agent 自动完成所有相关 sub-tasks 的搜索、研究、总结、优化循环。

---

## Topic 定义模板

每个 topic 包含：

```markdown
# Topic: TOPIC_NAME

## Sub-Tasks
1. [ ] sub-task-1: 描述
2. [ ] sub-task-2: 描述

## 搜索关键词
- keyword1
- keyword2

## 来源优先级
1. PubMed
2. Nature/Science
3. 预印本
```

---

## 执行流程

```
用户输入 "研究 topic: XXX"
        ↓
Agent 解析 topic 定义
        ↓
拆分 sub-tasks
        ↓
┌─────────────────────────────────┐
│  For each sub-task (并行/串行):  │
│    1. 搜索 (Search)              │
│    2. 研究 (Research)            │
│    3. 总结 (Summarize)           │
│    4. 优化 (Optimize)            │
│    5. 标记 [x]                   │
└─────────────────────────────────┘
        ↓
所有 sub-tasks 完成
        ↓
生成 FINAL_REPORT.md
        ↓
Git commit
```

---

## 自迭代循环控制

### 循环变量
- `current_subtask`: 当前执行的 sub-task
- `remaining_tasks`: 剩余未完成 sub-tasks

### 循环条件
```python
while remaining_tasks > 0:
    execute_subtask_loop(current_subtask)
    update_progress()
    if all_complete:
        generate_final_report()
```

### 进度输出
```
[Round 1/5] Sub-task 1: 搜索 ✓
[Round 1/5] Sub-task 1: 研究 ✓
[Round 1/5] Sub-task 1: 总结 ✓
[Round 1/5] Sub-task 1: 优化 ✓
[Progress] 1/5 sub-tasks complete
---
[Round 2/5] Sub-task 2: 搜索 ✓
...
```

---

## Topic 启动命令示例

| 命令 | 执行范围 |
|------|---------|
| `研究 topic: 肌肥大` | Topic 1 + 9.1 (4 sub-tasks) |
| `研究 topic: 蛋白质与营养` | Topic 2 (5 sub-tasks) |
| `研究 topic: 阻力训练` | Topic 3 (5 sub-tasks) |
| `研究 topic: 睡眠与恢复` | Topic 5 (4 sub-tasks) |
| `研究 topic: 碳水化合物` | Topic 12 (14 sub-tasks) |
| `研究 topic: 鱼油与Omega-3` | Topic 13 (8 sub-tasks) |
| `研究全部` | 执行 RESEARCH_MASTER_LIST.md 所有任务 |

---

## 输出文件结构

```
research/
├── 2026-04-28-topic-muscle-hypertrophy/
│   ├── subtask-1-search-results.md
│   ├── subtask-1-final-report.md
│   ├── subtask-2-search-results.md
│   ├── subtask-2-final-report.md
│   └── ...
└── 2026-04-28-FINAL-REPORT.md
```

---

## 预定义 Topics

| Topic ID | Topic Name | Sub-tasks |
|----------|-----------|-----------|
| TOPIC-01 | 肌肥大与肌肉增长 | 4 |
| TOPIC-02 | 营养与补剂 | 5 |
| TOPIC-03 | 阻力训练科学 | 5 |
| TOPIC-04 | 心肺与代谢适应 | 5 |
| TOPIC-05 | 睡眠与恢复 | 4 |
| TOPIC-06 | 健康与疾病 | 5 |
| TOPIC-07 | 训练方法与技术 | 5 |
| TOPIC-08 | 学术会议与期刊 | 4 |
| TOPIC-09 | 新兴研究领域 | 4 |
| TOPIC-10 | 综合研究轮次 | 16 |
| TOPIC-11 | 补剂科学 | 7 |
| TOPIC-12 | 碳水化合物研究 | 14 |
| TOPIC-13 | 鱼油与Omega-3 | 8 |