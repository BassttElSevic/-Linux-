# 特别篇，给小Ling的 ThinkPad T42 无线网卡调试日志

> **场景**：ThinkPad T42（i386 32 位老机器）+ 插入的 USB 无线网卡（RTL8188EU），Debian 12 bookworm。重装系统、重新编译外置驱动后，能扫到 WiFi，但一连接就报**"密码错误"**。
> **结论先行**：**驱动、编译、安装、加载全是好的。** 真凶是 T42 内置的 Cisco Aironet 老卡（`airo` 驱动）——它硬件上只支持 WEP/LEAP，做不了 WPA2。NetworkManager 把连接错误地激活到了这张卡上，它一失败就只会被 NM 显示成"密码错误"并反复弹窗。
> **状态**：已定位，已修复。
> **本日志定位**：帮小Ling 复盘这次排查思路，也留作下次遇到"老笔记本 WiFi 报密码错误"的通用排查手册。

---

## 1. 环境信息（复现此问题的系统配置）

| 项目 | 值 | 说明 |
| --- | --- | --- |
| 机器 | ThinkPad T42 | 23 年前的 i386 / 32 位老本 |
| 发行版 | Debian GNU/Linux 12 (bookworm) | `admin1@IBMThinkpad`，主机名 IBMThinkpad |
| **内核 ABI** | `6.1.0-52-686` | **注意：不是 `6.1.0-13-686`！** |
| 无线网卡 | USB RTL8188EU 小网卡 | 接口 `wlx000f00405b53` |
| 外置驱动源码 | `/opt/rtl8188eus`（aircrack-ng/rtl8188eus） | 带 `dkms.conf` / `dkms-install.sh` |
| 内置老卡 | Cisco Aironet（`airo` 驱动） | `enp2s2` / `wifi0` |
| 有线网口 | `enp2s1`（`e1000` 驱动） | 排查时曾用于 SSH |
| 网络管理 | NetworkManager + nmtui | 决定一切表象的地方 |

> ⚠️ 一个关键前提：**内核 ABI 已经变了。** 最初为 `6.1.0-13-686`，重装后跳到 `6.1.0-52-686`。这直接决定了"离线 headers 包还有没有用"。

---

## 2. 问题现象

- USB 网卡**能扫到**附近的 WiFi（`YZW_1`、`yzp_2`、`HUAWEI-888888`… 全都在）
- 输入正确密码后，连接**失败**，NetworkManager 报"密码错误"
- 反复弹密码输入框，重试依然失败
- dmesg 里疯狂刷屏：`airo(enp2s2): association failed (reason: 255)`

**注意点**：能扫到信号 ≠ 关联成功。扫到说明驱动转发扫描帧没问题；连不上是**关联/握手**这层挂了，和"密码对不对"完全不是一回事。

---

## 3. 踩过的弯路（反面教材）

### 3.1 以为是驱动没编对 → 白折腾编译

第一次排查把所有注意力放在"是不是外置驱动没编译好"上，反复 `sudo make`、`make install`、`modprobe`。**结果编译链从头到尾都是好的**，这条路完全浪费了。

> **教训**：`make` 成功 ≠ 模块被加载 ≠ 加载的是它 ≠ 问题在驱动。问题可能在"另一张卡抢着干活"。

### 3.2 依赖一个"只对单个内核 ABI 有效"的离线 headers 包

当时别人给的方案是：T42 的旧 ABI `6.1.0-13-686` 已被 Debian 官方仓库下架，必须用 snapshot.debian.org 的离线 deb 包才能编译。这解决了当时的问题，但**重装后内核跳成了 `6.1.0-52-686`，那份离线 headers 就彻底作废了**。

> 好在 `6.1.0-52` 是当前 ABI，仓库里有，直接 `apt install linux-headers-$(uname -r)` 就行。所以"旧脚本 apt 装不到 headers"的根因，在重装后**已经不存在**。

### 3.3 用 `nmcli device set enp2s2 managed no` 屏蔽老卡 —— 临时生效，重启就丢

这本来是**正确的方向**（就是要去掉老卡），但用错方法：

```bash
nmcli device set enp2s2 managed no   # 运行时设置，不写任何配置文件
```

这是运行时 D-Bus 设置，**不落盘**。重装当然没了，就算不重装，**一重启也会失效**。

> **教训**：要让 NetworkManager 持久化忽略某设备，必须写配置文件里的 `unmanaged-devices`，不是 `nmcli device set`。

