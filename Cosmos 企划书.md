### 2.1 项目定位

Cosmos 是以缠论为底层语言的全天候量化操作系统，处理 1m/15m/30m/4h 四个周期的结构识别。核心竞争力：**骨架层维护全局拓扑一致性，AI 感知层识别 Regime，肌肉层因子插件随市场状态动态路由**，三层解耦，互不侵入。

### 2.2 数据流向（对应文档中的架构图）

```
OKX REST (历史) / WebSocket (实时)
  └── Eye Layer (Ring Buffer 4个周期)
        ├── Mutable Structure Layer  (分型/笔/线段, 增量更新)
        │     └── InvalidationProposal → ProposalQueue
        │                              → SAL Authority Validation
        │                              → Frozen Topology Layer (已确认中枢/趋势)
        │
        └── AI Perception Layer
              ├── Regime Classifier  → RegimeTag (枚举, 5值)
              ├── Parameter Router   → VALIDATED_PARAM_LIBRARY 查表
              └── AI Watchdog        → 报警 (不执行)

Frozen Topology Layer + RegimeTag + ActiveParams
  └── Neuron Signal Graph (L1 4h → L2 30m → L3 15m → L4 1m)
        └── Signal Decay Engine  S(t) = S₀·e^(-λt)
              └── Score Gate (Divergence · Hurst · Entropy)
                    └── Risk Survival Gate
                          └── Execution Validation (5重校验)
                                └── Execution Arm (下单/止损/止盈)
```

### 2.3 模块解耦设计

#### 骨架层（永久封存，不可修改）

|模块|文件|职责|
|---|---|---|
|`eye_layer`|`core/eye_layer.py`|Ring Buffer 预分配，O(1) K线写入|
|`topology.mutable`|`core/topology/mutable.py`|分型/笔/线段增量识别，有限回溯|
|`topology.frozen`|`core/topology/frozen.py`|已确认中枢，只读，版本单调递增|
|`topology.proposal`|`core/topology/proposal.py`|`InvalidationProposal` + `ProposalQueue`|
|`topology.sal`|`core/topology/sal.py`|SAL 权威验证，Commit/Reject|
|`risk_gate`|`core/risk_gate.py`|`P_survival` 最大化，风险恒定模型|
|`execution_validation`|`core/exec_validation.py`|5重校验（版本/Regime/时效/强度/盘口）|
|`execution_arm`|`core/exec_arm.py`|下单，ATR 追踪止损/止盈|

#### AI 感知层（可热替换）

|模块|文件|职责|
|---|---|---|
|`regime_classifier`|`ai/regime_classifier.py`|输出 `RegimeTag` 枚举，Walk-Forward 训练|
|`parameter_router`|`ai/parameter_router.py`|查 `VALIDATED_PARAM_LIBRARY`，禁止生成新参数|
|`ai_watchdog`|`ai/watchdog.py`|异常监控，4级报警，不执行交易|

#### 肌肉层（可插拔）

|模块|文件|职责|
|---|---|---|
|`factor_registry`|`muscle/factor_registry.py`|因子插件注册，健康度管理|
|`neuron_l1~l4`|`muscle/neurons/`|L1(4h)~L4(1m) 分级神经元|
|`signal_decay`|`muscle/signal_decay.py`|波动率自适应衰减|
|`score_gate`|`muscle/score_gate.py`|Hurst 二值门控 + 熵门控|

#### 工具与基础设施

|模块|文件|职责|
|---|---|---|
|`data_client`|`infra/data_client.py`|OKX REST/WS 数据接入，**显式 1082 代理**|
|`visual_logger`|`infra/visual_logger.py`|异步心电图日志，不阻塞主流程|
|`walkforward`|`research/walkforward.py`|参数稳定岛搜索，多目标适应度|

### 2.4 状态一致性保证

`topology_version` 全局单调递增（`threading.atomic` / `asyncio.Lock` 双重保护）。每次 Execution Validation 的第一步必须检查信号携带的 `topology_version` 与当前版本一致，版本不符立即拒绝执行。

### 2.5 Rebuild 冷却机制

三触发条件（冲突率/中枢异常扩张/分型振荡）任一满足触发 Rebuild，触发后进入对应周期不应期（1m: 15min, 15m: 2h, 30m: 4h），期间拒绝同级 Rebuild 请求。