# CS2社区服搭建全流程：Docker 开服要几步、2 核 2G 够不够、端口和令牌怎么配（附云主机选购对照表）

CS2 社区服不需要在本机双击某个服务端程序。现在的主流做法是跑一个 Docker 镜像，把它放在一台 7×24 在线的机器上，镜像每次启动时自己从 Steam 拉服务端、自己更新。镜像官方 README 写明的底线是 **2 CPU、2GiB 内存、60GB 空闲磁盘**，外加 **27015 的 TCP 和 UDP、27020 的 UDP** 三个端口。下面把从零到能进服的整条链路走一遍，再说机器该怎么挑、哪些参数值得改、哪些坑前排教程基本不提。

## 先解决机器：为什么这件事绕不开云主机

家宽开服卡在三个地方：没有固定公网 IP，上行带宽要同时喂给十个玩家，机器得一直开着不能睡眠。租一台云主机是最省事的解法，也是后面所有步骤的前提。

10 人左右的竞技服，规格不用堆。官方最低线往上留一点余量就够，唯一需要往上加的是磁盘。

> 一句话版本：2 核 / 2GiB 内存 / 60GB 磁盘是官方写明的底线，选机器时磁盘别卡在 60GB 这个数上；带宽够十个玩家同步就行，实际体验更多取决于线路到玩家的延迟。具体在售规格和价格看雨云当前的机器列表，对着下面那张对照表挑。
>
> 👉 [打开雨云查看当前在售的云主机规格与价格](https://bit.ly/RaiNyun)

这里不贴一份套餐价格表。套餐、活动和续费价会变，抄一份可能已经过期的数字给你，比不给更糟。下面这张表是可以直接拿去和官网对照的规格清单。

| 项目 | 镜像官方最低线 | 10 人竞技服的建议 | 说明 |
| --- | --- | --- | --- |
| CPU | 2 核 | 2 核起 | 玩家数上去或开 CSTV 录像再加 |
| 内存 | 2GiB | 2GiB 起 | 装 mod、上工坊图会吃更多 |
| 磁盘 | 60GB 空闲 | 60GB 以上，装 mod 再往上留 | 官方明确写了“装 mod 可能需要更多空间” |
| 端口 | 27015/tcp、27015/udp、27020/udp | 三个都要放行 | 只开 UDP 会让 RCON 和查询出问题 |
| 主机权限 | 未规定 | 有 root / SSH，能自己装 Docker | 面板服不一定给这个权限，下单前问清楚 |
| 带宽 | 未规定 | 够十人 tick 同步即可 | README 没给带宽数字，别信“几人几兆”的换算表 |

## 开服前先确认这四件事

**一、SRCDS_TOKEN 必须先拿到。** 这是 CS2 专用服务器的游戏服务器令牌，没有它服务端起不来。获取地址是 Steam 的游戏服务器账号管理页：`steamcommunity.com/dev/managegameservers`，用你的 Steam 账号登录后申请一个。这个令牌后面要作为环境变量传给容器。

**二、三个端口一个都不能少。** `27015/tcp`（RCON 和查询走 TCP）、`27015/udp`（游戏流量）、`27020/udp`（CSTV / SourceTV）。很多人只记得放 UDP，结果 RCON 连不上，还以为是密码错了。

**三、磁盘按 60GB 起算。** 这个数字是镜像 README 里的原文要求，不是估算。服务端本体加上历次更新会占用相当一部分，如果你还打算装 mod，官方提示需要更多空间。磁盘买小了，跑几个月更新几次就爆，扩容比一开始多买 20GB 麻烦得多。

**四、主机要能跑 Docker。** 这个镜像是 Linux 容器镜像，宿主要么是 Linux，要么是能装 Docker 的 Windows。想省事就直接选 Linux 主机，装完 Docker 剩下的命令和本文一字不差。

## 搭建流程：从空机器到能进服

### 第 1 步：选镜像

镜像在两个仓库都有，内容一致：

- Docker Hub：`joedwards32/cs2`
- GitHub Container Registry：`ghcr.io/joedwards32/cs2`

`CM2Walki/CS2` 是同一镜像的镜像仓库，README 内容基本一样，看哪个都行。

### 第 2 步：建数据目录并改属主

容器以一个 uid 为 1000 的非特权用户 `steam` 运行。宿主机上挂载进去的目录如果属主不对，容器写不进去，服务端会起不来。这一步经常被跳过。

bash
mkdir -p $(pwd)/cs2-data
chown 1000:1000 $(pwd)/cs2-data


### 第 3 步：一条 docker run 拉起来

bash
export SRCDS_TOKEN="你的令牌"

docker run -d --name=cs2 \
  -e SRCDS_TOKEN="$SRCDS_TOKEN" \
  -e CS2_SERVERNAME="My Server 1\/3" \
  -e CS2_RCONPW="换成你自己的密码" \
  -e CS2_MAXPLAYERS=10 \
  -e CS2_GAMEALIAS="competitive" \
  -v $(pwd)/cs2-data:/home/steam/cs2-dedicated/ \
  -p 27015:27015/tcp \
  -p 27015:27015/udp \
  -p 27020:27020/udp \
  joedwards32/cs2


注意 `CS2_SERVERNAME` 里那个 `\/`。这是官方 README 里点名的规则：CS2 相关环境变量中的 `/` 必须转义成 `\/`，否则在配置处理阶段会出现意料之外的行为。上面这行在游戏内显示为 `My Server 1/3`。

不想写长命令的话，仓库的 `examples` 目录里有现成的 `docker-compose.yml`，改掉里面的令牌和密码，`docker compose --file examples/docker-compose.yml up -d cs2-server` 就行。

### 第 4 步：等首次下载

容器第一次启动会自动跑 steamcmd 下载 CS2 服务端，几十 GB，视线路快慢要等一阵。之后每次重启容器，它都会检查并应用游戏更新。**游戏出新版本时你不需要手动跑任何命令，重启容器就够了。**

### 第 5 步：几个值得改的变量

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `SRCDS_TOKEN` | 空 | 必填，没有它起不来 |
| `CS2_SERVERNAME` | `changeme` | 服务器显示名，`/` 写成 `\/` |
| `CS2_RCONPW` | `changeme` | RCON 密码，务必改 |
| `CS2_MAXPLAYERS` | `10` | 最大玩家数 |
| `CS2_GAMEALIAS` | 空 | 填 `casual` / `competitive` / `deathmatch` |
| `CS2_GAMETYPE` / `CS2_GAMEMODE` | `0` / `1` | 仅在 `CS2_GAMEALIAS` 为空时生效 |
| `CS2_PORT` | `27015` | 游戏端口 |
| `CS2_PW` | 空 | 进服密码，可选 |
| `CS2_LAN` | `0` | 设 1 为局域网模式 |

游戏模式优先用 `CS2_GAMEALIAS`，它比手动填 gametype 和 gamemode 两个数字直观得多。

## 五个新手最常踩的坑

**坑一：服务器名带斜杠。** 上面说过了，`/` 必须写成 `\/`。名字里带日期、带战队名分隔符的人最容易中招。

**坑二：为了省 CPU 开休眠。** `CS2_SERVER_HIBERNATE` 的说明里有一句官方警告，原文是“hibernation has been observed to trigger server crashes”，也就是说开休眠观察到会触发服务器崩溃。默认值是 0，别改。

**坑三：设了开局地图却没生效。** 如果你配置了工坊地图（`CS2_HOST_WORKSHOP_MAP` 或 `CS2_HOST_WORKSHOP_COLLECTION`），`CS2_MAPGROUP` 和 `CS2_STARTMAP` 会被直接忽略。这不是 bug，是设计如此。README 同时把 Steam Workshop 支持标为**实验性**功能，别把正式比赛挂在这上面。设了工坊合集之后可以在 RCON 里用 `ds_workshop_listmaps` 列图、`ds_workshop_changelevel $map_name` 换图。

**坑四：只放行 UDP。** 27015 的 TCP 和 UDP 是两个独立端口，CSTV 的 27020 是第三个。云厂商的安全组和系统防火墙都要放，两处漏一处都连不上。

**坑五：直接改 `gamemode_competitive.cfg`。** 各模式的默认配置在 `/home/steam/cs2-dedicated/game/csgo/cfg/` 下，比如竞技模式是 `gamemode_competitive.cfg`。想改规则，写进同目录的 `gamemode_competitive_server.cfg` 更稳，原有默认文件留着不动。比如想把比赛改短，就在里面加一行 `mp_maxrounds 16`。直接改默认文件并非官方禁止，但镜像更新时有被覆盖的风险，覆盖文件是更省心的做法。

## 开服之后：日常怎么管

**RCON 不一定非要装客户端。** 有一个第三方网页版 RCON 控制台 `rcon.srcds.pro`，作者自己标注是 beta 版本。用法是在文本框里填 `IP:PORT`（形如 `123.123.123.123:27015`）加 RCON 密码，点连接。它带服务器历史、快捷命令菜单（Cvars、竞技图一键切换、AimMap、Configs）和一个带输入补全的手动命令台，在 Steam 内置浏览器里就能开，不用切出游戏。作者在说明里提到它可以在连接游戏服务器时替代 RCON 协议，这个表述比较含糊，别理解成“官方 RCON 已经失效”，它只是一个更顺手的第三方入口。

**CSTV 录像要额外占磁盘。** `TV_ENABLE=0` 默认关闭，开了之后 `TV_PORT=27020` 是对外端口。想自动录下每一场，把 `TV_AUTORECORD` 设成 1，代价是磁盘会持续增长，这也是前面建议磁盘别卡最低线的原因之一。

**日志可以推到自己的服务上。** `CS2_LOG_HTTP_URL` 能把日志通过 HTTP POST 推给一个监听端点，做战绩统计或在线人数看板时有用。URI 必须用双引号包起来。

**装 metamod 用钩子脚本。** 容器提供两个脚本：`/home/steam/cs2-dedicated/pre.sh` 在 CS2 启动前执行，`post.sh` 在停止后执行。挂了持久卷之后可以直接编辑它们做自定义动作，README 举的例子就是启用 metamod。也可以从宿主机挂一个外部文件覆盖进去。

**配置批量下发用 `CS2_CFG_URL`。** 给它一个 HTTP/HTTPS 地址，容器会拉取 tar.gz、tar 或 zip 格式的配置包并解压，适合一次管多台服务器、配置想版本化的情况。

**改坏了就校验重下。** 把 `STEAMAPPVALIDATE` 设成 `1`，steamcmd 会校验并重新下载游戏文件，默认是 0（跳过校验）。排查“为什么我突然起不来了”时，这是最快的兜底手段。

**RCON 端口可以单独走。** 有些云厂商的端口映射不支持同一端口混合协议，这时用 `CS2_RCON_PORT` 让 RCON 走一个额外的 TCP 端口，记得在安全组里把那个 TCP 端口也映射出去。

## 该买什么规格：按你的实际情况对号入座

- **10 人左右、只打竞技或休闲、不录像**：2 核 / 2GiB 内存 / 60GB 磁盘，正好压在官方底线附近，够用。
- **要开 CSTV 自动录像**：CPU 和磁盘都往上加一档，录像文件是持续增长的。
- **要上创意工坊地图**：磁盘多留，工坊图和它的依赖会额外占空间；同时接受它目前是实验性功能这个前提。
- **要装 mod、metamod、跑一堆插件**：磁盘和内存都往上留，官方对 mod 场景的用词是“可能需要更多空间”。

下单前有两件事值得先问清楚：**27015 的 TCP 和 UDP、27020 的 UDP 能不能放行**，以及**这台机器给不给 root 或 SSH、能不能自己装 Docker**。面板服在这两点上经常有限制，买完才发现装不了容器，退起来最费时间。第三件是续费价，首单价和续费价不一样是常态，按续费价算一年总花费才准。

规格对上了，剩下的就是挑机器：

👉 [按这份清单去雨云选一台能跑 CS2 服务端的机器](https://bit.ly/RaiNyun)
