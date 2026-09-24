# Skills

个人编写的 Agent Skill 集合，每个 skill 以独立文件夹形式存放


## 使用方式

### 1. 克隆仓库

```bash
git clone https://github.com/<your-username>/skills.git
```

### 2. 安装到对应平台

根据你使用的 agent 平台，选择对应的安装方式：

#### Qoder

将需要的 skill 文件夹复制到 Qoder 的 skill 目录，或通过 `create-plugin` 命令将整个仓库打包为 Qoder 原生插件：

```
/create-plugin
```

#### Claude Code

将 skill 文件夹链接或复制到 `~/.claude/skills/` 目录：

```bash
# 链接单个 skill
ln -s /path/to/skills/grill-me ~/.claude/skills/grill-me

# 或链接整个仓库（所有 skill 一次性生效）
ln -s /path/to/skills ~/.claude/skills/my-skills
```

#### Codex CLI / Codex App

将 skill 文件夹链接或复制到 `~/.agents/skills/` 目录：

```bash
ln -s /path/to/skills/grill-me ~/.agents/skills/grill-me
```

#### Gemini CLI

通过 extensions 安装本地路径：

```bash
gemini extensions install /path/to/skills
```

#### Cursor

在 Cursor Agent 聊天中使用：

```text
/add-plugin /path/to/skills
```

### 3. 验证安装

安装完成后，在 agent 中输入 `/` 查看 skill 列表，确认目标 skill 已出现即可。

### 更新

由于是本地链接（symlink），直接在本仓库 `git pull` 即可同步更新，无需重新安装。

如果是复制方式，需要重新复制最新版本到对应平台目录。
