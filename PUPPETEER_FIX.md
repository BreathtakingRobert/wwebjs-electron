# Puppeteer 模块兼容性修复

## 问题分析

### 错误信息
```
Error: Cannot find module 'puppeteer'
Require stack:
- D:\qianou\qihang\wwebjs-electron\src\Client.js
- D:\qianou\qihang\wwebjs-electron\index.js
- D:\qianou\qihang\qihangCrmClient\electron\main.js
```

### 根本原因

**模块依赖冲突**：
1. wwebjs-electron 从 whatsapp-web.js 同步代码时，使用了 `require('puppeteer')`
2. 在 Electron 环境中，应该使用 `puppeteer-core` 而不是 `puppeteer`
3. qihangCrmClient 只安装了 `puppeteer-core`，没有安装完整的 `puppeteer`

### Puppeteer vs Puppeteer-Core

| 特性 | puppeteer | puppeteer-core |
|------|-----------|----------------|
| Chromium | ✅ 内置（~300MB） | ❌ 需要外部提供 |
| 大小 | 大（~300MB） | 小（~10MB） |
| 用途 | 独立使用 | 配合现有浏览器 |
| Electron | ❌ 不推荐 | ✅ 推荐（配合 puppeteer-in-electron） |

在 Electron 环境中：
- 使用 `puppeteer-core` + `puppeteer-in-electron`
- 利用 Electron 自带的 Chromium
- 避免重复下载浏览器，节省空间

## 解决方案

### 1. 修改 `src/Client.js`

添加智能模块加载逻辑，优先使用 `puppeteer-core`：

```javascript
// 尝试使用 puppeteer-core（Electron 环境），如果不存在则使用 puppeteer（标准环境）
let puppeteer;
try {
    puppeteer = require('puppeteer-core');
} catch (e) {
    puppeteer = require('puppeteer');
}
```

**优点**：
- ✅ 在 Electron 环境中使用 `puppeteer-core`
- ✅ 在标准 Node.js 环境中降级使用 `puppeteer`
- ✅ 保持向后兼容性

### 2. 重构 `package.json` 依赖

#### 修改前
```json
"dependencies": {
  "puppeteer": "^24.31.0",
  "puppeteer-in-electron": "^3.0.5"
}
```

#### 修改后
```json
"dependencies": {
  "@pedroslopez/moduleraid": "^5.0.2",
  "fluent-ffmpeg": "2.1.3",
  "mime": "^3.0.0",
  "node-fetch": "^2.6.9",
  "node-webpmux": "3.1.7"
},
"peerDependencies": {
  "puppeteer": ">=18.0.0",
  "puppeteer-core": ">=18.0.0"
},
"peerDependenciesMeta": {
  "puppeteer": {
    "optional": true
  },
  "puppeteer-core": {
    "optional": true
  }
},
"optionalDependencies": {
  "archiver": "^5.3.1",
  "fs-extra": "^10.1.0",
  "unzipper": "^0.10.11",
  "puppeteer-in-electron": "^3.0.5"
}
```

**说明**：
- 核心依赖中移除了 `puppeteer`
- 使用 `peerDependencies` 声明需要 `puppeteer` 或 `puppeteer-core`
- 两者都设为 `optional`，由使用者决定安装哪个
- `puppeteer-in-electron` 移到 `optionalDependencies`

### 3. 重新安装依赖

```bash
cd qihangCrmClient
rm -rf node_modules/wwebjs-electron
npm install
```

## 验证结果

### 检查点

1. ✅ wwebjs-electron 不再包含 puppeteer 包
2. ✅ Client.js 使用智能模块加载
3. ✅ 依赖安装成功，无错误
4. ✅ 文件大小减小（无需下载 Chromium）

### 模块加载流程

在 Electron 环境中：
```
Client.js 
  → 尝试 require('puppeteer-core') → 成功 ✅
  → 使用 qihangCrmClient 中的 puppeteer-core@24.19.0
  → 配合 puppeteer-in-electron 使用 Electron 的浏览器
```

在标准 Node.js 环境中：
```
Client.js 
  → 尝试 require('puppeteer-core') → 失败 ❌
  → 降级到 require('puppeteer') → 成功 ✅
  → 使用 puppeteer 自带的 Chromium
```

## 使用建议

### 在 Electron 项目中
```json
{
  "dependencies": {
    "puppeteer-core": "^24.19.0",
    "puppeteer-in-electron": "^3.0.5",
    "wwebjs-electron": "file:../wwebjs-electron"
  }
}
```

### 在标准 Node.js 项目中
```json
{
  "dependencies": {
    "puppeteer": "^24.31.0",
    "wwebjs-electron": "^1.34.6"
  }
}
```

## 技术细节

### Peer Dependencies

`peerDependencies` 表示：
- 这个包需要某个依赖，但不会自动安装
- 由使用者的项目提供这个依赖
- 避免重复安装和版本冲突

### Optional Peer Dependencies

`peerDependenciesMeta.optional: true` 表示：
- 这个 peer dependency 不是必需的
- 如果不存在也不会报错
- 适用于多个可选依赖中只需要一个的场景

在我们的场景中：
- `puppeteer` 和 `puppeteer-core` 都是可选的
- 只要有一个存在就能工作
- Electron 项目用 `puppeteer-core`
- Node.js 项目用 `puppeteer`

## 相关修改

1. ✅ `wwebjs-electron/src/Client.js` - 智能模块加载
2. ✅ `wwebjs-electron/package.json` - 依赖重构
3. ✅ `qihangCrmClient` - 重新安装依赖

## 下一步

现在可以尝试启动客户端：
```bash
npm run electron:dev
```

应该不再出现 "Cannot find module 'puppeteer'" 错误！
