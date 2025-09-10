# 使用 pnpm 创建和管理 monorepo 子项目

在单一仓库（monorepo）中管理多个项目是大型应用或需要代码共享的团队的强大解决方案。[pnpm](https://pnpm.io/) 是一个快速且节省磁盘空间的包管理器，其工作空间功能让 monorepo 管理变得更加高效。本文将带你一步步完成使用 pnpm 创建 monorepo 和子项目的流程，包含实用示例。

## 前提条件

- **Node.js**：确保已安装 Node.js（建议 v16 或更高版本）。
- **pnpm**：如果尚未安装 pnpm，可通过以下命令全局安装：
    
    ```bash
    npm install -g pnpm
    ```
    
- 对 JavaScript 和 `package.json` 有基本了解。

## 步骤 1：初始化 monorepo

monorepo 需要一个根目录和一个配置文件来定义工作空间。

1. **创建根目录**：  
    创建一个 monorepo 文件夹并进入：
    
    ```bash
    mkdir my-monorepo
    cd my-monorepo
    ```
    
2. **初始化 Node.js 项目**：  
    在根目录下运行以下命令，创建一个 `package.json` 文件：
    
    ```bash
    npm init -y
    ```
    
    这会生成一个基础的 `package.json` 文件。
    
3. **设置 pnpm 工作空间**：  
    在根目录下创建 `pnpm-workspace.yaml` 文件，指定子项目所在目录：
    
    ```yaml
    packages:
      - 'packages/*'
    ```
    
    这表示所有子项目都将位于 `packages/` 目录下，`packages/*` 会匹配该目录中的所有子文件夹。
    

## 步骤 2：创建子项目

每个子项目是一个独立的模块，通常存储在 `packages/` 目录下，拥有自己的 `package.json`。

1. **创建子项目目录**：  
    假设我们要创建一个名为 `sub-project1` 的子项目：
    
    ```bash
    mkdir -p packages/sub-project1
    cd packages/sub-project1
    ```
    
2. **初始化子项目**：  
    为子项目生成 `package.json`：
    
    ```bash
    npm init -y
    ```
    
    编辑 `package.json`，设置项目名称、版本等信息：
    
    ```json
    {
      "name": "@my-monorepo/sub-project1",
      "version": "1.0.0",
      "description": "monorepo 中的第一个子项目",
      "main": "index.js",
      "scripts": {
        "start": "node index.js"
      }
    }
    ```
    
3. **添加示例代码**：  
    在 `packages/sub-project1` 目录下创建 `index.js` 文件：
    
    ```javascript
    console.log("来自 sub-project1 的问候！");
    ```
    
4. **创建更多子项目**：  
    如果需要其他子项目，例如 `sub-project2`，重复上述步骤：
    
    ```bash
    mkdir packages/sub-project2
    cd packages/sub-project2
    npm init -y
    ```
    
    同样更新其 `package.json` 并添加 `index.js` 文件。
    

## 步骤 3：管理依赖

pnpm 的依赖管理高效，支持子项目共享依赖或独立安装。

1. **安装共享依赖**：  
    在根目录下为整个工作空间安装共享依赖。例如，安装 `lodash`：
    
    ```bash
    pnpm add lodash --workspace
    ```
    
    这会将 `lodash` 安装到根目录的 `node_modules`，并通过符号链接共享给所有子项目。
    
2. **为子项目安装特定依赖**：  
    如果某个子项目需要特定依赖，进入该子项目目录并安装。例如，为 `sub-project1` 安装 `axios`：
    
    ```bash
    cd packages/sub-project1
    pnpm add axios
    ```
    
3. **子项目间依赖**：  
    如果 `sub-project1` 需要依赖 `sub-project2`，在 `sub-project1` 的 `package.json` 中添加：
    
    ```json
    {
      "dependencies": {
        "@my-monorepo/sub-project2": "workspace:*"
      }
    }
    ```
    
    然后在根目录运行以下命令以链接依赖：
    
    ```bash
    pnpm install
    ```
    

## 步骤 4：配置脚本

在根目录的 `package.json` 中添加脚本，方便管理所有子项目：

```json
{
  "scripts": {
    "start:sub1": "pnpm --filter @my-monorepo/sub-project1 start",
    "start:sub2": "pnpm --filter @my-monorepo/sub-project2 start",
    "install:all": "pnpm install",
    "start:all": "pnpm --parallel run start"
  }
}
```

- `--filter`：用于指定某个子项目的命令。
- `--parallel`：并行运行所有子项目的命令。

## 步骤 5：安装和测试

1. **安装所有依赖**：  
    在根目录运行：
    
    ```bash
    pnpm install
    ```
    
    这会根据 `pnpm-workspace.yaml` 和各子项目的 `package.json` 安装所有依赖。
    
2. **运行子项目**：  
    使用配置的脚本运行子项目。例如：
    
    ```bash
    pnpm start:sub1
    ```
    
    这应该会输出 `来自 sub-project1 的问候！`。
    
3. **同时运行所有子项目**：  
    使用 `start:all` 脚本并行运行所有子项目：
    
    ```bash
    pnpm start:all
    ```
    

## 步骤 6：可选配置

1. **添加 `.gitignore`**：  
    在根目录创建 `.gitignore` 文件，忽略不需要的文件：
    
    ```gitignore
    node_modules/
    .pnpm-store/
    ```
    
2. **自定义 pnpm 配置**：  
    在根目录创建 `.npmrc` 文件，配置 pnpm 行为，例如：
    
    ```plaintext
    shamefully-hoist=true
    ```
    
    这会将依赖提升到根目录的 `node_modules`，方便某些工具使用。
    
3. **添加 TypeScript（可选）**：  
    如果子项目需要使用 TypeScript，可以在子项目中安装并初始化：
    
    ```bash
    cd packages/sub-project1
    pnpm add typescript
    npx tsc --init
    ```
    

## 步骤 7：验证设置

- 使用以下命令检查工作空间结构：
    
    ```bash
    pnpm list --depth 0
    ```
    
- 确保 `node_modules/.pnpm` 中的依赖正确生成。

## 示例目录结构

完成以上步骤后，目录结构可能如下：

```
my-monorepo/
├── packages/
│   ├── sub-project1/
│   │   ├── package.json
│   │   ├── index.js
│   ├── sub-project2/
│   │   ├── package.json
│   │   ├── index.js
├── node_modules/
├── package.json
├── pnpm-workspace.yaml
├── .gitignore
```

## 提示与最佳实践

- __使用 workspace:_ 引用本地依赖_*：确保子项目间通过 `workspace:*` 引用，避免版本冲突。
- **利用 pnpm 的高效性**：pnpm 使用硬链接节省磁盘空间，非常适合大型 monorepo。
- **调试**：如果依赖安装失败，检查 `pnpm-workspace.yaml` 和 `package.json` 是否正确配置。
- **扩展工具**：根据需要为根目录或子项目添加 Vite、ESLint 或 Jest 等工具。

## 总结

通过 pnpm，你可以轻松设置一个包含多个子项目的 monorepo，享受高效的依赖管理和清晰的项目结构。按照本文的步骤，你可以快速构建一个可扩展、可维护的开发环境。尝试一下，并探索 pnpm 的高级功能（如过滤和递归命令）来优化你的工作流！

更多详情，请参考 [pnpm 官方文档](https://pnpm.io/)，或在你的 monorepo 中实验更多工具。