# 子模块相关操作

## 常见操作

### 克隆仓库

#### 标准化克隆流程
- 基本命令
	``` bash
	git clone <父仓库远程URL> [本地文件名]
	cd <本地仓库名>
	git submodule init   # 初始化子模块配置
	git submodule update # 拉取子模块内容
	```

#### 一次性完成克隆和初始化流程
- 基本命令
	``` bash
	git clone --recurse-submodules <父仓库远程URL> [本地文件名]
	```

#### 其他克隆流程
- 基本命令
	``` bash
	git clone <远程仓库URL> [本地文件名]
	cd <本地仓库名>
	git submodule update --init --recursive
	```

### 添加子模块配置
#### 标准化添加流程
- 添加仓库命令
	``` bash
	git submodule add <子目录远程仓库URL> [子模块名]
	# 例：
	git submodule add $(cd blog && git remote get-url origin) blog # 将blog这个模块作为子模块添加到当前git仓库中；使用 git remote get-url <本地仓库名>
	```
- 成功添加后的表现
	- 生产或修改`.gitmodules`文件，若已有此文件，则直接写入新增的子模块配置，否则新建一个名为`.gitmodules`的文件并写入第一关配置
	- 提交时，子模块以特殊模式（`160000`）记录，表示其指向某个具体提交而非普通文件。

### 移除子模块配置
#### 标准化写在流程
- 命令行卸载子模块
	``` bash
	git submodule deinit <子模块的相对路径> # 在.git/config 和 .gitsubmodule中移除配置
	# 例：
	git submodule deinit "blog"
	```
- 删除子模块并暂存变更
	``` bash
	git rm <子模的相对块路径>
	```
- 清理残留文件：彻底删除子模块的Git缓存
	``` bash
	rm -rf .git/modules/<子模块名称>
	```
- 提交变更

### 修改子模块配置

#### 更新子模块路径以及名称
- 使用命令移动或重命名子模块目录
	``` bash
	git mv old/path new/path # 将旧的路由更名为新的路由
	```
	执行完毕后`.gitmodules`中的路由配置信息会自动更新
	``` bash
	git config -f .gitmodules submodule.<submodule-neme>.path new/path # 如果没有自动更新可使用此命令更新，或手动编辑 .gitmodules 文件
	```
- 使用文件编辑器方式更新子模块名称
	``` bash
	vim .gitmodules # 使用任何编辑方式均可
	```
	修改`[submodule "<子模块名>"]`中的配置
	``` gitmodules
	[submodule "old-name"]  => [submodule "new-name"] # 将名称配置修改为新的名称（最好与路径字符串统一）
	path = <new/path>
    url = <URL保持不变>
	```
- 在其他地方使用最新的仓库
	``` bash
	git pull # 拉取最新内容，修改后的路径会以一个全新的文件夹的方式新增在这里
	rm -rf old/path # 移除掉以及被淘汰的旧路径
	git submodule update # 更新子模块内容，如果此命令无效可先输入 git submoduel init
	cd new/path
	git switch <目标分支> # 新克隆的子模块的分支处于游离状态，所以需呀切换到有效分支在做操作
	```

#### 更新子模块远程地址

- 使用命令更新子模块路径
	``` bash
	git config -f .gitmodules submodule.<submodule-name>.url <new-url>
	```
	自动更新`.gitmodules`文件中的呢日欧能够，或使用文本编辑系做如下修改
	``` gitmodules
	[submodule "submodule-name"]
	path = <路径保持不变>
    url = <old-url> => url = <new-url> # 将URL配置修改为新的URL
	```
- 使用命令更新子模块
	``` bash
	git submodule update --remote
	```
- 在其他地方使用最新的仓库
	``` bash
	git pull # 拉取最新内容，修改后的路径会以一个全新的文件夹的方式新增在这里
	git submodule update # 更新子模块内容，如果此命令无效可先输入 git submoduel init
	```

#### 更新子模块内容，以及同步父模块

