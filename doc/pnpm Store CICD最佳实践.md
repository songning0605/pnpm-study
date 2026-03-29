# CI/CD 中 pnpm 缓存依赖最佳实践

基于 pnpm 独特的全局内容寻址存储（CAS）和硬链接机制，在 CI/CD 流水线（如 GitHub Actions, GitLab CI, Jenkins 等）中配置缓存的最佳实践与 npm/Yarn 有着本质的区别。

## 1. 核心理念：缓存全局 Store，而非 `node_modules`

*   **不要缓存 `node_modules`**：因为 pnpm 的 `node_modules` 内部充斥着大量的符号链接（Symlinks）和硬链接（Hard Links）。缓存和恢复这些链接不仅极易出错（导致链接断裂），而且无法跨操作系统或不同的磁盘挂载点工作。
*   **必须缓存全局 Store (`~/.pnpm-store`)**：这是 pnpm 的“灵魂”。只要全局 Store 被缓存，执行 `pnpm install` 时就可以实现秒级的硬链接复用，完全跳过网络下载。

## 2. 最佳实践流程

### 步骤 1：获取并设置 Store 路径
在 CI 环境中，默认的 store 路径可能不可靠或难以定位。最佳做法是显式配置一个相对于项目或工作区的 store 路径，或者获取 pnpm 的默认路径。

```bash
# 获取当前 pnpm 的 store 路径
STORE_PATH=$(pnpm store path --silent)
```

### 步骤 2：配置 CI 缓存规则
以 GitHub Actions 为例，你需要以 `pnpm-lock.yaml` 的哈希值作为缓存的 Key。

```yaml
- name: Get pnpm store directory
  id: pnpm-cache
  shell: bash
  run: |
    echo "STORE_PATH=$(pnpm store path --silent)" >> $GITHUB_ENV

- name: Setup pnpm cache
  uses: actions/cache@v3
  with:
    path: ${{ env.STORE_PATH }}
    key: ${{ runner.os }}-pnpm-store-${{ hashFiles('**/pnpm-lock.yaml') }}
    restore-keys: |
      ${{ runner.os }}-pnpm-store-
```

### 步骤 3：执行安装（使用无头模式）
在 CI 中，**永远不要使用 `pnpm install`**，而应该使用 `pnpm install --frozen-lockfile`（类似 npm 的 `npm ci`）。

*   **原因**：它严格依据 `pnpm-lock.yaml` 安装，不会因为包的微小版本更新而意外修改锁文件，保证了构建的一致性。

```yaml
- name: Install dependencies
  run: pnpm install --frozen-lockfile
```

## 3. 各大 CI 平台的简易配置示例

### GitHub Actions (最简推荐用法)
如今 GitHub Actions 有了官方推荐的 Setup 步骤，内置了缓存逻辑：

```yaml
steps:
  - uses: actions/checkout@v3
  
  - name: Install pnpm
    uses: pnpm/action-setup@v2
    with:
      version: 8 # 指定你的 pnpm 版本
      
  - name: Setup Node.js
    uses: actions/setup-node@v3
    with:
      node-version: 18
      cache: 'pnpm' # 神奇开关：自动为你处理 pnpm store 的缓存！
      
  - name: Install dependencies
    run: pnpm install --frozen-lockfile
```

### GitLab CI
在 GitLab CI 中，你需要把 store 路径设置在项目目录内（因为 GitLab 只能缓存工作区内的文件）。

```yaml
variables:
  # 将 store 设置在项目内部
  pnpm_config_store_dir: ${CI_PROJECT_DIR}/.pnpm-store

cache:
  key:
    files:
      - pnpm-lock.yaml
  paths:
    - .pnpm-store

install:
  script:
    - corepack enable
    - corepack prepare pnpm@latest --activate
    - pnpm config set store-dir $pnpm_config_store_dir
    - pnpm install --frozen-lockfile
```

## 4. 注意事项与避坑指南

