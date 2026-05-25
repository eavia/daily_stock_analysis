# 筹码分布数据修复指南（Tushare 最优方案）

## 📋 问题诊断

**症状**：运行分析时看到
```
筹码健康 - 筹码分布数据缺失，无法判断
```

**根本原因**：所有筹码数据源都无法获取数据，可能原因：
1. Tushare Token 未配置（最常见）
2. 筹码分布功能被关闭
3. 数据源 API 限流或不可用

---

## ✅ 最优解决方案：配置 Tushare Token

### 第1步：获取 Tushare Token

访问 **Tushare 官网** 获取免费 Token：
- 网址：https://tushare.pro
- 点击 **"立即注册"** 或 **"登录"**
- 进入个人中心 → **API 接口** → 复制你的 **Token**

**注意**：
- Tushare 免费用户每天 **500 次** API 调用配额
- 筹码分布接口：每天 **15 次**、每小时 **5 次**（仅 5000 积分以下用户）
- 足够日常个股分析使用

### 第2步：添加 Token 到 .env 配置

#### 方式 A：快速配置（推荐）

如果你已有 `.env` 文件，只需找到这一行并修改：

```dotenv
# 找到这行（可能在文件第 24 行左右）
TUSHARE_TOKEN=

# 替换为你的 Token
TUSHARE_TOKEN=你的Token值
```

**示例**（虚拟 Token）：
```dotenv
TUSHARE_TOKEN=tushare_token_abc123def456ghi789jkl0123456789mnop
```

#### 方式 B：使用提供的模板

1. 复制仓库中的配置模板：
   ```bash
   cat .env.tushare-chip-fix
   ```

2. 编辑 `.env` 文件，添加以下内容：
   ```dotenv
   # ===== 筹码分布修复：Tushare Token =====
   TUSHARE_TOKEN=YOUR_TUSHARE_TOKEN_HERE
   
   # 启用筹码分布
   ENABLE_CHIP_DISTRIBUTION=true
   ```

3. 保存并重启应用

### 第3步：验证配置

**方式 1：查看日志**
```bash
# 启动应用，查看初始化日志
tail -f logs/*.log | grep -i "tushare"

# 应该看到类似输出：
# ✅ 检测到 TUSHARE_TOKEN 且 API 初始化成功，Tushare 数据源优先级提升为最高
```

**方式 2：测试单只股票**
```bash
python -c "
from src.config import get_config
config = get_config()
print(f'TUSHARE_TOKEN: {config.tushare_token[:10]}...' if config.tushare_token else 'NOT SET')
"
```

**方式 3：直接分析**

运行分析后，查看日志是否出现：
```
[筹码分布] 600519 日期=2024-05-24: 获利比例=45.3%, 平均成本=45.2, 90%集中度=8.12%
```

---

## 📊 配置优化建议

### 基础配置（必须）
```dotenv
# Tushare Token - 最重要
TUSHARE_TOKEN=your_token_here

# 启用筹码分布
ENABLE_CHIP_DISTRIBUTION=true
```

### 推荐配置（提升体验）
```dotenv
# 实时行情优先级（Tushare 支持最全字段）
REALTIME_SOURCE_PRIORITY=tushare,tencent,akshare_sina,efinance,akshare_em

# 启用大盘复盘（Tushare 支持更多指数）
MARKET_REVIEW_ENABLED=true

# 启用回测（Tushare 高积分用户推荐）
BACKTEST_ENABLED=true
```

### 数据源优先级设置
```dotenv
# 自动（推荐）- Tushare Token 配置后，优先级自动提升为 -1
# TUSHARE_PRIORITY=-1

# 手动调整（可选）
EFINANCE_PRIORITY=0        # EastMoney 备选
AKSHARE_PRIORITY=1         # AkShare 备选
```

---

## 🚀 工作流验证

