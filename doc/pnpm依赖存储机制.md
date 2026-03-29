# pnpm 依赖存储机制详解

本文总结了 pnpm 如何通过虚拟存储、软链接和硬链接高效管理项目依赖。

## 1. 虚拟存储目录：`node_modules/.pnpm`
`node_modules/.pnpm` 被称为 **虚拟存储（Virtual Store）**，是 pnpm 的核心。

*   **平铺布局**：所有依赖（直接和间接）都以平铺方式存放在此目录下，避免了传统 `node_modules` 的嵌套深度问题。
*   **内容寻址**：此目录下的文件实际上是全局内容寻址存储（Content-addressable store）的**硬链接**。

## 2. 复杂的目录命名（依赖解析键）
在 `.pnpm` 下常看到类似 `antd@6.3.4_react-dom@19.2.4_react@19.2.4` 的复杂名称。

*   **原因**：这是为了处理 **Peer Dependencies（对等依赖）**。
*   **原理**：同一个包如果关联了不同版本的 Peer Dependencies，pnpm 会为每一组组合创建唯一的存储路径，确保依赖隔离，彻底解决“幽灵依赖”问题。

## 3. 软链接 vs 硬链接

pnpm 结合使用了两种链接技术：

| 特性 | 软链接 (Symbolic Link) | 硬链接 (Hard Link) |
| :--- | :--- | :--- |
| **本质** | 路径快捷方式 | 文件的另一个名称 |
| **pnpm 中的应用** | 将 `.pnpm` 中的包链接到项目根目录 `node_modules` | 将全局存储中的文件链接到项目的 `.pnpm` 目录 |
| **作用** | 控制代码对依赖的可见性 | 节省磁盘空间，实现极速安装 |
| **删除源文件** | 链接失效 | 只要还有一个硬链接存在，数据就不会丢失 |

## 4. 如何查看文件链接状态

### 查看软链接
在终端运行：
```bash
ls -l node_modules/package-name
# 输出示例：lrwxr-xr-x ... package-name -> .pnpm/xxx/node_modules/package-name
```

### 查看硬链接
在终端进入包目录后运行：
```bash
ls -l package.json
# 查看第二列的数字（链接计数）。如果数字 > 1，则通常是硬链接。

stat package.json
# 查看 Inode 编号。不同项目间相同包的相同文件，其 Inode 编号是一致的。
```

---
*本文由 GitHub Copilot 生成，基于对 pnpm 机制的深度解析。*
