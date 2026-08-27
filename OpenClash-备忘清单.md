# OpenClash 规则集问题备忘清单

适用场景：
- 规则集分流异常（例如 x.com / chatgpt.com 走错分组）
- 修改远端规则文件后需要让路由器重新拉取
- 规则集格式异常（classical 变成裸域名 / `+.domain`）

---

## 一、快速确认（规则集是否正常）

1. 查看规则集缓存格式（应为 classical：`DOMAIN-SUFFIX,xxx`）

```sh
head -n 10 /etc/openclash/rule_provider/Twitter_Domain
head -n 10 /etc/openclash/rule_provider/ChatGPT_Domain
```

预期：
```
DOMAIN-SUFFIX,x.com
DOMAIN-SUFFIX,chatgpt.com
```

如果看到裸域名或 `+.domain`，说明格式被转换，可能导致不命中。

---

## 二、强制重新拉取规则集（最常用）

当你修改了远端规则，或怀疑缓存异常时，执行以下命令强制重新拉取：

```sh
/etc/init.d/openclash stop
rm -rf /etc/openclash/rule_provider/*
/etc/init.d/openclash start
```

说明：
- 不要删除 `/etc/openclash/rule_provider` 目录本身，只清空目录内容。
- 这会让 OpenClash 在启动时重新下载所有规则集。

---

## 三、验证远端规则是否可访问

用于确认路由器可以正常拉取远端规则（含重定向）：

```sh
curl -IL "https://edgeone.gh-proxy.com/https://raw.githubusercontent.com/iBigQiang/clash/main/rule-refs/x.list"
```

若返回 200 OK，说明可访问。

---

## 四、下载远端规则并检查内容

用于确认远端规则文件内容本身是否正确：

```sh
curl -L -o /tmp/test.list "https://edgeone.gh-proxy.com/https://raw.githubusercontent.com/iBigQiang/clash/main/rule-refs/x.list"
head -n 5 /tmp/test.list
```

预期内容：
```
DOMAIN-SUFFIX,api.x.com
DOMAIN-SUFFIX,x.com
DOMAIN-SUFFIX,x.ai
DOMAIN-SUFFIX,twitter.com
DOMAIN-SUFFIX,t.co
```

---

## 五、检查系统时间与证书（SSL 失败时）

如果日志出现：
```
curl: (60) SSL certificate problem: self signed certificate
```
按以下顺序处理：

1. 校准系统时间：
```sh
date
/etc/init.d/sysntpd restart
date
```

2. 更新 CA 证书：
```sh
opkg update
opkg install ca-certificates ca-bundle
```

---

## 六、快速验证分流是否生效

访问目标网站后，在 OpenClash 连接列表中确认：
- `x.com` → `RuleSet: Twitter_Domain`
- `chatgpt.com` / `chat.openai.com` → `AI` 分组

---

## 七、常见结论提示

- 规则集文件可下载 ≠ 实际会命中，必须看格式是否 classical。
- 远端规则更新后，必须清空 `/etc/openclash/rule_provider` 并重启，确保拉取新版本。

