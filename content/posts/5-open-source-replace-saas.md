---
title: "每月被 SaaS 吸走 500 块？这 5 个开源项目先替掉"
description: "订阅费不是一笔大钱，但它最会悄悄流血。Paperless-ngx、Karakeep、Vaultwarden、Anytype、AdGuard Home 这 5 个开源项目，适合把文档、书签、密码、知识库和全屋去广告慢慢迁回自己手里。"
date: 2026-05-27T14:20:00+08:00
draft: false
author: "任博"
tags: ["开源工具", "自托管", "SaaS", "GitHub", "效率工具"]
categories: ["工具效率"]
cover: "/images/5-open-source-replace-saas/cover.png"
toc: true
---

翻一遍订阅账单，很多人会有点后背发凉。

Evernote 一笔，1Password 一笔，书签工具一笔，笔记工具一笔，DNS 去广告再来一笔。每一项单看都不贵，放在一起，一个月五六百块就没了。

更烦的是，你不是不用这些工具。

文档要归档，密码要同步，网页要收藏，知识库要整理，家里广告也确实要挡。问题不在工具，问题在于你把太多基础能力都租出去了。

订阅费最可怕的地方，不是贵，是它让你习惯了“不拥有”。

这篇不劝你一夜之间逃离所有 SaaS。那不现实，也没必要。我的建议更简单：先从最成熟、最容易落地的 5 类工具开始替换。

这 5 个项目都在 GitHub 上长期维护，星标数不低，也不是那种三天热度的玩具：

- Paperless-ngx：文档归档和 OCR
- Karakeep：书签、网页、图片和笔记收藏
- Vaultwarden：自托管密码库
- Anytype：本地优先的知识库
- AdGuard Home：全屋 DNS 去广告

按常见付费工具估算，它们能替掉每月大约 80 美元，折合人民币约 560 元。你不一定能全省下来，但只要替掉其中两三个，账单就会立刻轻一点。

![用开源项目替掉一部分 SaaS 订阅](/images/5-open-source-replace-saas/cover.png)

## 开源不是免费午餐

自托管不是把钱省掉就结束了。你只是把一部分现金成本，换成了时间成本、维护成本和责任。

这句话要放在前面。

如果你连 Docker 是什么都不想碰，或者家里没有 NAS、VPS、闲置小主机，那这篇文章更适合当作工具清单收藏，不适合今晚就全部开干。

但如果你已经有一台常年开机的机器，或者本来就爱折腾 HomeLab，那这些工具就很值。

你不需要一次搭完。先挑一个最痛的地方动手。

## Paperless-ngx：把纸质文件变成能搜索的档案库

Paperless-ngx 做的事很朴素：把扫描件、PDF、发票、合同、说明书这些文件吃进去，自动 OCR、打标签、归档，然后让你以后能搜出来。

它可以替代什么？

如果你现在用 Adobe Scan 扫文件，再把 PDF 丢进 Evernote、网盘或某个笔记软件里，Paperless-ngx 就是更适合长期归档的那一层。

这类工具的价值不在“存文件”。存文件谁都会。它真正解决的是三个月后你要找那张发票、那份合同、那张保修卡时，不用再翻聊天记录、邮箱和一堆叫 `scan_2024_final_final.pdf` 的文件。

Paperless-ngx 的 GitHub 星标已经超过 4.1 万。它不是新项目，文档、Docker 部署、OCR 流程都比较成熟。

最常见的安装方式是 Docker Compose。官方 compose 文件里已经把 Redis、PostgreSQL 和 Web 服务串好了。落地时可以这样做：

```bash
mkdir -p ~/apps/paperless-ngx
cd ~/apps/paperless-ngx

curl -O https://raw.githubusercontent.com/paperless-ngx/paperless-ngx/main/docker/compose/docker-compose.postgres.yml
mv docker-compose.postgres.yml docker-compose.yml

docker compose pull
docker compose up -d
```

默认 Web 端口通常是 `8000`。第一次跑起来后，重点不是疯狂导入旧文件，而是先建立一个固定入口：手机扫描、邮箱附件、下载目录，最后都进 Paperless。

注意事项也很明确：

第一，OCR 很吃 CPU。你如果丢进去几千个历史 PDF，别怪小主机风扇起飞。

第二，备份不能偷懒。Paperless 管的是发票、合同、证件、账单这类文件，丢了比普通笔记麻烦得多。至少要备份数据库、媒体文件和 consume/archive 目录。

第三，扫描件别只放在一个地方。自托管不是把云厂商换成你家的单点故障。

能省多少钱？

如果它替掉 Evernote 付费版和一部分扫描/归档工具，每月按 100 元人民币估算并不过分。但更大的收益不是钱，是你终于有了一个“家庭文件柜”。

## Karakeep：别再让收藏夹变成垃圾场

很多人的浏览器书签，本质上已经坏了。

你收藏了一堆文章、GitHub 项目、产品页面、教程和图片，过两个月再打开，不是链接失效，就是忘了当初为什么要收藏。

