# 子模块相关操作

## 命令

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
