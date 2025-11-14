# 第16章：Transforms 功能实践指南

本章介绍如何使用 Elasticsearch 的 Transform（数据转换）功能，将明细事件数据按维度聚合重塑为新的、开箱即可用于分析和加速查询的实体视角索引。内容包括：

1. 概念与使用场景
2. Transform 类型：Pivot 与 Latest
3. 批处理与持续（Continuous）模式
4. 示例数据与准备步骤
5. 使用 Kibana 控台或 API 全流程：预览 -> 创建 -> 启动 -> 查询状态 -> 更新 -> 停止 -> 删除
6. 常见问题与优化建议
7. 进阶：结合聚合、脚本、运行频率与性能调优
8. 清理与再现
9. 版本与兼容性注意
10. 总结
11. 滚动窗口与时间序列特征示例
12. 与机器学习异常检测结合
13. 预测（Forecasting）

---
## 1. 概念与使用场景
Transform 通过持续或一次性地读取源索引中的文档，使用聚合（通常是 terms + date_histogram + metrics 等）生成“实体化视图”(entity-centric index)。适用于：
- 构建用户画像：从用户行为日志提炼每个用户的汇总指标。
- 构建商品或设备统计快照：如销量、最近活动时间、平均评分。
- 预聚合加速报表：减少在查询时复杂聚合的开销。
- 制作用于机器学习的特征表：将大量明细点击/购买事件转换成每个用户一行的特征集。

Transform 的结果是一个新的索引；原始数据仍保留在源索引中。可以周期性/实时地刷新该实体索引。

## 2. Transform 类型：Pivot 与 Latest
- Pivot Transform：根据指定的分组（group_by）和聚合（aggregations）规则，把多行事件聚合为一行实体。适合统计指标、汇总。
- Latest Transform：在分组键下仅保留最新（基于时间字段）的一条文档，用于维护最新状态快照，如设备当前状态、订单最新状态。

选择依据：
- 需要聚合多个数值/计数 => Pivot。
- 只关心分组键下最新记录 => Latest。

## 3. 批处理与持续（Continuous）模式
- Batch（默认）：执行一次到结束，适合历史数据一次性转换。
- Continuous：开启后以内部 checkpoint 机制定期增量拉取新文档（依赖日期字段或时间同步条件），保持目标索引近实时更新。

Continuous 模式注意：
- 需要在配置中指定日期字段用于增量推进（sync）。
- 资源使用会更长期；需监控内存与搜索负载。

## 4. 示例数据与准备
示例场景：我们有一个 `sales_events` 索引，存储电商销售事件，每条文档包含：
- event_time（日期）
- user_id
- product_id
- category
- price
- quantity

我们要构建一个用户画像索引 `user_sales_profile`，每行对应一个用户，聚合其：
- 订单总数
- 总金额（price * quantity）
- 平均订单金额
- 最近一次购买时间
- 涉及的品类列表（示例利用 `terms` + `bucket_script`/后处理）

### 4.1 创建源索引映射（可简单示例）
```
PUT sales_events
{
  "mappings": {
    "properties": {
      "event_time": {"type": "date"},
      "user_id": {"type": "keyword"},
      "product_id": {"type": "keyword"},
      "category": {"type": "keyword"},
      "price": {"type": "double"},
      "quantity": {"type": "integer"}
    }
  }
}
```

### 4.2 导入示例数据（Bulk）
下面是一个简化的 bulk 数据片段，你可复制到 Kibana Dev Tools：
```
POST _bulk
{ "index": { "_index": "sales_events" } }
{ "event_time": "2025-10-30T10:00:00Z", "user_id": "u1", "product_id": "p1", "category": "book", "price": 30.5, "quantity": 1 }
{ "index": { "_index": "sales_events" } }
{ "event_time": "2025-10-30T10:05:00Z", "user_id": "u1", "product_id": "p2", "category": "electronics", "price": 199.0, "quantity": 1 }
{ "index": { "_index": "sales_events" } }
{ "event_time": "2025-10-30T11:00:00Z", "user_id": "u2", "product_id": "p3", "category": "book", "price": 45.0, "quantity": 2 }
{ "index": { "_index": "sales_events" } }
{ "event_time": "2025-10-30T12:00:00Z", "user_id": "u1", "product_id": "p4", "category": "home", "price": 80.0, "quantity": 1 }
{ "index": { "_index": "sales_events" } }
{ "event_time": "2025-10-30T12:30:00Z", "user_id": "u2", "product_id": "p5", "category": "electronics", "price": 250.0, "quantity": 1 }
```
刷新：
```
POST sales_events/_refresh
```

