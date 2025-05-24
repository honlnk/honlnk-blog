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
