# SmartSR-Pricing

[English](README.md) · 简体中文

SmartSR-Pricing 是供 [SmartSR-Core](https://github.com/bwm191/SmartSR-Core) 使用的公开、版本化模型价格注册表，用于估算大语言模型的 token 使用成本。仓库记录服务商、模型、API endpoint、价格、模型别名、价格来源及版本信息。

本仓库只保存价格元数据，不调用模型 API、不保存 API 密钥，也不决定某项研究必须使用哪个模型。

## 快速访问

- **最新版本索引：** [latest.json](latest.json)
- **当前价格快照：** [v2026-11-24.json](v2026-11-24.json)
- **索引原始文件：** https://raw.githubusercontent.com/bwm191/SmartSR-Pricing/main/latest.json
- **当前快照原始文件：** https://raw.githubusercontent.com/bwm191/SmartSR-Pricing/main/v2026-11-24.json

使用者应首先读取 `latest.json`，再根据其中的版本信息获取对应快照，不应假定最新快照始终使用同一个文件名。

## 注册表如何工作

仓库采用两层 JSON 文件结构：

1. `latest.json` 是版本索引，记录当前最新版本，并将每个版本映射到对应的价格文件及其 SHA-256 校验值。
2. `vYYYY-MM-DD.json` 是某一日期的价格快照。已发布快照应视为不可变文件；价格发生变化时，应新增版本，而不是直接修改旧版本。

标准读取流程：

1. 获取 `latest.json`。
2. 读取 `latest_version`。
3. 在 `versions` 中找到对应版本，取得其 `path` 和 `sha256`。
4. 获取对应的价格快照文件。
5. 对快照的原始文件字节计算 SHA-256，并与索引中的值核对。
6. 校验注册表结构，再根据项目实际配置匹配 provider、model 和 API endpoint。

SmartSR-Core 使用这一索引与快照机制，并在价格快照中记录解析得到的版本及来源信息。

## 注册表结构

价格快照包含以下顶层字段：

| 字段 | 含义 |
| --- | --- |
| `schema_version` | 注册表结构版本，目前为 `smartsr.pricing-registry.v1`。 |
| `registry_version` | 价格快照版本，必须与索引中选择的版本一致。 |
| `published_at` | 发布时间，采用 ISO 8601 格式。 |
| `currency` | 当前快照中所有价格使用的币种，例如 `USD`。 |
| `models` | provider/model/endpoint 价格记录列表。 |

每条模型记录包含：

| 字段 | 含义 |
| --- | --- |
| `provider` | 服务商标识，例如 `deepseek` 或 `qwen`。 |
| `model` | 用于匹配的标准模型标识。 |
| `aliases` | 允许匹配的其他模型标识。 |
| `base_url` | API endpoint，是模型匹配条件的一部分，必须与项目配置相符。 |
| `deployment_scope` | 价格适用的部署区域或范围。 |
| `input_per_1m` | 每百万输入 token 的价格，币种由快照的 `currency` 决定。 |
| `output_per_1m` | 每百万输出 token 的价格，币种由快照的 `currency` 决定。 |
| `input_basis` | 对顶层输入价格采用何种计价口径的简要说明。 |
| `source_url` | 用于核实价格的服务商定价文档或其他来源。 |
| `checked_at` | 最近一次核查价格来源的日期。 |

还可以保存服务商特有的补充信息，例如缓存命中价格、峰谷时段价格、上下文长度分档、模型版本说明及计价注意事项。这些字段用于保留来源和背景信息，不代表所有 SmartSR-Core 版本都会读取并参与计算。

### 价格单位与匹配规则

- 所有顶层价格均为**每百万 token 的价格**，不是每千 token 或每个 token 的价格。
- 同一个快照中的所有模型记录使用统一的 `currency`。
- SmartSR-Core 在成本计算时将每百万 token 的价格换算为每千 token 的价格。
- 模型匹配同时使用 provider、model 和 API endpoint。provider 匹配时不区分大小写，endpoint 会去除末尾的斜杠；首先匹配标准模型标识，只有标准标识未匹配时才检查别名。
- 模型标识和别名应与项目中的实际配置完全一致。即使模型名称正确，如果 endpoint 不同，也无法匹配。

价格注册表不与工作流角色绑定。TA 阶段的 `LLM_A`、`LLM_B` 以及其他阶段的模型角色，由 SmartSR-Core 根据项目配置映射到具体的 provider/model/endpoint。某个模型出现在注册表中，不代表研究必须使用该模型。

## 当前快照：2026-11-24

当前索引指向 `v2026-11-24.json`，该快照的币种为 USD。

| 注册表记录 | Endpoint | 顶层输入价格 | 顶层输出价格 | 计价口径 |
| --- | --- | ---: | ---: | --- |
| `deepseek/deepseek-flash`（别名：`deepseek-v4-flash`） | `https://api.deepseek.com` 或 `https://api.deepseek.com/v1` | $0.30 / 百万 | $1.20 / 百万 | 峰时段缓存未命中价格；作为保守的统一计价口径 |
| `qwen/qwen3.7-flash` | `https://dashscope.aliyuncs.com/compatible-mode/v1` | $0.028 / 百万 | $0.11 / 百万 | 标准实时计价，输入长度 0–32K 档 |

当前快照还保存了服务商提供的其他计价信息：

- **DeepSeek：** `pricing_details` 中记录了缓存命中和非高峰时段价格；顶层价格采用高峰时段缓存未命中的计价口径。
- **Qwen3.7 Flash：** `tiered_rates_per_1m` 中记录了更长上下文的价格档位，包括输入长度不超过 256K 和不超过 1M 的档位。

### 当前成本计算引擎的限制

目前 SmartSR-Core v0.3.1 对每个模型只使用 `input_per_1m` 和 `output_per_1m` 两个顶层价格，按单一固定价格计算。当前尚不支持根据缓存命中状态、峰谷时段或输入长度动态计算价格。

因此：

- DeepSeek 成本按顶层的峰时段/缓存未命中价格估算，并未根据实际缓存命中比例或非高峰请求动态调整。
- Qwen 输入长度超过 32K 时，仍按 0–32K 档的顶层价格估算，因此估算值可能低于服务商实际阶梯计费金额。

保留这些扩展字段是为了支持未来升级价格计算模块；不能因为字段已经存在，就认为当前计算器已经支持相应的动态计价。

## 在 SmartSR-Core 中使用指定版本

默认情况下，SmartSR-Core 可以通过版本索引解析最新价格。若希望将项目固定在某个价格快照，可在配置中设置：

```yaml
cost_tracking:
  pricing_version: "2026-11-24"
```

实际项目配置可能还包含其他 `cost_tracking` 设置；添加该字段时应保留其他无关配置。

注意：

- 显式配置的 `cost_tracking.rates` 优先级高于公开价格注册表。
- 设置 `pricing_version` 后，项目使用指定的历史快照，而不是跟随最新索引自动更新。
- 如果注册表中找不到对应的 provider/model/endpoint，解析器会报告价格未配置，不会自行编造默认价格。
- 注册表不包含 API 密钥或其他机密信息。API 凭据应通过服务商支持的密钥管理方式或环境变量单独管理。

工作流和配置细节请参阅 [SmartSR-Core README](https://github.com/bwm191/SmartSR-Core)。

## 发布新的价格版本

应新增价格快照，而不是直接修改已经发布的历史版本。

1. 核查服务商最新定价文档，记录来源 URL、核查日期、币种、部署区域、模型标识、endpoint 和计价口径。
2. 新建 `vYYYY-MM-DD.json`，并将 `registry_version` 设置为相同日期，保留所有必需字段。
3. 校验 JSON，并检查 provider/model/endpoint 组合是否唯一。
4. 根据最终快照文件的原始字节计算 SHA-256。
5. 更新 `latest.json`：将 `latest_version` 指向新版本，并在 `versions` 中添加对应相对路径和 SHA-256。
6. 将新快照与索引一起提交。

计算 SHA-256 的示例：

```python
from hashlib import sha256
from pathlib import Path

registry_path = Path("vYYYY-MM-DD.json")
digest = sha256(registry_path.read_bytes()).hexdigest()
print(digest)
```

必须在快照文件最终确定后计算校验值。之后如果文件内容发生任何字节级变化，都需要重新计算 SHA-256 并同步更新索引。为保证可审计性，更推荐直接发布新版本。

## 发布前检查清单

- [ ] JSON 为有效 UTF-8，且符合注册表结构。
- [ ] `registry_version` 与 `latest.json` 中的版本键一致。
- [ ] 每条模型记录均包含所有必需字段。
- [ ] 价格为有限、非负数值，单位为每百万 token。
- [ ] 币种正确，且整个快照保持一致。
- [ ] provider、标准模型标识、别名和 `base_url` 与目标 API 部署相符。
- [ ] 来源 URL 能够支持所记录的价格和计价口径。
- [ ] `checked_at` 是实际核查日期。
- [ ] `latest.json` 中的 SHA-256 与快照原始字节一致。
- [ ] 未直接修改已发布的历史版本文件。

## 适用范围与免责声明

本仓库是供软件进行成本估算的版本化价格参考。服务商价格、折扣、缓存策略、区域可用性和计费规则均可能发生变化。注册表为每条记录保留来源和核查日期，但它不是账单系统，也不保证估算金额与服务商最终账单完全一致。进行重要成本决策前，应重新核查所链接的服务商定价文档。
