# pnpm Catalogs 与 `catalog:` 协议

## Catalogs 是什么

**Catalogs** 是 pnpm 的**工作空间（workspace）功能**：在 monorepo 里把一批依赖的版本写成**可复用的常量**，集中维护；各子包在 `package.json` 里用 `**catalog:` 协议**引用这些常量。

- 只在配置了 `**pnpm-workspace.yaml`** 的多包仓库里有意义；单包项目没有 workspace 级 catalog，也就谈不上用 `catalog:`。
- 版本表定义在 **workspace root** 的 `pnpm-workspace.yaml`（`catalog` / `catalogs`），子包不写死 semver，而是写 `"react": "catalog:"`，解析时等价于在 catalog 里写好的范围（如 `^18.3.1`）。

官方文档：[https://pnpm.io/zh/catalogs](https://pnpm.io/zh/catalogs)

## 为什么说「工作空间功能」


| 含义   | 说明                                                                                                                 |
| ---- | ------------------------------------------------------------------------------------------------------------------ |
| 定义位置 | 在 **workspace 根** 的 `pnpm-workspace.yaml`，不是某个子包单独一份                                                               |
| 使用方式 | 各 **workspace 包** 的 `dependencies` / `devDependencies` / `peerDependencies` / `optionalDependencies` 里写 `catalog:` |
| 目的   | 多包共享同一依赖时，**统一版本、少改多处 `package.json`、减少升级时的合并冲突**                                                                  |


## 「Catalog 协议」指什么

在 `package.json` 里，依赖的版本不是只能写 `^18.0.0` 这种纯 semver，还可以用**带前缀的写法**，表示「版本从哪来、怎么解析」。例如：

- `workspace:`* —— workspace 协议
- `file:../foo` —— file 协议
- `catalog:` —— **catalog 协议**

所以 **「Catalog 目录协议」里的「协议」**，指的就是 `catalog:` **这种依赖声明方式**：pnpm 看到它，就去工作区里的 **catalog 定义**里查版本，而不是把 `catalog:` 当成字面版本号。

### 发布时的行为

执行 `pnpm publish` 或 `pnpm pack` 时，`catalog:` 会被**展开成普通 semver**，和 `workspace:` 类似，便于其他环境或其他包管理器消费已发布的包。

## 在 `pnpm-workspace.yaml` 里怎么写

**默认 catalog**（名为 `default`，子包里可写 `catalog:` 简写）：

```yaml
catalog:
  react: ^18.2.0
  react-dom: ^18.2.0
```

**多个具名 catalog**：

```yaml
catalogs:
  react17:
    react: ^17.0.2
    react-dom: ^17.0.2
  react18:
    react: ^18.2.0
    react-dom: ^18.2.0
```

也可与顶层 `catalog` 同时存在；迁移旧项目可用：`pnpx codemod pnpm/catalog`。

## 相关设置（可选）

- `**catalogMode**`（v10.12.1+）：控制 `pnpm add` 是否/如何把依赖写入默认 catalog，取值 `manual`（默认）/ `strict` / `prefer`。
- `**cleanupUnusedCatalogs**`（v10.15.0+）：为 `true` 时安装过程中会删掉未使用的 catalog 条目。