### 3.4 抓"随机 MAC"写进屏蔽名单 —— 抓到的是快照，一次就失效

朋友给的 `block.conf` 用了 `mac:BE:FF:48:30:25:6F;mac:5E:69:16:45:46:35`，但这两个 MAC 都是**扫描时的随机 MAC**（见第 5 节），下一轮扫描就换成别的了。所以它**实际上没生效**——nmtui 里 `Wi-Fi (enp2s2)` 依然出现。

---

## 4. 诊断命令（按顺序执行，附输出解读）

### 4.1 摸清网络设备全家福

```bash
ip -brief addr
# 3: wlx000f00405b53  UP  inet 192.168.0.104/24 ...   <- USB WiFi，有 IP，正常
# 2: enp2s1          DOWN NO-CARRIER  inet 192.168.0.102/24 ...  <- 有线，网线拔了（租约未过期）
# 4: enp2s2          DOWN ... mac 1e:ba:cc:1f:b3:a1  permaddr 00:0e:9b:93:c8:6b  <- Aironet
# 5: wifi0           DOWN ... mac 1e:ba:cc:1f:b3:a1  permaddr 00:0e:9b:93:c8:6b  <- 同一张 Aironet
```

> `enp2s2` 和 `wifi0` 是**同一张物理卡**暴露的两个接口（`permaddr` 相同 `00:0e:9b:93:c8:6b`）。

### 4.2 看 dmesg —— 决定性的证据

```bash
dmesg | grep -iE 'rtl|8188|wlx|airo|wlan|usb'
```

**关键输出 1：你编的 USB 驱动是好的**

```
==> rtl8188e_iol_efuse_patch                                   <- RTL8188EU 初始化特征
IPv6: ADDRCONF(NETDEV_CHANGE): wlx000f00405b53: link becomes ready  <- 接口起来了
```

`rtl8188e_iol_efuse_patch` 是 RTL8188EU 驱动加载时特有的打印，**说明驱动加载成功、接口 ready**。编译/安装/加载这条链没问题。

**关键输出 2：老卡在疯狂自残**

```
airo(enp2s2): association failed (reason: 255)   <- 85 分钟内出现 2129 次！
airo(enp2s2): Bad MAC enable reason=88, rid=ff10, offset=49392
airo(enp2s2): cmd:1 status:7f01 rsp0:88 ...
```

`air_o` 驱动在反复尝试关联并全部失败，`Bad MAC enable` 说明它的 MAC 层根本起不来。这张卡只支持 WEP/LEAP，永远连不上现代 WPA2 网络。

### 4.3 数一下失败次数、看时间跨度（判断严重性）

```bash
dmesg | grep -c 'airo.*association failed'
# 2129
```

> 这个数字很重要：**它不仅说明老卡一直在捣乱，还意味着它把内核环形缓冲区挤爆了**——开机时的模块加载、USB 枚举记录全被这 2129 条消息推出去，日志里看不到任何模块加载信息。

### 4.4 看 NetworkManager 到底激活到了哪个接口

```bash
journalctl -b -u NetworkManager | grep -iE 'enp2s2|wifi0|wlx|secrets|WRONG_KEY' | tail -30
```

如果你看到连接被激活到 `enp2s2`，或者一堆 `WRONG_KEY` / 握手失败，基本就能钉死方向。

---

## 5. 根因分析

### 5.1 真正干活的驱动没问题，"密码错误"是假象

完整因果链：

```
用户输入正确密码
→ NetworkManager 找可用的无线设备
→ THINKPAD 上存在两块无线设备：USB 卡(wlx...) + 内置 Aironet(enp2s2/wifi0)
→ NM 把连接激活到了 Aironet 上（它也在扫描列表里）
→ Aironet 硬件只支持 WEP/LEAP，无法完成 WPA2/WPA 四次握手
→ airo 驱动反复 association failed（2139 次）
→ NM 无法区分"密码错"和"驱动做不了这加密"，一律显示"密码错误"并反复弹窗
```

**所以答案从一开始就是：驱动没毛病，是 NM 选错了卡。**

### 5.2 为什么 MAC 那个写法失效（`block.conf` 的坑）

看 `ip a` 里 Aironet 的两个地址：

```
permaddr 00:0e:9b:93:c8:6b    <- 烧死的真实 MAC（首字节 0x00，全球唯一）
mac      1e:ba:cc:1f:b3:a1    <- 当前 MAC（首字节 0x1E，本地管理地址 = 随机）
```

