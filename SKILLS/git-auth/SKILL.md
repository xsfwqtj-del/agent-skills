---
name: git-auth
description: GitHub 认证信息。当需要 git push/pull/remote 操作或 GitHub API 调用时自动触发。
trigger: auto
---

# Git 认证配置

> 记录本机的 Git 认证方式，确保每次操作知道用什么权限。

## 认证方式

**gh CLI + Classic Token**

```bash
# 安装
winget install GitHub.cli

# 登录方式
echo "ghp_<经典token>" | gh auth login --with-token
```

## Token 权限

Classic token (`ghp_...`)，在 github.com/settings/tokens 生成，勾选了：
- `repo` — 完整仓库读写
- `delete_repo` — 删除仓库
- `admin` — 管理组织/仓库设置
- `user` — 用户级别操作

**权限全开。** 能建仓库、删仓库、改可见性、强推。

## 使用规则

1. **每次 git push 前确认** — 推送的是哪个仓库，包含什么内容
2. **强推前必须确认** — `git push --force` 需要爸爸明确同意
3. **删仓库前必须确认** — 不可逆操作
4. **用完随时可 revoke** — github.com/settings/tokens 里撤销

## 仓库清单

| 仓库 | 可见性 | 本地路径 | 用途 |
|------|--------|---------|------|
| mioplugin | PRIVATE | d:/mio/ | Mio 记忆管线插件 + 老 Agent 归档 |
| jingxiju | PRIVATE | d:/烬汐居/ | 角色卡生产工作区 |
| agent-skills | PUBLIC | d:/skills/ | 自定义 Agent Skills |
| mio | PRIVATE | d:/godot-ai-tavern/ | LLM 酒馆（Godot 客户端） |

## 安全检查

推送前自动检查：
- `.gitignore` 是否排除了密钥/令牌/会话记录
- 是否包含 NSFW 内容
- 是否包含个人隐私数据（会话日志、配置文件等）
