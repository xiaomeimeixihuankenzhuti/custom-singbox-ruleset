# Custom Sing-box Ruleset

Sing-box 自定义规则集，用于管理网络流量的路由规则。

## 项目结构

| 文件 | 说明 |
|------|------|
| `direct.json` | 直连规则 - 存放 `ChinaMaxNoIP` 未覆盖、需要强制直连的长尾域名 |
| `proxy.json` | 代理规则 - 定义需要走代理的域名 |
| `iflytek.json` | 科大讯飞内部规则 - 包含 IP 段和域名规则 |
| `iflytek-domain.json` | 科大讯飞域名规则（与 iflytek.json 部分重复，建议合并） |

## 规则格式

每个文件遵循 sing-box ruleset v3 格式：

```json
{
  "version": 3,
  "rules": [
    {
      "domain_suffix": ["example.com", "..."]
    },
    {
      "ip_cidr": ["192.168.0.0/16", "..."]
    }
  ]
}
```

### 支持的规则类型

- `domain_suffix` - 域名后缀匹配
- `ip_cidr` - IP 段匹配

## 使用方法

1. 将规则文件部署到 sing-box 配置目录
2. 在 sing-box 配置中引用对应的 ruleset
3. 重启 sing-box 服务使配置生效

## 注意事项

- 域名后缀不要包含前导点号（如 `.cn` 应写为 `cn`）
- 域名不要包含尾部空格
- 避免在不同规则文件中重复定义相同域名

## 与 DNS 广告拦截的交互

`custom-direct` 在 `dns.rules` 中排在广告拦截规则**之后**（见
`/etc/sing-box/config.json`，reject 规则在前，`custom-direct` 在第 74 行）：

1. `custom-block`、`hagezi-pro` / `anti-ad-srs` 先执行 `reject`；
2. 未被拦截的域名才轮到 `custom-direct`，命中后选择 `local` DNS 服务器。

因此 **`direct.json` 不产生任何拦截豁免**：把域名写进来只能决定"用哪个 DNS
解析器"，不能决定"要不要拦截"。

需要豁免拦截时，必须新增一条**排在 reject 之前**的规则。现有先例是微信
PCDN 白名单（config.json 中内联的 `domain_suffix` + `server: local`）；
条目较多时可建独立 `allowlist.json` 规则集并前置引用。

### domain_suffix 的整树匹配

任何**前置白名单**（内联或独立规则集）的 `domain_suffix` 都匹配整棵子域树，
不只是基域本身：

| 白名单条目 | 同时被豁免拦截的子域（实测） |
|---|---|
| `xiaomi.com` | `tracker.ai.xiaomi.com` |
| `mi.com` | `ot.io.mi.com` 等 |

所以写 `xiaomi.com` 的意图是放行小米主站，实际效果却是
**所有 `*.xiaomi.com` 都不再被广告拦截**。前置白名单应尽量写最窄的主机名；
如需恢复拦截，从前置白名单中移除对应条目，而不是改 `direct.json`。

## 冗余清理记录

- **2026-11-08**：以 `ChinaMaxNoIP` 为基准逐条执行 `sing-box rule-set match`，
  `direct.json` 从 269 条精简到 52 条。被移除的 217 条均已被 `ChinaMaxNoIP`
  覆盖（含裸 `cn`、`xiaomi.com`、`tencent-cloud.com` 等），DNS 解析器选择与
  路由结果不变。
- 同日第二轮：再移除 9 条与广告/推送拦截列表重叠的死路由条目
  （`getui.com`、`getui.net`、`jpush.io`、`pangle.io`、`sigmob.com`、
  `anythinktech.com`、`easytomessage.com`、`gepush.com`、`trace.rtbasia.com`），
  `direct.json` 从 52 条精简到 43 条。这些域名已被 `hagezi-pro` /
  `anti-ad-srs` 在 DNS 层拦截，`direct.json` 中的路由条目永远不会生效。
- 保留的 43 条是 `ChinaMaxNoIP` 未覆盖的长尾域名，主要是国内 CDN/DNS 调度域名
  （如 `kwcdn.com`、`queniu*.com`、`kunlun*.com`）和必须强制直连的境外服务
  （如 `tailscale.com`、`ntp.org`、`ipify.org`）。它们的作用是**在解析阶段
  选对 DNS 服务器**：一旦落到 `final: google`，CDN 会返回海外节点，后续
  `ChinaIPs` 无法纠正。
- 复查方法：`sing-box rule-set decompile ChinaMaxNoIP.srs -o cm.json`，
  再对每个域名执行 `sing-box rule-set match cm.json <domain>`，有输出即可删。