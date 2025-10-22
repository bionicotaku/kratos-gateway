# Kratos Gateway 使用手册

> 适用场景：本地或内网环境下作为微服务入口的反向代理 / API Gateway。仓库基于 go-kratos 官方 gateway 模块，已按项目规范梳理架构、配置加载、链路观测与容错策略。

## 1. 核心能力概览

- **协议转换**：支持 `HTTP -> HTTP`、`HTTP -> gRPC`、`gRPC -> gRPC`，内置 gRPC-JSON 转码中间件。
- **路由热更新**：配置由 `config.yaml` 与可选优先级目录组成，FileLoader/控制面监听变更并热重载路由树，无需重启。
- **链路治理**：集成熔断、限流（BBR）、重试、重写、CORS、Tracing、Logging 等中间件，可按端点或全局装配。
- **服务发现**：对接 Kratos Registry 抽象（示例实现 Consul），支持 `discovery:///service` 与 `direct:///<host>` 目标地址。
- **可观测性**：Prometheus 指标、结构化访问日志、调试端点 (`/debug/*`、`/metrics`)，便于排障与巡检。
- **客户端负载均衡**：内置 Kratos `selector`，默认使用 Power-of-Two-Choices (P2C) 策略，并支持配置权重及重试过滤。

## 2. 目录结构速览

```
kratos-gateway/
├── cmd/gateway          # 入口程序（配置解析、路由热更新、调试面板）
├── config               # 文件配置 & 控制面加载器
├── client               # 后端节点选择、服务发现 watcher、HTTP 客户端池
├── proxy                # 主请求转发逻辑、重试/熔断、debug handler
├── router               # gorilla/mux 封装，负责路由注册与资源释放
├── middleware           # 内置 RoundTripper 中间件库
├── server               # h2c HTTP Server 封装（读写超时等参数）
├── api/gateway          # Proto 定义（配置、middleware options）
└── config.yaml          # 默认示例配置
```

> 目录与项目 4MVC 规范兼容，可作为 `services/gateway` 服务落地骨架。

## 3. 构建与运行

### 3.1 环境要求

- Go **1.24+**（go.mod 指定 toolchain `go1.24.3`）
- 可选：Consul（服务发现演示）、OTLP Collector（Tracing 演示）、Prometheus（指标采集）
- 项目根目录已配置 `go.work`，在仓库根运行 `go env GOPATH` 检查模块生效

### 3.2 安装依赖

仓库只依赖 Go Modules，进入根目录执行：

```bash
cd /Users/evan/Code/learning-app/back-end
go mod tidy
```

### 3.3 启动命令

```bash
cd kratos-gateway
go run ./cmd/gateway \
  -conf ./cmd/gateway/config.yaml \
  -addr :8080 \
  -debug=true
```

常用 flag：

| Flag | 说明 |
| ---- | ---- |
| `-conf` | 主配置文件路径（YAML） |
| `-conf.priority` | 优先级配置目录，覆盖或新增端点 |
| `-addr` | 可重复传入多个监听地址 |
| `-debug` | 启用 `/debug/*` 诊断接口 |
| `-ctrl.service` | 控制面地址，支持逗号分隔 |
| `-ctrl.name` | 对控制面自我标识（默认为 `ADVERTISE_NAME` 环境变量） |
| `-discovery.dsn` | 注册中心 DSN（如 `consul://127.0.0.1:8500`） |

## 4. 配置体系

### 4.1 主配置 (`config.yaml`)

```yaml
name: helloworld           # 应用名称
version: v1                # 版本号，透传到路由与 debug
middlewares:               # 全局中间件，后注册的后执行
  - name: tracing
    options:
      '@type': type.googleapis.com/gateway.middleware.tracing.v1.Tracing
      httpEndpoint: localhost:4318
  - name: logging
endpoints:
  - path: /helloworld/*
    protocol: HTTP
    method: '*'
    timeout: 1s
    backends:
      - target: 127.0.0.1:8000         # direct 直连
    middlewares:                       # 仅对该端点生效
      - name: circuitbreaker
        options:
          '@type': type.googleapis.com/gateway.middleware.circuitbreaker.v1.CircuitBreaker
          successRatio:
            request: "1"
            success: 0.6
          backupService:
            endpoint:
              backends:
                - target: 127.0.0.1:8001
      - name: rewrite
        options:
          '@type': type.googleapis.com/gateway.middleware.rewrite.v1.Rewrite
          stripPrefix: /helloworld
```

