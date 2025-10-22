# TODO 列表（Gateway 自定义功能）

1. **gRPC Metadata 注入中间件**
   - 目标：在 Gateway 链路中统一添加自定义 `Grpc-Metadata-*` 头部，满足业务鉴权 / 追踪需求。
   - 方案草案：
     - 新增 RoundTripper 中间件 `metadata_injector`，在请求进入后端前设置所需 Header。
     - 支持从 `RequestOptions.Values` 读取上下文信息（如用户标识、Trace ID），并写入 Metadata。
     - 提供配置示例，允许按端点或全局启用。
   - 负责人：自定义功能（待指派）
   - 状态：待实现

2. **Supabase JWT 认证中间件**
   - 目标：在 Gateway 请求链最前段验证 Supabase JWT，并解析用户信息。
   - 方案草案：
     - 新增认证中间件 `supabase_jwt_auth`，解析 `Authorization: Bearer`，校验签名与过期时间。
     - 成功后将 `user_id`、`role` 等信息写入 `RequestOptions.Values`，供后续 metadata 注入等中间件使用。
     - 认证失败时返回统一 Problem Details 响应，避免继续调用下游服务。
   - 负责人：自定义功能（待指派）
   - 状态：待实现

3. **Authorize 多来源认证中间件**
   - 目标：在单一中间件中支持三方来源校验，统一处理：
     1. 用户请求：复用 Supabase JWT（Access Token / Service Role）完成身份验证与上下文字段写入。
     2. Google Cloud Pub/Sub 推送：校验 `Ce-Type`、`Ce-Source` 等签名/证书（使用 GCP 公钥）确认事件来源。
     3. Supabase Webhook：校验签名（`x-supabase-signature`）或使用共享密钥/公钥验证请求真实性。
   - 方案草案：
     - 设计 `authorize` 中间件，按来源匹配规则（Header / IP allowlist / URL 前缀）调用对应的验证器。
     - 将认证结果（`principal_type`、`principal_id`、`scopes`）写入 `RequestOptions.Values`，供后续链路使用。
     - 认证失败统一返回 Problem Details，记录审计日志，避免进入后端。
   - 负责人：自定义功能（待指派）
   - 状态：待实现

4. **来源感知限流中间件**
   - 目标：对不同来源（用户、Pub/Sub、Webhook 等）实施独立的限流策略，防止单一来源耗尽网关资源。
   - 方案草案：
     - 新增 `source_rate_limit` 中间件，读取 `authorize` 中间件输出的来源信息或请求头，选择相应的令牌桶/漏斗参数。
     - 支持本地内存速率限制（令牌桶）和未来可扩展的分布式限流适配器。
     - 透出指标（限流命中次数）和调试接口，便于运维观测。
   - 负责人：自定义功能（待指派）
   - 状态：待实现

5. **统一 Problem Details 错误输出**
   - 目标：对齐 `services/gateway` 中的 RFC 7807 规范，实现网关层统一的错误映射与响应头。
   - 方案草案：
     - 在 Proxy 层或自定义中间件中拦截错误，映射成标准 Problem Details（`type/title/status/detail/instance/trace_id`）。
     - 定义错误分类（认证、授权、限流、后端失败等）与标准化文档链接。
     - 保证响应头附带 `X-Request-ID` / `X-Trace-ID`，便于客户端排查。
   - 负责人：自定义功能（待指派）
   - 状态：待实现

6. **审计日志流水线（Pub/Sub → BigQuery）**
   - 目标：记录所有请求的审计事件，并按 README 所述支持三级可靠性（实时发布、磁盘降级、补发）。
   - 方案草案：
     - 设计 `audit` 中间件，生成审计事件（路由、用户、来源、限流结果、TraceID）。
     - 实现批量异步发布到 Pub/Sub，失败落盘 JSONL，并提供后台补发任务。
     - 暴露 `go_gateway_audit_*` 指标，跟踪发布成功率与降级情况。
   - 负责人：平台（待指派）
   - 状态：规划中

7. **Trace / Request ID 对齐与响应头输出**
   - 目标：确保链路追踪、日志、审计使用同一个 TraceID，并通过响应头返回。
   - 方案草案：
     - 在请求进入时生成/提取 TraceID，写入 `RequestOptions.Values`，后续中间件与客户端复用。
     - 响应统一附加 `X-Trace-ID`、`X-Request-ID`，日志、Problem Details、审计引用同一 ID。
     - 结合 OTel Tracer 设定 `TraceID = RequestID`，方便端到端关联。
   - 负责人：平台（待指派）
   - 状态：规划中

8. **安全增强：来源网段校验与 HTTP 安全头**
   - 目标：支持内部回调路由的 IP Allowlist，并预留上游安全头配置。
   - 方案草案：
     - 在 `authorize` 中间件或独立 `ip_allowlist` 中间件中增加 CIDR 校验，未匹配返回 403 并写审计。
     - 在响应阶段增加 HSTS、CSP、X-Content-Type-Options 等安全头的配置入口（可按环境启用）。
     - 为 Google Pub/Sub / Supabase Webhook 回调补充来源网段验证。
   - 负责人：安全（待指派）
   - 状态：规划中

9. **有界并发与过载保护**
   - 目标：在高负载时限制网关并发，避免拖垮后端，同时给客户端明确的重试指引。
   - 方案草案：
     - 引入带缓冲的信号量（如 `chan struct{}`）控制最大并发数，超过上限返回 503 并设置 `Retry-After`。
     - 结合现有限流策略，输出 Prometheus 指标（当前并发、拒绝次数）。
     - 与平台监控阈值对齐，保障 SLO。
   - 负责人：平台（待指派）
   - 状态：规划中
