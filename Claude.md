# Cosmos — Claude Code 员工手册

## 你的角色
你是 Cosmos 全天候量化操作系统的开发执行者。
Cosmos 分为三层：骨架（不可替换）、肌肉（可插拔）、AI感知（可替换）。
**骨架层的任何接口和逻辑，未经 Chief Architect 书面确认，禁止修改。**

---

## 绝对铁律

### 1. 骨架层封存原则
以下文件一旦通过 Review 进入代码库，禁止在未获批准的情况下修改：
- `core/topology/frozen.py`
- `core/topology/sal.py`
- `core/topology/proposal.py`
- `core/risk_gate.py`
- `core/exec_validation.py`

如需修改，必须在 commit message 中注明 `[SKELETON CHANGE - APPROVED]`，并附上 Chief Architect 的批准说明。

### 2. 异步代理硬编码
\```python
# 所有 OKX 数据客户端初始化
self._exchange = ccxt.okx({
    "apiKey": self._cfg["api_key"],
    "secret": self._cfg["secret"],
    "password": self._cfg["passphrase"],
    "aiohttp_proxy": "http://127.0.0.1:1082",  # 硬编码，必须
    "options": {"defaultType": "swap"},
})
\```

### 3. topology_version 单调性
任何写入 `FrozenTopologyLayer` 的操作必须通过 `SAL.commit()`，禁止直接赋值：
\```python
# ✅ 正确
sal.commit(proposal)  # 内部自动递增 topology_version

# ❌ 错误
frozen_layer.topology_version += 1  # 绕过 SAL
\```

### 4. AI 层不可生成新参数
`ParameterRouter` 只能从 `VALIDATED_PARAM_LIBRARY` 查表，禁止动态生成未经 Walk-Forward 验证的参数值：
\```python
# ✅ 正确
params = VALIDATED_PARAM_LIBRARY[regime]

# ❌ 错误
params["wma_period"] = model.predict(features)  # 未验证参数
\```

### 5. 目录隔离
- 禁止写入 `cosmos/` 目录外的任何文件
- `cosmos/data/` 已被 `.gitignore` 覆盖，Tick 数据不得上传

---

## Python 3.12 规范

- `slots=True` 的 dataclass 用于所有高频路径上的数据结构
- `InvalidationProposal` 等核心结构使用 `__slots__` 保证内存效率
- `typing.Protocol` 定义插件接口，不使用 ABC（Protocol 更灵活，支持鸭子类型）
- 骨架层全部使用同步代码 + `asyncio.Lock`，避免复杂的异步状态机

## Polars 优先原则

所有 K线、分型、中枢的批量计算必须使用 Polars：

\```python
# ✅ K线数据处理
klines = pl.DataFrame({
    "open": ..., "high": ..., "low": ..., "close": ..., "volume": ...
}).with_columns([
    pl.col("high").rolling_max(20).alias("hh20"),
    pl.col("low").rolling_min(20).alias("ll20"),
])

# WMA 计算使用 numba JIT 加速（单根K线增量更新场景）
\```

---

## 错误捕获规范

### Proposal 拒绝处理
\```python
result = sal.validate(proposal)
if result == ValidationResult.REJECTED:
    # 只记录，不抛异常，不影响主流程
    logger.info(f"Proposal {proposal.proposal_id} 被拒绝: {result.reason}")
    return
elif result == ValidationResult.COMMITTED:
    # 通知下游订阅者（通过事件总线，不直接调用）
    event_bus.emit(TopologyUpdatedEvent(version=frozen.topology_version))
\```

### 黑天鹅紧急解冻
\```python
if check_emergency_unfreeze(zhongshu, price, atr_30m):
    logger.critical("触发紧急解冻！价格偏离 > 3×ATR，启动 Rebuild")
    # 写入 Rebuild 冷却记录，防止振荡陷阱
    rebuild_manager.trigger(timeframe="30m")
\```

### WS 数据流重连（与 Minisha 相同的指数退避策略）
所有外部 IO 使用指数退避重连，上限 60s。

---

## 层级开发规范

### 新增肌肉层因子（FactorPlugin）
1. 在 `muscle/neurons/` 新建文件，继承 `FactorPlugin`
2. 实现 `compute(data: pl.DataFrame) -> float`，输出范围严格限制在 `[-1, 1]`
3. 实现 `health_check() -> float`，使用近 N 期 IC 滚动均值
4. 在 `FactorRegistry.register()` 中注册
5. 禁止在因子内部调用任何骨架层接口

### 替换 AI 感知层
1. 新建实现文件，遵守 `RegimeClassifierProtocol`
2. 输出必须是 `RegimeTag` 枚举，禁止输出连续评分
3. Regime 变更必须通过 `RegimeChangeProposal`，置信度 < 0.70 自动被 SAL 拒绝
4. 替换时旧实现保留（不删除），通过配置文件切换

---

## 核心依赖版本

| 包 | 版本 | 用途 |
|----|------|------|
| polars | 1.40.1 | 主力数据处理 |
| torch | 2.12.0 | AI 感知层（MPS加速） |
| scikit-learn | 1.8.0 | Regime Classifier |
| numba | 0.61.2 | JIT 加速热点计算 |
| ccxt | 4.5.54 | OKX 接口 |

---

## 禁止事项速查

| 禁止 | 原因 |
|------|------|
| 直接修改 frozen_layer 属性 | 必须通过 SAL.commit() |
| AI 层生成未验证参数 | 只能查 VALIDATED_PARAM_LIBRARY |
| Watchdog 执行任何交易动作 | Watchdog 只报警 |
| 双向持仓 | 止损规则明确禁止 |
| 跨仓库写文件 | 目录隔离铁律 |
| `import pandas` | 用 Polars |