## 5. 全流程操作
### 5.1 预览 Pivot Transform
```
POST _transform/_preview
{
  "source": {"index": "sales_events"},
  "pivot": {
    "group_by": {
      "user_id": {"terms": {"field": "user_id"}}
    },
    "aggregations": {
      "order_count": {"value_count": {"field": "product_id"}},
      "total_revenue": {
        "sum": {
          "script": {
            "source": "doc['price'].value * doc['quantity'].value"
          }
        }
      },
      "avg_order_value": {
        "bucket_script": {
          "buckets_path": {
            "total": "total_revenue",
            "count": "order_count"
          },
          "script": "params.total / params.count"
        }
      },
      "last_purchase_time": {"max": {"field": "event_time"}},
      "categories": {
        "scripted_metric": {
          "init_script": "state.list = new HashSet();",
          "map_script": "state.list.add(doc['category'].value);",
          "combine_script": "return state.list;",
          "reduce_script": "Set all = new HashSet(); for (s in states) { all.addAll(s) } return all;"
        }
      }
    }
  },
  "description": "User sales profile preview"
}
```
预览结果会返回示例文档结构，可用于确认字段。

注意：`scripted_metric` 产生的集合结果在 transform 最终索引中会是一个数组。若需更简单，可改用 `terms` 聚合 + `top_hits` 或后续在应用层处理。

### 5.2 创建 Transform（Batch）
```
PUT _transform/user_sales_profile_batch
{
  "source": {"index": "sales_events"},
  "description": "Batch transform building user sales profile",
  "dest": {"index": "user_sales_profile"},
  "pivot": {
    "group_by": {
      "user_id": {"terms": {"field": "user_id"}}
    },
    "aggregations": {
      "order_count": {"value_count": {"field": "product_id"}},
      "total_revenue": {
        "sum": {
          "script": {"source": "doc['price'].value * doc['quantity'].value"}
        }
      },
      "avg_order_value": {
        "bucket_script": {
          "buckets_path": {"total": "total_revenue", "count": "order_count"},
          "script": "params.total / params.count"
        }
      },
      "last_purchase_time": {"max": {"field": "event_time"}}
    }
  }
}
```

### 5.3 启动 Transform
```
POST _transform/user_sales_profile_batch/_start
```
查询任务状态：
```
GET _transform/user_sales_profile_batch/_stats
```
查看生成的目标索引文档：
```
GET user_sales_profile/_search
{
  "size": 10
}
```

### 5.4 使用 Continuous 模式
Continuous 增量需要 `sync` 部分：
```
PUT _transform/user_sales_profile_cont
{
  "source": {"index": "sales_events"},
  "description": "Continuous user profile transform",
  "dest": {"index": "user_sales_profile_cont"},
  "sync": {
    "time": {
      "field": "event_time",
      "delay": "60s"   
    }
  },
  "pivot": {
    "group_by": {"user_id": {"terms": {"field": "user_id"}}},
    "aggregations": {
      "order_count": {"value_count": {"field": "product_id"}},
      "total_revenue": {"sum": {"script": {"source": "doc['price'].value * doc['quantity'].value"}}},
      "last_purchase_time": {"max": {"field": "event_time"}}
    }
  }
}
```
启动：
```
POST _transform/user_sales_profile_cont/_start
```
随着新事件写入 `sales_events`，在 delay 窗口后会被纳入聚合。

### 5.5 更新（例如修改描述）
```
POST _transform/user_sales_profile_cont/_update
{
  "description": "Continuous user profile transform (updated description)"
}
```
注意：不能随意改变 pivot 的核心结构，只能更新部分元数据。若需结构调整，建议新建一个 Transform。

### 5.6 停止与删除
```
POST _transform/user_sales_profile_cont/_stop?wait_for_completion=true
DELETE _transform/user_sales_profile_cont
```
删除仅会删除任务定义，不会删除目标索引 `user_sales_profile_cont`，如需删除索引：
```
DELETE user_sales_profile_cont
```