判断"是否随机 MAC"看**首字节第 2 个十六进制位的最后一位**（二进制第 2 位，LAA 位）：

| MAC | 首字节 | 二进制 | LAA 位 | 含义 |
| --- | --- | --- | --- | --- |
| `00:0e:9b:...`（permaddr） | `0x00` | `0000 0000` | **0** | 全球唯一（真实永久地址） |
| `1e:ba:cc:...`（当前） | `0x1E` | `0001 1110` | **1** | 本地管理地址（随机） |
| `BE:FF:...`（block.conf） | `0xBE` | `1011 1110` | **1** | 本地管理地址（随机） |
| `5E:69:...`（block.conf） | `0x5E` | `0101 1110` | **1** | 本地管理地址（随机） |

> NetworkManager 的 `wifi.scan-rand-mac-address` 默认是 **yes**，扫描时会临时给网卡换随机 MAC。Aironet 一直连不上、一直在扫描，所以它每轮扫描 MAC 都在变。朋友写进 `block.conf` 的两个随机 MAC 只是**某一瞬间的快照**，下一轮就失效了——所以那些 `mac:` 规则根本没匹配到。

（对照：USB 卡 `wlx000f00405b53` 的 MAC `00:0f:00:40:5b:53` 首字节是 `0x00`，是真实地址。因为它已连接，不在扫描状态，NM 没随机化它。）

---

## 6. 修复步骤

### 6.1 彻底禁掉只会 WEP 的 Aironet 老卡（推荐）

```bash
sudo tee /etc/modprobe.d/blacklist-airo.conf >/dev/null <<'EOF'
blacklist airo
blacklist airo_cs
EOF
sudo modprobe -r airo
```

> 直接从模块层面禁掉，`enp2s2` / `wifi0` 根本不会出现，nmtui 里只剩一个 WiFi 设备，以后不可能再点错。

### 6.2 顺手屏蔽内核自带的 staging `r8188eu`，避免和编的 `8188eu` 抢设备

```bash
sudo tee /etc/modprobe.d/blacklist-r8188eu.conf >/dev/null <<'EOF'
blacklist r8188eu
EOF
```

> Debian 内核自带一个 `r8188eu`（DRIVER 名带 "r"），和 aircrack-ng 的 `8188eu`（无 "r"）是两个不同模块，可能同时想绑同一张 USB 卡。blacklist 掉自带的，确保用的是你编的那个。

### 6.3 NetworkManager 层面持久化屏蔽（用配置文件，不用 `nmcli device set`）

```bash
sudo tee /etc/NetworkManager/conf.d/99-unmanage-airo.conf >/dev/null <<'EOF'
[keyfile]
unmanaged-devices=interface-name:enp2s2;interface-name:wifi0
EOF
```

> ⚠️ 这里用的是 **`interface-name:`**（接口名，稳定不变），不是 **`mac:`**（会因 MAC 随机化而变）。这正是上面 5.2 踩过的坑。**不要用 `nmcli device set ... managed no`**，那玩意儿重启就丢。

### 6.4 重启并对账

```bash
sudo systemctl restart NetworkManager
sleep 8
nmcli device status
ip -brief addr
```

期望结果：

- `enp2s2` / `wifi0` 消失，或显示 `unmanaged`
- 只剩 `enp2s1`（有线）、`wlx000f00405b53`（USB WiFi）
- USB 卡拿到 `192.168.0.104/24`，`UP`

**还原方法**（万一有问题）：

```bash
sudo rm /etc/modprobe.d/blacklist-airo.conf \
        /etc/modprobe.d/blacklist-r8188eu.conf \
        /etc/NetworkManager/conf.d/99-unmanage-airo.conf
sudo systemctl restart NetworkManager
```

### 6.5 强烈建议：改用 DKMS 装驱动

`make install` 把模块放进了 `/lib/modules/6.1.0-52-686/kernel/drivers/net/wireless/` —— 这是**发行版内核自己的目录**，下次内核一更新（`-52` → `-53`）就会被新内核包覆盖，驱动又没了。而内核已经从 `-13` 跳到 `-52` 一次了，还会再跳。

`/opt/rtl8188eus` 里现成有 DKMS 支持：

```bash
cd /opt/rtl8188eus
sudo ./dkms-install.sh
dkms status
```

配好后内核每次升级**自动重编**，不会再出现"某天忽然又没网了"。

---

## 7. 避坑清单（分享给别人的重点）

