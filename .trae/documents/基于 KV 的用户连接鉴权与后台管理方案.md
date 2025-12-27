## 需求摘要
- 支持多用户配置，用户数据存 KV，可在管理面板增删改查。
- 可设定到期时间；到期或删除后禁止连接。
- 在“连接握手”阶段进行用户鉴权（不是仅限制订阅）。
- 实现方式稳定、简单，优先复用现有 `_worker.js` 的协议栈与 `/admin` 面板。

## 总体架构
- 存储：使用 Cloudflare KV（已有 `env.KV` 绑定），新增 users 相关键空间。
- 管理面板：复用 `/admin` 登录态，新增用户管理 UI 与 JSON API。
- 连接鉴权：在 `_worker.js` 的 WebSocket 入站处理函数中，对 VLESS/Trojan 的首包完成用户校验与过期判断。
- 订阅：`/sub` 继续保留，但不作为准入条件；订阅仅是便捷入口，真正准入在握手层完成。

## KV 数据模型
- `users/<uuid>`: { uuid, expiresAt, disabled, note, createdAt, trojanPassword? }
- `users/index`: 存放用户 uuid 列表与简要元数据用于后台列表。
- `users-trojan/<sha224>`: 可选二级索引，映射 Trojan 密码哈希到 uuid（更安全的 Trojan 校验路径）。
- 设计说明：
  - VLESS 使用 `uuid` 直接匹配；Trojan 默认使用同一 `uuid` 作为密码，或单独配置 `trojanPassword`。
  - 简化实现可先存明文 `trojanPassword`，后续切换为仅存哈希并用二级索引查找。

## 接入鉴权流程
- 入口：`_worker.js` 的入站分支 `处理WS请求(request, yourUUID)`，位置：`_worker.js:357`。
- Trojan 路径：
  - 解析：`解析木马请求(buffer, passwordPlainText)`，位置：`_worker.js:412`。
  - 方案：先从首包提取 56 字节哈希；用 `users-trojan/<sha224>` 查找用户；校验用户状态（未过期/未禁用）；再继续转发。
  - 简化备选：若不建索引，使用明文 `trojanPassword` 与现有 `sha224(passwordPlainText)` 对比。
- VLESS 路径：
  - 解析：`解析魏烈思请求(chunk, token)`，位置：`_worker.js:471`。
  - 方案：从首包读取 16 字节 UUID；根据 `users/<uuid>` 查 KV；未命中或到期/禁用则返回握手错误并断开（WebSocket 1008）。
- 调用点参考：
  - Trojan 分流：`_worker.js:389`（当前把 `yourUUID` 作为密码传入）→ 改为按用户匹配。
  - VLESS 分流：`_worker.js:393`（当前把 `yourUUID` 作为 token）→ 改为按用户匹配。

## 管理面板与 API
- 保留现有登录 `/login` 与校验逻辑；后台入口 `/admin`（`_worker.js:192`）。
- 新增页面：`/admin/users`（列表+表单）；`/admin/users/<uuid>`（编辑）；支持生成 UUID、一键禁用/删除、设置到期。
- 新增 API（返回 JSON，需 `auth` Cookie）：
  - `GET /admin/api/users`: 列出所有用户（读取 `users/index`）。
  - `GET /admin/api/users/<uuid>`: 读取单个用户（`users/<uuid>`）。
  - `POST /admin/api/users`: 创建/更新用户（写 `users/<uuid>`、更新 `users/index`、按需写 `users-trojan/<sha224>`）。
  - `DELETE /admin/api/users/<uuid>`: 删除用户（清理相关 KV 键与索引）。
- UI 技术：沿用当前 Pages 静态页结构与简单表单，前端用原生 fetch 调用上述 API。

## 连接拒绝与日志
- 未注册/已到期/被禁用：握手阶段直接返回错误并关闭 WS（1008），并写入 `log.json`。
- 日志扩展字段：`userUuid`、`reason`（expired/disabled/not_found）、`protocol`（vless/trojan）。

## 订阅适配（可选）
- `/sub` 支持 `user=<uuid>&token=<MD5MD5(host+uuid)>`；按用户生成节点（如需区分用户路径或限速）。
- 未通过订阅校验也能连接？否。订阅只影响展示；真正准入在握手。

## 迁移与兼容
- 初始迁移：将现有 `env.UUID` 写入一个默认用户 `users/<env.UUID>`，到期设为远期；保证老链接可用。
- 渐进替换：把 `_worker.js` 中的 `yourUUID` 依赖改为基于用户查 KV，避免单用户限制。

## 验证与测试
- 单元测试思路：
  - VLESS：构造首包含合法/非法/过期 UUID，断言握手行为。
  - Trojan：构造首包含合法/非法哈希，断言握手行为。
- 集成验证：
  - 创建两个用户，一个有效一个已过期；用两种协议分别连接；观察 `log.json` 与连接状态。

## 风险与取舍
- KV 一致性与延迟：KV 读写具备最终一致性，鉴权读为主，稳定可靠。
- Trojan 查找：为“简单稳定”，推荐二级索引按哈希查找；明文方案更简单但安全性较低。
- 订阅与连接逻辑分离：订阅仅便捷入口，避免把订阅 token 作为连接准入。

## 实施要点（最小改动）
1. 在 `_worker.js` 入站处（`_worker.js:334`、`_worker.js:357`）接管 `yourUUID` 源：从首包提取凭据→查 KV 用户→校验状态。
2. 替换 Trojan/VLESS 解析调用参数（`_worker.js:389`、`_worker.js:393`），不再用单一 `yourUUID`，改为“按用户匹配”。
3. 增加 `/admin/api/users*` 路由与 KV 读写；在 `/admin` 页面嵌入用户管理 UI。
4. 更新 `/sub` 支持按用户生成与校验 token（不影响握手准入）。

如同意该方案，我将按上述步骤在代码中落地实现并提供完整变更与验证结果。