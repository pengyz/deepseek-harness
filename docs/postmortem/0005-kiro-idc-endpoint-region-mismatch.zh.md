# 事故复盘 0005：`dsh-kiro` 把 IdC 认证的请求发向了 AWS 从未部署过的 region 和主机名

[English](0005-kiro-idc-endpoint-region-mismatch.md) | 中文

状态：已解决（修复已应用于社区 `dsh-kiro` 插件；已向上游提交 issue）

## 执行摘要

社区版 `dsh-kiro` provider 按 `authMethod` 选择生成对话的主机名（IAM Identity Center 走 `codewhisperer.<region>`，其他方式走 `q.<region>`），并用登录 token 里的 region 填充 `<region>`。对于一个 token region 为 `ap-southeast-1` 的 IdC 部署，这会拼出 `codewhisperer.ap-southeast-1.amazonaws.com`——一个在任何解析器里都没有公开 DNS 记录的主机名，因为 AWS 从未在该区域部署过 CodeWhisperer 的流式 API。token 里的 region 是 IAM Identity Center 实例所在的 region，而不是模型 API 部署的 region；无论 `authMethod` 是什么，都必须调用 AWS 实际上线的那一个主机名（`q.us-east-1.amazonaws.com`），这一点已通过厂商自家 CLI 与另一个独立的开源逆向实现交叉验证。修复方案是把 `us-east-1` 硬编码进去，并去掉按 `authMethod` 分支选主机名的逻辑。

## 影响

对 Kiro provider 的每一次对话都会失败。用户看到的表现是 `Invalid model. Please select a different model to continue.`（`INVALID_MODEL_ID`），这个措辞读起来像是模型目录的问题，把排查方向带偏到了完全错误的地方：这个响应确实来自一个真实的、能通过认证的 AWS 端点，但它拒绝了这个模型 id——因为这个端点根本没听说过任何 Kiro 模型 id，它本身就是错的服务。同一账号自带的 CLI（`kiro-cli`）在整个过程中一直能正常工作，这最终推翻了"账号配置有问题"的假设。

## 时间线

- 用户报告：在一次与之无关的 harness 升级/回退之后，`dsh-kiro` 的刷新用量按钮和对话请求都失败了。
- 第一个假设：harness 升级改动了什么。已排除——在升级前的 harness commit 上同样能复现这个失败。
- 第二个假设：需要把账号的 AWS IAM Identity Center region（`ap-southeast-1`，从存储的登录 token 中读取）传入插件的 `region` 配置。应用后，刷新用量开始正常（该端点对两种 region 都容忍），但对话请求依旧失败，此时报的是 `codewhisperer.ap-southeast-1.amazonaws.com` 不可达——`getent hosts` 查不到任何记录，用 Google 的公共解析器独立验证过（不是本地 DNS 过滤造成的）。
- 第三个假设（这次站住了）：网络失败是症状，不是原因。`kiro-cli`——厂商自己的客户端，用的是同一个已登录账号——始终能正常工作。用 `strace -f -e trace=network` 追踪一次真实的 `kiro-cli` 对话调用，显示所有 DNS 查询和连接全部指向 `*.us-east-1.amazonaws.com`，从未出现 `ap-southeast-1`。从 `kiro-cli` 二进制里提取可读字符串，找到了它内置的端点表——`q.eu-central-1`、`q.us-gov-east-1`、`q.us-gov-west-1`、`codewhisperer.us-east-1`——里面根本没有 `ap-southeast-1` 这一项。
- 与一个独立维护的开源 Kiro-to-OpenAI/Anthropic 代理项目做了交叉核对（跟本项目毫无关系的代码库，针对同一个逆向出来的协议）：它的 provider 把 `KIRO_API_URL = "https://q.us-east-1.amazonaws.com/generateAssistantResponse"` 直接硬编码——没有按账号区分 region，也没有按 `authMethod` 分支。
- 用直接的 `curl` 调用把剩下的两个变量隔离开，其余条件全部保持不变：`q.us-east-1` 返回 `200`，带回一段真实的流式回答；`codewhisperer.us-east-1`——同一个账号、同一个 bearer token、同一份请求体——返回 `400 INVALID_MODEL_ID`，跟用户最初报的错误一字不差。这就仅凭主机名的选择复现出了最初的症状。

