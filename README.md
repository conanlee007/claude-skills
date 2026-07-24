# Claude Skills — 分发仓库

可直接安装到 Claude(Claude Code / Claude Desktop / claude.ai)的技能包。

## 技能列表

| Skill | 用途 |
|---|---|
| [sourcing-candidates-with-zoominfo](sourcing-candidates-with-zoominfo/) | 用 ZoomInfo 按 JD 三段式搜索招聘候选人:免费搜名单 → LinkedIn 人工核实 → 确认后 enrich 拿联系方式(最省 credit 的打法) |

## 安装方法

### Claude Code(CLI / 桌面版)

把技能文件夹拷贝到用户级 skills 目录:

```bash
git clone https://github.com/conanlee007/claude-skills.git
mkdir -p ~/.claude/skills
cp -r claude-skills/sourcing-candidates-with-zoominfo ~/.claude/skills/
```

重启会话后,直接说"帮我根据 JD 找候选人"即可触发。

### claude.ai 网页版 / Claude Desktop

Settings → Capabilities(功能)→ Skills → Upload skill,上传对应技能文件夹的 zip 包。

## 前置条件(sourcing-candidates-with-zoominfo)

1. 公司需有 **ZoomInfo 订阅**,且管理员已开通 MCP / AI Copilot 访问权限
2. 安装 **ZoomInfo MCP 连接器**:
   - claude.ai / Claude Desktop:Settings → Connectors → 搜索 "ZoomInfo" → Connect → OAuth 授权
   - Claude Code:交互式会话运行 `/mcp` 添加 ZoomInfo connector
3. 技能会在运行时自动检测 MCP 是否就绪,未安装时会引导完成安装

## 使用要点

- 输入是**明确的 JD**:技能会强制逐项确认 8 项搜索条件,信息不全会反复追问
- **搜索阶段免费**,不消耗 ZoomInfo credit
- **enrich(获取联系方式)扣 credit**:技能会先展示候选人明细表,经你确认后才执行,绝不擅自扣费
- 同一联系人首次 enrich 后一年内重复获取免费

## License

MIT
