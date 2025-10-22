# TODO 列表（Gateway 自定义功能）

## MVP 阶段

1. **gRPC Metadata 注入中间件**
   - 目标：在 Gateway 链路中统一添加自定义 `Grpc-Metadata-*` 头部，满足业务鉴权 / 追踪需求。
   - 方案草案：
     - 新增 RoundTripper 中间件 `metadata_injector`，在请求进入后端前设置所需 Header。
     - 支持从 `RequestOptions.Values` 读取上下文信息（如用户标识、TraceID），并写入 Metadata。
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
   - 目标：在单一中间件中支持用户、Pub/Sub 推送、Supabase Webhook 等三方来源校验。
   - 方案草案：
     - 设计 `authorize` 中间件，按来源匹配规则（Header / IP allowlist / URL 前缀）调用对应的验证器。
     - 将认证结果（`principal_type`、`principal_id`、`scopes`）写入 `RequestOptions.Values`，供后续链路使用。
     - 认证失败统一返回 Problem Details，记录审计日志，避免进入后端。
   - 负责人：自定义功能（待指派）
   - 状态：待实现

4. **来源感知限流中间件**
   - 目标：对不同来源（用户、Pub/Sub、Webhook 等）实施独立的限流策略。
   - 方案草案：
     - 新增 `source_rate_limit` 中间件，读取 `authorize` 输出的来源信息或请求头，选择相应的令牌桶/漏斗参数。
     - 支持本地内存速率限制（令牌桶）和未来可扩展的分布式限流适配器。
     - 透出指标（限流命中次数）和调试接口，便于运维观测。
   - 负责人：自定义功能（待指派）
   - 状态：待实现

5. **统一 Problem Details 错误输出**
   - 目标：对齐 `services/gateway` 的 RFC 7807 规范，实现网关层统一的错误映射与响应头。
   - 方案草案：
     - 在 Proxy 层或自定义中间件中拦截错误，映射成标准 Problem Details（`type/title/status/detail/instance`，并额外附带 TraceID）。
     - 定义错误分类（认证、授权、限流、后端失败等）与文档链接。
     - 响应头附带 `X-Trace-ID`（入口 TraceID），便于客户端排查。
   - 价值说明：
     - **客户端一致性**：统一 JSON 结构，调用方无需适配各服务文本报错。
     - **跨服务抽象**：可在 Problem JSON 承载“认证失败/限流”等语义，并保留业务错误码。
     - **排障效率**：Problem Details + `X-Trace-ID` 让排障更快。
     - **网关自有错误**：认证、路由等错误在 Problem JSON 中明确化。
   - 负责人：自定义功能（待指派）
   - 状态：待实现

## MVP 之后

6. **审计日志**
   - 目标：补齐审计能力，记录关键访问行为，并扩展到 Pub/Sub → BigQuery 的可靠链路。
   - 方案草案：
     - 阶段一：实现 `audit` 中间件，收集核心字段（timestamp、traceId、userId、sourceIp、endpoint、httpMethod、responseStatus、outcome、processingTimeMs 等），输出结构化 JSON。
     - 阶段二：接入 Pub/Sub，失败时落盘 JSONL 并提供补发任务；暴露 `go_gateway_audit_*` 指标监控发布成功率与降级情况。
     - 字段参考 README 6.1，可按业务添加 `resourceId`、`operation`、`requestPayload`、`authorizationDecision` 等，并注意脱敏。
   - 负责人：平台（待指派）
   - 状态：规划中

7. **Trace ID 生成与响应头输出**
   - 目标：在入口统一生成新的 TraceID，贯穿日志、审计，并通过 `X-Trace-ID` 返回给客户端。
   - 方案草案：
     - 请求进入时无条件生成 TraceID（忽略客户端 header），存入 `RequestOptions.Values`。
     - 响应阶段附加 `X-Trace-ID`，并在 Problem Details、审计、日志中引用。
     - 对后端转发时注入该 TraceID，保持上下游链路一致。
   - 负责人：平台（待指派）
   - 状态：规划中

8. **安全增强**
   - 目标：在 MVP 之后补齐来源网段校验与 HTTP 安全头能力。
   - 方案草案：
     - 来源校验：在 `authorize` 或独立中间件中增加 CIDR 白名单，内部回调不匹配即 403 并写审计。
     - 安全头：按需添加 HSTS、CSP、X-Content-Type-Options 等响应头，与 WAF/Cloud Armor 配合作为兜底。
   - 负责人：安全（待指派）
   - 状态：规划中

9. **有界并发与过载保护**
   - 目标：加入网关层并发上限，避免高峰拖垮后端。
   - 方案草案：
     - 引入信号量/请求队列控制最大并发，超限返回 503 并设置 `Retry-After`。
     - 输出 Prometheus 指标（当前并发、拒绝次数）。
     - 结合现有限流策略，对齐 SLO 与告警阈值。
   - 负责人：平台（待指派）
   - 状态：规划中

10. **网关→后端服务间认证（重点）**
    - 目标：在网关向后端 gRPC 服务发起调用时，提供统一的服务间认证能力，避免仅依赖客户端传入的凭据。
    - 方案草案：
      - 支持 mTLS：在 `tls_store` 配置客户端证书，后端启用双向 TLS。
      - 支持自定义凭据：在转发前注入服务级 Token（JWT/API Key），以 gRPC Metadata/HTTP Header 方式传递。
      - 可与 metadata 注入/authorize 中间件结合，根据路由或来源生成不同凭据。
    - 负责人：平台（待指派）
    - 状态：规划中
