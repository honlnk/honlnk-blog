## 隐藏更改
### 基本命令
- **`git stash`**  
  临时存储未提交的更改
  ```bash
  git stash
  ```

- **`git stash pop`**  
  恢复最近一次隐藏的更改并删除存储记录
  ```bash
  git stash pop
  ```

## 查看当前状态
### 命令参数
- **`git status`**  
  显示工作目录和暂存区状态
  ```bash
  git status
  ```

### 输出说明
``` bash
git status -> 查看状态

----- 输出结果 -----
--- 1. 当前分支信息（On barnch） ---
On barnch <你的当前分支>
Your bransh is up to date with '<当前分支以跟踪的远程分支>'
--- 2. 未‘跟踪’的文件（Untracked files） --- 
Untracked files:
  (use "git add <file>..." to include in what will be committed)
	<未跟踪的文件全名（地址加文件名）>
nothing added to commit but untracked files present (use "git add" to track) -> 提示可以使用 'git add' 命令暂存更改
--- 3. 已‘修改’但‘未暂’存的文件（Changes not staged for commit:） ---
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed) -> 提示如果还有未提交的更改会提示 使用 'git add <file>' 将未提交的文件提交
  (use "git restore <file>..." to discard changes in working directory) -> 提示可以使用 'git restore <file>' 命令放弃跟踪（取消暂存）
    modified: <修改的文件>
    new file: <新增的文件>
    deleted: <删除的文件>
--- 4. 已暂存的文件（Changes to be committed:） ---
Changes to be committed:
  (use "git restore --staged <file>..." to unstage) -> 提示 使用'git restore --staged <file>'命令可以取消暂存
    new file: <新增的文件>
    modified: <修改的文件>
    deleted: <删除的文件>
```

## 提交
### 1. 基本参数
- **`-m <message>`**
  - 直接在命令行中提供提交信息。
  - 示例：
    ```bash
    git commit -m "Add new feature"
    ```

- **`-a` 或 `--all`**
  - 自动暂存所有已跟踪文件的更改（包括修改和删除），但不会包含未跟踪的新文件。
  - 示例：
    ```bash
    git commit -a -m "Update existing files"
    ```

- **`--amend`**
  - 修改最近一次提交的内容或提交信息。
  - 如果暂存区有更改，这些更改会被合并到最近一次提交中。
  - 示例：
    ```bash
    git commit --amend -m "Updated commit message"
    ```

---

### 2. 跳过钩子
- **`--no-verify`**
  - 跳过提交前的钩子脚本（如 `pre-commit` 和 `commit-msg` 钩子）。
  - 示例：
    ```bash
    git commit --no-verify -m "Bypass hooks"
    ```

---

### 3. 指定作者信息
- **`--author=<author>`**
  - 指定提交的作者信息（覆盖默认的配置信息）。
  - 格式：`Name <email>`
  - 示例：
    ```bash
    git commit -m "Fix bug" --author="John Doe <john.doe@example.com>"
    ```

---

### 4. 干运行 (Dry Run)
- **`--dry-run`**
  - 显示将要提交的内容，但不实际执行提交操作。
  - 示例：
    ```bash
    git commit --dry-run
    ```

---

### 5. 控制输出
- **`--quiet`**
  - 减少输出信息，仅显示错误消息。
  - 示例：
    ```bash
    git commit --quiet -m "Silent commit"
    ```

- **`--verbose` 或 `-v`**
  - 在提交时显示详细的差异信息。
  - 示例：
    ```bash
    git commit -v -m "Verbose commit"
    ```

---

### 6. 提交模板
- **`-t <file>` 或 `--template=<file>`**
  - 使用指定文件作为提交信息模板。
  - 示例：
    ```bash
    git commit -t commit_template.txt
    ```

- **`-F <file>` 或 `--file=<file>`**
  - 从文件中读取提交信息。
  - 示例：
    ``` bash
    git commit -F commit_message.txt
    ```

---

## 追踪
### 停止追踪（忽略更改）
``` bash
git update-index --assume-unchanged <文件路径>
```
### 恢复追踪（监听更改）
``` bash
git update-index --no-assume-unchanged <文件路径>
```


****