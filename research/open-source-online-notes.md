# 在线笔记开源仓库调研

## 高完成度

### NoteGen
- 地址：[https://github.com/codexu/note-gen](https://github.com/codexu/note-gen)
- 使用方式：跨平台的桌面APP，移动端适配度低
- **技术栈**:
	- **后端打包**: Tauri2
	- **前端**: Next.js（React技术栈、实现用户界面和应用逻辑）
	- **其他**: TypeScript、Vue3（部分前端脚手架）、多种 AI 模型集成。
- **文件存储**：Markdown文件
- **AI**：集成多种AI模型，支持问答、文本续写、优化、简化、翻译等功能。
- **图片管理**：贴图到Markdown，自动上传图床，存储图床链接。
- 斜杠指令：不支持斜杠指令操作，仅支持手写Markdown语法以及富文本编辑选项

### Zettlr
- **仓库地址**: [https://github.com/Zettlr/Zettlr](https://github.com/Zettlr/Zettlr)
- **描述**: 开源 Markdown 编辑器，适合研究人员、记者和作家，支持 Zettelkasten 知识管理和多格式导出。
- **使用方式**: 跨平台桌面应用（macOS、Windows、Linux），未优化移动端。
- **技术栈**:
	- **后端打包**: Electron
    - **前端**: Vue.js
    - **其他**: Node.js、Pandoc、CodeMirror 6
- **文件存储**: Markdown 文件，支持 Zettelkasten 笔记 ID 和内部链接
- AI：未集成任何AI功能
- 斜杠指令：不支持斜杠指令操作，仅支持手写Markdown语法以及富文本编辑选项




## 低完成度

### Lotion
- **地址**：[https://github.com/Dashibase/lotion](https://github.com/Dashibase/lotion)
- **使用方式**：Web应用，支持浏览器访问，适合构建类Notion编辑器，一个半成品。
- **技术栈**：
  - **前端**：Vue 3（实现用户界面和交互逻辑）
  - **其他**：TypeScript、Tiptap（编辑器核心）、Tailwind CSS（样式）
- **文件存储**：基于块（Block）的结构化数据，类似Notion
- **功能**：提供Notion风格的块编辑器，支持文本、标题等块类型，可拖拽重排序，扩展性强
- **斜杠指令**：支持斜杠（/）快捷指令切换块类型，如/heading
- **社区与扩展**：可通过npm安装（@dashibase/lotion），支持自定义块开发，已停止维护。

### Dashibase
- 地址：[https://github.com/Dashibase/dashibase](https://github.com/Dashibase/dashibase)
- 使用方式：开源的Web应用，适合快速构建Supabase用户的管理面板
- **技术栈**：
    - **前端**：Vue.js（用于交互式用户界面）、Tailwind CSS（样式设计）
    - **后端**：集成Supabase（提供数据库和认证支持）
    - **其他**：TypeScript、Next.js（部分插件支持）
- **文件存储**：基于Supabase数据库，配置通过JSON或TypeScript文件
- **功能**：支持快速构建多页面CRUD仪表板，无需编写SQL或JavaScript，支持表格过滤、排序、隐藏列等功能，集成Notion风格UI
- **插件系统**：支持自定义插件（如Stripe、Zendesk），通过斜杠指令添加
- **授权**：GNU General Public License v3.0
- 社区：项目已停止维护。


## 无头编辑器

### Novel
- 地址：[https://github.com/steven-tey/novel](https://github.com/steven-tey/novel)
- 使用方式：Web 应用，适配桌面和移动端
- **技术栈**:
  - **前端**: Next.js（React 技术栈，负责 UI 和交互逻辑）
  - **后端**: Node.js（支持服务器端功能）
  - **其他**: TypeScript、Tailwind CSS（样式）、Vercel（部署平台）
- **文件存储**：Markdown 文件
- **AI 功能**：集成 AI 模型，支持文本生成、编辑和优化
- **图片管理**：支持图片插入 Markdown，链接存储
- **编辑方式**：支持富文本编辑和 Markdown 语法，无斜杠指令


### Tiptap
- 地址：[https://github.com/ueberdosis/tiptap](https://github.com/ueberdosis/tiptap)
- 使用方式：JavaScript 库，适用于 Web 应用的富文本编辑器，易于集成到前端项目
- **技术栈**:
  - **核心**: JavaScript、ProseMirror（提供强大的文本编辑基础）
  - **前端**: 支持与 React、Vue 等框架无缝集成
  - **其他**: TypeScript（类型安全）、模块化设计
- **功能**：提供高度可定制的富文本编辑体验，支持 Markdown、实时协作、扩展插件（如表格、图片、代码块等）
- **扩展性**：丰富的插件生态，可自定义编辑器功能和样式
- **存储**：支持 JSON 或 Markdown 格式输出，便于数据处理和存储