Karakeep 做的是“什么都能收”：链接、图片、笔记都可以放进去；它还支持全文搜索、自动标签、页面存档。这个项目以前叫 Hoarder，现在改名 Karakeep，GitHub 星标已经超过 2.5 万。

它可以替代 Raindrop Pro、Pocket 这类收藏工具，尤其适合“我不是只存链接，我要把网页内容也留一份”的人。

部署方式也很直接。官方 Docker Compose 里会启动 Web 服务、Meilisearch 和一个 Chrome 容器，用来抓取页面和生成预览。

```bash
mkdir -p ~/apps/karakeep
cd ~/apps/karakeep

git clone https://github.com/karakeep-app/karakeep.git .
cd docker
cp .env.example .env

docker compose up -d
```

默认端口是 `3000`。如果你已经有反向代理，可以再挂到自己的域名下面。

Karakeep 好用的地方，是它把“收藏”从浏览器里解放出来。浏览器书签适合放每天要打开的站点，不适合当资料库。资料库需要全文搜索、标签、预览、归档，还要能解释“我为什么收藏它”。

这里有个小坑：如果你开启 AI 自动标签，需要自己提供对应模型或 API Key。没有也能用，只是自动整理能力会弱一些。

另外，收藏工具最容易变成新的垃圾场。别指望换个开源项目就突然变自律。我的建议是给 Karakeep 定一条规矩：只收以后可能复用的内容，不收“看起来挺有用”的内容。

有用感，是收藏夹最大的噪音。

它能替掉 Raindrop Pro + Pocket 这一类服务，每月按 100 元人民币估算。但真正省下来的，是你下次找资料时少翻 20 分钟。

## Vaultwarden：1Password 很好，但你可以先把家庭密码库掌握住

密码管理器这件事，我不会劝普通人轻易自托管。

因为密码库不是博客，不是 RSS，也不是书签。它一旦搞砸，代价很高。

但如果你已经理解反向代理、HTTPS、备份、2FA，也愿意为它做维护，Vaultwarden 是开源世界里最成熟的选择之一。

Vaultwarden 是一个用 Rust 写的 Bitwarden 兼容服务端。你可以继续使用 Bitwarden 的浏览器插件和移动端 App，只是服务端换成你自己的。它的 GitHub 星标已经超过 6.1 万。

它替代的是 1Password Family、Bitwarden 官方高级方案，或者其他家庭密码同步服务。

最小 Docker Compose 可以这样写：

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: always
    environment:
      SIGNUPS_ALLOWED: "true"
    volumes:
      - ./vw-data:/data
    ports:
      - 11001:80
```

启动：

```bash
docker compose up -d
```

创建完自己的账号后，第一件事不是导入密码，而是把注册关掉：

```yaml
SIGNUPS_ALLOWED: "false"
```

然后再配置域名、HTTPS、反向代理和备份。

这里必须泼冷水：Vaultwarden 不适合“照着教程复制一下就完事”的人。密码库至少要做到三件事：

- HTTPS 正常，不裸奔
- 管理后台和注册入口收紧
- 数据目录定期离线备份，并且真的演练过恢复

如果这三件事你不想做，继续用 1Password 或 Bitwarden 官方服务更稳。

省钱只是 Vaultwarden 最浅的一层理由。更深一层，是你能控制家庭密码库的存放位置、备份策略和迁移路线。

1Password Families 官方价格常见是每月 4.99 美元或按年折扣后每月约 4.49 美元，折合人民币三四十元。任务里按每月 70 元估算偏保守地覆盖了家庭套餐和汇率/税费/渠道差异。

## Anytype：不是另一个 Notion，而是本地优先的个人知识库

Anytype 最容易被误解成“开源 Notion”。

这不准确。

Notion 是一个很好用的云端协作工具。Anytype 的核心气质不一样：本地优先、端到端加密、对象化知识库、离线可用。它更像是给个人搭一个私密知识系统，而不是给团队搭一个在线协作文档中心。

Anytype 官方桌面客户端仓库 `anyproto/anytype-ts` 星标接近 8 千。更关键的是，它背后还有 any-sync 这一套同步网络，自托管不是单个 Docker 容器那么简单。

如果你只想替代个人 Notion，用 Anytype 客户端就能开始，不一定立刻自托管。

如果你要自托管同步网络，官方的 any-sync-dockercompose 给的是个人、测试、评估用途的部署方式。它会启动 coordinator、多个 sync node、filenode、consensus node、MongoDB、Redis、MinIO 等组件。

```bash
git clone https://github.com/anyproto/any-sync-dockercompose.git
cd any-sync-dockercompose
cp .env.example .env