- 从本地更新到远程
	``` bash
	cd path
	git add .
	git commit -m "<提交信息>"
	git pull [仓库名称] [目标分支] # 提交之前拉取一次最新版的远程仓库，防止提交冲突
	git push [仓库名称] [目标分支]
	cd <父仓库的根目录>
	git commit -am "更新子模块"
	git pull
	git push
	``` 
- 从远程更新到本地
	``` bash
	cd path
	git switch <分支名称>
	git pull [仓库名称] [目标分支]
	git submodule update --remote [--init] [--recursive] # 远程指令（必填）、初始化指令（选填）、递归指令（选填）
	```

## 命令 -- 列表版

**Git子模块指令查询手册**

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

### 最佳实践

- 克隆时用`git clone --recursive`。
	确保一次性克隆主仓库和所有子模块，避免后续手动初始化和更新。
- 设置子模块跟踪分支并定期更新。
	通过在`.gitmodules`中设置`branch`属性，并使用`git submodule update --remote`定期更新子模块。
- 启用`git config submodule.recurse true`简化操作。
	使Git命令默认递归到子模块，减少手动指定`--recurse-submodules`的需要。
- 子模块变更后，先提交子模块，再提交主仓库。
	确保子模块的变更先被提交和推送，然后在主仓库中记录新的提交哈希。
- 推送时用`git push --recurse-submodules=on-demand`。
	自动推送子模块的变更，确保主仓库和子模块的提交同步。




## 命令 -- 表格版

**Git子模块指令查询手册**

### 基本操作指令

| 指令                                         | 功能       | 说明                                | 示例                                                                    |
| ------------------------------------------ | -------- | --------------------------------- | --------------------------------------------------------------------- |
| `git submodule add <仓库地址> [路径]`            | 添加子模块    | 将外部仓库作为子模块添加到主项目，生成`.gitmodules`。 | `git submodule add https://github.com/user/repo.git submodule_folder` |
| `git clone --recursive <主仓库地址>`            | 克隆含子模块项目 | 克隆主仓库及其所有子模块。                     | `git clone --recursive https://github.com/user/main-repo.git`         |
| `git submodule init`                       | 初始化子模块   | 将`.gitmodules`配置写入`.git/config`。  | `git submodule init`                                                  |
| `git submodule update`                     | 更新子模块    | 拉取子模块代码并检出记录的提交。                  | `git submodule update`                                                |
| `git submodule sync`                       | 同步子模块URL | 更新子模块URL配置。                       | `git submodule sync`                                                  |
| `git rm <子模块路径>`  <br>`git deinit <子模块路径>` | 删除子模块    | 删除子模块记录并解除初始化。                    | `git rm submodule_folder`  <br>`git deinit submodule_folder`          |

### 状态查看指令

|指令|功能|说明|示例|
|---|---|---|---|
|`git submodule status`|查看子模块状态|显示子模块路径、提交哈希和分支信息。|`git submodule status`|
|`git submodule summary`|查看子模块更新摘要|显示子模块与主仓库记录的差异。|`git submodule summary`|
|`git status --submodules`|查看项目状态（含子模块）|显示主仓库和子模块状态。|`git status --submodules`|
|`git diff --submodule`|查看子模块差异|显示主仓库与子模块差异。|`git diff --submodule`|

### 更新与同步指令

|指令|功能|说明|示例|
|---|---|---|---|
|`git submodule update --remote`|更新子模块到远程最新|拉取子模块远程分支最新提交。|`git submodule update --remote`|
|`git submodule foreach git pull`|拉取所有子模块最新代码|遍历子模块执行`git pull`。|`git submodule foreach git pull`|
|`git pull --recurse-submodules`|拉取主仓库并更新子模块|同时更新主仓库和子模块。|`git pull --recurse-submodules`|
|`git submodule foreach git fetch`|获取子模块最新元数据|拉取子模块最新元数据。|`git submodule foreach git fetch`|

### 分支管理指令

