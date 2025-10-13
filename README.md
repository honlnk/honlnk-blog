

# honlnk-blog

## 项目简介

honlnk-blog 是一个基于 Obsidian 的个人博客项目，用于记录与分享在软件开发、AI测试、Git子模块管理、前端自动化测试等方面的技术笔记和实践总结。

## 目录结构说明

- `algorithm/`: 字符串编辑距离算法相关的实现与说明
- `git/`: Git子模块管理、常见命令与配置指南
- `obsidian/`: Obsidian插件安装、翻译和使用技巧
- `remoteServer/`: SSH连接服务器及 VSCode 远程开发配置
- `research/`: 开源在线笔记软件调研
- `vue-component/`: Vue 3组件实现与使用示例
- `web-testing/`: Web自动化测试相关技术讨论
- `.obsidian/`: Obsidian项目配置目录，包含插件与样式文件
- `.smtcmp_json_db/`: 存储 Smart Composer 插件相关数据
- `.smtcmp_vector_db.tar.gz`: Smart Composer 插件的向量数据库压缩包
- `LICENSE`: 开源项目授权协议
- `README.md`: 本文件

## 安装与部署

本项目主要用于 Obsidian 内部笔记管理与博客写作，如需部署，请参考以下步骤：

1. 安装 [Obsidian](https://obsidian.md/)
2. 将本项目克隆到 Obsidian 的 `vaults` 目录下
3. 启动 Obsidian，打开本项目
4. 安装并启用 `.obsidian/plugins/obsidian-git` 插件，用于 Git 版本同步
5. 如需 AI 辅助写作，请启用 `.obsidian/plugins/smart-composer` 插件

## 使用说明

- **Git操作**: 请参考 `git/common-commands` 目录下的文档，包含子模块、分支、提交等常用命令
- **Obsidian插件**: 项目中使用了多个 Obsidian 插件，具体配置请参考 `obsidian/obsidian-plugins.md`
- **AI辅助写作**: 启用 `smart-composer` 插件后，可在笔记编辑器中使用 AI 生成 Markdown 内容、代码片段、文件结构等
- **前端组件**: `vue-component/marquee-scroll.vue.md` 提供了一个 Vue 3 的横向滚动组件实现与使用示例

## 贡献指南

- 请遵循 `dev-standards/code-commit-standards.md` 中的代码提交规范
- 新增内容建议使用 Markdown 格式，并保持清晰的目录结构
- 对于 AI 相关内容，建议注明模型版本与生成逻辑
- 提交 Pull Request 前，请确保 `.obsidian/plugins/smart-composer` 与 `.obsidian/plugins/obsidian-git` 插件配置无误

## 开源协议

本项目遵循开源协议，请查看根目录下的 `LICENSE` 文件。

## 参考资料

- [Obsidian 官方文档](https://help.obsidian.md/)
- [Git Submodules 文档](https://git-scm.com/book/zh/v2/Git-%E5%B7%A5%E5%BA%95%E7%94%A8%E9%80%89%E9%A1%B9%E7%9B%AE)
- [Vue 3 组件开发文档](https://vuejs.org/guide/components/registration.html)

## 联系方式

如需交流或反馈，请联系项目维护者 [hong-ying-19](https://gitee.com/hong-ying-19)。欢迎提交 Issue 或 Pull Request！

## 更新日志

- 2025-10-13: 优化了仓库的动态路径管理，支持在 GitHub 和 Gitee 之间轻松切换