1.  **定期清理缓存**：随着时间推移，全局 store 会不断膨胀。如果 CI 平台不支持自动清理旧缓存，可以通过 `pnpm store prune` 命令定期修剪不再使用的孤儿包。
2.  **跨操作系统限制**：缓存的 Store **不能**在 Linux、macOS 和 Windows 之间混用（注意缓存 Key 中的 `${{ runner.os }}`）。
3.  **不要同时缓存 Store 和 `.next`/`dist`**：缓存策略要清晰。Store 负责依赖极速恢复，构建产物缓存负责代码加速。混合缓存可能导致路径错乱。

## 5. Kubernetes (K8s) Pod 临时环境中的缓存方案

在基于 Kubernetes 调度的动态 CI/CD 环境（如 Tekton, Jenkins Kubernetes Plugin, GitLab Runner Kubernetes executor）中，每次构建都在一个全新的 Pod 中运行，构建结束后 Pod 立即销毁。由于容器的临时性，本地磁盘的 `.pnpm-store` 无法留存，导致每次都会全量下载。

针对这种“阅后即焚”的环境，最佳实践是**持久化存储挂载**或**分布式缓存系统**。

### 方案 1：使用持久化卷 (Persistent Volume Claim, PVC) 共享 Store
这是 K8s 环境下最简单、性能最高的方案。

*   **原理**：创建一个全局共享的 PVC（如 NFS, CephFS 等支持 `ReadWriteMany` 的存储），将其挂载到所有执行构建任务的 Pod 中的同一个路径。
*   **操作**：
    1. 在 Pod 模板中挂载 PVC（例如挂载到 `/mnt/pnpm-store`）。
    2. 配置 pnpm 使用这个路径：`pnpm config set store-dir /mnt/pnpm-store`。
*   **优点**：真正实现了多项目、多并发构建时的秒级硬链接复用（因为挂载的 PVC 依然支持硬链接）。
*   **缺点**：如果使用网络存储（NFS），高频小文件的 I/O 性能可能成为瓶颈。

### 方案 2：将 Store 打包上传至对象存储 (S3/MinIO) 
如果 K8s 集群不方便配置共享存储，可以退化为“下载-解压-上传”的传统 CI 缓存模式。

*   **原理**：
    1. **恢复缓存（构建前）**：从 S3/MinIO 下载之前打包的 `.pnpm-store.tar.gz` 到当前 Pod 的本地磁盘并解压。
    2. **执行安装**：配置 `store-dir` 指向解压后的本地目录，运行 `pnpm install`。
    3. **保存缓存（构建后）**：如果 `pnpm-lock.yaml` 发生变化，将新的 `.pnpm-store` 重新打包上传回 S3。
*   **工具支持**：大多数 CI 引擎的 K8s 运行器（如 GitLab Runner Cache）默认支持对接 S3，只要配置了缓存路径，底层会自动完成上传下载。
*   **优点**：架构解耦，不依赖底层存储驱动，适用于多集群环境。
*   **缺点**：每次构建都需要传输巨大的包，网络带宽和打包/解压时间消耗较大。

### 方案 3：在基础镜像中预置“部分 Store” (预热镜像)
如果你的公司有特定的技术栈（比如所有项目都用 React, Antd, Lodash），可以考虑构建一个包含这些基础包 Store 的 Docker 镜像。

*   **原理**：
    1. 编写一个 `Dockerfile`，在构建时运行 `pnpm fetch` 拉取常用包并生成 Store。
    2. CI 任务的 Pod 使用这个预热镜像作为基础镜像。
*   **操作**：后续在 Pod 中执行 `pnpm install` 时，因为基础镜像里已经包含了大部分常用包，pnpm 会瞬间硬链接这些包，只下载项目中特有的少量依赖。
*   **优点**：不需要网络下载大包，启动极快。
*   **缺点**：需要定期维护基础镜像，不够灵活。

