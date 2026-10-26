# 火山方舟 Coding Plan 额度（5h / 周 / 月）

本 fork 在小鲸鱼挂件里新增了一个订阅额度厂商模板：**「火山方舟 Coding Plan（订阅）」**
（`provider: volcengine_ark_coding`，`kind: 'quota'`）。一次请求同时拿到三个窗口：

| 窗口 | 后端 Level | 语义 |
|---|---|---|
| 5h | `session` | 5 小时滚动窗口（按窗口内首次请求起算，满 5 小时重置） |
| 周 | `weekly` | 每周一 00:00 重置 |
| 月 | `monthly` | 订阅月对应日 00:00 重置 |

展示已用百分比 + 各窗口重置倒计时；接口只返回百分比（`Cap` 固定 100），**没有绝对次数**。

## 接口

```http
POST https://ark.cn-beijing.volcengineapi.com/?Action=GetCodingPlanUsage&Version=2024-01-01
Content-Type: application/json

{"ProjectName":"default"}
```

响应（火山引擎控制面 OpenAPI，服务 `ark`，版本 `2024-01-01`）：

```json
{"Result":{"Status":"Running","UpdateTimestamp":1792986700,"HasReward":false,
  "QuotaUsage":[
    {"Level":"session","Cap":100,"Percent":96.69,"ResetTimestamp":1792995766,"RewardTotalPercent":0},
    {"Level":"weekly", "Cap":100,"Percent":12.89,"ResetTimestamp":1793548800,"RewardTotalPercent":0},
    {"Level":"monthly","Cap":100,"Percent":17.22,"ResetTimestamp":1795363199,"RewardTotalPercent":0}]}}
```

- `ResetTimestamp` 是秒级 epoch；前端 `apiPlanResetMs` 已兼容秒/毫秒。
- 必须 **AK/SK 做 V4 签名**；数据面 API key（`ark-…`）打不通这个接口。
- 该接口未收录进公开 API 文档，接口若变更需要同步调整模板。

## 配置（两种取数方式）

1. **AK/SK 签名直连（推荐）**：添加模型时凭据填 `AK:SK`（火山控制台 → IAM → 访问密钥，中间用英文冒号）。
   若用 STS 临时凭证，可写 `AK:SK:会话令牌`，第三段会自动作为 `X-Security-Token`。
2. **复用本机 arkcli 登录态**：凭据留空即可 —— 机器上装了 `arkcli` 且已 `arkcli auth login` 时，
   会退回执行 `arkcli api usage.get_coding_plan_usage` 取同一份数据（无需 AK/SK，但依赖登录态有效）。

## 代码改动（仅 `lib/index.js`）

- 新增模板 `volcengine_ark_coding`：`quota.url / method / body / sign:'volc-v4' / region / service`，
  `json.windows` 用 `pick: { list:'Result.QuotaUsage', by:'Level', value:'session|weekly|monthly' }` 选条。
- `pickQuotaEntry()` 增加通用选条 `pick.by` + `pick.value/values`（原来只按 `type`/`unit` 选）。
- 新增 `volcParseCredential()` / `volcSignHeaders()`（V4 签名）/ `volcSignedFetchJson()` / `volcQuotaViaArkcli()`（兜底）。
- `fetchModelQuota()`：`q.sign === 'volc-v4'` 走签名或 arkcli 分支；无凭据（`NO_KEY`）不算失败。
- `apiModelsPayload()`：带 `sign` 的额度模板即使 `hasKey=false` 也尝试取额度（否则 arkcli 兜底永远不触发）。
- `apiProbeModel()`：`kind='quota'` 的模板「测试连通性」直接打额度接口，返回各窗口已用%。
- 额度请求失败时带上上游 `Error.Code/Message`（如 `InvalidAuthorization`），便于排障。

## 安装（本地开发）

```powershell
dsh plugin --profile desktop remove dsh-whale-widget
dsh plugin --profile desktop add link:<本仓库绝对路径>
```

Host 代码改完需要重新挂载插件（DSH HMR 或关/开一次 bundle）；`assets/whale-widget.js` 的改动刷新页面即可。
