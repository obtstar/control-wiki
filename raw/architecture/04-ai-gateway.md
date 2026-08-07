# 04 AI 网关层（L3）：企业内 LiteLLM 代理

## 现状

企业内已搭建并**启动** LiteLLM 代理，接入两条上游：
- **Anthropic**：`api.anthropic.com`（Claude 系列）
- **GitHub 企业版**：`ghe.com`（Copilot Business/Enterprise，经 `api.githubcopilot.com`）

平台**不重复部署 LiteLLM**，`control-api` / pi.dev 作为消费端直连企业内代理地址（如 `http://litellm.internal:4000`）。

## 优化后的代理配置（litellm_config.yaml）

```yaml
# litellm_config.yaml（企业内代理，优化版）
model_list:
  # ── Anthropic（api.anthropic.com）──
  - model_name: claude-sonnet
    litellm_params:
      model: anthropic/claude-sonnet-4
      api_base: https://api.anthropic.com
      api_key: os.environ/ANTHROPIC_API_KEY
    model_info:
      supports_function_calling: true
      max_tokens: 64000

  - model_name: claude-opus
    litellm_params:
      model: anthropic/claude-opus-4
      api_base: https://api.anthropic.com
      api_key: os.environ/ANTHROPIC_API_KEY
    model_info:
      supports_function_calling: true
      max_tokens: 128000

  # ── GitHub 企业版（ghe.com，Copilot Enterprise）──
  - model_name: copilot-chat
    litellm_params:
      model: github_copilot/gpt-4o
      api_base: https://api.githubcopilot.com
      api_key: os.environ/COPILOT_ACCESS_TOKEN
      user: os.environ/COPILOT_USER
    model_info:
      supports_function_calling: true

router_settings:
  routing_strategy: usage-based-routing-v2   # 按成本/延迟智能路由
  num_retries: 3                             # 失败重试
  request_timeout: 600                       # Agent 长任务（生成代码）需长超时
  allowed_fails: 3                           # 连续失败触发冷却
  cooldown_time: 30                          # 故障模型冷却秒
  enable_pre_call_checks: true               # 上下文超限等提前拦截，省无效调用
  fallbacks:                                 # 模型降级，保证任务不中断
    claude-sonnet: ["copilot-chat"]
    claude-opus:   ["claude-sonnet", "copilot-chat"]
  # 不配置 context_window_fallbacks：copilot-chat（gpt-4o，128K）窗口小于
  # claude 系列（200K），超长降级必然失败；超长上下文由应用层截断/分块处理

model_group_alias:                           # 业务别名，调用方不改配置即可换模型
  coding:   claude-sonnet                    # 默认编码
  cheap:    copilot-chat                     # 简单/补全任务
  heavy:    claude-opus                      # 复杂设计/重构

general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
  database_url: os.environ/DATABASE_URL      # 用量/预算持久化
  budget_duration: 30d                       # 月度预算周期
  default_max_budget: 1000                   # 每 key 月度预算上限（成本治理）
  max_parallel_requests: 32
  drop_params: false                         # 不丢弃厂商不支持的参数（保持一致性）
```

## 优化要点

| 优化项 | 收益 |
|-------|------|
| `fallbacks` | opus 故障自动降级到 sonnet/copilot，任务不中断；超长上下文由应用层截断/分块处理（不走模型降级，copilot 窗口更小） |
| `model_group_alias`（coding/cheap/heavy） | pi.dev 与 control-api 按业务名调用，模型升级零改动 |
| `database_url` + `budget_duration` + `default_max_budget` | 用量记录与月度配额，治理外接模型成本 |
| `request_timeout=600` + `num_retries=3` + `cooldown` | 适配 Agent 长任务，故障自愈 |
| `enable_pre_call_checks` | 提前拦截上下文超限调用，省 token |
| ghe.com 企业版经 `api.githubcopilot.com` + 企业 token | 合规使用企业 Copilot 授权，模型为 gpt-4o |

## pi.dev 接入（消费企业代理）

```json
// ~/.pi/models.json
{
  "providers": [{
    "name": "litellm-enterprise",
    "base_url": "http://litellm.internal:4000/v1",
    "api_key": "${LITELLM_API_KEY}",
    "models": ["coding", "cheap", "heavy"]
  }]
}
```

## 职责

- 消费端直连企业代理，不重复部署；模型路由/降级/预算由代理统一管控
- 密钥集中在代理服务端（ANTHROPIC_API_KEY / COPILOT_ACCESS_TOKEN），不外泄到终端
- 调用审计：`work_log.model_used` 记录实际命中的模型（经代理返回的 model 名）