docker compose up -d
```

第一次启动后，会生成 `./etc/client.yml`，你需要把这个配置导入 Anytype 客户端，切换到自己的 self-hosted network。

Anytype 的好处是：你不再把所有知识库都押在一个云端工作区上。网络断了，很多内容仍然在本地。隐私和数据控制也更符合个人长期积累的需求。

但它的代价也很清楚：如果你是重度团队协作、多人同时编辑、复杂权限管理，Notion 仍然更顺手。Anytype 更适合个人知识库、家庭资料库、长期笔记，而不是替代公司 Wiki。

所以别把它当成 Notion 平替。把它当成一套更偏个人、私密、本地优先的知识系统，你就不会用错。

按 Notion Plus 加 Roam Research 这类订阅组合估算，每月 140 元人民币是能算出来的。但实际是否能省，取决于你到底用不用 Roam 那套双链思维，以及团队是否依赖 Notion 协作。

工具不是越开源越高级。适合你的工作流，才算替代成功。

![五类订阅工具的替代路线](/images/5-open-source-replace-saas/inline-01.png)

## AdGuard Home：广告拦截别只装在一台电脑上

浏览器插件只能管浏览器。

手机 App、电视、平板、游戏机、智能音箱、家里老人用的设备，它们不一定能装插件，也不一定愿意折腾。

AdGuard Home 的思路是从 DNS 层面拦截广告和追踪域名。部署好之后，让路由器或设备把 DNS 指向它，整个家庭网络都会受益。

它的 GitHub 星标超过 3.4 万，是这几个项目里最适合“搭一次，全家用”的工具之一。

Docker 启动命令可以从最小版本开始：

```bash
mkdir -p ~/apps/adguard/{work,conf}

docker run --name adguardhome \
  --restart unless-stopped \
  -v ~/apps/adguard/work:/opt/adguardhome/work \
  -v ~/apps/adguard/conf:/opt/adguardhome/conf \
  -p 53:53/tcp -p 53:53/udp \
  -p 3000:3000/tcp \
  -d adguard/adguardhome
```

浏览器打开：

```text
http://127.0.0.1:3000/
```

配置完成后，真正关键的一步是把路由器 DHCP 下发的 DNS 改成 AdGuard Home 的地址。这样家里的设备不用一个个设置。

但 DNS 去广告也不是魔法。

YouTube、B 站、很多 App 内广告，并不一定能靠 DNS 全部拦掉。因为它们可能和正常内容走同一个域名。DNS 只能拦“域名级别”的东西，不能理解页面里的每一段内容。

还有一个坑：别把它部署在全家唯一 DNS 上却没有备用方案。AdGuard Home 挂了，全家网络就会像断网一样。至少要知道怎么临时切回路由器默认 DNS。

它能替代 NextDNS Premium 这类服务。NextDNS 个人 Pro 在不同地区价格会显示成本地货币，任务里按每月 140 元人民币估算偏高，更像覆盖家庭多人、多设备和其他广告拦截订阅的组合成本。单看 NextDNS 官方个人版，实际通常没这么贵。

这一点要算清楚：AdGuard Home 最大收益不是省下 NextDNS 那几块钱，而是你终于能控制家里 DNS 规则。

## 最适合这个周末开工的顺序

如果你不知道先装哪个，我建议按这个顺序来：

第一，AdGuard Home。见效最快，搭好后全家都能感受到。

第二，Karakeep。迁移风险低，坏了也不会伤筋动骨，适合练手。

第三，Paperless-ngx。价值很高，但要认真做备份。

第四，Anytype。先用客户端，再考虑自托管同步网络。

第五，Vaultwarden。最后再动密码库。不是因为它不好，而是因为它最不该随便搭。

很多人玩自托管会犯一个错：一上来就想把所有云服务一次性替掉。

别这样。

真正可持续的迁移，是一个工具一个工具地换。每换一个，都问三件事：

- 我有没有稳定入口？
- 我有没有备份？
- 我有没有恢复方案？

没有恢复方案的自托管，只是换了个地方裸奔。

## 这 500 块，不一定全该省

最后再把话说重一点。

不是所有 SaaS 都该被替代。你每天赖以吃饭的团队协作、客户资料、公司生产系统，该付费就付费。专业服务的稳定性、合规、客服和团队协作，不是几个 Docker 容器能轻松替掉的。

但个人生活里的很多基础能力，确实可以慢慢拿回来。

文档归档可以自己管。

书签资料可以自己管。

家庭密码库在有能力时可以自己管。

个人知识库可以先本地优先。

全屋 DNS 规则也可以自己管。

这不是为了省每月 500 块而折腾。省钱只是入口。

真正重要的是：你开始重新区分，哪些能力值得租，哪些能力应该拥有。

如果你这个周末只想动手一个，我会选 AdGuard Home；如果你想先做长期收益，我会选 Paperless-ngx；如果你最怕账单继续流血，就从 Karakeep 和 Anytype 这类低风险替代开始。

别一口气全迁。

先替掉一个，跑稳一个。你的订阅账单，就是这样一点点瘦下来的。

如果你对这类开源替代、自托管工具、HomeLab 实战感兴趣，可以关注我。后面我会继续把能真正落地的工具筛出来，不写“看起来很美”的工具清单，只写普通人周末能跑起来的东西。