关键字段说明：

- `protocol`：`HTTP` 或 `GRPC`
- `timeout`：端点级调用超时（整体重试窗口）
- `backends`：
  - `target`: `host:port` 或 `discovery:///service`
  - `weight`: 仅 `direct://` 模式生效
  - `tls` / `tls_config_name`: 使用预加载的 TLS client
- `retry`：配置重试次数、单次超时、触发条件（状态码 / header）
- `metadata`: 自定义标签（用于 metrics 维度，如 `service`, `basePath`）

### 4.2 优先级配置 (`-conf.priority`)

- 适合金丝雀、灰度，文件命名 `xxx.yaml`。
- 解析后按 `method+path` 覆盖主配置同名端点，未匹配则前置插入。
- 可通过 Feature Flag `gw:PriorityConfig` 控制加载；控制面同步会删除过期文件。

### 4.3 控制面加载

- `-ctrl.service` 支持传入多个逗号分隔 URL，失败自动轮换。
- 控制面返回结构：
  - `config`: 主配置 JSON
  - `priorityConfigs`: 额外配置数组
  - `features`: Feature flag 列表
- Loader 会落地到本地 `dstPath`、`dstPriorityConfigDir`，写入临时文件后原子替换。
- 支持定时拉取最新版本并写入 `lastVersion` 指标可供巡检。

## 5. 请求流水线与中间件链

Gateway 的一次请求严格串行执行，从监听器到上游再回到客户端不会出现并行分支。完整链路如下。

### 5.1 启动与入口

1. **应用初始化**：`cmd/gateway/main.go` 解析 flag，创建 `clientFactory` 与 `proxy.New`，加载配置后调用 `p.Update` 构建初始路由。
2. **监听服务**：`server/proxy.go` 启动 h2c `http.Server`；`PROXY_*_TIMEOUT` 环境变量控制读写与空闲超时。

### 5.2 路由与 Handler 构造

3. **路由匹配**：`router/mux/mux.go` 按 `path/method/host` 匹配请求；`/metrics` 默认注册且对带 `X-Forwarded-For` 的请求返回 403；热更新通过 `atomic.Value` 平滑替换。
4. **Handler 装配**：`proxy/proxy.go:223-244` 为每个端点组装 RoundTripper 链，顺序为“端点级中间件（逆序）→ 全局中间件（逆序）→ client.RoundTripper”。

### 5.3 请求进入链路

5. **入口准备**：`proxy/proxy.go:262-283` 设置 `X-Forwarded-For`（JSON→gRPC 场景也会保留并转换为 gRPC metadata），缓存请求体并重写 `req.GetBody`，构造 `RequestOptions` 存放端点元数据、选路结果与重试状态。
6. **整体超时**：`proxy/proxy.go:268-310` 应用端点级超时，并在每次尝试前使用 `prepareAttemptTimeoutContext` 衍生 per-try timeout。

### 5.4 中间件线性链（按执行顺序）

1. **cors**  
   - 注册名：`cors`，选项类型 `gateway.middleware.cors.v1.Cors`。  
   - 责任：实现浏览器跨域策略控制，包括来源校验、允许方法、允许/暴露头、预检缓存、是否允许携带凭证、是否允许私有网络访问。  
   - 请求阶段：读取 `Origin`、`Access-Control-Request-Method`、`Access-Control-Request-Headers` 等头部进行判定；若为预检请求直接构造响应并终止链路；否则继续调用下游。  
   - 响应阶段：向成功响应注入 `Access-Control-*` 与 `Vary` 头，确保缓存行为符合规范。所有逻辑依赖配置项，未列出的字段保持默认值。

2. **rewrite**  
   - 注册名：`rewrite`，选项类型 `gateway.middleware.rewrite.v1.Rewrite`。  
   - 责任：对请求和响应进行结构化改写，包括路径、主机头、Header 集。  
   - 请求阶段：根据配置执行 `strip_prefix`、`path_rewrite`、`host_rewrite`；对 Header 支持 `set`、`add`、`remove` 操作，同时更新 `RequestOptions.Metadata` 中的头部信息；保证修改后的 `URL.Path` 始终保持以 `/` 开头。  
   - 响应阶段：依照配置对返回 Header 做相同的增删改处理，适配上游与客户端契约差异。