|指令|功能|说明|示例|
|---|---|---|---|
|`git submodule add -b <分支名> <仓库地址> [路径]`|添加子模块并指定分支|添加子模块并设置跟踪分支。|`git submodule add -b main https://github.com/user/repo.git submodule_folder`|
|`git config -f .gitmodules submodule.<name>.branch <分支名>`|设置子模块跟踪分支|在`.gitmodules`中设置跟踪分支。|`git config -f .gitmodules submodule.submodule_folder.branch main`|
|`git submodule foreach git checkout <分支名>`|切换子模块分支|遍历子模块切换分支。|`git submodule foreach git checkout main`|
|`git checkout --recurse-submodules <分支名>`|切换主仓库分支并同步子模块|切换主仓库分支并更新子模块。|`git checkout --recurse-submodules feature-branch`|

### 高级操作指令

|指令|功能|说明|示例|
|---|---|---|---|
|`git submodule foreach --recursive <命令>`|递归遍历子模块|递归执行命令于所有子模块。|`git submodule foreach --recursive git status`|
|`git submodule foreach --init <命令>`|初始化并执行命令|初始化未初始化的子模块并执行命令。|`git submodule foreach --init git pull`|
|`git submodule absorbgitdirs`|合并子模块.git目录|将子模块`.git`目录合并到主仓库。|`git submodule absorbgitdirs`|
|`git submodule deabsorb`|恢复子模块独立.git目录|恢复子模块独立`.git`目录。|`git submodule deabsorb`|

### 协同操作指令

|指令|功能|说明|示例|
|---|---|---|---|
|`git add <子模块路径>`|记录子模块变更|记录子模块新提交哈希到主仓库。|`git add submodule_folder`|
|`git push --recurse-submodules=check`|推送前检查子模块|确保子模块变更已推送。|`git push --recurse-submodules=check`|
|`git push --recurse-submodules=on-demand`|推送主仓库并自动推送子模块|推送主仓库并自动推送子模块变更。|`git push --recurse-submodules=on-demand`|

### 配置设置指令

|指令|功能|说明|示例|
|---|---|---|---|
|`git config submodule.<name>.update <策略>`|设置子模块更新策略|设置更新策略（`none`、`checkout`、`rebase`、`merge`）。|`git config submodule.submodule_folder.update rebase`|
|`git config submodule.<name>.branch <分支名>`|设置子模块跟踪分支|设置跟踪分支。|`git config submodule.submodule_folder.branch main`|
|`git config submodule.recurse true`|全局启用子模块递归|启用子模块命令默认递归。|`git config submodule.recurse true`|

### 实用组合指令

| 指令                                                                                                                                                           | 功能           | 说明                       | 示例                                                                                                                                                                                             |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------ | ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `git submodule foreach --recursive 'git checkout main && git pull origin main'`                                                                              | 批量更新子模块到主分支  | 遍历子模块切换到`main`并拉取最新代码。   | `git submodule foreach --recursive 'git checkout main && git pull origin main'`                                                                                                                |
| `git submodule foreach --recursive 'git fetch origin main && [ "$(git rev-parse HEAD)" = "$(git rev-parse origin/main)" ] && echo "✅ $name 一致"               |              | echo "❌ $name 不一致"'`     | 检查子模块与远程主分支同步                                                                                                                                                                                  |
| `git clone --recursive <主仓库地址>`  <br>`git submodule update --init --recursive`                                                                               | 递归克隆并更新子模块   | 克隆主仓库并初始化、更新所有子模块。       | `git clone --recursive https://github.com/user/main-repo.git`  <br>`git submodule update --init --recursive`                                                                                   |
| `git config -f .gitmodules submodule.<name>.branch main`  <br>`git submodule update --remote`  <br>`git add <子模块路径>`  <br>`git commit -m "Update submodule"` | 设置跟踪分支并更新子模块 | 设置跟踪`main`分支，更新子模块并记录变更。 | `git config -f .gitmodules submodule.submodule_folder.branch main`  <br>`git submodule update --remote`  <br>`git add submodule_folder`  <br>`git commit -m "Update submodule to latest main"` |

---

此表格按功能分类，包含指令、功能、说明和示例，旨在帮助您快速掌握和使用Git子模块相关命令。
