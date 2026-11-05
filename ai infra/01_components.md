# AI Infra 学习组件索引

> 第一轮按 [[阅读目录]] 的路线推进。组件按系统问题组织，不机械照抄章节目录。

## 组件地图

| 编号 | 组件 | 主要正文 | 主要代码 / 实验入口 | 状态 |
| --- | --- | --- | --- | --- |
| 02 | [[02_ai_infra_panorama\|AI Infra 全景与六层结构]] | 1.1 | `experiments/ch01/01-01` | 已共同阅读 |
| 03 | [[03_resource_numbers\|容量、带宽、算力与延迟]] | 1.2～1.3 | `resource_basics.py`、`decode_budget.py`、实验 1-2～1-4 | 共同阅读中 |
| 04 | [[04_prefill_decode_workload\|Prefill、Decode 与请求负载]] | 3.1 | `request_trace.py`、`batch_reuse.py`、实验 3-1～3-2 | 已共同阅读 |
| 05 | [[05_agent_critical_path\|Agent 关键路径、工具等待与状态驻留]] | 3.2 | `agent_trace.py`、`execution_dag.py`、实验 3-3～3-4 | 待共同阅读 |
| 06 | [[06_inference_request_lifecycle\|推理请求生命周期与指标]] | 8.1 | `decode_budget.py`、实验 8-1 | 待共同阅读 |
| 07 | [[07_batch_and_scheduling\|Batch、连续批处理与调度]] | 8.2 | `iteration_batching.py`、`service_replay.py`、实验 8-1～8-2 | 待共同阅读 |
| 08 | [[08_kv_cache_and_prefix_reuse\|KV Cache 与 Prefix 复用]] | 8.3 | `kv_pages.py`、`kv_trace.py`、`apc_trace.py`、实验 8-3～8-4 | 待共同阅读 |
| 09 | [[09_compression_quantization_offload\|压缩、量化与 Offload]] | 8.4 | `kv_codec.py`、`kv_quality.py`、`kv_tiers.py`、实验 8-7～8-8 | 待共同阅读 |
| 10 | [[10_speculative_decoding\|推测解码]] | 8.5 | `speculative_round.py`、`speculative_budget.py`、实验 8-5 | 待共同阅读 |
| 11 | [[11_queue_slo_configuration\|排队、SLO、吞吐与配置选择]] | 8.6 | `service_replay.py`、`energy_ledger.py`、实验 8-9 | 待共同阅读 |
| 12 | [[12_agent_resource_lifecycle\|Agent 任务的资源生命周期]] | 11.1 | `environment_resources.py`、实验 11-1 | 待共同阅读 |
| 13 | [[13_sandbox_environment_lifecycle\|Sandbox 与环境生命周期]] | 11.2 | `environment_lifecycle.py`、实验 11-2、11-6 | 待共同阅读 |
| 14 | [[14_resource_pool_preemption_scheduling\|资源池、抢占与调度]] | 11.3 | `completion_reclaim.py`、`feedback_queue.py`、实验 11-3～11-5 | 待共同阅读 |
| 15 | [[15_model_routing_thinking_budget_cost\|模型路由、Thinking Budget 与成本]] | 11.4 | `routing_cost.py`、`strategy_record_cost.py`、实验 11-7～11-8 | 待共同阅读 |
| 16 | [[16_retry_recovery_idempotency_cost\|Retry、恢复、幂等与完整任务成本]] | 11.5 | `retry_paths.py`、`operation_ordering.py`、实验 11-4、11-9 | 待共同阅读 |

## 第一轮研究顺序

```text
02 全景
  ↓
03 数量级与资源下界
  ↓
04 请求负载
  ↓
05 Agent 关键路径
  ↓
06～11 推理服务核心
  ↓
12～16 Agent Infra 与运行环境
```

第 2、5、9、12 章只作为依赖回查，不单独建立主线组件。若后续发现某个支线足够复杂，再单独扩展组件。

## 单个组件的固定结构

每份组件笔记使用同一组章节：

1. 学习目标与边界
2. 先用直白的话讲
3. 最小模型或运行链
4. 关键输入、输出和状态
5. 公式与数量级
6. 正文、计算代码与实验入口
7. 与 Agent Harness 的连接
8. 共同阅读时要回答的问题
9. 已确认结论
10. 待验证与暂不展开

## 状态纪律

- 开始讨论前保持“待共同阅读”；
- 讨论中改为“共同阅读中”；
- 只有核心结论和证据均确认后才能标为“已共同阅读”；
- 仅建立框架不代表组件已经完成；
- 组件文件中的结论必须能回到正文、函数、固定输入或实验记录。