### 5.7 Latest Transform 示例
场景：维护每个设备的最新状态。
```
PUT device_events
{
  "mappings": {
    "properties": {
      "device_id": {"type": "keyword"},
      "status": {"type": "keyword"},
      "updated_at": {"type": "date"},
      "battery": {"type": "integer"}
    }
  }
}

POST _bulk
{ "index": { "_index": "device_events" } }
{ "device_id": "d1", "status": "online", "updated_at": "2025-10-30T09:00:00Z", "battery": 95 }
{ "index": { "_index": "device_events" } }
{ "device_id": "d1", "status": "offline", "updated_at": "2025-10-30T12:00:00Z", "battery": 80 }
{ "index": { "_index": "device_events" } }
{ "device_id": "d2", "status": "online", "updated_at": "2025-10-30T11:30:00Z", "battery": 60 }

POST device_events/_refresh
```
Latest Transform：
```
PUT _transform/device_latest_state
{
  "source": {"index": "device_events"},
  "description": "Latest status per device",
  "dest": {"index": "device_state"},
  "latest": {
    "unique_key": ["device_id"],
    "sort": "updated_at"
  },
  "sync": {
    "time": {"field": "updated_at", "delay": "30s"}
  }
}
```
启动：
```
POST _transform/device_latest_state/_start
```
查询：
```
GET device_state/_search
```

## 6. 常见问题与陷阱
1. 大量分组键导致内存峰值：使用过滤、拆分 Transform 或调整 `max_page_search_size`。
2. Continuous Transform 延迟：`delay` 太大或源索引写入时间戳晚到（应用写入延迟）。可调低 delay 或确保事件时间戳与真实写入接近。
3. 无法更新核心结构：Transform 设计为不可变结构；迭代时创建新 Transform，旧的可停用保留历史。
4. Script 开销大：尽量用预计算字段，避免复杂脚本；可在 Ingest Pipeline 中先生成衍生字段。
5. 权限不足：确保执行用户具备对源索引的 read 权限与对目标索引的 write 权限，以及 transform 管理权限。

## 7. 优化建议
- 预聚合字段：在写入时计算 price*quantity 减少脚本。
- 减少分组数量：对高基数字段先做过滤或采样。
- 调整 checkpoint 间隔：Continuous 模式自动管理，若滞后可查看 `_stats` 中的 checkpoint 信息。
- 监控指标：使用 Stack Monitoring 观察 Transform 运行资源。
- 索引模板：为目标索引定义模板（mapping、settings）确保字段类型正确，如日期、数值。
- 冷热分层：源索引历史可转冷/冻结层，Transform 主要处理热数据。

## 8. 清理与再现
当你需要重新演练：
```
DELETE _transform/user_sales_profile_batch
DELETE user_sales_profile
DELETE _transform/user_sales_profile_cont
DELETE user_sales_profile_cont
DELETE sales_events
```

## 9. 版本与兼容性注意
- 8.x 版本中 Transform 功能成熟；早期 7.x 可能部分字段或统计输出格式稍有差异。
- 安全（X-Pack Security）开启时需具备 `manage_transform` 或 `transform_admin` 权限角色。

## 10. 总结
Transform 让你可以把原始的事件流转换为实体索引，降低查询时的聚合成本并提供近实时更新能力。通过合理设计 pivot 或 latest 结构、选择 batch 或 continuous 模式，可以实现高效的分析与特征抽取工作流。

## 11. 滚动窗口与时间序列特征示例
本节展示如何使用 Transform 生成按天聚合的用户收入，并在结果索引中直接包含最近7天滚动累计金额特征，用于后续分析或 ML。

### 11.1 设计思路
- 源索引：`sales_events`（同前）
- 目标索引：`user_daily_sales`
- 分组维度：用户ID + 日期（日粒度）
- 指标：当日总收入、当日订单数
- 滚动特征：最近7天总收入（包含当天）

当前 Pivot Transform 不原生支持跨行的滚动窗口计算，因为每个 bucket 只看到自己聚合范围。实现方案有两种：
1. 使用新的 Elasticsearch 功能（如果版本支持）`moving_fn` 在聚合阶段做移动计算（注意：Transform 的聚合需要是单次 bucket 聚合，直接 moving_fn 可能受限）。
2. 将 Pivot 生成的日度索引后，再用第二个 Transform 或使用 Rollup/再加工脚本更新 7 日窗口字段。

