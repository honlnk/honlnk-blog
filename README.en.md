# honlnk-blog

## Project Introduction

honlnk-blog is a personal blog project based on Obsidian, used to record and share technical notes and practical summaries in software development, AI testing, Git submodule management, front-end automation testing, and related areas.

## Directory Structure

- `algorithm/`: Implementation and explanation of string edit distance algorithms  
- `git/`: Git submodule management, common commands, and configuration guide  
- `obsidian/`: Obsidian plugin installation, translation, and usage tips  
- `remoteServer/`: SSH connection to servers and VSCode remote development configuration  
- `research/`: Research on open-source online note-taking software  
- `vue-component/`: Vue 3 component implementation and usage examples  
- `web-testing/`: Discussion on web automation testing technologies  
- `.obsidian/`: Obsidian project configuration directory, including plugins and style files  
- `.smtcmp_json_db/`: Storage directory for Smart Composer plugin-related data  
- `.smtcmp_vector_db.tar.gz`: Compressed vector database file for the Smart Composer plugin  
- `LICENSE`: Open-source project license file  
- `README.md`: This file  

## Installation and Deployment

This project is primarily used for internal note management and blog writing within Obsidian. If deployment is required, please follow the steps below:

1. Install [Obsidian](https://obsidian.md/)  
2. Clone this project into Obsidian's `vaults` directory  
3. Launch Obsidian and open this project  
4. Install and enable the `.obsidian/plugins/obsidian-git` plugin for Git version synchronization  
5. To use AI-assisted writing, enable the `.obsidian/plugins/smart-composer` plugin  

## Usage Instructions

- **Git Operations**: Please refer to the documentation under `git/common-commands`, which includes commonly used commands for submodules, branches, commits, etc.  
- **Obsidian Plugins**: This project uses multiple Obsidian plugins. For detailed configuration, please refer to `obsidian/obsidian-plugins.md`  
- **AI-Assisted Writing**: After enabling the `smart-composer` plugin, you can use AI to generate Markdown content, code snippets, file structures, etc., within the note editor  
- **Front-End Components**: `vue-component/marquee-scroll.vue.md` provides an implementation and usage example of a Vue 3 horizontal scrolling component  

## Contribution Guidelines

- Please follow the code submission standards outlined in `dev-standards/code-commit-standards.md`  
- New content is recommended to be in Markdown format, maintaining a clear directory structure  
- For AI-related content, it is recommended to specify the model version and generation logic  
- Before submitting a Pull Request, ensure that the configurations for `.obsidian/plugins/smart-composer` and `.obsidian/plugins/obsidian-git` plugins are correct  

## Open Source License

This project follows an open-source license. Please check the `LICENSE` file in the root directory.

## References

- [Obsidian Official Documentation](https://help.obsidian.md/)  
- [Git Submodules Documentation](https://git-scm.com/book/zh/v2/Git-%E5%B7%A5%E5%BA%95%E7%94%A8%E9%80%89%E9%A1%B9%E7%9B%AE)  
- [Vue 3 Component Development Documentation](https://vuejs.org/guide/components/registration.html)  

## Contact Information

For communication or feedback, please contact the project maintainer [hong-ying-19](https://gitee.com/hong-ying-19). We welcome you to submit Issues or Pull Requests!