### 验证 1：单只股票分析
```bash
python -c "
from data_provider.base import DataFetcherManager
from data_provider.tushare_fetcher import TushareFetcher

manager = DataFetcherManager()
chip = manager.get_chip_distribution('600519')

if chip:
    print(f'✅ 筹码获取成功！')
    print(f'   获利比例: {chip.profit_ratio:.1%}')
    print(f'   平均成本: {chip.avg_cost}')
    print(f'   90%集中度: {chip.concentration_90:.2%}')
else:
    print('❌ 筹码获取失败')
"
```

### 验证 2：批量股票分析
```bash
# 运行完整分析（含筹码分布）
python main.py --stocks 600519,300750,002594 --debug
```

### 验证 3：查看分析报告
检查输出报告中的筹码分析部分：
```
【筹码健康】
- 获利比例: 45.3%
- 平均成本: ¥45.20
- 90%筹码集中度: 8.12%
```

---

## 🔧 常见问题排查

### Q1: 仍显示"筹码分布数据缺失"

**原因可能**：
1. Token 未正确保存到 `.env`
2. 应用未重启（旧配置仍在运行）
3. Tushare API 限流（检查配额）

**解决**：
```bash
# 1. 验证 Token 是否正确
grep "TUSHARE_TOKEN" .env

# 2. 重启应用
# (取决于你的运行方式)
# 本地: Ctrl+C 然后重新运行
# Docker: docker-compose restart

# 3. 查看日志
tail -50 logs/*.log | grep -i "chip\|筹码"
```

### Q2: 提示"Tushare 配额超限"

**原因**：达到每日/每小时限额

**解决**：
- 等待配额重置（每天 00:00 重置）
- 或升级到付费账户获得更高配额
- 暂时使用备选数据源

### Q3: 其他数据源仍然失败

**检查配置**：
```bash
# 确保备选数据源启用
grep -E "ENABLE_CHIP_DISTRIBUTION|AKSHARE_PRIORITY|EFINANCE_PRIORITY" .env

# 预期输出：
# ENABLE_CHIP_DISTRIBUTION=true
# EFINANCE_PRIORITY=0
# AKSHARE_PRIORITY=1
```

---

## 📈 性能与成本

| 项目 | 免费用户 | 付费用户 |
|------|---------|---------|
| 日 API 配额 | 500 次 | 更高 |
| 筹码接口/天 | 15 次 | 无限 |
| 筹码接口/小时 | 5 次 | 无限 |
| 成本 | ¥0 | 根据套餐 |

**建议**：
- 日常个股分析用免费账户足够
- 需要高频分析时升级付费
- 多账户轮询（高级用户）

---

## 🎯 后续优化

### 本地缓存优化
```dotenv
# 筹码分布缓存 TTL（秒）
FUNDAMENTAL_CACHE_TTL_SECONDS=120

# 缓存最大条数
FUNDAMENTAL_CACHE_MAX_ENTRIES=256
```

### 多数据源降级
```dotenv
# 若 Tushare 不可用，自动降级到 AkShare
# (自动处理，无需额外配置)
```

### 监控与告警
```bash
# 监控 Tushare 接口状态
grep "tushare" logs/*.log | tail -20
```

---

## 📝 参考资料

- **Tushare 官网**：https://tushare.pro
- **Tushare 文档**：https://tushare.pro/document/1
- **筹码分布 API**：https://tushare.pro/document/2?doc_id=281
- **项目 GitHub**：https://github.com/eavia/daily_stock_analysis

---

## ✨ 总结

| 步骤 | 操作 | 耗时 |
|------|------|------|
| 1 | 注册 Tushare 获取 Token | 2 分钟 |
| 2 | 编辑 `.env` 添加 Token | 1 分钟 |
| 3 | 重启应用 | <1 分钟 |
| 4 | 验证筹码分布 | 1 分钟 |
| **总计** | | **5 分钟** |

**效果**：
- ✅ 筹码分布正常显示
- ✅ 获利比例、平均成本等指标可用
- ✅ 完整的筹码健康评估
- ✅ 自动备选数据源降级

现在即刻开始修复吧！🚀
