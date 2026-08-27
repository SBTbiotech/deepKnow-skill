# DeepKnow Skill

DeepKnow 知识库助手 Skill，支持知识库浏览、文件或 URL 导入、Markdown 写入、知识检索和表格原始数据查询。

## 安装

需要本机已安装 Node.js 和 `npx`。使用 Skills CLI 从 GitHub 全局安装到 Codex：

```bash
npx --yes skills@latest add SBTbiotech/deepknow-skill --skill deepknow -a codex -g -y
```

配置 DeepKnow API Key：

```bash
export DEEPKNOW_API_KEY="sk-your-api-key"
```

API 基础地址已经固定在 Skill 中，不需要配置 `DEEPKNOW_BASE_URL`。

## 更新

手动更新已安装的 Skill：

```bash
npx --yes skills@latest update deepknow -g -y
```

如果旧版本不是通过 Skills CLI 安装、因而没有安装记录，请重新执行上面的安装命令。后端不托管或分发 Skill ZIP，GitHub 仓库是安装来源。

## 自动升级协议

Skill 调用 API 时会发送：

```text
X-DeepKnow-Channel: skill
X-DeepKnow-Skill-Version: <SKILL.md frontmatter 中的 version>
```

当该版本低于后端允许的最低兼容版本时，后端会在执行业务逻辑前返回 HTTP `426 Upgrade Required`：

```json
{
  "success": false,
  "error": {
    "code": "SKILL_UPGRADE_REQUIRED",
    "message": "DeepKnow skill 需要升级"
  },
  "upgrade_info": {
    "minimum_version": "<minimum-compatible-version>",
    "request_executed": false
  }
}
```

Skill 随后通过固定的 `npx skills update` 命令升级到 GitHub 最新版本，并将原请求完整重试一次。`request_executed: false` 保证写请求不会因为自动重放产生重复副作用。

后端配置 `DEEPKNOW_SKILL_MIN_VERSION` 表示最低兼容版本，不是当前发布版本。只有后端 API 与旧 Skill 不再兼容时才提高它，因此不需要在两个仓库同步维护“最新版本”。

## 目录结构

```text
skills/
├── SKILL.md             # Skill 入口、公共调用规范和唯一版本号
├── import.md            # 导入与内容管理 API
├── knowledge-bases.md   # 知识库与内容浏览 API
├── search.md            # 检索与引用规范
└── tabular.md           # 表格查询 API
```

## 开发与发布

- 发布版本只修改 `skills/SKILL.md` frontmatter 的 `version`，不要在其他文件复制当前版本号。
- API 参数或工作流按能力拆分到对应 Markdown 文档，并保持现有文档格式。
- 本地安装当前工作区进行验证：

  ```bash
  npx --yes skills@latest add . --skill deepknow -a codex -g -y
  ```
