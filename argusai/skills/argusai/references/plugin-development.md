# ArgusAI Plugin 开发规范

本文档说明如何为 ArgusAI 开发自定义断言 Plugin。

## Plugin 是什么

Plugin 是一个 ES Module，导出实现 `PluginModule` 接口的默认对象。ArgusAI 在运行测试前 `import` 这个模块，调用 `setup()`，并把 `assertionPlugins` 里的每个 plugin 注册到全局断言注册表。

测试用例中遇到 plugin 注册的断言类型时，框架会调用对应 plugin 的 `assert()` 方法。

## 最简 Plugin 结构

```typescript
// my-plugin.ts
import type { PluginModule, AssertionPlugin, AssertionResult } from 'argusai-core';

// 一个断言 plugin：验证某个文件里是否包含特定字符串
const myFileContainsPlugin: AssertionPlugin = {
  name: 'my-file-contains',

  assert(type: string, input: unknown, config: unknown): AssertionResult[] {
    if (type !== 'my-file-contains') return [];

    const filePath = typeof input === 'string' ? input
      : (input as Record<string, unknown>).input as string;
    const { expected } = config as { expected: string };

    const content = fs.readFileSync(filePath, 'utf-8');
    const passed = content.includes(expected);

    return [{
      path: 'my-file-contains.content',
      operator: 'contains',
      expected,
      actual: content.slice(0, 100),
      passed,
      message: passed
        ? `File contains "${expected}"`
        : `File does not contain "${expected}"`,
    }];
  },
};

const plugin: PluginModule = {
  name: 'my-plugin',

  async setup() {
    // 可选：初始化操作（连接 DB、启动 stub 等）
    console.log('[my-plugin] loaded');
  },

  async teardown() {
    // 可选：清理操作
  },

  assertionPlugins: [myFileContainsPlugin],
};

export default plugin;
```

## PluginModule 接口

```typescript
interface PluginModule {
  name: string;                          // 显示名，用于日志
  setup?: () => Promise<void> | void;    // 所有 suite 开始前调用一次
  teardown?: () => Promise<void> | void; // 所有 suite 结束后调用一次
  assertionPlugins?: AssertionPlugin[];  // 断言 plugin 列表
}
```

## AssertionPlugin 接口

```typescript
interface AssertionPlugin {
  name: string;   // 断言类型名，在 YAML 中用这个名字

  assert(
    type: string,    // 断言类型（与 name 一致时才处理）
    input: unknown,  // primary input（路径、数据等）
    config: unknown, // 额外配置（YAML 里的其他字段）
  ): AssertionResult[];
}

interface AssertionResult {
  path: string;      // 断言路径（用于错误定位），如 "session.hasToolCalls.write_file"
  operator: string;  // 比较操作符，如 "contains" / "exists" / "gte"
  expected: unknown; // 期望值
  actual: unknown;   // 实际值
  passed: boolean;   // 是否通过
  message: string;   // 人可读的结果描述
}
```

## input 的两种形式

YAML 里的断言步骤有两种写法，`assert()` 的 `input` 参数对应不同结构：

**写法 1：简单字符串**
```yaml
cases:
  - name: "check session"
    my-plugin:
      /tmp/sessions-01    # input 是 string
```
→ `input = "/tmp/sessions-01"`, `config = {}`

**写法 2：带配置的对象**
```yaml
cases:
  - name: "check session"
    my-plugin:
      container: recursive-e2e   # 特殊字段，框架处理
      input: /tmp/sessions-01    # 主输入
      expected: "ok"             # 自定义配置
      minScore: 7
```
→ `input` 是完整的对象（含 `container`、`input` 字段），需要在 `assert()` 里解包：

```typescript
assert(type, input, config) {
  if (type !== 'my-plugin') return [];

  let dataPath: string;
  let options: MyConfig;

  if (typeof input === 'object' && input !== null && 'input' in input) {
    // 写法 2：对象形式，解包
    const stepBody = input as Record<string, unknown>;
    dataPath = stepBody.input as string;
    const { container: _, input: __, ...rest } = stepBody;
    options = rest as MyConfig;
  } else {
    // 写法 1：字符串形式
    dataPath = input as string;
    options = config as MyConfig;
  }
  // ...
}
```

