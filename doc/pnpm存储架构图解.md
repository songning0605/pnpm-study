# pnpm 依赖存储与关联引用机制图解

本文档通过 Mermaid 架构图详细展示了 pnpm 是如何通过全局存储（CAS）、虚拟存储区（`.pnpm`）以及各种链接机制（硬链接与软链接）来管理项目依赖的。

## 机制图解

```mermaid
graph TD
    %% 定义全局仓库
    subgraph Global_Store ["~/.pnpm-store (全局内容寻址存储 CAS)"]
        H_Antd[("antd@6.3.4\n(真实数据块)")]
        H_Dayjs[("dayjs@1.11.11\n(真实数据块)")]
        H_React[("react@18.2.0\n(真实数据块)")]
    end

    %% 定义项目结构
    subgraph Project_Workspace ["项目工作区 (pnpm-test)"]
        
        %% 项目根 node_modules
        subgraph Node_Modules ["node_modules/"]
            S_Root_Antd{{"antd\n(软链接)"}}
            S_Root_React{{"react\n(软链接)"}}
        end

        %% 虚拟存储区 .pnpm
        subgraph Virtual_Store [".pnpm/ (虚拟存储区)"]
            
            %% antd 的专有解析目录
            subgraph Antd_Resolution ["antd@6.3.4_react.../node_modules/"]
                S_Antd_Self{{"antd\n(硬链接/自引用)"}}
                S_Antd_Dayjs{{"dayjs\n(软链接)"}}
                S_Antd_React{{"react\n(软链接)"}}
            end
            
            %% dayjs 的专有解析目录
            subgraph Dayjs_Resolution ["dayjs@1.11.11/node_modules/"]
                S_Dayjs_Self{{"dayjs\n(硬链接)"}}
            end
            
            %% react 的专有解析目录
            subgraph React_Resolution ["react@18.2.0/node_modules/"]
                S_React_Self{{"react\n(硬链接)"}}
            end
        end
    end

    %% -------------------- 链接关系绘制 --------------------

    %% 全局存储到虚拟存储的硬链接 (Hard Links)
    H_Antd == "硬链接 (物理共享)" ===> S_Antd_Self
    H_Dayjs == "硬链接 (物理共享)" ===> S_Dayjs_Self
    H_React == "硬链接 (物理共享)" ===> S_React_Self

    %% 项目根目录到虚拟存储的软链接 (Symlinks)
    S_Root_Antd -. "软链接 (控制项目可见性)" .-> S_Antd_Self
    S_Root_React -. "软链接 (控制项目可见性)" .-> S_React_Self

    %% 虚拟存储内部的软链接 (处理依赖关系)
    S_Antd_Dayjs -. "软链接 (antd依赖dayjs)" .-> S_Dayjs_Self
    S_Antd_React -. "软链接 (antd依赖react)" .-> S_React_Self

    %% -------------------- 样式设置 --------------------
    classDef store fill:#f9f9f9,stroke:#333,stroke-width:2px;
    classDef hardlink fill:#e1f5fe,stroke:#0288d1,stroke-width:2px;
    classDef symlink fill:#f1f8e9,stroke:#689f38,stroke-width:2px,stroke-dasharray: 5 5;
    
    class Global_Store store;
    class H_Antd,H_Dayjs,H_React hardlink;
    class S_Root_Antd,S_Root_React,S_Antd_Dayjs,S_Antd_React symlink;
    class S_Antd_Self,S_Dayjs_Self,S_React_Self hardlink;
```

## 核心原理解析

1. **最上层（全局仓库）**：只存储真正的物理数据块，按内容寻址（CAS），确保全电脑只存一份相同的数据。
2. **中间层（虚拟存储区 `.pnpm`）**：
   * 通过 **硬链接** 从全局拉取数据（如 `antd(硬链接)`）。
   * 通过内部的 **软链接** 构建包与包之间的依赖树（如 `antd` 指向 `dayjs`）。
   * **自引用机制**：`antd` 在自己的目录下还有一个硬链接，用以利用 Node.js 模块查找逻辑并支持包的内部自引用。
3. **最下层（项目根目录 `node_modules`）**：仅包含在 `package.json` 中明确声明的依赖的 **软链接**。这严格控制了代码的可见性，杜绝了“幽灵依赖”问题。