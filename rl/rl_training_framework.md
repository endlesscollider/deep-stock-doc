# RL Training Framework Documentation

This document provides comprehensive documentation for the reinforcement learning training framework used in the Deep Stock project, covering PPO algorithm configuration, training loop execution, model architecture, callbacks, checkpoint management, and TensorBoard monitoring.

## Table of Contents

1. [PPO Algorithm Configuration](#1-ppo-algorithm-configuration)
2. [Training Loop and Execution Flow](#2-training-loop-and-execution-flow)
3. [Model Architecture and Network Design](#3-model-architecture-and-network-design)
4. [Callback Functions and Their Roles](#4-callback-functions-and-their-roles)
5. [Checkpoint Saving and Model Persistence](#5-checkpoint-saving-and-model-persistence)
6. [Training Monitoring with TensorBoard](#6-training-monitoring-with-tensorboard)

---

## 1. PPO Algorithm Configuration

### 1.1 Configuration Class Overview

All training parameters are managed through the `RlStockConfig` dataclass, ensuring type safety and validation:

```python
from src.train.rl_stock.config import RlStockConfig

config = RlStockConfig(
    stock_pool="hs300",
    total_timesteps=500000,
    learning_rate=3e-4,
    batch_size=4096,
    n_epochs=8,
    gamma=0.99,
    gae_lambda=0.95,
    clip_range=0.2,
    ent_coef=0.02,
)
```

### 1.2 Core PPO Hyperparameters

| Parameter | Default | Range | Description |
|-----------|---------|-------|-------------|
| `learning_rate` | 3e-4 | > 0 | Learning rate for optimizer. Can be a schedule function. |
| `batch_size` | 4096 | > 0, must divide `n_steps × n_envs` | Mini-batch size for gradient updates. |
| `n_epochs` | 8 | > 0 | Number of optimization epochs per update. |
| `gamma` | 0.99 | (0, 1] | Discount factor for future rewards. |
| `gae_lambda` | 0.95 | (0, 1] | GAE (Generalized Advantage Estimation) parameter. |
| `clip_range` | 0.2 | (0, 1) | PPO clipping parameter. |
| `ent_coef` | 0.02 | >= 0 | Entropy coefficient for exploration bonus. |
| `vf_coef` | 0.5 | >= 0 | Value function loss coefficient. |
| `n_steps` | 2048 | > 0 | Steps per environment per update. |

### 1.3 Parameter Validation

The configuration class includes comprehensive validation:

```python
def __post_init__(self) -> None:
    """Validate configuration parameters after initialization."""
    self._validate_stock_pool()
    self._validate_environment_params()
    self._validate_training_params()
    self._validate_ppo_params()
    self._validate_reward_params()
    self._validate_backtest_params()
    self._validate_split_ranges()
```

Example validation for PPO parameters:

```python
def _validate_ppo_params(self) -> None:
    if not 0 < self.gamma <= 1:
        raise ValueError(f"gamma must be in (0, 1], got {self.gamma}")
    if not 0 < self.gae_lambda <= 1:
        raise ValueError(f"gae_lambda must be in (0, 1], got {self.gae_lambda}")
    if not 0 < self.clip_range < 1:
        raise ValueError(f"clip_range must be in (0, 1), got {self.clip_range}")
```

### 1.4 Environment Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `stock_pool` | "hs300" | Stock universe (hs300/zz500/zz1000). |
| `initial_balance` | 1,000,000 | Starting portfolio value. |
| `transaction_cost` | 0.001 | Transaction fee rate (0.1%). |
| `lookback` | 30 | Historical window size for observations. |
| `trade_threshold` | 0.01 | Minimum weight change to execute trade. |
| `rebalance_interval_days` | 1 | Days between rebalancing (1=daily, 5=weekly). |

### 1.5 Reward Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `reward_scaling` | 10.0 | Scale factor for reward signal. |
| `reward_clip` | 5.0 | Maximum absolute reward value. |

### 1.6 Parallel Environment Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `n_envs` | 8 | Number of parallel environments. |
| `vec_backend` | "auto" | Vectorization backend (auto/dummy/subproc). |
| `device` | "auto" | Compute device (auto/cpu/cuda). |

The backend is automatically resolved:

```python
def resolve_vec_backend(config: RlStockConfig) -> str:
    if config.vec_backend == "auto":
        return "subproc" if config.n_envs > 1 else "dummy"
    return config.vec_backend
```

---

## 2. Training Loop and Execution Flow

### 2.1 Complete Training Pipeline

The training process follows a structured pipeline:

```
┌─────────────────────────────────────────────────────────────┐
│                    Configuration Parse                       │
│                  parse_args_to_config()                      │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Data Preparation                          │
│              load_training_dataframe()                       │
│              prepare_datasets()                              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Preprocessing                             │
│              RLPreprocessor.fit()                            │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Environment Creation                      │
│              _make_env() → VecNormalize                      │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Model Creation                            │
│              PPO("MlpPolicy", train_env, ...)               │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Training Loop                             │
│              model.learn(total_timesteps, callbacks)        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Model Persistence                         │
│              model.save(), vec_normalize.save()             │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Data Preparation Stage

```python
def prepare_datasets(config: RlStockConfig) -> Tuple[pd.DataFrame, pd.DataFrame]:
    """Prepare training and validation datasets."""
    df = load_training_dataframe(config).copy()
    df["trade_date"] = df["trade_date"].astype(str)
    
    train_df = _filter_by_date_range(df, config.start_date, config.end_date)
    eval_df = _filter_by_date_range(
        df,
        config.val_start_date or config.start_date,
        config.val_end_date or config.end_date,
    )
    
    # Fallback handling
    if train_df.empty:
        train_df = df.copy()
    if eval_df.empty:
        tail_size = max(int(len(train_df) * 0.2), config.lookback * 2)
        eval_df = train_df.tail(tail_size)
    
    return train_df.reset_index(drop=True), eval_df.reset_index(drop=True)
```

### 2.3 Preprocessing Stage

The `RLPreprocessor` handles feature normalization:

```python
preprocessor = RLPreprocessor(lookback=config.lookback)
preprocessor.fit(train_df)
```

Feature groups and normalization methods:

| Group | Features | Normalization |
|-------|----------|---------------|
| price | open, high, low, close | StandardScaler |
| volume | vol, amount | RobustScaler |
| ratio | pe, pb, ps, turnover_rate | RobustScaler |
| market_value | total_mv, circ_mv | RobustScaler |
| financial | roe, eps, bps | RobustScaler |

### 2.4 Environment Creation

```python
def _make_env(df, config, preprocessor, mode="train"):
    """Factory function for environment creation."""
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

Vectorized environment setup:

```python
train_env_fns = [
    _make_env(train_df, config, preprocessor, "train") 
    for _ in range(config.n_envs)
]

if backend == "subproc":
    train_env = SubprocVecEnv(train_env_fns, start_method="spawn")
else:
    train_env = DummyVecEnv(train_env_fns)

train_env = VecNormalize(
    train_env, 
    norm_obs=True, 
    norm_reward=True, 
    clip_obs=10.0
)
```

### 2.5 Main Training Loop

The core training call:

```python
model.learn(
    total_timesteps=config.total_timesteps,
    callback=callbacks,
)
```

PPO internally performs:

1. **Rollout Phase**: Collect `n_steps × n_envs` transitions
2. **Advantage Computation**: Calculate GAE advantages
3. **Optimization Phase**: Update policy `n_epochs` times with mini-batches
4. **Logging**: Record metrics to TensorBoard

---

## 3. Model Architecture and Network Design

### 3.1 Policy Network Architecture

The PPO model uses a separated actor-critic architecture:

```python
policy_kwargs = dict(
    net_arch=dict(
        pi=[128, 64],   # Actor network: 128 → 64 hidden units
        vf=[128, 64],   # Critic network: 128 → 64 hidden units
    ),
    log_std_init=-2.0,  # Initial action standard deviation (exp(-2) ≈ 0.135)
)

model = PPO(
    "MlpPolicy",
    train_env,
    learning_rate=lr_schedule,
    n_steps=config.n_steps,
    batch_size=config.batch_size,
    n_epochs=config.n_epochs,
    gamma=config.gamma,
    gae_lambda=config.gae_lambda,
    clip_range=config.clip_range,
    ent_coef=config.ent_coef,
    vf_coef=config.vf_coef,
    max_grad_norm=0.5,
    policy_kwargs=policy_kwargs,
    verbose=1,
    tensorboard_log=tb_log_dir,
    device=device,
)
```

### 3.2 Network Structure Diagram

```
Observation (lookback, n_stocks, n_features)
                    │
                    ▼
              ┌───────────┐
              │  Flatten  │
              └───────────┘
                    │
        ┌───────────┴───────────┐
        │                       │
        ▼                       ▼
┌───────────────┐       ┌───────────────┐
│ Actor (π)     │       │ Critic (V)    │
│ Linear 128    │       │ Linear 128    │
│ ReLU          │       │ ReLU          │
│ Linear 64     │       │ Linear 64     │
│ ReLU          │       │ ReLU          │
│ Linear n_act  │       │ Linear 1      │
└───────────────┘       └───────────────┘
        │                       │
        ▼                       ▼
   Action (n_stocks+1)    Value Estimate
```

### 3.3 Learning Rate Schedule

A linear decay schedule is used:

```python
class LinearLRSchedule:
    def __init__(self, initial_lr: float, total_timesteps: int):
        self.initial_lr = initial_lr
        self.total_timesteps = total_timesteps
    
    def __call__(self, progress_remaining: float) -> float:
        """Progress remaining: 1.0 at start, 0.0 at end."""
        return progress_remaining * self.initial_lr

lr_schedule = LinearLRSchedule(config.learning_rate, config.total_timesteps)
```

### 3.4 Action Space

The action space is a continuous Box representing portfolio weights:

```python
self.action_space = spaces.Box(
    low=0.0,
    high=1.0,
    shape=(self.n_stocks + 1,),  # n_stocks + 1 for cash
    dtype=np.float32,
)
```

Actions are normalized via softmax:

```python
def _normalize_action(self, action: np.ndarray) -> np.ndarray:
    action = np.clip(action, 0, None)
    exp_action = np.exp(action - np.max(action))
    normalized = exp_action / (np.sum(exp_action) + 1e-8)
    return normalized
```

### 3.5 Observation Space

The observation is a 3D tensor:

```python
self.observation_space = spaces.Box(
    low=-np.inf,
    high=np.inf,
    shape=(self.lookback, self.n_stocks, self.n_features),
    dtype=np.float32,
)
```

Shape breakdown:

- `lookback`: Historical window (default 30 days)
- `n_stocks`: Number of stocks in universe
- `n_features`: Number of features per stock (default 15)

---

## 4. Callback Functions and Their Roles

### 4.1 Callback Overview

The framework uses four callbacks for comprehensive monitoring:

```python
callbacks = CallbackList([
    trading_metrics_callback,  # Custom: Trading metrics
    backtest_callback,         # Custom: Periodic backtest
    eval_callback,             # SB3: Best model saving
    checkpoint_callback,       # SB3: Periodic checkpoints
])
```

### 4.2 TradingMetricsCallback

Records trading-specific metrics during training:

```python
class TradingMetricsCallback(BaseCallback):
    def __init__(self, verbose: int = 0, log_freq: int = 1):
        super().__init__(verbose)
        self.log_freq = log_freq
        self._rollout_count = 0
        self._initial_portfolio_value = None
    
    def _on_step(self) -> bool:
        """Called after each environment step."""
        infos = self.locals.get("infos", [])
        for info in infos:
            if isinstance(info, dict) and "episode" in info:
                episode_info = info["episode"]
                episode_return = episode_info.get("r", 0)
                episode_length = episode_info.get("l", 0)
                
                self.logger.record("portfolio/episode_return_mean", float(episode_return))
                self.logger.record("rollout/ep_len_mean", float(episode_length))
        return True
    
    def _on_rollout_end(self) -> None:
        """Called after each rollout (n_steps collected)."""
        weights = self._get_current_weights()
        if weights is None:
            return
        
        # Log position metrics
        self.logger.record("trading/cash_weight", float(weights[-1]))
        position_count = np.sum(weights[:-1] > 0.01)
        self.logger.record("trading/position_count", int(position_count))
        self.logger.record("trading/max_position", float(np.max(weights[:-1])))
```

**Logged Metrics:**

| Metric | Description |
|--------|-------------|
| `portfolio/episode_return_mean` | Mean episode return |
| `trading/cash_weight` | Cash allocation ratio |
| `trading/position_count` | Number of positions > 1% |
| `trading/max_position` | Largest single position |
| `portfolio/total_return` | Cumulative return since training start |

### 4.3 BacktestCallback

Performs periodic backtests on validation data:

```python
class BacktestCallback(BaseCallback):
    def __init__(
        self,
        eval_env: VecEnv,
        eval_freq_episodes: int = 10,
        n_eval_episodes: int = 1,
        deterministic: bool = True,
        verbose: int = 0,
    ):
        super().__init__(verbose)
        self.eval_env = eval_env
        self.eval_freq_episodes = eval_freq_episodes
        self.n_eval_episodes = n_eval_episodes
        self._episode_count = 0
    
    def _on_step(self) -> bool:
        """Check if backtest should run."""
        for info in self.locals.get("infos", []):
            if isinstance(info, dict) and "episode" in info:
                self._episode_count += 1
        
        if self._episode_count >= self.eval_freq_episodes:
            self._run_backtest()
            self._episode_count = 0
        return True
```

**Backtest Metrics:**

```python
def _run_episode(self, env: VecEnv) -> Dict:
    """Run a complete episode and compute metrics."""
    portfolio_values = [self._get_initial_balance(env)]
    
    while not done:
        action, _ = self.model.predict(obs, deterministic=self.deterministic)
        obs, _, done, info = env.step(action)
        portfolio_values.append(info["portfolio_value"])
    
    values = np.array(portfolio_values)
    
    # Compute metrics
    return_multiple = values[-1] / values[0]
    annual_return = (return_multiple ** (252 / n_days) - 1)
    max_drawdown = np.min((values - np.maximum.accumulate(values)) / np.maximum.accumulate(values))
    sharpe_ratio = np.mean(returns) / np.std(returns) * np.sqrt(252)
    
    return {
        "return_multiple": return_multiple,
        "annual_return": annual_return,
        "max_drawdown": max_drawdown,
        "sharpe_ratio": sharpe_ratio,
    }
```

**Logged Metrics:**

| Metric | Description |
|--------|-------------|
| `backtest/eval_return` | Return multiple (final/initial) |
| `backtest/eval_annual` | Annualized return |
| `backtest/eval_max_dd` | Maximum drawdown |
| `backtest/eval_sharpe` | Sharpe ratio |

### 4.4 EvalCallback (SB3 Built-in)

Saves the best model based on evaluation performance:

```python
eval_freq = max(config.n_steps * 10 // config.n_envs, 1)
eval_callback = EvalCallback(
    eval_env,
    best_model_save_path=os.path.join(experiment_dir, "best_model"),
    log_path=os.path.join(experiment_dir, "eval_results"),
    eval_freq=eval_freq,
    n_eval_episodes=5,
    deterministic=True,
)
```

### 4.5 CheckpointCallback (SB3 Built-in)

Saves periodic training checkpoints:

```python
checkpoint_freq = max(50000 // config.n_envs, 1)
checkpoint_callback = CheckpointCallback(
    save_freq=checkpoint_freq,
    save_path=os.path.join(experiment_dir, "checkpoints"),
    name_prefix="rl_stock",
)
```

---

## 5. Checkpoint Saving and Model Persistence

### 5.1 Saved Artifacts

After training, the following artifacts are saved:

```
runs/rl_stock/rl_stock_{timestamp}/
├── config.json              # Training configuration
├── final_model.zip          # Final trained model
├── preprocessor.pkl         # Feature preprocessor
├── vec_normalize.pkl        # VecNormalize statistics
├── best_model/              # Best model by evaluation
│   └── best_model.zip
├── eval_results/            # Evaluation logs
│   └── evaluations.npz
└── checkpoints/             # Training checkpoints
    ├── rl_stock_50000_steps.zip
    ├── rl_stock_100000_steps.zip
    └── ...
```

### 5.2 Model Saving

```python
# Save final model
model_path = os.path.join(experiment_dir, "final_model.zip")
model.save(model_path)

# Save VecNormalize statistics
vec_normalize_path = os.path.join(experiment_dir, "vec_normalize.pkl")
train_env.save(vec_normalize_path)

# Save preprocessor
preprocessor_path = os.path.join(experiment_dir, "preprocessor.pkl")
preprocessor.save(preprocessor_path)
```

### 5.3 Configuration Serialization

```python
def _serialize_config(config: RlStockConfig) -> dict:
    """Serialize config to JSON-compatible dict."""
    config_dict = asdict(config)
    config_dict_serializable = {}
    for key, value in config_dict.items():
        if value is None:
            config_dict_serializable[key] = None
        elif isinstance(value, (list, dict, str, int, float, bool)):
            config_dict_serializable[key] = value
        else:
            config_dict_serializable[key] = str(value)
    return config_dict_serializable

# Save config
config_path = os.path.join(experiment_dir, "config.json")
with open(config_path, "w", encoding="utf-8") as f:
    json.dump(_serialize_config(config), f, ensure_ascii=False, indent=2)
```

### 5.4 Loading a Trained Model

```python
from stable_baselines3 import PPO
from stable_baselines3.common.vec_env import VecNormalize
from src.train.rl_stock.preprocessor import RLPreprocessor

# Load model
model = PPO.load("path/to/model.zip")

# Load VecNormalize
vec_normalize = VecNormalize.load("path/to/vec_normalize.pkl")

# Load preprocessor
preprocessor = RLPreprocessor.load("path/to/preprocessor.pkl")

# For inference
obs, _ = env.reset()
normalized_obs = vec_normalize.normalize_obs(obs)
action, _ = model.predict(normalized_obs, deterministic=True)
```

### 5.5 Resuming Training from Checkpoint

```python
from stable_baselines3 import PPO

# Load checkpoint
model = PPO.load("path/to/checkpoint.zip")

# Continue training
model.learn(total_timesteps=additional_steps, callback=callbacks)
```

---

## 6. Training Monitoring with TensorBoard

### 6.1 Enabling TensorBoard

```python
config = RlStockConfig(
    use_tensorboard=True,  # Default
)

# In training script
if config.use_tensorboard:
    tb_log_dir = os.path.join(experiment_dir, "logs")
    os.makedirs(tb_log_dir, exist_ok=True)
```

### 6.2 Starting TensorBoard

```bash
# Start for all experiments
tensorboard --logdir=runs/rl_stock/

# Start for specific experiment
tensorboard --logdir=runs/rl_stock/rl_stock_20240101_120000/logs
```

### 6.3 PPO Internal Metrics

| Metric | Description | Healthy Range |
|--------|-------------|---------------|
| `rollout/ep_rew_mean` | Mean episode reward | Increasing, then stable |
| `rollout/ep_len_mean` | Mean episode length | Stable |
| `train/policy_gradient_loss` | Policy gradient loss | Decreasing trend |
| `train/value_loss` | Value function loss | Decreasing trend |
| `train/entropy_loss` | Entropy (exploration) | Positive, decreasing |
| `train/approx_kl` | Approximate KL divergence | < 0.01 |
| `train/clip_fraction` | Fraction of clipped updates | 0.1 - 0.3 |
| `train/explained_variance` | Value function quality | Near 1.0 |
| `train/learning_rate` | Current learning rate | Per schedule |

### 6.4 Trading Metrics

| Metric | Description | Interpretation |
|--------|-------------|----------------|
| `portfolio/total_return` | Cumulative return | Should increase over time |
| `portfolio/episode_return_mean` | Mean episode return | Positive = profitable |
| `trading/cash_weight` | Cash allocation | High = conservative |
| `trading/position_count` | Number of positions | Diversification indicator |
| `trading/max_position` | Largest position | Concentration risk |

### 6.5 Backtest Metrics

| Metric | Description | Target |
|--------|-------------|--------|
| `backtest/eval_return` | Return multiple | > 1.0 (profitable) |
| `backtest/eval_annual` | Annual return | > 0.1 (10%+) |
| `backtest/eval_max_dd` | Maximum drawdown | < 0.2 (20%) |
| `backtest/eval_sharpe` | Sharpe ratio | > 1.0 |

### 6.6 Training Health Indicators

**Healthy Training:**

- `rollout/ep_rew_mean` increases and stabilizes
- `train/approx_kl` stays below 0.01
- `train/clip_fraction` between 0.1-0.3
- `train/explained_variance` approaches 1.0
- `backtest/eval_return` > 1.0 and increasing

**Warning Signs:**

- `train/approx_kl` > 0.02: Policy updates too aggressive
- `train/clip_fraction` > 0.5: Too many clipped updates
- `train/explained_variance` < 0: Value function failing
- `rollout/ep_rew_mean` decreasing: Learning instability

**Common Issues:**

| Issue | Symptom | Solution |
|-------|---------|----------|
| NaN rewards | Sudden spikes to NaN | Reduce `reward_scaling`, check data |
| No learning | Flat `ep_rew_mean` | Increase `ent_coef`, check reward |
| Overfitting | Train up, eval down | Reduce model size, add regularization |
| Slow learning | Very gradual improvement | Increase `learning_rate`, `batch_size` |

### 6.7 Custom TensorBoard Logging

You can add custom metrics via callbacks:

```python
class CustomMetricsCallback(BaseCallback):
    def _on_step(self) -> bool:
        # Custom metric logging
        self.logger.record("custom/my_metric", some_value)
        return True
```

---

## Quick Reference

### Command Line Usage

```bash
# Basic training
python -m src.train.rl_stock.train_rl_stock

# Custom configuration
python -m src.train.rl_stock.train_rl_stock \
    --stock-pool hs300 \
    --total-timesteps 500000 \
    --learning-rate 1e-4 \
    --n-envs 4 \
    --batch-size 2048

# Debug mode with mock data
python -m src.train.rl_stock.train_rl_stock \
    --use-mock-data \
    --total-timesteps 10000

# Disable TensorBoard
python -m src.train.rl_stock.train_rl_stock --no-tensorboard
```

### Key Configuration Snippets

```python
# Conservative configuration (lower risk)
config = RlStockConfig(
    stock_pool="hs300",
    transaction_cost=0.002,
    trade_threshold=0.05,
    rebalance_interval_days=5,
    reward_scaling=5.0,
    ent_coef=0.01,
)

# Aggressive configuration (more exploration)
config = RlStockConfig(
    stock_pool="zz1000",
    transaction_cost=0.0005,
    trade_threshold=0.01,
    rebalance_interval_days=1,
    reward_scaling=20.0,
    ent_coef=0.05,
)

# Fast debug configuration
config = RlStockConfig(
    use_mock_data=True,
    max_stocks=10,
    total_timesteps=10000,
    n_envs=1,
)
```