## container 路径处理

当 `container` 字段存在时，`input` 里的路径指向容器内的文件。框架不会自动拷贝——你需要在 plugin 里手动 `docker cp`：

```typescript
import { execSync } from 'child_process';
import * as fs from 'fs';
import * as os from 'os';
import * as path from 'path';

function copyFromContainer(container: string, containerPath: string): string {
  const tmpDir = fs.mkdtempSync(path.join(os.tmpdir(), 'argusai-'));
  execSync(`docker cp "${container}:${containerPath}" "${tmpDir}/"`, { stdio: 'pipe' });
  return path.join(tmpDir, path.basename(containerPath));
}

// 在 assert() 里：
if (container && dataPath.startsWith('/')) {
  dataPath = copyFromContainer(container, dataPath);
}
```

## 注册和加载

在 `e2e.yaml` 里加载编译后的 plugin：

```yaml
plugins:
  - ./plugins/dist/index.js
```

Plugin 文件必须是 **ES Module**（`"type": "module"` 或 `.mjs` 扩展名），默认导出 `PluginModule` 对象。

## 多个断言 Plugin 共享一个 PluginModule

一个 `PluginModule` 可以注册多个断言 plugin：

```typescript
const plugin: PluginModule = {
  name: 'my-suite',
  assertionPlugins: [
    myFileContainsPlugin,
    mySessionPlugin,
    myCostPlugin,
  ],
};
export default plugin;
```

## setup() 中启动 Docker 容器（aimock 模式）

如果你的 plugin 需要在测试开始时启动一个 Docker 容器（如 aimock），在 `setup()` 里做，并确保加入正确的 Docker 网络：

```typescript
async setup() {
  const { execSync } = await import('child_process');
  const allNetworks = execSync('docker network ls --format "{{.Name}}"', { encoding: 'utf-8' })
    .trim().split('\n');

  // 优先使用 WORKTREE_ID 对应的隔离网络
  const worktreeId = process.env['WORKTREE_ID'];
  const namespacedNet = worktreeId ? `argusai-${worktreeId}-network` : null;
  const network = (namespacedNet && allNetworks.includes(namespacedNet) ? namespacedNet : null)
    || (allNetworks.includes('e2e-network') ? 'e2e-network' : null)
    || '';
  const networkFlag = network ? `--network ${network}` : '';

  const running = execSync('docker ps --filter name=aimock --format "{{.Names}}"', { encoding: 'utf-8' }).trim();
  if (!running.includes('aimock')) {
    execSync(`docker run -d --name aimock ${networkFlag} -v ./fixtures:/fixtures ghcr.io/copilotkit/aimock -f /fixtures`, { stdio: 'pipe' });
    await new Promise(r => setTimeout(r, 2000));
  }
},
```

**注意**：`process.env['WORKTREE_ID']` 在 self-improve.sh 里会被设置为当前 worktree 的短 ID，确保并行运行时 aimock 加入正确的隔离网络。

## 最佳实践

1. **type 守卫在第一行**：`if (type !== 'my-type') return [];`，避免处理不属于自己的断言
2. **返回空数组而非抛错**：`assert()` 不应该 throw，失败用 `passed: false` 的 `AssertionResult`
3. **path 字段要有层次**：如 `session.hasToolCalls.write_file`，方便定位哪个断言失败
4. **message 写清楚**：通过时说"通过了什么"，失败时说"期望什么，实际是什么"
5. **临时文件及时清理**：`docker cp` 出来的临时目录在 teardown 或用完后删掉

## 项目结构建议

```
e2e/
├── e2e.yaml
├── fixtures/           # aimock fixtures
├── tests/              # 测试 yaml 文件
└── plugins/
    ├── package.json    # "type": "module", "main": "dist/index.js"
    ├── tsconfig.json
    └── src/
        ├── index.ts    # 导出 PluginModule
        ├── session-plugin.ts
        └── cost-plugin.ts
```

`package.json` 最小配置：
```json
{
  "type": "module",
  "main": "dist/index.js",
  "scripts": { "build": "tsc" },
  "devDependencies": { "argusai-core": "^0.12.2", "typescript": "^5" }
}
```
