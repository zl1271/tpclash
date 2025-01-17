# Transparent proxy tool for Clash

> 这是一个用于 Clash Premium 的透明代理辅助工具, 由于众所周知周知的原因(**手笨**)而创建的.

## 一、TPClash 是什么

TPClash 可以自动安装 Clash Premium, 并自动配置基于 TProxy/Tun 的透明代理; 透明代理同时支持 TCP 和 UDP 协议, 包括对 DNS 的自动配置和 ICMP 的劫持等.

**TPClash 的透明代理规则、日志配置、Dashboard(UI) 配置等全部从标准的 Clash Premium 配置文件内读取, 并完成自适应; TPClash 暂时不会创建自己的自定义
配置文件(减轻使用负担).**

**同时 TPClash 在终止后会清理自己创建的 iptables 规则和路由表(防止把机器搞没); 这种清理不会简单的执行 `iptables -F/-X`, 而是进行 "定点清除", 以防止误删用户规则.**

## 二、TPClash 使用

TPClash 只有一个二进制文件, 直接从 Release 页面下载二进制文件运行即可. TPClash 二进制内嵌入了目标平台的 Clash 二进制文件以及其他资源文件(All in one), 
启动后会自动释放, 所以无需再下载 Clash. 

**注意: TPClash 默认会读取 位于 `/etc/clash.yaml` 的 clash 配置文件, 如果 clash 配置文件在其他位置请自行修改.**

```sh
./tpclash -c /etc/clash.yaml
```

**根据使用模式不同, TPClash 对 Clash 配置文件有不同的要求; 目前默认为 TUN 模式(对宿主机的 docker 等兼容性比较好), 可以通过 `-m` 参数自行切换.**

### 2.1、TProxy 模式配置

```yaml
# 需要开启 tproxy 端口
tproxy-port: 7893

# 请指定自己实际的借口名称
interface-name: ens160

# 开启 DNS 配置, 且使用 fake-ip 模式
# DNS 监听地址至少保证 127.0.0.1 可达
dns:
  enable: true
  listen: 0.0.0.0:1053
  enhanced-mode: fake-ip
  default-nameserver:
    - 114.114.114.114
  nameserver:
    - 114.114.114.114
```

### 2.2、TUN 模式配置

```yaml
# 请指定自己实际的借口名称
interface-name: ens160

# 需要开启 TUN 配置
tun:
  enable: true
  stack: system # or gvisor
  dns-hijack:
    - any:53
  #   - 8.8.8.8:53
  #   - tcp://8.8.8.8:53

# 需要指定全局 routing-mark(值可更改)
routing-mark: 666

# 开启 DNS 配置, 且使用 fake-ip 模式
# DNS 监听地址至少保证 127.0.0.1 可达
dns:
  enable: true
  listen: 0.0.0.0:1053
  enhanced-mode: fake-ip
  default-nameserver:
    - 114.114.114.114
  nameserver:
    - 114.114.114.114
```

### 2.3、启动 TPClash

**初次使用的用户推荐命令行执行并增加 `--test` 参数, 该参数保证 TPClash 在启动 5 分钟后自动退出, 如果出现断网等情况也能自行恢复. TPClash 支持的所有命令可以通过 `--help` 查看:**

```sh
root@test62 ~ # ❯❯❯ ./tpclash --help
Transparent proxy tool for Clash

Usage:
  tpclash [flags]

Flags:
      --clash-user string     clash runtime user (default "tpclash")
  -c, --config string         clash config path (default "/etc/clash.yaml")
      --debug                 enable debug log
      --direct-group string   skip tproxy group (default "tpdirect")
      --disable-extract       disable extract files
  -h, --help                  help for tpclash
      --hijack-dns strings    hijack the target DNS address (default "0.0.0.0/0")
      --hijack-ip ipSlice     hijack target IP traffic (default [])
  -d, --home string           clash home dir (default "/data/clash")
      --local-proxy           enable local proxy (default true)
  -m, --proxy-mode string     clash proxy mode(tproxy|tun) (default "tun")
      --test                  run in test mode, exit automatically after 5 minutes
      --tproxy-mark string    tproxy mark (default "666")
  -u, --ui string             clash dashboard(official|yacd|meta) (default "yacd")
  -v, --version               version for tpclash
```

### 2.4、Meta 用户

从 `v0.0.16` 版本开始支持 Clash Meta 分支版本, Meta 用户**需要在配置文件中关闭 iptables 配置**:

```yaml
iptables:
  enable: false # default is false
  inbound-interface: eth0 # detect the inbound interface, default is 'lo'
```

此外, Meta 用户如果期望使用 Meta 专用的 Dashboard 可以通过 `--ui meta` 选项指定.

### 2.5、自动流量接管

从 `v0.0.13` 版本起, TPClash 内置了一个 ARP 流量劫持功能, 可以通过 `--hijack-ip` 选项指定需要劫持的 IP 地址:

```sh
# 可以指定多次
./tpclash --hijack-ip 172.16.11.92 --hijack-ip 172.16.11.93
```

当该选项被设置后, TPClash 将会对目标 IP 发起 ARP 攻击, 从而强制接管目标地址的流量. 需要注意的是, 当目标 IP 被设置为 `0.0.0.0`
时, TPClash 将会劫持所有内网流量, 这可能会因为配置错误导致整体断网, 所以请谨慎操作.

## 三、TPClash 做了什么

**TPClash 在启动后会进行如下动作:**

- 1、创建 `/data/clash` 目录, 并将其作为 Clash 的 `Home Dir`
- 2、将 Clash Premium 二进制文件、Dashboard(官方+yacd)、必要的 ruleset、Country.mmdb 释放到 `/data/clash` 目录
- 3、创建 `tpclash` 普通用户用于启动 clash, 该用户用于配合 iptables 进行流量过滤
- 4、添加透明代理的路由表和 iptables 配置
- 5、启动官方的 Clash Premium, 并设置必要参数, 比如 `-ext-ui`、`-d` 等

## 四、如何编译 TPClash

由于 TPClash 是一个集成工具, 所以在编译前请安装好以下工具链:

- git
- curl
- jq
- tar
- gzip
- nodejs(用于编译 Dashboard)
- pnpm、yarn(Dashboard 编译所需依赖工具, 可通过 `npm i -g xxx` 安装)
- golang 1.17+
- [go-task](https://github.com/go-task/task)(类似 Makefile 的替代工具)

TPClash 项目内的 `Taskfile.yaml` 内已经写好了自动编译脚本, 只需要执行 `task` 命令即可:

```sh
git clone https://github.com/mritd/tpclash.git
cd tpclash
task # go-task 安装成功后会包含此命令
```

## 五、其他说明

TPClash 默认释放的文件包含了 [Loyalsoldier/clash-rules](https://github.com/Loyalsoldier/clash-rules) 相关文件, 可在规则中直接使用;

**TPClash 同时也释放了 [Hackl0us/GeoIP2-CN](https://github.com/Hackl0us/GeoIP2-CN) 项目的 Country.mmdb 文件, 该 GeoIP 数据库
仅包含中国大陆地区 IP, 所以如果使用 `GEOIP, US, PROXY` 等其他国家规则会失败.**
