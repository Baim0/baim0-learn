# ifcfg 备份文件会被 network 服务当配置执行（机制分析 + 源码）

- **记录日期**：2026-09-24
- **结论**：在 CentOS/RHEL 7 系（老式 initscripts）环境下，`/etc/sysconfig/network-scripts/` 里面的 ifcfg 备份文件**会被 network 服务枚举并真实执行**，只要满足两个条件：
  1. 文件名以 `ifcfg-` 开头、且不以 `~ / .bak / .old / .orig / .rpmnew / .rpmorig / .rpmsave` **结尾**（即 `ifcfg-eth0.bak0923`、`ifcfg-eth0.bakup0921` 这类"bak+日期"命名会**逃过过滤**）；
  2. 文件里写了 `DEVICE=eth0`（有它，配置就作用到真实 eth0；没有它才会被跳过）。
- **影响**：地址按"累加"方式生效、默认路由"后者覆盖前者"，最终状态取决于文件执行顺序，**行为不可预期**。

## 1. 完整执行链条（4 步）

| 步骤 | 位置 | 行为 |
|---|---|---|
| 1. 枚举 + 过滤 | `network` 服务脚本 | `ls ifcfg-*`，用正则剔除"忽略文件"——**只匹配结尾**的 `.bak/.old/...`，`bak+日期` 命名逃过过滤 |
| 2. 取 DEVICE、查 ONBOOT | `network` 服务脚本 | 从每个文件里 `grep DEVICE=`；DEVICE 为空才回退用文件名；`ONBOOT=yes` 则拉起 |
| 3. 执行 ifup | `network` 服务脚本 | `./ifup eth0.bak0923 boot` |
| 4. 作用对象由文件内 DEVICE 决定 | `ifup` / `network-functions` / `ifup-eth` | 载入备份文件全部变量后，`REALDEVICE=${DEVICE%%:*}` = `eth0`，后续加地址/装路由全部落在 eth0 上 |

## 2. 源码证据（逐段）

