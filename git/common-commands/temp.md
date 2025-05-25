根据用户的要求，我将参考“基本操作指令”部分的前两个命令（“添加子模块”和“克隆含子模块项目”）的编写风格，将整个文档中的类似命令列表改为四级标题格式。每条命令将使用四级标题（`####`），后跟代码块中的命令和简短描述。以下是全文重写后的版本：

---

### 一、基本操作指令

#### 添加子模块
`git submodule add <仓库地址> [路径]`  
添加子模块并生成`.gitmodules`。

#### 克隆含子模块项目
`git clone --recursive <主仓库地址>`  
克隆主仓库及所有子模块。

#### 初始化子模块
`git submodule init`  
将`.gitmodules`配置写入`.git/config`。

#### 更新子模块
`git submodule update`  
拉取子模块代码并检出记录的提交。

#### 同步子模块URL
`git submodule sync`  
更新子模块URL配置。

#### 删除子模块
`git rm <子模块路径>`  
`git deinit <子模块路径>`  
删除子模块记录并解除初始化。

---

### 二、状态查看指令

#### 查看子模块状态
`git submodule status`  
显示子模块路径、提交哈希和分支信息。

#### 查看子模块更新摘要
`git submodule summary`  
显示子模块与主仓库记录的差异。

#### 查看项目状态（含子模块）
`git status --submodules`  
显示主仓库和子模块状态。

#### 查看子模块差异
`git diff --submodule`  
显示主仓库与子模块差异。

---

### 三、更新与同步指令

#### 更新子模块到远程最新
`git submodule update --remote`  
拉取子模块远程分支最新提交。

#### 拉取所有子模块最新代码
`git submodule foreach git pull`  
遍历子模块执行`git pull`。

#### 拉取主仓库并更新子模块
`git pull --recurse-submodules`  
同时更新主仓库和子模块。

#### 获取子模块最新元数据
`git submodule foreach git fetch`  
拉取子模块最新元数据。

---

### 四、分支管理指令

#### 添加子模块并指定分支
`git submodule add -b <分支名> <仓库地址> [路径]`  
添加子模块并设置跟踪分支。

#### 设置子模块跟踪分支
`git config -f .gitmodules submodule.<name>.branch <分支名>`  
在`.gitmodules`中设置跟踪分支。

#### 切换子模块分支
`git submodule foreach git checkout <分支名>`  
遍历子模块切换分支。

#### 切换主仓库分支并同步子模块
`git checkout --recurse-submodules <分支名>`  
切换主仓库分支并更新子模块。

---

### 五、高级操作指令

#### 递归遍历子模块
`git submodule foreach --recursive <命令>`  
递归执行命令于所有子模块。

#### 初始化并执行命令
`git submodule foreach --init <命令>`  
初始化未初始化的子模块并执行命令。

#### 合并子模块.git目录
`git submodule absorbgitdirs`  
将子模块`.git`目录合并到主仓库。

#### 恢复子模块独立.git目录
`git submodule deabsorb`  
恢复子模块独立`.git`目录。

---

### 六、协同操作指令

#### 记录子模块变更
`git add <子模块路径>`  
记录子模块新提交哈希到主仓库。

#### 推送前检查子模块
`git push --recurse-submodules=check`  
确保子模块变更已推送。

#### 推送主仓库并自动推送子模块
`git push --recurse-submodules=on-demand`  
推送主仓库并自动推送子模块变更。

---

### 七、配置设置指令

#### 设置子模块更新策略
`git config submodule.<name>.update <策略>`  
设置更新策略（`none`、`checkout`、`rebase`、`merge`）。

#### 设置子模块跟踪分支
`git config submodule.<name>.branch <分支名>`  
设置跟踪分支。

#### 全局启用子模块递归
`git config submodule.recurse true`  
启用子模块命令默认递归。

---

### 八、实用组合指令

#### 批量更新子模块到主分支
`git submodule foreach --recursive 'git checkout main && git pull origin main'`  
遍历所有子模块，切换到主分支并拉取最新代码。

#### 检查子模块与远程主分支同步
`git submodule foreach --recursive 'git fetch origin main && [ "$(git rev-parse HEAD)" = "$(git rev-parse origin/main)" ] && echo "✅ $name 一致" || echo "❌ $name 不一致"'`  
检查每个子模块的当前提交是否与远程主分支一致。

#### 递归克隆并更新子模块
`git clone --recursive <主仓库地址>`  
`git submodule update --init --recursive`  
递归克隆主仓库并初始化、更新所有子模块。

#### 设置跟踪分支并更新子模块
`git config -f .gitmodules submodule.<name>.branch main`  
`git submodule update --remote`  
`git add <子模块路径>`  
`git commit -m "Update submodule"`  
设置子模块跟踪分支，更新到最新提交，并记录到主仓库。

---

### 九、最佳实践

#### 克隆时使用`git clone --recursive`
确保一次性克隆主仓库和所有子模块，避免后续手动初始化和更新。

#### 设置子模块跟踪分支并定期更新
通过在`.gitmodules`中设置`branch`属性，并使用`git submodule update --remote`定期更新子模块。

#### 启用`git config submodule.recurse true`简化操作
使Git命令默认递归到子模块，减少手动指定`--recurse-submodules`的需要。

#### 子模块变更后先提交子模块再提交主仓库
确保子模块的变更先被提交和推送，然后在主仓库中记录新的提交哈希。

#### 推送时使用`git push --recurse-submodules=on-demand`
自动推送子模块的变更，确保主仓库和子模块的提交同步。

---

### 说明
- 所有命令均按照“基本操作指令”中“添加子模块”和“克隆含子模块项目”的风格重写，使用四级标题（`####`），后接代码块中的命令和简短描述。
- 对于多命令操作（如“删除子模块”或“设置跟踪分支并更新子模块”），将所有相关命令放在同一标题下。
- “最佳实践”部分原为简短的建议列表，现改为四级标题并附上简要说明，以保持一致性。
- 文档结构保持原有的大章节（二级标题），仅将命令列表改为四级标题格式。

这样，文档已完全按照要求重写，风格统一且清晰易读。