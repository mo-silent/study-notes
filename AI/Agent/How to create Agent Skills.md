---
title: 'Agent Skills: 如何创建自定义 Agent Skills'
slug: how-to-create-custom-skills
categories:
  - Agent
tags:
  - AI
halo:
  site: https://blog.silentmo.cn
  name: e0b3407a-a1fa-4ae8-8244-0545b5697753
  publish: true
---
# 如何创建自定义 Agent Skills

## 简单说一下

这里不讲 Agent Skills 的定义和概念，可以看上一篇博客：[Agent Skills: 如何为大语言模型构建可复用技能](https://blog.silentmo.cn/archives/agent-skills-details)

这里使用 Gemini CLI 演示，因为博主的 Claude 过期了，没续期，Gemini 是白嫖的一年学生免费使用。

不过文档来看，还是 Anthropic 的更全，步骤更详细。大家想看官方文档的，直接看 Anthropic 的就好了。

自定义 SKills 详细说明：[Claude - How to create custom skills](https://support.claude.com/en/articles/12512198-how-to-create-custom-skills)

## Skills 存放的位置

> 目前就看到了 Claude Agent SDK 有说明 skills 的使用（[Agent skill in the sdk](https://platform.claude.com/docs/en/agent-sdk/skills)），其他官方框架没找到，注意是官方框架哦。
>
> 如果有其他大佬知道 [OpenAI Agents sdk](https://platform.openai.com/docs/guides/agents-sdk) 和 [Google Agent Development Kit](https://docs.cloud.google.com/agent-builder/agent-development-kit/overview) 对 skill 的使用说明，麻烦贴一下连接在评论区，将不胜感激。

|                  | 个人 Skills                                            | 项目 Skills                   |
| ---------------- | ------------------------------------------------------ | ----------------------------- |
| Claude Agent SDK | 默认：`~/.claude/skills/`，或任意目录，通过 `cwd` 指定 | 项目目录下  `.claude/skills/` |
| Claude Code      | `~/.claude/skills/`                                    | 项目目录下  `.claude/skills/` |
| Codex(Openai)    | `~/.codex/skills/`                                     | 项目目录下`.codex/skills/`    |
| Gemini CLI       | `~/.gemini/skills/`                                    | 项目目录下`.gemini/skills/`   |

## 创建第一个 Skill

> 示例是用的 Gemini 官方的：[Gemini CLI - Create a skill](https://geminicli.com/docs/cli/skills/#creating-a-skill)
>
> Claude Code: [Create your first skill](https://code.claude.com/docs/en/skills#create-your-first-skill)
>
> Openai(gpt): [codex-create custom skill](https://developers.openai.com/codex/skills/create-skill)

1. 创建 Skill 目录

   > 每一个 Skill 就是一个单独的目录

   ```shell
   mkdir -p ~/.gemini/skills/code-reviewer-example
   ```

2. 在 `~/.gemini/skills/code-reviewer-example` 目录下创建一个 `SKILL.md` 文件，内容示例：
   ```markdown
   ---
   name: code-reviewer
   description:
     Expertise in reviewing code for style, security, and performance. Use when the
     user asks for "feedback," a "review," or to "check" their changes.
   ---
   
   # Code Reviewer
   
   You are an expert code reviewer. When reviewing code, follow this workflow:
   
   1.  **Analyze**: Review the staged changes or specific files provided. Ensure
       that the changes are scoped properly and represent minimal changes required
       to address the issue.
   2.  **Style**: Ensure code follows the project's conventions and idiomatic
       patterns as described in the `GEMINI.md` file.
   3.  **Security**: Flag any potential security vulnerabilities.
   4.  **Tests**: Verify that new logic has corresponding test coverage and that
       the test coverage adequately validates the changes.
   
   Provide your feedback as a concise bulleted list of "Strengths" and
   "Opportunities."
   ```

3.  Gemini CLI 首次使用 Skill 需要先启用 (Claude code 不需要)

   > 也可以直接改 `~/.gemini/settings.json` 文件，添加：
   > ```
   > {
   >   "experimental": {
   >     "skills": true
   >   }
   > }
   > ```

   - 在终端运行 `gemini`，进入交互界面，在交互界面输入 `/settings`

   - 在 `settings` 中输入 `skills`，查看  `Enable Agent Skills`。

     - 如果是 false，则回车一次会变成 true, 按 `esc`
     - 如果是 true，就直接按 `esc`

     ![gemini enable skills](https://gallery-lsky.silentmo.cn/i_blog/2026/01/Agent-skills-custom-gemini-1.png)

4. 在上一步如果是交互设置，设置完成后，输入`/quit` 退出，再进入 `gemini` 交互终端。这时候就会有显示加载的 skill 和 存在 `/skills` 命令了，执行 `/skills list` 查看存在的 skills
   ![gemini skills list](https://gallery-lsky.silentmo.cn/i_blog/2026/01/Agent-skills-custom-gemini-2.png)

5. 输入关键词触发skill，观察skill的过程，如：`please check the code changes`。

   > 可以看出 Gemini 对 skill 的使用过程跟 Claude 是一样的，细节可以看上一篇：[Agent Skills: 如何为大语言模型构建可复用技能](https://blog.silentmo.cn/archives/agent-skills-details)

   ![gemini skills discovery](https://gallery-lsky.silentmo.cn/i_blog/2026/01/Agent-skills-custom-gemini-3.png)

   当允许后，它就会使用 skill 执行相应的任务
   ![gemini skills discovery](https://gallery-lsky.silentmo.cn/i_blog/2026/01/Agent-skills-custom-gemini-4.png)

上面步骤是最简单的 Skill 示例，但是复杂的 Skill 不只是 `SKILL.md` 就能完成的。还有下面的资源定义：

- **`scripts/`**：代理可以运行的可执行脚本（bash、Python、Node）。
- **`references/`**：供代理查阅的静态文档、模式（schemas）或示例数据。
- **`assets/`**：代码模板、样板代码或二进制资源。

完整的 Skill 目录可能如下所示（Claude 官方 pdf 示例）：
```
pdf/
├── SKILL.md              # 主要说明（触发时加载）
├── FORMS.md              # 表单填充指南（根据需要加载）
├── reference.md          # API 参考（根据需要加载）
├── examples.md           # 使用示例（根据需要加载）
└── scripts/
    ├── analyze_form.py   # 实用脚本（执行，不加载）
    ├── fill_form.py      # 表单填充脚本
    └── validate.py       # 验证脚本
```

### SKILL.md 元数据字段

在 YAML 前置部分中使用以下字段：

| 字段             | 必需 | 描述                                                         |
| :--------------- | :--- | :----------------------------------------------------------- |
| `name`           | 是   | Skill 名称。必须仅使用小写字母、数字和连字符（最多 64 个字符）。应与目录名称匹配。 |
| `description`    | 是   | Skill 的功能和何时使用它（最多 1024 个字符）。Claude 使用这个来决定何时应用 Skill。 |
| `allowed-tools`  | 否   | 当此 Skill 活跃时，Claude 可以使用而无需请求权限的工具。支持逗号分隔的值或 YAML 风格的列表。参见[限制工具访问](https://code.claude.com/docs/zh-CN/skills#restrict-tool-access-with-allowed-tools)。 |
| `model`          | 否   | 当此 Skill 活跃时使用的[模型](https://docs.claude.com/zh-CN/docs/about-claude/models/overview)（例如，`claude-sonnet-4-20250514`）。默认为对话的模型。 |
| `context`        | 否   | 设置为 `fork` 以在具有自己对话历史的分叉子代理上下文中运行 Skill。 |
| `agent`          | 否   | 指定当设置 `context: fork` 时使用哪个[代理类型](https://code.claude.com/docs/zh-CN/sub-agents#built-in-subagents)（例如，`Explore`、`Plan`、`general-purpose` 或来自 `.claude/agents/` 的自定义代理名称）。如果未指定，默认为 `general-purpose`。仅在与 `context: fork` 结合时适用。 |
| `hooks`          | 否   | 定义限定于此 Skill 生命周期的 hooks。支持 `PreToolUse`、`PostToolUse` 和 `Stop` 事件。 |
| `user-invocable` | 否   | 控制 Skill 是否出现在斜杠命令菜单中。不影响[`Skill` 工具](https://code.claude.com/docs/zh-CN/slash-commands#skill-tool)或自动发现。默认为 `true`。参见[控制 Skill 可见性](https://code.claude.com/docs/zh-CN/skills#control-skill-visibility)。 |

有关完整的编写指导，包括验证规则，请参阅[最佳实践指南](https://docs.claude.com/zh-CN/docs/agents-and-tools/agent-skills/best-practices)。

## 最佳实践

> 这里不单独写出来是因为篇幅很长，而且 Anthropics 的文档都支持切换语言，就没必要了。

文档说明：[Claude Best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)

Github Skills 参考：[Anthropics skills](https://github.com/anthropics/skills/tree/main)