## 根因

`dsh-kiro` 的 `kiroRequestEndpoint(token, region)` 根据 `token.authMethod` 选择生成对话用的主机名：

```ts ignore-check
function kiroRequestEndpoint(token: KiroToken, region: string): string {
  return token.authMethod === 'idc' || token.authMethod === 'external_idp'
    ? `https://codewhisperer.${region}.amazonaws.com/generateAssistantResponse`
    : `https://q.${region}.amazonaws.com/generateAssistantResponse`
}
```

`region` 来自 `connection.region ?? token.region`，而 `token.region` 是直接从持久化的登录 token 里读出来的——它是调用方 IAM Identity Center 实例所在的 region，是组织在搭建 SSO 时选定的。这个值跟 AWS 把 CodeWhisperer/Q Developer 流式 API 部署在哪个 region 完全没有关系：AWS 只在一个很小的、固定的 region 集合里上线这个服务（`us-east-1`，外加厂商 CLI 自带端点表里列出的少数政府云/中国分区），跟任何客户的 Identity Center 实例部署在哪里毫无关联。还有两个额外的选择加重了这次错配：

- **IdC 走了错误的主机名族。** 即便固定用 `us-east-1`，`codewhisperer.us-east-1.amazonaws.com` 也是一个跟 `q.us-east-1.amazonaws.com` 完全不同、真实存在且能通过认证的端点——它会对每一个 Kiro 模型 id 都报 `400 INVALID_MODEL_ID`，因为 IdC 账号无论 `authMethod` 是什么，都是由 `q.` 端点提供服务的。
- **一旦发现了某个账号的 IAM Identity Center profile ARN，它会悄无声息地覆盖任何已配置的 region。** `dsh-kiro` 的 profile 自动发现流程（只要 token 里还没有 `profileArn` 就会触发）会调用 `ListAvailableProfiles`，当匹配到多个 profile 时，会优先选择 ARN 里的 region 恰好等于 `token.region` 的那一个，而不是第一个（或按配置 region 匹配到的）结果。一旦这个 profile 被缓存下来，之后每一次请求的主机名就由 `profileRegion(profileArn)` 决定，而不是 `region` 配置项。所以只在 `cordis.yml` 里配置 `region` 的修复方式扛不住 profile 发现流程；还必须考虑被发现的 profile ARN 里到底带着什么。

## 防护措施

- **永远不要把某个凭证自身的元数据当成某个 provider 固定服务拓扑的依据。** 登录 token 的 `region` 描述的是*认证*锚定在哪里（Identity Center 实例、OAuth 发行方、SSO 租户）——不是该 token 所授权的*产品 API* 部署在哪里。在把一个来自账号的值接入请求 URL 之前，先去读厂商自己客户端里真实的端点表；多区域服务是 provider 需要明确声明的例外情况，不是默认可以假设的常态。
- **逆向出来的协议在被信任为已确认的约定之前，需要两个独立来源的相互印证。** 厂商自己的 CLI（用 `strace -f -e trace=network` 追踪，再用二进制 `strings` 提取内置端点表）和一个无关的、针对同一套协议的开源客户端，都一致指向同一个不带 region 参数的硬编码端点——正是这种相互印证，而不是任何单一来源本身，才是这次修复里把 `us-east-1` 硬编码进去的依据。
- **当一个症状可能有多个原因时，每次请求只隔离一个变量。** 用完全相同的 header、请求体和 bearer token，只改动主机名的 `curl` 调用，仅凭主机名这一项差异就复现出了用户报告的确切错误，一次性排除了账号、模型、Content-Type 这几个理论——比再加一个配置项、重新跑一遍完整 harness 要更便宜，也更有说服力。
- 一个来自真实、能通过认证的端点的 HTTP 400，并不能证明请求到达了正确的*服务*——只能证明它到达了*某个*理解了请求内容、并因此拒绝了某个字段的服务。在把一次成功或失败的响应当作诊断信号来信任之前，先确认 DNS 记录确实存在（要用公共解析器核实，不能只信本地的）是一道前置检查。
