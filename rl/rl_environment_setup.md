# RL Stock 环境设置详解

本文档详细介绍 RL Stock 训练脚本中的环境创建和配置。

## 环境创建架构

```
数据 (DataFrame)
    ↓
预处理器 (RLPreprocessor)
    ↓
环境工厂 (_make_env) × N
    ↓
向量环境 (VecEnv)
    ↓
归一化包装器 (VecNormalize)
    ↓
训练/评估环境
```

## 核心组件

### 1. 环境工厂函数 `_make_env`

```python
def _make_env(
    df: pd.DataFrame,
    config: RlStockConfig,
    preprocessor: RLPreprocessor,
    mode: str = "train",
):
    """创建环境工厂函数（避免 lambda 闭包问题）。"""
    from stable_baselines3.common.monitor import Monitor

    def _init():
        env = PortfolioTradingEnv(
            df=df,
            config=config,
            preprocessor=preprocessor,
            mode=mode,
        )
        return Monitor(env)

    return _init
```

**设计要点：**

- **避免 Lambda 闭包问题**：使用嵌套函数而非 lambda
- **Monitor 包装**：记录 episode 统计数据
- **参数隔离**：每个环境独立初始化

### 2. 向量环境选择

脚本支持两种向量环境后端：

#### DummyVecEnv（串行）

```python
train_env = DummyVecEnv([_make_env(...) for _ in range(config.n_envs)])
```

**特点：**
- 单进程串行执行
- 内存共享，效率高
- 适合调试和小规模训练
- 当数据量大时自动降级

#### SubprocVecEnv（并行）

```python
train_env = SubprocVecEnv(env_fns, start_method="spawn")
```

**特点：**
- 多进程并行执行
- 每个进程独立内存空间
- 训练速度更快
- 内存占用更大

**启动方法选择：**

```python
# 使用 spawn 方法避免 CUDA fork 问题
train_env = SubprocVecEnv(env_fns, start_method="spawn")
```

### 3. 观测归一化 VecNormalize

```python
# 训练环境 - 归一化观测和奖励
train_env = VecNormalize(
    train_env,
    norm_obs=True,      # 归一化观测
    norm_reward=True,   # 归一化奖励
    clip_obs=10.0,      # 裁剪观测值
)

# 评估环境 - 只归一化观测，不归一化奖励
eval_env = VecNormalize(
    eval_env,
    norm_obs=True,
    norm_reward=False,  # 评估时使用原始奖励
    clip_obs=10.0,
    training=False,     # 不更新统计量
)

# 共享统计量
eval_env.obs_rms = train_env.obs_rms
eval_env.clip_obs = train_env.clip_obs
eval_env.training = False
```

**关键参数：**

| 参数 | 训练环境 | 评估环境 | 说明 |
|------|----------|----------|------|
| norm_obs | True | True | 归一化观测 |
| norm_reward | True | False | 评估用原始奖励 |
| clip_obs | 10.0 | 10.0 | 裁剪范围 |
| training | True | False | 是否更新统计量 |

## 内存优化策略

### 自动检测和降级

```python
obs_mem_bytes = _estimate_observation_bytes(train_df, preprocessor, config)
estimated_total_bytes = obs_mem_bytes * config.n_envs
gb = 1024**3
backend = resolve_vec_backend(config)

if backend == "subproc" and estimated_total_bytes > 6 * gb:
    print("⚠️ 数据体积较大，多进程会复制观测张量")
    print(f"   单个环境 ~{obs_mem_bytes / gb:.1f} GB，总计 ~{estimated_total_bytes / gb:.1f} GB")
    print("   自动降级为 DummyVecEnv 并将 n_envs 调整为 1")
    config.n_envs = 1
    backend = "dummy"
```

**内存计算公式：**

```
观测内存 = n_days × lookback × n_stocks × n_features × 4 bytes (float32)

示例：
- 交易日: 500 天
- 回看窗口: 30 天
- 股票数: 300 只
- 特征数: 20 个
- 内存 = 500 × 30 × 300 × 20 × 4 = 360,000,000 bytes ≈ 343 MB
```

### 后端选择逻辑

```python
def resolve_vec_backend(config: RlStockConfig) -> str:
    if config.vec_backend == "auto":
        return "subproc" if config.n_envs > 1 else "dummy"
    return config.vec_backend
```

**选择策略：**
- `auto`: 根据 n_envs 自动选择
- `n_envs > 1`: 使用 SubprocVecEnv
- `n_envs = 1`: 使用 DummyVecEnv

## 训练环境与评估环境的区别

### 训练环境

```python
train_env_fns = [
    _make_env(train_df, config, preprocessor, "train") 
    for _ in range(config.n_envs)
]
train_env = SubprocVecEnv(train_env_fns, start_method="spawn")
train_env = VecNormalize(train_env, norm_obs=True, norm_reward=True, clip_obs=10.0)
```

**特点：**
- 使用训练数据
- 多环境并行
- 归一化观测和奖励
- 实时更新统计量

### 评估环境

```python
eval_env = DummyVecEnv([_make_env(eval_df, config, preprocessor, "val")])
eval_env = VecNormalize(eval_env, norm_obs=True, norm_reward=False, clip_obs=10.0, training=False)
eval_env.obs_rms = train_env.obs_rms  # 共享统计量
eval_env.training = False
```

**特点：**
- 使用验证数据
- 单环境串行
- 不归一化奖励（真实奖励）
- 使用训练环境的归一化统计量
- 不更新统计量

## 环境配置参数

### 关键配置项

```python
@dataclass
class RlStockConfig:
    n_envs: int = 4                    # 并行环境数量
    vec_backend: str = "auto"          # 向量环境后端
    device: str = "auto"               # 计算设备
    
    # 环境参数
    lookback: int = 30                 # 观测历史窗口
    initial_balance: float = 1_000_000.0  # 初始资金
    transaction_cost: float = 0.001    # 交易成本
    trade_threshold: float = 0.01      # 交易阈值
```

### 环境变量

```bash
# 限制股票数量（调试时使用）
export RL_STOCK_MAX_CODES=50
```

## 常见问题

### Q: 为什么训练时会内存溢出？

**原因：**
- SubprocVecEnv 会复制观测数据到每个进程
- 数据量大时内存占用剧增

**解决：**
- 自动降级机制会在内存超过 6GB 时切换到 DummyVecEnv
- 或手动设置 `--n-envs 1`

### Q: 评估时奖励和训练时不一致？

**原因：**
- 训练环境归一化了奖励
- 评估环境使用原始奖励

**解决：**
- 这是设计如此，评估时使用真实奖励更有意义

### Q: 为什么使用 Monitor 包装环境？

**原因：**
- Monitor 记录 episode 长度和奖励
- 用于 TensorBoard 和日志输出

## 最佳实践

1. **调试时使用单环境**：`--n-envs 1`
2. **生产环境使用多环境**：`--n-envs 4` 或更高
3. **监控内存使用**：大股票池时降低 n_envs
4. **验证集独立**：确保评估环境使用独立数据