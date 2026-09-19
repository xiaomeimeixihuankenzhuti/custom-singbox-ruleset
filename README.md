# Custom Sing-box Ruleset

Sing-box 自定义规则集，用于管理网络流量的路由规则。

## 项目结构

| 文件 | 说明 |
|------|------|
| `direct.json` | 直连规则 - 定义不需要走代理的域名 |
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

`direct.json` 中的域名在 sing-box 的 `dns.rules` 中**优先于广告拦截规则**匹配：
命中白名单的域名走本地 DNS 解析，不会被 `adguard-ads` / `anti-ad-srs` 的
`reject` 规则拦截。

**`domain_suffix` 匹配的是整棵子域树，不只是基域本身：**

| 白名单条目 | 同时被豁免拦截的子域（实测） |
|---|---|
| `xiaomi.com` | `tracker.ai.xiaomi.com` |
| `mi.com` | （`ot.io.mi.com` 等） |

也就是说，写 `xiaomi.com` 的意图是放行小米主站，但实际效果是
**所有 `*.xiaomi.com` 都不再被广告拦截**。加入白名单前请确认这一连带效果可接受。

如需恢复某个追踪子域的拦截，将该基域从 `direct.json` 中移除即可——
但请先确认其正常服务域名能被 `geosite-cn` 规则集覆盖，避免影响业务连通性。