## 6. 特殊优化方案：`store-dir=node_modules/.pnpm-store`

将 pnpm 的 Store 强制配置在项目内部（如 `.npmrc` 中设置 `store-dir=node_modules/.pnpm-store`）是一个非常有争议但在特定场景下极为有效的优化手段。

### 怎么上传？
在构建完成并且确定 pnpm-lock.yaml 有更新后，利用 CI 工具的内置缓存机制（如 GitLab CI 的 cache 块）或自己编写脚本（如 tar 命令结合 aws s3 cp 等工具），将整个包含 .pnpm-store 的 node_modules 目录打包压缩并上传到统一的存储系统（S3、MinIO 等分布式存储，或 GitLab Runner 的本地/网络缓存系统）。
注意点：之所以连着整个 node_modules 一起打包，是因为解压缩时，底层压缩工具（如 tar）能够原样保留所有内部硬链接关系。相当于直接保存了一个安装好的快照，下次解压出来就可以直接用。

### 怎么重新恢复？
在每次新的构建任务（新的 K8s Pod 或隔离环境）启动、但在执行 pnpm install 之前，通过脚本（aws s3 cp 或 CI 工具原生拉取命令）将刚才上传的压缩包下载并解压回项目目录。
随后，只要确保执行了 pnpm config set store-dir node_modules/.pnpm-store 并运行 pnpm install --frozen-lockfile，pnpm 就会瞬间确认所需的包都在解压回来的 .pnpm-store 中（并且很可能硬链接都已经恢复好了），几乎只需几秒就能完成所谓的“安装”过程。

### 核心原理：解决“跨盘硬链接失效”
这是该方案最主要的价值所在。
* **痛点**：硬链接（Hard Link）在操作系统层面**绝对不能跨越不同的物理磁盘分区**。如果你的全局 Store 在 `C:` 盘，而项目代码在 `D:` 盘，执行 `pnpm install` 时，操作系统会拒绝创建硬链接。
* **pnpm 的退化行为**：一旦无法创建硬链接，pnpm 被迫回退到“文件复制（Copy）”模式。这会导致大量的磁盘 I/O 读写，安装极慢，并且完全失去了 pnpm 节省磁盘空间的优势。
* **优化结果**：设置 `store-dir=node_modules/.pnpm-store` 后，Store 必然和项目代码处于同一个磁盘分区，**强行恢复了硬链接能力**，瞬间解决跨盘复制的性能灾难。

### 场景评估与建议

| 场景 | 是否推荐 | 详细评估说明 |
| :--- | :---: | :--- |
| **项目与全局 Store 不在同一磁盘** | **✅ 强烈推荐** | 必须使用。这是解决跨盘导致 pnpm 退化为“龟速复制模式”的唯一有效方案。 |
| **K8s Pod / Docker 隔离构建** | **✅ 推荐** | 在容器内，将 Store 放在项目内部意味着环境是完全**自包含（Self-contained）**的。这避免了挂载外部 Volume 的权限问题，也使得基于项目目录的缓存策略变得极其简单（只需打包整个工作区）。 |
| **部分 CI/CD 流水线 (GitLab CI)** | **✅ 推荐** | GitLab CI 的 cache 机制默认只能缓存项目工作区内的文件。将 Store 移入项目内部，是让 GitLab CI 能够成功缓存 pnpm Store 的标准做法。 |
| **单磁盘的普通个人开发环境** | **❌ 坚决反对** | 这样做会**彻底废掉 pnpm 最核心的全局共享优势**。如果你的电脑上有 10 个项目，硬盘上就会出现 10 份冗余的包数据，磁盘占用将极其恐怖，且对单盘安装速度没有任何提升。 |

**总结结论**：
该方案是一把“双刃剑”。它是为了对抗物理隔离（跨磁盘、跨容器）导致硬链接失效而诞生的“特权解药”，绝不能作为通用的默认配置在个人开发机上滥用。