这里演示“二阶段”方法：

### 11.2 第1阶段：日度 Pivot Transform
```
PUT _transform/user_daily_sales_stage1
{
  "source": {"index": "sales_events"},
  "description": "User daily sales pivot (stage1)",
  "dest": {"index": "user_daily_sales_stage1"},
  "pivot": {
    "group_by": {
      "user_id": {"terms": {"field": "user_id"}},
      "day": {"date_histogram": {"field": "event_time", "calendar_interval": "day"}}
    },
    "aggregations": {
      "daily_revenue": {
        "sum": {
          "script": {"source": "doc['price'].value * doc['quantity'].value"}
        }
      },
      "daily_orders": {"value_count": {"field": "product_id"}}
    }
  },
  "sync": {
    "time": {"field": "event_time", "delay": "60s"}
  }
}
```
启动：
```
POST _transform/user_daily_sales_stage1/_start
```
结果文档结构示例（简化）：
```
{
  "user_id": "u1",
  "day": "2025-10-30T00:00:00.000Z",
  "user_id.distinct": "u1",
  "day.date_histogram": "2025-10-30T00:00:00.000Z",
  "daily_revenue.sum": 309.5,
  "daily_orders.value_count": 3
}
```
可使用 ingest pipeline 或应用端重命名字段使更友好。

### 11.3 第2阶段：滚动窗口计算 Transform
第二个 Transform 读取 `user_daily_sales_stage1`，再次按 user_id 分组，并聚合最近 N 天（这里演示 7 天）数据。由于单纯 terms 分组无法限制“最近7天”窗口，我们采用 `filter` + `scripted_metric`，仅在 continuous 模式下对时间范围内的文档计算。

做法：在第二阶段保持 Latest（不合适）或 Pivot 但 group_by 仅 user_id，然后用 `top_hits` 拉出最近7天文档再脚本求和。注意：大量 top_hits 可能影响性能，谨慎使用。

示例：
```
PUT _transform/user_7day_sales
{
  "source": {
    "index": "user_daily_sales_stage1",
    "query": {
      "range": {
        "day.date_histogram": {
          "gte": "now-7d/d"
        }
      }
    }
  },
  "description": "Compute 7-day rolling revenue per user",
  "dest": {"index": "user_daily_sales_7day"},
  "pivot": {
    "group_by": {
      "user_id": {"terms": {"field": "user_id"}}
    },
    "aggregations": {
      "recent_days": {
        "top_hits": {
          "size": 30,
          "sort": [{"day.date_histogram": {"order": "desc"}}],
          "_source": {"includes": ["daily_revenue.sum", "day.date_histogram"]}
        }
      },
      "rolling_7day_revenue": {
        "scripted_metric": {
          "init_script": "state.total = 0;",
          "map_script": "if (params['_source'].containsKey('daily_revenue.sum')) { state.total += params['_source']['daily_revenue.sum']; }",
          "combine_script": "return state.total;",
          "reduce_script": "double t = 0; for (s in states) { t += s } return t;"
        }
      }
    }
  },
  "sync": {
    "time": {"field": "day.date_histogram", "delay": "120s"}
  }
}
```
说明：
- 使用 range query 限制源数据到最近7天。
- `top_hits` 用于后续若要取每日明细（这里也可省略，仅保留滚动和统计）。
- 此方案近似7天滚动窗口，但需要 continuous 模式和正确时间推进。

启动：
```
POST _transform/user_7day_sales/_start
```
查询：
```
GET user_daily_sales_7day/_search
```
性能注意：
- 若用户数巨大，`top_hits.size` 应尽量小。
- 可替换成多个 sum 聚合拆分天数，再 bucket_script 合并（复杂度高）。
- 更优方案：直接使用原始事件索引加 `date_histogram` + `moving_fn` 查询实时计算，而 Transform 仅保留基础日度事实。

### 11.4 替代方案：使用查询时 moving_fn
对实时需求，可跳过第二阶段，直接查询：
```
POST sales_events/_search
{
  "size": 0,
  "query": {"range": {"event_time": {"gte": "now-30d/d"}}},
  "aggs": {
    "per_user": {
      "terms": {"field": "user_id", "size": 100},
      "aggs": {
        "days": {
          "date_histogram": {"field": "event_time", "calendar_interval": "day"},
          "aggs": {
            "revenue": {"sum": {"script": {"source": "doc['price'].value * doc['quantity'].value"}}},
            "rolling_7d": {
              "moving_fn": {
                "buckets_path": "revenue",
                "window": 7,
                "script": "return Collections.sum(values)"
              }
            }
          }
        }
      }
    }
  }
}
```