3. **bbr**  
   - 注册名：`bbr`，无选项。  
   - 责任：利用 go-kratos aegis 的 BBR 算法执行自适应限流，防止网关与后端在高负载下过载。  
   - 行为：在请求进入时尝试申请访问许可，若当前负载超出 BBR 估算的安全窗口，则立即返回 HTTP 429，响应体使用静态空 Body；请求完成后将调用结果（成功或上游错误）反馈给 limiter，用于更新下一批次的吞吐估算。

4. **circuitbreaker**  
   - 注册名：`circuitbreaker`，选项类型 `gateway.middleware.circuitbreaker.v1.CircuitBreaker`。  
   - 责任：围绕单个端点实施熔断逻辑，支持成功率阈值（SRE breaker）或按概率拒绝，提供备用回退机制。  
   - 行为：请求进入时执行 `Allow` 判断；若处于熔断状态，依据配置调用备用 endpoint 的 RoundTripper 或构造静态响应数据；若通过，则继续执行下游并根据 `assert_conditions` 判断成功与否；失败会调用 `MarkFailed`，成功调用 `MarkSuccess`；所有拒绝事件通过 `go_gateway_requests_circuit_breaker_denied_total` 指标记录。
   - 额外能力：可与 Feature Flag 配合（控制面下发），实现运行期启停。

5. **logging**  
   - 注册名：`logging`，无选项。  
   - 责任：统一记录网关访问日志，输出到 Kratos slog JSON 管道。  
   - 行为：在请求进入链路时记录起始时间；在响应返回或发生错误时采集客户端信息（Host、Method、Scheme、Path、Query、User-Agent）、网关状态码、错误信息、耗时；从 `RequestOptions` 中读取后端地址列表、上游状态码序列、上游耗时序列、是否为最后一次重试等字段，并全部写入日志；错误时自动切换日志等级。

6. **tracing**  
   - 注册名：`tracing`，选项类型 `gateway.middleware.tracing.v1.Tracing`。  
   - 责任：接入 OpenTelemetry 链路追踪体系，确保上下游调用共享 Trace Context。  
   - 行为：初始化或复用全局 TracerProvider；为每个请求开启 Span，并写入 HTTP 方法、Target、RemoteAddr、上游状态码等属性；在请求头中注入 Trace Context 和 Baggage；异常时记录 `RecordError` 并设置 `codes.Error`；根据配置选择 OTLP HTTP Endpoint、是否忽略 TLS 验证、采样比例与导出批处理超时。

7. **transcoder**（端点协议为 GRPC 时自动插入）  
   - 注册名：`transcoder`，无选项。  
   - 责任：在客户端以 JSON 或 Protobuf over HTTP 访问时，将请求转换为 gRPC over HTTP/2 数据帧，并在返回阶段还原为客户端可识别的格式。  
   - 行为：构造带 5 字节帧头的 gRPC 请求体，重写 `Content-Type` 与 `Content-Length`，确保可重试；对上游响应读取 Data 帧和 Trailer，将 `grpc-status`、`grpc-message`、`grpc-status-details-bin` 映射到 JSON 结构体或保持原始二进制数据；清理 `Content-Length` 以兼容 Trailer 传输；保障 gRPC 与 HTTP 的状态码语义一致。
   
当配置未启用某个中间件时，该中间件不会加入链路；端点级与全局链均按上述顺序串行执行。

### 5.5 后端选择与调用

8. **节点选择**：`client/client.go` 调用 Kratos `selector` 组件，默认使用 **P2C (power-of-two-choices)** 算法在可用节点间择优；支持 `weight`（direct 模式）与 `RequestOptions.Filters`（重试时剔除失败节点）；`client/node.go` 根据 `target` 解析 direct/discovery 并加载 TLS 客户端。
9. **服务发现监听**：`client/servicewatch.go` 管理 registry watcher，缓存实例并暴露 `/debug/watcher/*`；实例变更时刷新 selector 节点列表。
10. **HTTP RoundTrip**：设置 `req.URL/Host/Scheme`，调用对应 `http.Client`（HTTP、HTTPS、h2c）发起上游请求；响应返回后执行 `selector.DoneFunc` 上报结果。

### 5.6 重试与响应返回

11. **重试循环**：`proxy/proxy.go:286-327` 根据端点 `Retry` 配置与 Feature Flag `gw:Retry` 判定是否重新执行整条链路；熔断拒绝或达到 `attempts` 后停止。
12. **响应写回**：`proxy/proxy.go:333-375` 将上游 Header 写入客户端，必要时 `Flush`；按需选择无缓冲复制，统计上下行字节并写回 Trailer，调用 `DoneFunc` 传递元信息。
13. **链路收尾**：`logging` 输出日志，`tracing` 结束 Span，`circuitbreaker`/`bbr` 根据结果更新状态；Proxy 汇总 Prometheus 指标（请求计数、耗时、重试状态、上下行字节）。

