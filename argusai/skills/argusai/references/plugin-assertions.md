# ArgusAI 断言类型完整参考

## 一、内置断言类型

### `request` — HTTP 请求断言

向服务发送 HTTP 请求并验证响应。

```yaml
cases:
  - name: "GET /health returns 200"
    request:
      method: GET
      path: /health
      expectedStatus: 200
      expectedBody:
        status: ok

  - name: "POST /users creates user"
    request:
      method: POST
      path: /users
      body:
        name: Alice
        email: alice@example.com
      expectedStatus: 201
      expectedBody:
        name: Alice
      save:
        userId: "$.id"    # 用 JSONPath 保存响应字段供后续用例使用
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `method` | string | HTTP 方法（GET/POST/PUT/DELETE/PATCH） |
| `path` | string | 请求路径 |
| `body` | object? | 请求 body（自动设 Content-Type: application/json） |
| `headers` | object? | 额外请求头 |
| `expectedStatus` | number | 期望 HTTP 状态码 |
| `expectedBody` | object? | 期望响应 body（支持嵌套、部分匹配） |
| `save` | object? | JSONPath 表达式，从响应中提取值供后续用例使用 |

**值比较操作符**（在 `expectedBody` 里用）：

```yaml
expectedBody:
  count: { gt: 0 }         # 大于
  name: { contains: "Al" } # 包含子串
  items: { length: 3 }     # 数组长度
  id: { exists: true }     # 字段存在
  role: { in: ["admin", "user"] }  # 枚举之一
```

---

### `file` — 容器内文件断言

```yaml
cases:
  - name: "output file exists with correct content"
    file:
      container: my-service       # 可选：容器名（省略则读宿主机路径）
      path: /workspace/output.txt
      exists: true
      contains: "hello"
      matches: "^hello .+"        # 正则
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `path` | string | 文件路径 |
| `container` | string? | 从哪个容器读文件 |
| `exists` | boolean? | 期望文件存在/不存在 |
| `contains` | string? | 文件内容包含此字符串 |
| `matches` | string? | 文件内容匹配正则 |
| `json` | object? | 解析 JSON 后的字段断言 |

---

### `exec` — 容器内执行命令

```yaml
cases:
  - name: "database has records"
    exec:
      container: my-service
      command: "sqlite3 /data/app.db 'SELECT COUNT(*) FROM users'"
      expected: "3"
      contains: "3"
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `command` | string | 要执行的 shell 命令 |
| `container` | string? | 在哪个容器执行 |
| `expected` | string? | stdout 完全匹配 |
| `contains` | string? | stdout 包含此字符串 |
| `notContains` | string? | stdout 不包含此字符串 |
| `matches` | string? | stdout 匹配正则 |

---

### `sse` — Server-Sent Events 断言

```yaml
cases:
  - name: "stream emits correct events"
    sse:
      path: /stream
      method: POST
      body: { prompt: "hello" }
      events:
        - type: start
        - type: delta
          data: { contains: "hel" }
        - type: done
      timeout: 10000
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `path` | string | SSE 端点路径 |
| `method` | string? | HTTP 方法（默认 GET） |
| `body` | object? | 请求 body |
| `events` | array | 期望的事件序列（按顺序匹配） |
| `timeout` | number? | 超时毫秒数（默认 30000） |

---

### `process` — 进程存在断言

```yaml
cases:
  - name: "worker process is running"
    process:
      container: my-service
      name: "worker"     # 进程名（支持部分匹配）
      exists: true
```

---

### `port` — 端口监听断言

```yaml
cases:
  - name: "service is listening on 8080"
    port:
      port: 8080
      listening: true
      timeout: 5000
```

---

## 二、Plugin 断言

Plugin 通过 `e2e.yaml` 的 `plugins:` 字段加载，加载后其注册的断言类型与内置断言用法一致。

### 加载 Plugin

```yaml
plugins:
  - ./plugins/dist/index.js    # 本地编译好的 ESM 模块
  - my-argusai-plugin           # npm 包
```

### 使用 Plugin 断言

Plugin 断言在 `cases:` 里直接用 plugin 注册的类型名作为 key：

```yaml
cases:
  - name: "custom assertion passes"
    my-plugin-type:              # plugin 注册的断言类型名
      container: my-service      # 可选：从容器读取路径
      input: /path/to/data       # 主输入
      someOption: value          # plugin 自定义配置
```

每个 plugin 的具体字段由 plugin 自行定义，参见 plugin 自己的文档。

### 常见 Plugin 示例

- **Recursive Agent Plugin** (`e2e/plugins/`)：提供 `recursive-session`（验证 agent session）、`recursive-cost`（验证 token 消耗）、`llm-judge`（LLM 质量评审）等断言，专门用于测试 AI agent 行为。
- 你可以参照 [plugin-development.md](plugin-development.md) 开发自己的 plugin。

---

## 三、断言值操作符速查

在 `expectedBody`、`file.json` 等支持对象断言的字段里，可以用操作符做复杂断言：

| 操作符 | 示例 | 说明 |
|--------|------|------|
| `eq` | `{ eq: 42 }` | 严格相等（默认） |
| `ne` | `{ ne: 0 }` | 不等于 |
| `gt` | `{ gt: 0 }` | 大于 |
| `gte` | `{ gte: 1 }` | 大于等于 |
| `lt` | `{ lt: 100 }` | 小于 |
| `lte` | `{ lte: 99 }` | 小于等于 |
| `contains` | `{ contains: "ok" }` | 字符串包含 |
| `notContains` | `{ notContains: "error" }` | 字符串不包含 |
| `matches` | `{ matches: "^[0-9]+" }` | 正则匹配 |
| `startsWith` | `{ startsWith: "ok" }` | 以...开头 |
| `endsWith` | `{ endsWith: ".json" }` | 以...结尾 |
| `exists` | `{ exists: true }` | 字段存在 |
| `length` | `{ length: 3 }` | 数组/字符串长度 |
| `in` | `{ in: ["a","b"] }` | 枚举值之一 |