## 12. 与机器学习异常检测结合
Transform 生成的聚合索引可直接作为 ML Anomaly Detection Job 的数据源（减少原始高基数事件对 Job 的压力）。

### 12.1 场景
基于 `user_daily_sales_stage1` 日度数据，检测用户每日收入是否异常（突出增长或下降）。

### 12.2 创建 Anomaly Detection Job（API）
前提：已开启 X-Pack ML 功能，且有权限。

1. 创建 Job：
```
PUT _ml/anomaly_detectors/user_daily_revenue_job
{
  "description": "Detect anomalies in user daily revenue",
  "analysis_config": {
    "bucket_span": "1d",
    "detectors": [
      {
        "function": "sum",
        "field_name": "daily_revenue.sum",
        "by_field_name": "user_id.distinct"
      }
    ]
  },
  "data_description": {"time_field": "day.date_histogram"}
}
```
注意：字段名需与 Transform 产出一致，可能需要在写入时通过 ingest pipeline 重命名为更简洁（例如 `daily_revenue`）。

2. 创建 Datafeed：
```
PUT _ml/datafeeds/datafeed-user_daily_revenue
{
  "job_id": "user_daily_revenue_job",
  "indices": ["user_daily_sales_stage1"],
  "query": {"match_all": {}},
  "scroll_size": 1000,
  "aggregations": {
    "user": {
      "terms": {"field": "user_id.distinct", "size": 10000},
      "aggs": {
        "by_day": {
          "date_histogram": {"field": "day.date_histogram", "calendar_interval": "day"},
          "aggs": {
            "revenue": {"sum": {"field": "daily_revenue.sum"}}
          }
        }
      }
    }
  }
}
```
如果 Transform 输出已经是日度行（每用户每日期一行），可以简化 Datafeed，不再需要复杂聚合：
```
PUT _ml/datafeeds/datafeed-user_daily_revenue_simple
{
  "job_id": "user_daily_revenue_job",
  "indices": ["user_daily_sales_stage1"],
  "query": {"match_all": {}},
  "scroll_size": 1000
}
```

3. 启动 Datafeed：
```
POST _ml/datafeeds/datafeed-user_daily_revenue_simple/_start
```
或指定时间范围：
```
POST _ml/datafeeds/datafeed-user_daily_revenue_simple/_start
{
  "start": "2025-10-01T00:00:00Z",
  "end": "2025-10-31T00:00:00Z"
}
```

4. 查询 Job 结果（anomalies）：
```
GET _ml/anomaly_detectors/user_daily_revenue_job/results/records
{
  "size": 20,
  "sort": ["record_score"],
  "desc": true
}
```

### 12.3 解释与调优
- 使用 Transform 降低维度：原始事件 -> 日度聚合，减少 Anomaly Detection 需要扫描的文档数量。
- bucket_span 与 Transform 的时间粒度保持一致（1d）。
- 若用户数量极大，可先筛选活跃用户子集索引。

### 12.4 额外建议
- 对高基数实体做拆分多个 Job，避免单 Job 内模型过多。
- 使用自定义前置 Transform 生成更多特征（例如 7 日滚动、变化率），在 Job 中用 `metric` 或多 detector 检测。

## 13. 预测（Forecasting）
利用 Transform 输出的结构化、降噪数据可以更容易做未来趋势预测（如未来7天收入、设备用量）。这里介绍两条路径：

### 13.1 使用 Elasticsearch 内置 ML Forecast
当你已有一个基于 Transform 日度聚合数据的 Anomaly Detection Job（例如 `user_daily_revenue_job`），可以直接调用 `_forecast` 接口让 Elastic ML 在同一建模框架中给出未来区间的预测。

示例：
```
POST _ml/anomaly_detectors/user_daily_revenue_job/_forecast
{
  "duration": "7d"
}
```
返回将包含 forecast_id，你可以使用：
```
GET .ml-anomalies-* /_search
{
  "query": {
    "term": {"forecast_id": "<返回的id>"}
  }
}
```
或在 Kibana ML UI 中查看预测曲线与上下界置信区间。