整条链路保持串行执行，只有在重试触发时才会按相同顺序重新走一遍完整流程。

### 5.7 请求上下文与状态传递

- **RequestOptions**：`middleware/request.go` 在入口阶段注入 `RequestOptions`，记录当前 endpoint、已访问的 backend 列表、上游状态码与耗时、是否为最后一次尝试、Selector 过滤器、`DoneFunc` 等。每个中间件、客户端都从该结构读取/写入状态，避免通过 context 反复取值。
- **重试过滤**：当某次尝试失败时，`RequestOptions.Filters` 会回写自定义 NodeFilter，下一次重试自动跳过失败节点，结合 P2C 进一步减轻雪崩。
- **Selector.DoneFunc**：`client/client.go` 在完成上游请求后调用 `done(ctx, selector.DoneInfo{...})`，向负载均衡器反馈成功/失败，便于权重调整。
- **上下行指标**：`proxy/proxy.go` 在复制响应体时，调用 `sentBytesAdd`、`receivedBytesAdd`、`requestsDurationObserve`、`requestsRetryStateIncr` 等函数积累 Prometheus 指标，所有标签通过 `middleware.NewMetricsLabels` 从 endpoint metadata (`service`、`basePath` 等) 衍生。
- **gRPC 特殊处理**：gRPC 请求通过 `transcoder` 中间件封装帧头、剥离 Trailer，并确保在错误场景下填充 `grpc-status`/`grpc-message`。`writeError` 针对 gRPC 入口将 HTTP 错误映射为 gRPC status code，再返回 200，以满足 gRPC 协议要求。
- **HTTP 客户端池**：`client/node.go` 为 HTTP、HTTPS、h2c 分别维护 `*http.Client`，TLS 模式可引用 `tls_store` 中的证书配置；默认拒绝自动跳转，可通过 `PROXY_FOLLOW_REDIRECT` 开启，并统计重定向次数。
- **Feature Flag**：部分链路（如重试、优先级配置）依赖 `github.com/go-kratos/feature` 注册的开关（`gw:Retry`、`gw:PriorityConfig` 等），可通过控制面动态调整。

## 6. 可观测性与调试

- **Prometheus 指标**：`/metrics`（禁止带 `X-Forwarded-For`）提供 `go_gateway_requests_code_total`、`go_gateway_requests_duration_seconds`、`go_gateway_requests_retry_state`、`go_gateway_requests_tx_bytes`、`go_gateway_requests_rx_bytes`、`go_gateway_client_redirect_total`、`go_gateway_requests_circuit_breaker_denied_total` 等。
- **日志**：所有链路日志通过 Kratos `log.Context(ctx)` 输出 JSON，可直接接入统一日志管线。
- **调试端点**（`-debug=true`）：
  - `/debug/proxy/router/inspect` 查看当前路由树。
  - `/debug/config/*` 检查配置摘要、强制重载或查看版本。
  - `/debug/watcher/*` 观测服务发现缓存与 Applier 列表。
  - `/debug/pprof/*` 获取 CPU/内存等性能分析数据。

### 6.1 审计日志（MVP 之后）

- **目标**：在 MVP 之后补齐审计能力，记录关键访问行为并为后续 Pub/Sub → BigQuery 链路做准备。
- **推荐字段**（建议统一使用 ISO-8601 UTC 时间）：