> 源码取自 [fedora-sysv/initscripts](https://github.com/fedora-sysv/initscripts)（main 分支，与 CentOS/RHEL 7 系 initscripts 同源，行号可能有小差异）。
> 本目录 `initscripts-source/` 下有 5 个文件的完整副本，可离线核对。

### 2.1 枚举 ifcfg-*（`etc/rc.d/init.d/network`，第 42-49 行）

```sh
# find all the interfaces besides loopback.
# ignore aliases, alternative configurations, and editor backup files
interfaces=$(ls ifcfg-* | \
        LC_ALL=C sed -e "$__sed_discard_ignored_files" \
               -e '/\(ifcfg-lo$\|:\|ifcfg-.*-range\)/d' \
               -e '{ s/^ifcfg-//g;s/[0-9]/ &/}' | \
        LC_ALL=C sort -k 1,1 -k 2n | \
        LC_ALL=C sed 's/ //')
```

注释里写明"ignore … editor backup files"——**意图是忽略备份文件**，但实际过滤规则有漏洞（见 2.2）。

### 2.2 过滤规则：只匹配"结尾"（`etc/rc.d/init.d/functions`，第 609-614、678-679 行）

```sh
# Copy of the function is present in usr/sbin/service
is_ignored_file() {
    case "$1" in
    *~ | *.bak | *.old | *.orig | *.rpmnew | *.rpmorig | *.rpmsave)
        return 0
        ;;
    esac
```

```sh
# A sed expression to filter out the files that is_ignored_file recognizes
__sed_discard_ignored_files='/\(~\|\.bak\|\.old\|\.orig\|\.rpmnew\|\.rpmorig\|\.rpmsave\)$/d'
```

> 正则以 `$` 结尾锚定：
> - `ifcfg-eth0.bak` → 结尾是 `.bak` → **被忽略**（正确行为）
> - `ifcfg-eth0.bak0923` → 结尾是 `0923` → **不被忽略**，进入后续流程
> - `ifcfg-eth0.bakup0921` → 结尾是 `0921` → **不被忽略**
> - `ifcfg-ens18.bakup0921` → 同样不被忽略（但 ens18 不存在，后续会被跳过，见 2.5）

### 2.3 DEVICE 取自文件内容 + ONBOOT 检查 + 拉起（`etc/rc.d/init.d/network`，第 91-134 行，节选）

```sh
    # bring up all other interfaces configured to come up at boot time
    for i in $interfaces; do
        unset DEVICE TYPE SLAVE NM_CONTROLLED
        eval $(LANG=C grep -F "DEVICE=" ifcfg-$i)
        eval $(LANG=C grep -F "TYPE=" ifcfg-$i)
        eval $(LANG=C grep -F "SLAVE=" ifcfg-$i)
        eval $(LANG=C grep -F "NM_CONTROLLED=" ifcfg-$i)

        if [ -z "$DEVICE" ] ; then DEVICE="$i"; fi
        ...
        if ( . ./ifcfg-"$i" ; is_false "$ONBOOT" ) ; then
            # this loads the module, to preserve ordering
            is_available $i
            continue
        fi
        action $"Bringing up interface $i: " ./ifup $i boot
        [ $? -ne 0 ] && rc=1
    done
```

> 备份文件里 `DEVICE=eth0`、`ONBOOT="yes"` → 走到 `./ifup eth0.bak0923 boot`。

### 2.4 作用对象 = 文件里的 DEVICE（`network-scripts/ifup` 第 27、53、168 行；`network-scripts/network-functions` 第 115-120、173-175 行）

```sh
# ifup
CONFIG=${1}
...
source_config
```

```sh
# network-functions
source_config ()
{
    CONFIG=${CONFIG##*/}
    DEVNAME=${CONFIG##ifcfg-}
    . /etc/sysconfig/network-scripts/$CONFIG
    [ -r "keys-$DEVNAME" ] && . /etc/sysconfig/network-scripts/keys-$DEVNAME
    ...
    [ -z "$REALDEVICE" -a -n "$PARENTDEVICE" ] && REALDEVICE=$PARENTDEVICE
    [ -z "$REALDEVICE" ] && REALDEVICE=${DEVICE%%:*}
```

> `ifup eth0.bak0923` 载入的是 `ifcfg-eth0.bak0923` 的全部变量（`DEVICE=eth0`、`IPADDR=…`、`GATEWAY=…` 全部生效），然后真正的操作对象 `REALDEVICE` = **eth0**。

```sh
# ifup 末尾把控制权交给 ifup-eth
exec ${OTHERSCRIPT} ${CONFIG} $2
```

### 2.5 "真实执行"还是"被跳过"的分界（`network-scripts/ifup-eth`）

**有 `DEVICE=eth0` → 真实执行，操作落在 REALDEVICE 上（第 329 行起）：**

```sh
            # set lifetime of address to forever
            ip addr change ${ipaddr[$idx]}/${prefix[$idx]} dev ${REALDEVICE} valid_lft forever preferred_lft forever
```

**没有 `DEVICE=` → DEVICE 回退为文件名派生的 `eth0.bak0923` → 设备不存在，报错并跳过（第 132-139 行）：**

```sh
# now check the real state
is_available_wait ${REALDEVICE} ${DEVTIMEOUT} || {
    if [ -n "$alias" ]; then
        net_log $"$alias device ${DEVICE} does not seem to be present, delaying initialization."
    else
        net_log $"Device ${DEVICE} does not seem to be present, delaying initialization."
    fi
    exit 1
}
```

> **判读提醒**：日志里看到 `Bringing up interface eth0.bak0923:` 不代表它没生效，要看整行有没有以 `does not seem to be present, delaying initialization` 收尾——**有这句才是"被跳过"**。

### 2.6 为什么"执行顺序不可预期"（`network-scripts/ifup-eth` 第 332-340 行）

```sh
        # Set a default route.
        if ! is_false "${DEFROUTE}" && [ -z "${GATEWAYDEV}" -o "${GATEWAYDEV}" = "${REALDEVICE}" ]; then
            # set up default gateway. replace if one already exists
            if [ -n "${GATEWAY}" ] && [ "$(ipcalc --network ${GATEWAY} ${netmask[0]} 2>/dev/null)" = "NETWORK=${NETWORK}" ]; then
                ip route replace default ... via ${GATEWAY} ... ||
                    net_log $"Error adding default gateway ${GATEWAY} for ${DEVICE}."
```

- 地址：`ip addr change/add` → **累加**（多份文件的不同地址会都加上去）；
- 默认路由：`ip route replace` → **后者覆盖前者**；
- 多个文件按 `ls` 字母序依次执行 → 最终状态取决于内容与顺序，不可预期。

## 3. 在机器上实锤的方法

```bash
# 1) 本次启动/重启实际拉起了哪些"接口名"
grep -i "Bringing up interface" /var/log/messages | tail -30
journalctl -b -u network --no-pager | tail -80
# 判读：出现 eth0.bak0923 等名字且整行无 "does not seem to be present" → 真实执行了

# 2) 文件差异对比（谁和谁不一样、哪个"赢了"）
cd /etc/sysconfig/network-scripts
md5sum ifcfg-eth0 ifcfg-eth0.bak0923 ifcfg-eth0.bakup0921
grep -H -E "DEVICE|ONBOOT|IPADDR|PREFIX|GATEWAY" ifcfg-eth0*
```

## 4. 规避建议

- **备份文件不要放在 `/etc/sysconfig/network-scripts/` 里**；非要放，至少不要以 `ifcfg-` 开头（例如改成 `eth0_backup_0921`）。
- 如果用 `ifcfg-` 开头命名，只能靠"结尾"后缀才能被忽略：`ifcfg-eth0.bak` 可以，`ifcfg-eth0.bak0923` **不行**。
- 已存在的备份建议统一移走：

```bash
mkdir -p /root/backup-network-scripts
mv /etc/sysconfig/network-scripts/ifcfg-*bak* /root/backup-network-scripts/
ls /etc/sysconfig/network-scripts/ifcfg-*   # 预期只剩本体配置和 ifcfg-lo
```

**文件命名规则速查表：**

| 文件名 | 是否会被当设备配置执行 |
|---|---|
| `ifcfg-<设备名>`（如 `ifcfg-eth0`） | 是（设备本体配置） |
| `ifcfg-<设备名>:<n>`、`ifcfg-<设备名>-range<n>`、`ifcfg-<设备名>.<vlan数字>` | 是（别名/批量别名/VLAN，有专门逻辑） |
| 以 `~`、`.bak`、`.old`、`.orig`、`.rpmnew`、`.rpmorig`、`.rpmsave` **结尾** | 否（被过滤规则忽略） |
| `*.bak0923`、`*.bakup0921`、`*.bak_日期` 等"bak+后缀" | **是，会被枚举执行**（本次踩坑点） |
| 不以 `ifcfg-` 开头的任意命名 | 否 |

## 5. 参考资料

- [Does 'ifcfg-eth0.bak' file load at the boot time? — Red Hat 知识库（官方专题）](https://access.redhat.com/solutions/68127)
- [ifcfg-eth0.bak, bug? — VMware/Broadcom 社区（实测案例：备份文件被当作真实接口配置，新老 IP 同时生效）](https://community.broadcom.com/vmware-cloud-foundation/discussion/ifcfg-eth0bak-bug)
- [多网卡配置文件导致 network 启动失败处理 — 华为云](https://support.huaweicloud.com/trouble-ecs/ecs_trouble_0319.html)
- [启动网络服务时提示 Job for network.service failed… — 阿里云文档](https://help.aliyun.com/zh/ecs/user-guide/prompt-when-start-network-service-job-for-the-network-the-service-failed-because-the-control-process-exited-with-the-error-code-what-should-i-do)
- initscripts 源码（本目录 `initscripts-source/` 有完整副本）：
  - [etc/rc.d/init.d/network](https://github.com/fedora-sysv/initscripts/blob/main/etc/rc.d/init.d/network)
  - [network-scripts/ifup](https://github.com/fedora-sysv/initscripts/blob/main/network-scripts/ifup)
  - [network-scripts/ifup-eth](https://github.com/fedora-sysv/initscripts/blob/main/network-scripts/ifup-eth)
  - [network-scripts/network-functions](https://github.com/fedora-sysv/initscripts/blob/main/network-scripts/network-functions)
  - [etc/rc.d/init.d/functions](https://github.com/fedora-sysv/initscripts/blob/main/etc/rc.d/init.d/functions)
