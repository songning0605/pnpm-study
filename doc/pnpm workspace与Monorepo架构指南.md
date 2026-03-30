# pnpm workspace 与 Monorepo 架构指南

## 一、 概念篇：什么是 Monorepo 与 pnpm workspace？

### 1. 什么是 Monorepo？
**Monorepo**（单体仓库）是一种代码管理策略，将多个独立的项目（通常相互有依赖关系）放到**同一个 Git 仓库**中进行管理。
*   **痛点（Multirepo）**：如果有多个包相互依赖（比如 `utils` 包和依赖它的前端 `app` 项目），修改了 `utils` 需要发布到 npm，然后在 `app` 中更新依赖，开发调试极其繁琐。
*   **优势（Monorepo）**：代码都在一个仓库里，可以实现跨项目的本地即时调试。A 项目依赖 B 项目，改了 B 之后 A 能**直接生效**，不需要发布，并且所有项目可以统一管理依赖版本和构建脚本。

### 2. 什么是 pnpm workspace？
**pnpm workspace** 是 pnpm 原生提供的对 Monorepo 的支持机制。它通过在项目根目录创建一个 `pnpm-workspace.yaml` 文件来声明这是一个工作区，并圈定哪些目录下的包属于这个工作区。

---

## 二、 原理与架构篇：它是怎么运作的？

pnpm 对 Monorepo 的实现非常优雅，底层机制依赖于**软链接（Symlink）**、**工作区协议** 和 **统一的 Store**。

### 1. 软链接机制（Symlink）
当工作区内的 `App` 项目依赖 `Math` 项目时，pnpm **不会**去外网下载，也**不会**把 `Math` 的代码硬复制过去。
pnpm 会在 `App/node_modules/Math` 创建一个**软链接**，直接指向同一仓库下的 `Math` 文件夹。这意味着你修改 `Math` 中的源码，`App` 中引用的代码是同步更新的！

### 2. 工作区协议 (`workspace:*` 和 `workspace:^`)
当你在 `package.json` 的依赖中看到 `"@my-workspace/math": "workspace:^"` 时，这是 pnpm 的专属语法。
*   `workspace:*` 或 `workspace:^` 强制告诉 pnpm：“去当前工作区里找这个包，千万不要去 npm 源里下载！”。
*   默认情况下，如果你在命令行使用 `pnpm add @my-workspace/math@workspace:*`，pnpm 会根据你的配置或默认行为（如匹配 semver 策略），在 `package.json` 中保存为 `workspace:^` 或其他匹配版本。
*   在发布项目（publish）时，pnpm 会自动把 `workspace:*` 或 `workspace:^` 替换为该包当前的真实版本号。

### 3. 唯一的 Lockfile 和依赖共享
不管你的工作区下面有10个还是100个子项目，根目录下通常只有一个 `pnpm-lock.yaml` 文件。
*   pnpm 会分析所有子项目的依赖，提取它们的公共部分，全部链接到全局的 `.pnpm` Store 中。这保证了极度节省磁盘空间和极快的安装速度。

---

## 三、 实操篇：一步步动手体验

项目结构如下：
```text
pnpm-workspace.yaml     <-- 声明工作区范围
packages/
  math/                 <-- 一个基础的数学工具包
    package.json
    index.js
apps/
  app/                  <-- 一个业务应用包
    package.json
    index.js
```

### 第 1 步：安装工作区依赖，构建连接
在根目录下运行以下命令将本地工作区连接起来：
```bash
pnpm install
```
*💡 原理：即使没有外部依赖，pnpm 也会扫描整个工作区，生成统一的 `pnpm-lock.yaml` 文件。*

### 第 2 步：给 app 添加对 math 的本地依赖
我们要让 `apps/app` 使用 `packages/math`，使用 `--filter` 参数指定只针对哪个包执行命令：
```bash
pnpm --filter @my-workspace/app add @my-workspace/math@workspace:*
```
*💡 观察：命令执行完后，点开 `apps/app/package.json`，它的 `dependencies` 增加了一行 `"@my-workspace/math": "workspace:^"`。此时，去 `apps/app/node_modules` 目录下看，你会发现 `@my-workspace/math` 是一个带有快捷方式箭头的**软链接**！*

### 第 3 步：运行并验证结果
在 `apps/app/index.js` 中引用了 `math` 包并输出了结果。在终端运行它：
```bash
node apps/app/index.js
```
输出：
```text
1 + 2 = 3
5 - 3 = 2
```

### 第 4 步：体验 Monorepo 最强特性（即时生效）
1. 修改 `packages/math/index.js`，给加法或减法添加特殊的打印或返回值。
2. **不用执行任何重新安装或构建操作**，直接再次运行 `node apps/app/index.js`。
3. 你会发现修改立刻生效了，因为 `app` 只是链接到了 `math` 的源代码！这就是 Monorepo 最提升开发体验的地方。

### 第 5 步：熟悉工作区全局指令
未来如果你的依赖变多了，或者需要给所有包同时运行某些命令（比如所有包都要执行 build），可以使用递归参数 `-r` 或 `--recursive`：
```bash
# 给工作区所有包含 build 脚本的项目执行构建
pnpm -r run build 

# 往所有工作区子项目中都安装 lodash 作为依赖
pnpm -r add lodash
```
