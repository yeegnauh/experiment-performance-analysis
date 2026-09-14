# 实验清单与指标口径

实验清单是报告前的事实层：它不是替代分析的数据库，也不是要求每次临时排障都填写的表单。对正式基线、性能回归、多轮 A/B、跨环境复现和需要汇报的结果，使用清单减少口径漂移、遗漏产物和“看似可比”的误读。

## 最小 YAML 模板

```yaml
schema_version: 1
id: qwen14b-fsdp16-bf16-20260914
status: validated # planned | running | validated | failed | exploratory
kind: training # training | inference | microbenchmark
objective: 在双机 16 卡条件下测量训练稳态吞吐

identity:
  code:
    repository: org/project
    commit: abcdef123456
  environment:
    image: registry.example.com/project:20260914
    runtime: torch-musa 2.7.1
    driver: 3.3.5-server
  hardware:
    topology: 2 nodes x 8 S5000

workload:
  model: Qwen2.5-14B Dense
  input: synthetic language tokens
  data_boundary: 不含真实数据解码、预处理和 I/O
  shape: seq=4096, micro_batch=1
  seed: 42

configuration:
  precision: bf16
  parallelism: FSDP2 full-shard x16
  effective_global_batch: 128
  key_overrides:
    reshard_after_forward: false

measurement:
  scope: training step
  includes: [forward, backward, optimizer]
  excludes: [model_load, compilation, data_preprocess, io]
  warmup: steps 1-4
  steady_window: steps 5-30
  aggregation: weighted mean; report p50 and p95 when available

metrics:
  - name: iteration_time
    value: 27.6492
    unit: s/iter
    statistic: weighted_mean
    denominator: one optimizer step
    scope: steady_window
  - name: throughput
    value: 18908.76
    unit: tokens/s
    statistic: weighted_mean
    denominator: cluster
    scope: steady_window

validation:
  correctness: completed optimizer steps with expected loss behavior
  stability: 30-step window completed
  resource_boundary: peak allocated 57.77 GiB per GPU

artifacts:
  config: configs/qwen14b_fsdp16.yaml
  logs: s3://experiment-bucket/qwen14b/run-42/train.log
  trace: s3://experiment-bucket/qwen14b/run-42/steady-step.trace
  outputs: null
  checksums: {}

conclusion_boundary:
  comparable_with: bf16 runs using the same model, input, sequence length and measurement scope
  not_comparable_with: runs whose FLOPs formula, precision or E2E inclusion differs
```

## 视频推理补充字段

视频/图像生成的清单应保留上面的 `identity`、`workload`、`configuration`、`measurement`、`validation` 与 `artifacts`，并补充：

```yaml
workload:
  model: MAGI-2 Preview
  input: assets/sample_enhanced_t2v.json
  output: 448x256, 10 s video
  seed: 42

measurement:
  scope: one fresh inference process
  includes: [text_encode, request_init, diffusion, vae, save]
  excludes: [model_load]
  warmup: none
  steady_window: diffusion steps 2-100

metrics:
  - name: e2e_time
    value: 265.655
    unit: s/request
    statistic: single_run
    denominator: one 100-step T2V request
    scope: excludes model load
  - name: stable_diffusion_step
    value: 1.021564
    unit: s/step
    statistic: mean
    denominator: diffusion step
    scope: steps 2-100

validation:
  correctness: 50-step I2V and 100-step T2V passed visual-quality gate
  media_preview: results/run-42/contact-sheet.png

artifacts:
  outputs: results/run-42/final.mp4
  preview: results/run-42/contact-sheet.png
  logs: results/run-42/inference.log
```

## 填写规则

- `status` 描述证据成熟度，不是进度猜测。未完成长稳的实验保持 `running` 或 `exploratory`，不能提供给正式最优表。
- `data_boundary`、`includes` 与 `excludes` 是比较边界的一部分。两个指标数值相近并不等于同一口径。
- `metric` 的 `denominator` 必填：例如单 GPU、集群、一个 optimizer step、一个请求或一个 diffusion step。
- `aggregation`、`warmup` 和 `steady_window` 必须能够让别人复算报告所引的统计值。
- `artifacts` 记录可定位路径；可访问时补充不可变版本或 hash。受权限或敏感数据限制时，记录合规的内部引用，不写入密钥、token 或受限数据。
- `conclusion_boundary` 明确哪些对照可解释、哪些只能并列展示。归因仍需要 profile 和控制变量，不能仅依赖清单。

## 从清单到报告

1. 从同一 `workload`、`configuration` 和 `measurement` 条件的清单生成性能总表。
2. 将 `status: validated` 的结果用于正式结论；把 `running`、`failed` 或 `exploratory` 放入排查记录或更新日志。
3. 报告引述清单中的测量窗口、指标单位和 `data_boundary`，而不是重新手工转写。
4. 需要把报告发布到飞书时，按 [飞书报告与媒体交付](lark-report-delivery.md) 生成原生表格、代码块和附件区。