注意：
- Forecast 依赖现有 Job 的时间序列建模能力（基于季节性、趋势等）。
- 适用单指标或少量分割字段（by_field）。分割过多会影响预测效果与资源使用。
- 建议先让 Job 运行足够的历史窗口（至少多个周期）再做预测。

### 13.2 外部建模（ARIMA/Prophet/深度学习）流程
如果需要更复杂特征或多变量建模，可导出 Transform 产出的索引数据到外部环境（Python/R），进行建模后再写回预测结果索引。

基本步骤：
1. 使用 Scroll 或 `_search` 导出数据：
```
POST user_daily_sales_stage1/_search
{
  "size": 0,
  "aggs": {
    "per_user": {
      "terms": {"field": "user_id.distinct", "size": 1000},
      "aggs": {
        "daily": {
          "date_histogram": {"field": "day.date_histogram", "calendar_interval": "day"},
          "aggs": {
            "revenue": {"sum": {"field": "daily_revenue.sum"}}
          }
        }
      }
    }
  }
}
```
或直接行式数据（如果字段已简化）：
```
GET user_daily_sales_stage1/_search
{
  "size": 10000,
  "sort": [{"day.date_histogram": {"order": "asc"}}]
}
```
2. 在外部（例如 Python Pandas）读入并 pivot 成 `date x user` 或按用户循环训练模型。使用常见库：statsmodels (ARIMA / SARIMAX), Prophet, NeuralProphet, GluonTS, PyTorch.
3. 生成未来 n 天预测数据及区间。结构示例：
```
{
  "user_id": "u1",
  "forecast_date": "2025-11-01",
  "revenue_forecast": 320.4,
  "revenue_lower": 290.0,
  "revenue_upper": 360.0,
  "model": "prophet_v1",
  "generated_at": "2025-10-31T10:00:00Z"
}
```
4. Bulk 写入回 Elasticsearch：
```
POST _bulk
{ "index": {"_index": "user_revenue_forecast"} }
{ "user_id": "u1", "forecast_date": "2025-11-01", "revenue_forecast": 320.4, "revenue_lower": 290.0, "revenue_upper": 360.0, "model": "prophet_v1", "generated_at": "2025-10-31T10:00:00Z" }
```
5. 可视化：在 Kibana Lens 将真实 `daily_revenue` 与预测值合并展示，添加误差带（使用上下界）。

### 13.3 与 Transform 的协同策略
- 使用 Transform 产生“干净”日度或小时级别特征（含滚动窗口、变化率），外部模型只需处理已聚合结果，降低高频噪声。
- 对于多指标预测（收入 + 订单数 + 转化率），可将 Transform 输出合并为单行多列特征索引，再外部多变量建模。
- 预测结果回写后还可再建一个 Latest Transform（按 user + forecast_date）确保每日期只有最新预测版本。

### 13.4 模型效果监控
- 定期将预测值与实际值差异写入另一个误差索引 `user_revenue_forecast_error`。
- 用 Transform 汇总每个用户的平均绝对百分比误差 (MAPE) 与 RMSE：
```
PUT _transform/user_forecast_error_metrics
{
  "source": {"index": "user_revenue_forecast_error"},
  "dest": {"index": "user_forecast_error_metrics"},
  "pivot": {
    "group_by": {"user_id": {"terms": {"field": "user_id"}}},
    "aggregations": {
      "mape": {"avg": {"field": "mape"}},
      "rmse": {"avg": {"field": "rmse"}},
      "count": {"value_count": {"field": "forecast_date"}}
    }
  }
}
```
帮助快速定位预测退化的实体。

### 13.5 何时选内置 Forecast vs 外部模型
| 场景 | 内置 Forecast | 外部模型 |
|------|---------------|----------|
| 快速集成，无需复杂特征 | ✅ | ❌ |
| 多变量、交叉特征 | ❌ | ✅ |
| 可在同一 ML Job 中使用 | ✅ | ❌ |
| 需要自定义神经网络/深度学习 | ❌ | ✅ |
| 运维简单（Kibana UI） | ✅ | ❌ |

### 13.6 小结
- Transform 为时间序列预测提供结构化输入。
- 内置 Forecast 快速、易用，但功能聚焦单指标/简单分割。
- 外部模型灵活，适合多特征与先进算法；需额外基础设施与数据导出流程。