| 字段 | 说明 | 是否建议 |
| ---- | ---- | -------- |
| `timestamp` | 请求开始或处理完成时间（ISO-8601） | ✅ |
| `traceId` | 当前请求的 Trace/Correlation ID（与 `X-Trace-ID` 一致） | ✅ |
| `userId` | 发起者的用户 ID / 服务账户 / 系统账户 | ✅ |
| `userRole` | 用户角色或权限组 | 建议 |
| `sourceIp` | 请求来源 IP | ✅ |
| `clientId` | 客户端标识（App 版本、调用方服务、网关节点等） | 建议 |
| `endpoint` | 访问的 API 路径或资源 URI | ✅ |
| `httpMethod` | HTTP 方法（GET/POST/…） | ✅ |
| `resourceId` | 被访问或变更的业务资源 ID | 建议 |
| `operation` | 业务动作描述（如 `TRANSFER_FUNDS`） | 建议 |
| `requestPayload` | 主要请求参数（敏感信息需脱敏/掩码） | 建议 |
| `responseStatus` | HTTP 状态码或业务错误码 | ✅ |
| `responsePayload` | 响应概要（敏感信息脱敏） | 建议 |
| `outcome` | 操作结果（`SUCCESS`/`FAILURE`/`DENIED` 等） | ✅ |
| `processingTimeMs` | 处理耗时（毫秒） | 建议 |
| `service` | 执行请求的服务或网关节点名称/版本 | 建议 |
| `environment` | 环境标识（prod/stage/dev） | 建议 |
| `tags` | 可选标签（如 `sensitive`、`financial`） | 可选 |
| `authorizationDecision` | 授权检查结果（`AUTHORIZED`/`UNAUTHORIZED`） | 建议 |
| `reason` | 失败或拒绝原因（错误码、消息） | 建议 |
| `deviceInfo` | 设备信息（User-Agent、客户端版本） | 可选 |
| `location` | 地理位置/区域（如基于 IP 推断） | 可选 |
| `auditLevel` | 审计级别（`INFO`/`AUDIT`/`SECURITY`） | 建议 |
| `prevValue/newValue` | 修改操作的前后关键字段 | 可选 |

- **实现建议**：先以结构化 JSON（stdout/落盘）验证字段与脱敏策略，再接入消息队列与数据仓库构建可靠链路（实时发布 + 落盘降级 + 补发）。

## 7. 常见运维要点

- **大请求体重试**：Proxy 会 `io.ReadAll` 缓存请求体，大文件上传建议关闭重试或限制 `Content-Length`。
- **优雅关闭**：路由热更新和进程退出都会等待正在处理的请求，超过 120 秒后强制关闭，可按业务调整。
- **Feature Flag**：`gw:Retry`、`gw:PriorityConfig` 等开关通过 `github.com/go-kratos/feature` 控制，需在控制面同步管理。
- **TLS 管理**：在配置 `tls_store` 后可重用证书，后端 `tls_config_name` 引用即可，避免重复加载。
- **安全考量**：默认拒绝带 `X-Forwarded-For` 的 `/metrics` 请求；来源网段校验、HTTP 安全头等增强能力计划在 MVP 之后补齐（详见 TODO 8）。
- **过载保护规划**：当前未限制整体并发，后续将按 TODO 9 引入信号量/`Retry-After` 控制，避免突发流量拖垮后端。

## 8. 集成建议

1. **配置治理**：将 Gateway 配置纳入 CI/CD，结合控制面或 GitOps 确保主配置与优先级配置可回溯。
2. **故障排查**：搭配 `/debug/proxy/router/inspect`、Prometheus 指标和访问日志定位异常节点或重试热点。
3. **中间件扩展**：自定义中间件需实现 `middleware.Middleware`/`MiddlewareV2` 并通过 `middleware.Register` 注册，遵循同一 RoundTripper 嵌套模式。
4. **对接 4MVC**：将 Gateway 作为 `services/gateway` 服务骨架使用时，可在 Controller 层封装 HTTP 接入，复用现有客户端与中间件实现。

若需进一步验证，可在仓库内执行：

```bash
cd kratos-gateway
go test ./proxy ./config ./middleware/... ./client/...
```

覆盖面包括重试、熔断、配置热更新等关键链路，建议在接入自定义中间件或控制面前补充对应集成测试。

## 9. 当前限制与后续计划

- **Problem Details**：目前错误仍为默认文本/状态码输出，计划在 TODO 5 中补齐 RFC 7807 结构化响应，附带 TraceID 等上下文。
- **入口 TraceID**：现有 tracing 中间件只在下游请求创建 Span，尚未在入口统一生成/响应 `X-Trace-ID`，对应 TODO 7。
- **审计日志**：尚未实现，MVP 之后按 TODO 6 落地审计字段和可靠链路。
- **安全增强**：来源网段校验、HTTP 安全头未上线，参考 TODO 8。
- **过载保护**：缺少网关级并发上限，计划在 TODO 9 引入信号量和 `Retry-After`。
- **RateLimit 响应头**：当前仅记录限流指标，未在响应中返回 `RateLimit-*`/`Retry-After` 头，后续由 TODO 4 落地。
- **RequestID**：当前未输出 `X-Request-ID`，后续只保留统一的 TraceID。
