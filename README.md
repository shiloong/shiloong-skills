# shiloong-skills

shile.zhang 的项目规范、开发习惯配置及 Agent Skills 集合。

## 目录结构

```
shiloong-skills/
├── CLAUDE.md                # 本仓库自身指令
├── README.md                # 项目说明
├── configs/                 # 各项目的 CLAUDE.md 规范（部署到对应项目目录）
│   └── tokenless/           # tokenless 项目
│       └── CLAUDE.md
│   └── sec-core/            # agent-sec-core 项目
│       └── CLAUDE.md
│   └── ...                  # 其他项目
├── skills/                  # 可复用 Agent Skill（SKILL.md 格式）
│   └── <skill-name>/
│       └── SKILL.md
├── .claude-plugin/          # 插件发布配置（可选）
│   ├── marketplace.json
│   └── plugin.json
└── notes/                   # 开发经验总结
```

### configs/

存放各子项目的 CLAUDE.md，包含项目级编码规范、构建/测试命令、架构说明等。工作时将对应文件部署到项目目录（如 `configs/tokenless/CLAUDE.md` → `anolisa/src/tokenless/CLAUDE.md`）。

当前已收录：

| 项目 | 路径 |
|------|------|
| tokenless | `configs/tokenless/` |

### skills/

可复用的 Agent Skill 定义，按 SKILL.md 格式组织。

### .claude-plugin/

插件 marketplace 配置，用于将 skill 发布为 Claude Code 插件。

### notes/

按主题组织的开发经验总结。

---

*Maintained by shile.zhang*