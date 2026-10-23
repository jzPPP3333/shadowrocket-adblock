# Shadowrocket 全能去广告配置

面向 iOS Shadowrocket 的去广告模块。覆盖开屏广告、信息流广告、弹窗、横幅、以及背后拉取广告的第三方 SDK。

## 特性

- **零依赖**：一个文件，作为模块叠加到你现有配置上，不破坏你原有的节点和分流
- **三层拦截**：DNS 域名层 → URL 重写层 → QUIC 阻断，逐层收口
- **精准范围**：只对需要改写的域名开 MITM 解密，其余纯域名拦截，不做 TLS 解密，性能开销极低
- **优雅降级**：`reject-dict` 返回 `{}` 而非直接断开，App 不会白屏也不会卡在加载态
- **持续更新**：内置社区云端规则集，链接文件自动跟随上游更新

## 快速开始

### 1. 安装模块

Shadowrocket → 底部「模块」→ 右上角 `+` → 粘贴下面的地址 → 保存并启用：

```
https://raw.githubusercontent.com/2xz7vmvpv4-art/shadowrocket-adblock/main/AdBlock.sgmodule
```

国内网络如果拉取失败，用镜像：

```
https://raw.gitmirror.com/2xz7vmvpv4-art/shadowrocket-adblock/main/AdBlock.sgmodule
```

### 2. 装并信任证书

模块里的 `[URL Rewrite]` 依赖 MITM 解密，必须信任证书：

1. Shadowrocket → 「配置」→ 当前配置 → 「编辑配置」→ 「HTTPS 解密」
2. 打开开关 → 「生成新的 CA 证书」→ 「安装证书」
3. iOS 设置 → 「已下载描述文件」→ 安装（输密码）
4. iOS 设置 → 「通用」→「关于本机」→ 最底部「证书信任设置」→ 打开 Shadowrocket 的**完全信任**

第 4 步不做，模块的 URL 重写全部无效。

### 3. 生效

重新开关一次 Shadowrocket，杀掉 App 后台，重新打开。

## 配置结构

模块分四段，按优先级从高到低：

| 段 | 作用 | 说明 |
|---|---|---|
| `[Rule]` QUIC 阻断 | 打掉 UDP 443 | App 走 HTTP/3 会绕过 MITM，必须先逼它回落 TCP |
| `[Rule]` 域名 REJECT | 已知广告域名直接丢弃 | 不解密、零开销，覆盖绝大部分第三方广告 SDK |
| `[Rule]` RULE-SET | 云端规则集 | 社区维护的持续更新清单 |
| `[URL Rewrite]` | 接口级改写 | 广告混在业务 API 里的场景，返回 `{}` |

顺序不能调。QUIC 阻断和域名 REJECT 必须排在 `RULE-SET` 和你自己的 `FINAL` 之前。

## 覆盖范围

### 已内置精准规则的应用

| 应用 | 覆盖内容 |
|---|---|
| 知乎 | 开屏、切后台、浮层、下拉刷新、预取、横幅、云控配置 |
| 车来了 | 开屏、线路页横幅、信息流广告 |
| 哔哩哔哩 | 开屏、推荐页广告插入 |
| 微博 | 信息流广告 |
| 小红书 | 开屏、信息流 |

### 已阻断的广告联盟

穿山甲（字节）、优量汇（腾讯）、百青藤（百度）、快手联盟、拓飞聚合、浩瑞互联、Google AdMob、AppLovin、Unity Ads、Vungle、ironSource、Chartboost、Criteo、Taboola、Outbrain 等 40+ 域名。

海外主流联盟走 `RULE-SET` 之外的显式域名 REJECT，不走云端清单，保障确定性。

## 常见问题

### 配置完还有广告

三个可能，按顺序排查：

1. **本地已缓存**。开屏广告提前几小时就下载到手机沙盒里了，断网也能播。卸载重装该 App 或清理其缓存。
2. **走了 HTTP/3**。检查 `[Rule]` 最顶部的 `AND,((DEST-PORT,443),(PROTOCOL,UDP)),REJECT` 是否被你的配置覆盖掉了。
3. **证书没信任**。回到「快速开始」第 2 步的第 4 小步核对。

### 某个 App 功能异常

把 `[Rule]` 里对应的 `DOMAIN-SUFFIX,...,REJECT` 注释掉（行首加 `#`）。常见误伤：

- `umeng.com` — 友盟统计。部分 App 会因为拿不到统计配置而启动变慢，但功能正常
- `getui.com` — 个推推送。注释掉可恢复推送
- `bytedance.com` — 字节全域。如果装了抖音、今日头条，这条会误伤，建议注释

### MITM 会不会泄露隐私

解密只发生在你手机本地，解密后的流量经你的节点转发，不经任何第三方。本配置的 `[MITM]` 只列了必要域名，其他流量完全不解密。

不想用 MITM 的话，把整个 `[URL Rewrite]` 和 `[MITM]` 段删掉即可，域名级拦截仍然生效，只是拦不住混在业务 API 里的广告。

### 如何抓漏掉的广告

Shadowrocket → 「日志」→ 清空 → 复现广告 → 搜索框输入 `ad` 或 `splash`。

看哪个请求显示 `DIRECT` 且 URL 里带广告特征，把它加进 `[Rule]` 或 `[URL Rewrite]`。

## 贡献

发现漏网之鱼，带日志片段提 Issue。三段信息能直接定位：

1. 出广告的 App 与版本号
2. Shadowrocket 日志里对应的请求 URL 与命中的规则
3. 是开屏、信息流、还是弹窗

## 免责声明

仅供学习与个人使用。绕过第三方广告属于违反其服务条款的行为，使用风险自负。

## 许可

MIT