### 13.7 多指标（多 detector）预测示例
当你希望同时预测或监控多个相关指标（例如收入与订单数、或设备的电量平均值），可以在同一个 Job 中配置多个 detector。注意：Elasticsearch 的内置 forecast 是针对指定 job 的整体建模，但结果会为每个 detector 生成预测。复杂多变量相关性建模有限，仍是单变量加分割的组合；若需要真正的多变量联合预测，请参考外部模型方案。

示例：创建一个聚合到用户日度层面的多 detector Job：
```
PUT _ml/anomaly_detectors/user_daily_multi_metric_job
{
  "description": "User daily multi-metric (revenue + orders) forecasting",
  "analysis_config": {
    "bucket_span": "1d",
    "detectors": [
      {
        "function": "sum",
        "field_name": "daily_revenue.sum",
        "by_field_name": "user_id.distinct"
      },
      {
        "function": "value_count",
        "field_name": "product_id",    
        "by_field_name": "user_id.distinct"
      }
    ]
  },
  "data_description": {"time_field": "day.date_histogram"}
}
```
简单 Datafeed（如果 Transform 输出一行即一日一用户）：
```
PUT _ml/datafeeds/datafeed-user_daily_multi_metric
{
  "job_id": "user_daily_multi_metric_job",
  "indices": ["user_daily_sales_stage1"],
  "query": {"match_all": {}},
  "scroll_size": 1000
}
```
启动：
```
POST _ml/datafeeds/datafeed-user_daily_multi_metric/_start
```
获得足够历史后发起预测：
```
POST _ml/anomaly_detectors/user_daily_multi_metric_job/_forecast
{
  "duration": "14d"
}
```
分析要点：
- 每个 detector 独立建模与预测；对多指标间的协方差不做深度推断。
- 可增加一个 detector 监控“sum revenue”与另一个监控“sum revenue”差分（变化率）——差分可通过 Transform 预计算字段写入。
- Detector 数量过多会增加内存占用与计算时间；保持精简。

### 13.8 季节性与 bucket_span 选择指南
`bucket_span` 是 ML Job 的核心参数，决定时间切片粒度。选择得当可让模型更好捕捉季节性（seasonality）与趋势。

常见季节性类型：
- 日内（例如流量在一天中早晚峰值）
- 每周（工作日 vs 周末模式）
- 每月/季度（账单周期、促销）
- 业务自定义周期（例如交易时段）

选择原则：
1. bucket_span 要与指标自然稳定的时间窗口对齐：日度数据用 1d，小时级用 1h；避免过细造成噪声。
2. 若原始事件非常多但有明显分钟级模式，可 Transform 到 5m 或 15m，再让 Job bucket_span=15m。
3. 需要检测快速突变：稍微减少 bucket_span（例如 1h 改为 30m）。
4. 需要平滑并聚焦长周期趋势：增大 bucket_span。

调优步骤：
- 初始设定：使用与 Transform 输出相同的时间粒度（例如 date_histogram interval）。
- 观察结果：若异常过度（太多 false positives），尝试加大 bucket_span 或添加 Influencers 减少维度分裂。
- 对于明显的周季节性：保证有多周历史样本；bucket_span 不要跨越季节性分界过大（例如使用 3d 可能掩盖周模式）。
- 利用 Kibana 诊断：在 Job 结果的“Job messages”与“Model plot”查看是否存在 delayed data 或 bucket processing 占用过高。

实践建议：
- 收入类日度指标：bucket_span=1d；如果需要更细的日内监控，建立第二个 Transform 按小时聚合并新建 Job。
- 流量类高频：Transform到15m或1h，再建 Job；直接用原始事件往往过噪且资源成本高。
- 当不确定时：从较细粒度开始（1h），测算模型质量与资源，再收敛到日度或更粗。

快速检查：
- 是否有足够历史（至少多个季节周期，如周季节性需要>4周）。
- Bucket processing time 与数据写入延迟是否稳定；延迟过高可能需调大 delay 或调整 ingest 时间戳策略。

将这些指南写入你的内部运维文档，形成标准流程：选粒度 -> 运行试验 Job -> 评估异常质量 (precision/recall) -> 调整 bucket_span / detectors / Transform。

## （完）