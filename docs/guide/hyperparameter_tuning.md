# 超参数调优指南

## 使用 Optuna 进行超参数搜索

```bash
python -m src.train.hyperparameter_tuner \
    --n-trials 100 \
    --study-name my_study
```

## 参数

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `--n-trials` | 搜索次数 | 50 |
| `--study-name` | 研究名称 | hyperparam_search |
| `--timeout` | 超时时间(秒) | None |
| `--device` | 计算设备 | auto |

## 调优范围

- `learning_rate`: 1e-5 ~ 1e-2
- `hidden_size`: 32 ~ 256
- `dropout`: 0.1 ~ 0.5
- `batch_size`: 256, 512, 1024
- `window_size`: 30, 60, 90
