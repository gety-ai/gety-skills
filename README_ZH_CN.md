# Gety Skills

[Gety](https://gety.ai) 的 Agent Skills — 让 AI 代理连接你的本地文档。

<p align="center">
  <img src="https://img.shields.io/badge/Skills-2-blue" alt="2 Skills" />
</p>

## 可用 Skills

| Skill                                 | 描述                    | 安装                                                     |
| ------------------------------------- | --------------------- | ------------------------------------------------------ |
| [create-custom-connector](skills/create-custom-connector/SKILL.md) | 构建自定义 connector，把任意 API、SaaS、数据库或本地数据源索引进 Gety | `npx skills add gety-ai/gety-skills --skills create-custom-connector` |
| [gety-cli](skills/gety-cli/README.md) | 通过 Gety CLI 搜索和检索本地文档 | `npx skills add gety-ai/gety-skills --skills gety-cli` |

## 快速开始

```bash
npx skills add gety-ai/gety-skills
```

## 前置要求

- 安装并运行 [Gety](https://gety.ai) 桌面应用
- 在 [Gety](https://gety.ai) 桌面应用设置中启用 AI 集成