### 7.1 `sudo make` 成功 ≠ 模块被加载

`make` 只在源码目录生成 `8188eu.ko`；真正生效还要 `make install` + `/sbin/depmod -a` + `modprobe`。排查时先确认这三个都做了，再谈"驱动是不是有问题"。

### 7.2 内核 ABI 会变，离线 headers 会作废

`6.1.0-13-686` 曾被 Debian 下架（只剩 snapshot.debian.org），但重装后内核跳到 `6.1.0-52-686`，仓库里又有这个 ABI 的 headers 了。**遇到"apt 装不上 headers"先 `uname -r` 看当前 ABI 是否还在仓库，再决定是离线包还是直接从仓库装。**

### 7.3 `nmcli device set ... managed no` 不持久化

它是运行时 D-Bus 设置，不写盘，重装或重启都会失效。要持久化必须写 `/etc/NetworkManager/conf.d/*.conf` 的 `unmanaged-devices`。

### 7.4 屏蔽设备优先用 `interface-name:`，别用 `mac:`

MAC 随机化（`wifi.scan-rand-mac-address=yes`）会让网卡 MAC 反复变，抄下来的随机 MAC 一扫描就失效。接口名（`enp2s2`/`wifi0`）才是稳定不变的。

### 7.5 "能扫到信号" ≠ "能连上"

扫描成功只说明驱动能发探测帧/收响应；关联、四次握手是更高层的事。老卡往往能扫到却连不上现代 WPA2/AES。

### 7.6 老式无线卡的硬件限制

老笔记本内置卡（Aironet、部分早期 Intel）只支持 WEP/LEAP，**永远做不了 WPA2/WPA3**。这是硬件能力，不是配置能解决的问题，只能屏蔽或弃用。

### 7.7 dmesg 被刷屏会冲垮日志

一条报错刷两千次，会把开机时的模块加载、USB 枚举等关键记录**全部挤出环形缓冲区**，排查时你会看到"日志里什么都没有"。这也是为什么要先禁掉刷屏源。

---

## 8. 通用判断流程（下次 5 分钟定位）

```
老笔记本 WiFi 报"密码错误"
│
├─ dmesg 里 grep 无线相关：有没有别家驱动在疯狂报错？
│   ├─ 有（如 airo/某老卡 association failed 刷屏）
│   │   └─ 是"另一张卡抢着干活"，不是密码 → 屏蔽那张卡（黑名单 / unmanaged）
│   └─ 没有 → 往下
│
├─ ip -brief addr：有几块无线设备？哪块拿到了 IP？
│   ├─ 拿到 IP 的 = 真正在工作（比如 wlx...）
│   └─ 没拿到 IP 却也在扫描的 = 可能是干扰源
│
├─ journalctl -u NetworkManager | grep -E 'enp2s2|wlx|WRONG_KEY'
│   ├─ 激活到了老卡 → 把它 blacklist/unmanaged
│   └─ 激活到了正确的 USB 卡但仍失败 → 查握手/驱动能力
│
├─ 确认驱动是否真的被加载（不是只编译了）
│   ├─ lsmod | grep 8188     有 → 加载了
│   └─ modinfo 8188eu | grep filename → 看是不是你编的那个
│
└─ 都可能没关系时：手动 wpa_supplicant -D nl80211/-D wext 排除密码变量
```

---

## 9. 参考链接

- aircrack-ng/rtl8188eus（外置驱动源）：<https://github.com/aircrack-ng/rtl8188eus>
- Debian snapshot（历史 ABI headers）：<https://snapshot.debian.org/package/linux/6.1.55-1/>
- NetworkManager 配置文件参考（`man 5 NetworkManager.conf` 里的 `unmanaged-devices` / `device*.managed`）
- `airo` 驱动（Cisco Aironet，仅 WEP/LEAP）：<https://wiki.debian.org/airo>

---

*记录时间：2026-08-29 · 环境：ThinkPad T42 / Debian 12 bookworm / 内核 6.1.0-52-686 / aircrack-ng rtl8188eus*
*结论一句话：**老笔记本连不上 WiFi 报"密码错误"，先查是不是内置老卡在抢连接——多数时候密码根本没输错，是 NM 把连接给了那张只会 WEP 的卡。***

<img width="500" height="500" alt="bigbasstt-removebg-preview" src="https://github.com/user-attachments/assets/4d11970a-2c7c-4c6b-9322-d97c59bb84f4" />
