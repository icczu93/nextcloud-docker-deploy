# nextcloud搭建个人网盘教程：按人数选服务器、用 Docker 一键部署，再配好 HTTPS 与备份

想给自己搭一个网盘，Nextcloud 是最常被提到的开源方案之一：开源、数据存在自己机器上，文件同步、日历、联系人和在线文档都能放进同一套系统。真正动手时，卡住大多数人的是三个问题：装在哪、怎么装、装完还要做什么。这篇 nextcloud搭建个人网盘教程就按这个顺序走，先定服务器，再用官方 AIO 一条 Docker 命令把服务跑起来，最后处理域名、HTTPS、性能和安全。

> 大多数人最短的路径：在腾讯云开一台 Linux 服务器（轻量应用服务器或云服务器 CVM 都行），装好 Docker，用 Nextcloud AIO 一条命令起服务。单用户 512MB 内存起步，多人完整 Hub 建议 3–8GB；域名可以后补，AIO 新版能直接申请免费域名。[👉 查看腾讯云服务器当前配置与价格](https://bit.ly/yunhuodong)

## 先选服务器：内存按人数算，别凭感觉买

Nextcloud 官方给出的内存要求很明确：绝对最低 128MB，基本单用户建议至少 512MB，多人使用的完整 Hub 安装需要 3–8GB。买服务器时按这个范围挑，再给系统和 Docker 留一点余量，就不容易一开始就买错配置。

| 使用场景 | 内存 | 依据 |
| --- | --- | --- |
| 单人测试、尝鲜 | 128MB 起 | 官方给出的绝对最低要求 |
| 单用户日常使用 | 建议 512MB 以上 | 官方对最基本单用户环境的推荐值 |
| 多人 + 完整 Hub（Office、Talk 等） | 3–8GB | 官方对多用户完整安装的说明 |

另外两个实际约束是磁盘和系统。数据目录里放的是你的真实文件，硬盘多大，网盘的上限就多大；系统方面，AIO 官方同时提供 Linux 和 Windows 两套安装指南，长期跑服务选 Linux 服务器更常见。如果服务器在中国大陆、又要用域名对外提供访问，备案这类合规事项记得提前处理。

还没有服务器的话，可以 [👉 在腾讯云开通一台能装 Docker 的 Linux 服务器](https://bit.ly/yunhuodong)，配置照着上面的表挑。

## 三种搭建方式怎么选：AIO、手动 Compose、Web Installer

第一次搭、又希望以后升级省心，直接选官方 AIO 的 Docker 包；服务器上已经有反向代理，或者想自己指定数据库和缓存版本，用手动 Docker Compose；手里只有共享虚拟主机、没有整台服务器，才轮到 Web Installer。

| 方式 | 适合谁 | 需要自己管什么 | 上手难度 |
| --- | --- | --- | --- |
| Nextcloud AIO | 绝大多数新建实例 | 基本只在网页向导里点选，数据库、Redis、反向代理由它编排 | 低 |
| 手动 Docker Compose | 已有反向代理、想控制组件版本的人 | compose 文件、数据库、Redis、TLS 都要自己配 | 中 |
| Web Installer | 只有共享主机、没有服务器的人 | 受主机环境限制，可调空间小 | 视主机而定 |

## 方法一：Nextcloud AIO 一键部署（推荐）

AIO 把 Nextcloud、数据库、Redis 和可选应用打包成一组 Docker 容器，由一个 mastercontainer 提供网页界面统一配置和启停。2026 年多数新装实例的推荐起点就是它，也是这份教程里最省事的一条路。

1. 准备一台装好 Docker 的服务器。官方对 Linux 和 Windows（Docker Desktop）都给了命令，下面以 Linux 为例。
2. 启动 mastercontainer：

bash
sudo docker run \
--sig-proxy=false \
--name nextcloud-aio-mastercontainer \
--restart always \
--publish 80:80 \
--publish 8080:8080 \
--publish 8443:8443 \
--volume nextcloud_aio_mastercontainer:/mnt/docker-aio-config \
--volume /var/run/docker.sock:/var/run/docker.sock:ro \
ghcr.io/nextcloud-releases/all-in-one:latest


3. 浏览器打开 `https://你的服务器IP:8080`，输入 passphrase 进入 AIO 管理界面。
4. 跟着向导设置 DDNS、端口转发，勾选要用的附加容器：Nextcloud Office（在线文档）、Talk（聊天和视频通话）、备份、杀毒等。用不上的先不装，以后也能加。AIO 会自动下载并启动所有容器，列表全绿就说明服务就绪。
5. 处理域名。从 v13.3.1 起，AIO 内置了 deSEC（柏林的非营利 DNS 服务）集成：在界面里点几下就能申请一个免费域名，DNS 记录、域名指向和 HTTPS 证书都会自动配好，caddy 反向代理和 dnsmasq 本地解析也由它设置。证书走 DNS 验证，所以在你不打算对外公开访问之前，不需要提前开放端口。用自己的域名也完全可以，AIO 本来就是按「已有域名和有效证书」的生产环境设计的。

这套方式的代价是灵活性：AIO 替你做了很多决定，想深度定制组件的人会觉得束手束脚，那类需求看下一节。

## 方法二：手动 Docker Compose，适合想自己掌控组件的人

服务器上已经有反向代理，或者你想自己挑 PostgreSQL 版本、单独调 Redis，用官方 nextcloud 镜像手写一份 compose 更合适。下面是一份能跑通的最小组合：

yaml
services:
  db:
    image: postgres:16
    restart: unless-stopped
    environment:
      POSTGRES_DB: nextcloud
      POSTGRES_USER: nextcloud
      POSTGRES_PASSWORD: CHANGE_ME
    volumes:
      - ./db:/var/lib/postgresql/data

  redis:
    image: redis:alpine
    restart: unless-stopped

  app:
    image: nextcloud:latest
    restart: unless-stopped
    ports:
      - "8081:80"
    environment:
      POSTGRES_HOST: db
      POSTGRES_DB: nextcloud
      POSTGRES_USER: nextcloud
      POSTGRES_PASSWORD: CHANGE_ME
      REDIS_HOST: redis
    volumes:
      - ./nextcloud:/var/www/html
    depends_on:
      - db
      - redis


几处要改的地方：两个 `CHANGE_ME` 换成同一个强密码；`8081:80` 是宿主机端口映射，前面有反向代理的话可以改成只监听内网；`./nextcloud` 这个卷对应容器里的 `/var/www/html`，迁移和备份时它要和数据库成对处理。TLS 和域名需要你在自己的反向代理里配。

只有共享虚拟主机的话，官方还有 Web Installer 方式，能把 Nextcloud 装在共享主机上。省了服务器，但环境受主机限制，适合尝鲜，不太适合当长期主力。

## 域名和 HTTPS：对外访问与只在家里用，是两条路线

要从公网访问，就绑域名、配好 HTTPS；只想在家里或局域网用，也有不暴露公网的做法，不必为了「私有」硬开端口。

- 有域名：把 DNS 指向服务器，再在 AIO 里完成配置。用 deSEC 免费域名时，DNS 记录和证书由 AIO 自动处理。
- 没域名、只想先跑起来：用 AIO 界面里的 deSEC 入口申请一个免费域名，直接拿到可用的 HTTPS 地址。
- 完全不对外：AIO 文档给了三种本地化方案——Tailscale、自建反向代理加本地 DNS、ACME DNS challenge（用 DNS 验证签证书，不开放 80/443）。

## 装完别急着用：几项影响体验和安全的设置

最少做四件事：启用 Redis、把后台任务切到 cron、用 PostgreSQL 或 MariaDB 而不是 SQLite、给账号开两步验证。单用户测试时感觉不出差别，多设备、多用户一起同步时就会体现出来。

### 性能：Redis、cron、数据库、预览

- Redis 负责文件锁和缓存，第二个人开始用之后基本就是必需品，否则多个客户端同时同步会明显变慢。
- 后台任务用系统 cron：`*/5 * * * * php -f /var/www/html/cron.php`，别依赖 AJAX 轮询。
- SQLite 只适合单用户测试环境，正式使用换 PostgreSQL 或 MariaDB。
- 大照片库生成缩略图很吃 CPU，照片多的可以上 Memories 或 Immich 这类专门应用。

### 安全与备份：成对备份是底线

- 开两步验证，定期看「安全与设置警告」面板，把标红项逐条解决，Nextcloud 和插件保持更新。
- 数据库和数据目录必须成对备份、成对恢复，只备份其中一个等于没备份。
- 数据敏感、底层存储没做加密的话，可以开服务端加密。
- 日常排查用 `sudo -u www-data php /var/www/nextcloud/occ status` 看服务状态；occ 是 Nextcloud 的命令行管理工具，配置文件 `config.php` 在安装目录的 `config/` 文件夹里，很多修复都要碰它。

外部存储是另一个常用功能：设置里可以把 S3 兼容对象存储（MinIO、Backblaze B2）、SMB/CIFS、SFTP 挂进文件列表，不用把数据全部搬进服务器。

## 版本和升级：Hub 26 和 34.0.3 说的是同一个东西

Nextcloud 每 4 个月发布一个大版本，每个大版本支持约一年，其间有月度维护更新。按 2026 年 8 月的记录，当时稳定版是 34.0.3，33 支持到 2027 年 2 月。市场名和版本号指的是同一套软件：34 是 Hub 26 Spring，33 是 Hub 26 Winter，35 是 Hub 26 Summer。

这意味着一年里你大概要升级两三次。选安装方式时，把「以后能不能从容升级」算进去，比省下安装的十分钟更重要。

## 成本怎么算：软件免费，钱花在服务器上

Nextcloud 服务端是 AGPL 授权，免费，Talk、Office 这些功能也在免费版里；自建的成本就是服务器、域名（可选）和维护时间。Enterprise 订阅购买的是支持合同和更长的维护窗口，不解锁功能。

- 存储上限取决于你的硬盘，不取决于软件授权，这是自建和买网盘会员最大的区别。
- 想压低成本：先用最低配跑起来，域名用 deSEC 免费的，附加容器只装用得上的，不够再升配置。
- 各家云厂商、各个配置的价格差异不小，按前面的内存表挑好规格，再去 [👉 对比腾讯云当前的服务器配置与价格](https://bit.ly/yunhuodong)，自己算一年总花费。

## 常见问题

**Nextcloud 免费吗？** 免费。核心代码是 AGPLv3 开源，个人自建不花钱，官方卖的是企业支持服务。

**AIO 和手动安装选哪个？** 大多数人用 AIO；已有反向代理、想指定组件版本，用手动 Compose。

**最低需要多少内存？** 官方底线 128MB，单用户至少 512MB，多人完整 Hub 建议 3–8GB。

**不想暴露在公网可以吗？** 可以，AIO 支持 Tailscale、本地反向代理加本地 DNS、ACME DNS challenge 三种方式。

**一定要买域名吗？** 对外访问建议有；可以先不买，AIO 能申请 deSEC 免费域名；纯局域网用本地方案即可。

**只同步一两台设备的文件，也要装 Nextcloud 吗？** 不一定。只需要点对点同步、不用日历和在线文档的话，Syncthing 这类工具更轻。

## 按你的情况选

- 一个人用，主要存照片和文件：最低配 Linux 服务器 + AIO + deSEC 免费域名，先跑起来，不够再加配置。
- 小团队要在线文档和视频会议：内存按 3–8GB 选，AIO 里把 Office 和 Talk 容器勾上。
- 已经有 NAS 或反向代理：手动 Compose，或者用外部存储把现有磁盘挂进 Nextcloud。

下单前确认两件事：服务器是 Linux、能跑 Docker；如果打算公网访问，域名和备案提前处理。

[👉 去腾讯云把服务器定下来，再回来照着上面的命令装](https://bit.ly/yunhuodong)
