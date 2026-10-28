# 实体机（真物理机）Linux 忘密码？两种方法重置 root

> 适用于：装在**真实电脑/物理机/裸机**上的 Linux（Ubuntu / Debian / CentOS / Rocky 等都行），不是虚拟机。
>
> 和虚拟机的区别：实体机没有 `.vmx` 文件，也没 vmrun/VNC 那套自动化，要手动操作；但好处是**通常能看到 GRUB 菜单**，反而有一条更快的捷径。
>
> 按你的情况选：
> - **开机能看到 GRUB 菜单** → 方法一：单用户模式，最快，不用任何 U 盘；
> - **看不到 GRUB（直接进系统）** → 方法二：U 盘应急盘，和虚拟机那套原理一样。

---

## 动手前，先搞清楚一件事：root 可能根本没开

很多 Linux 系统（尤其 **Ubuntu 24.04 用 autoinstall / cloud-init 装出来的**），**root 默认是锁定的**——压根没设密码，日常靠一个带 sudo 的普通用户管理。

所以动手前想清楚要改**哪个账号**：

| 你的情况 | 该改谁 | 说明 |
|---|---|---|
| 忘了 root 密码，想直接用 root 登录 | root | root 可能被锁、甚至不存在，下面会讲判断 |
| 忘了普通用户的密码 | 那个普通用户 | 通常一定存在，改它最简单，改完 `sudo -i` 照样拿 root 权限 |

判断 root 状态：看 `/etc/shadow` 里 root 那行第二个字段——`!`/`*` 是锁着、`$6$...`/`$y$...` 是有密码、没有这一行是不存在。

---

## 方法一：单用户模式（能看到 GRUB 时，最快）

> 原理：让内核以 `init=/bin/bash` 或 single 模式启动，直接进一个 root shell，跳过登录，改密码。全程不用 U 盘。

### 第 1 步：重启，在 GRUB 菜单停下

开机/重启时，一出现 GRUB 菜单就**快速按方向键（↑/↓）或 Esc**，让倒计时停住，选中你要进的那一项（通常是第一个「Ubuntu」）。

> 如果开机一闪而过、来不及按：开机后立刻**狂按 Esc 或 Shift**，有的机器要长按 Shift 才出 GRUB。

### 第 2 步：编辑启动项，进入单用户

在选中的启动项上按 **`e`** 键（编辑）。找到以 `linux` 开头的那一行（很长，末尾有 `quiet splash` 之类）：

1. 把光标移到这行末尾
2. 删除 `quiet splash`（可选，只为看清输出）
3. 在这行末尾加上 `init=/bin/bash`（或者某些发行版用 `single` / `rd.break`，见下方对照）

### 第 3 步：按 `Ctrl+X`（或 `F10`）启动

系统会启动并**直接进入一个 root shell**，提示符类似 `root@(none):/#`，无需密码。

### 第 4 步：重新挂载根分区 + 改密码

这时根分区是**只读**的，先重新挂成可写，再改密码：

```bash
# 重新挂载根分区为可写
mount -o remount,rw /

# 改 root 密码（交互式，输两遍新密码）
passwd root
```

> 如果 `passwd root` 报「用户不存在」，先补建：
> ```bash
> useradd -m -s /bin/bash root
> passwd root
> ```

改完重启：

```bash
exec /sbin/init        # 正常继续启动
# 或者直接
reboot -f
```

### 各发行版的启动参数对照

| 发行版 | 加的参数 | 说明 |
|---|---|---|
| Debian / Ubuntu（旧版，sysvinit） | `single` | 进单用户模式，可能提示输入 root 密码 |
| Ubuntu（新版，systemd） | `init=/bin/bash` | 直接进 root shell，最稳 |
| CentOS 7 / RHEL 7 | `rd.break` | 停在内核挂载阶段，需手动 `mount -o remount,rw /sysroot` 再 `chroot /sysroot` |
| CentOS 8+ / Rocky / Alma | `rd.break` | 同上 |
| Arch / 大部分 systemd 发行版 | `init=/bin/bash` | 直接进 root shell |

> **CentOS/RHEL 用 `rd.break` 的额外步骤**（因为停在了 `/sysroot`）：
> ```bash
> mount -o remount,rw /sysroot
> chroot /sysroot
> passwd root
> touch /.autorelabel        # SELinux 系统必须执行，否则改完可能起不来
> exit
> reboot -f
> ```

---

## 方法二：U 盘应急盘（看不到 GRUB，或方法一失败时）

> 原理和虚拟机一模一样：用 SystemRescue 从 U 盘启动，挂载硬盘改密码。区别只是「刻 U 盘」替代「挂 ISO」。

### 第 1 步：下载 SystemRescue

```
https://www.system-rescue.org/download/
```

得到 `systemrescue-13.02-amd64.iso`。

### 第 2 步：刻录到 U 盘

用 **Rufus**（Windows）或 **balenaEtcher**（跨平台）把 ISO 写进 U 盘（会清空 U 盘数据，注意备份）：

- Rufus：选 U 盘 → 选 ISO → 分区类型选 **GPT 或 MBR**（一般默认即可）→ 开始
- balenaEtcher：选 ISO → 选 U 盘 → Flash

### 第 3 步：从 U 盘启动

插入 U 盘 → 开机 → 进 **BIOS/UEFI 引导菜单**（常用键：`F2`/`F12`/`F11`/`Del`/`Esc`，看开机画面提示）→ 选 U 盘启动（名字里带 `USB` 或你的 U 盘品牌名）。

### 第 4 步：挂载硬盘 + 改密码

进入 SystemRescue 的命令行（自动 root 登录），然后：

```bash
# 找根分区（一般 sda2 或 nvme0n1p2，用下面命令看）
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT

# 挂载根分区
mount /dev/sda2 /mnt

# 先看一眼 root 状态（对应开头）
grep '^root:' /mnt/etc/shadow

# 改密码（自动解锁+设密码）
echo 'root:新密码' | chroot /mnt chpasswd

# 验证
grep '^root:' /mnt/etc/shadow
```

> 如果 `grep '^root:'` 没输出，说明 root 不存在，先补建：
> ```bash
> chroot /mnt useradd -m -s /bin/bash root
> echo 'root:新密码' | chroot /mnt chpasswd
> ```

收尾：

```bash
umount /mnt
poweroff
```

拔掉 U 盘，正常开机即可。

### 特殊磁盘：LVM / LUKS 加密盘

如果 `lsblk` 看到的是 `lvm` 或 `crypto_LUKS`，需要多一步：

```bash
# LVM：激活卷组后挂载
vgchange -ay
mount /dev/mapper/<卷组名>-root /mnt

# LUKS 加密：先解密（会提示输入加密密码）
cryptsetup open /dev/sda2 rootfs
mount /dev/mapper/rootfs /mnt
```

> ⚠️ LUKS 加密盘需要**加密密码**才能解开，没这个密码就无解（这正是加密的意义）。

---

## 改完密码，SSH 可能还是登不上

和虚拟机一样，**root 密码对了 ≠ SSH 能登**。Ubuntu 默认 `PermitRootLogin prohibit-password`（禁止 root 用密码 SSH）。改完密码后，如果还想 root 直接 SSH 登录：

```bash
sudo sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
sudo sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
sudo systemctl restart ssh        # CentOS/RHEL 用 sshd
```

---

## 常见坑

| 坑 | 原因 | 解决办法 |
|---|---|---|
| 抢不到 GRUB | 开机太快，或 GRUB 被隐藏 | 狂按 Esc/Shift，或干脆用 U 盘方案 |
| 改完密码进不去 | root 默认被锁 | 用 `passwd root` / `chpasswd` 会自动解锁 |
| CentOS 改完起不来 | SELinux 没重标记 | `rd.break` 流程里必须 `touch /.autorelabel` |
| U 盘启动没反应 | BIOS 没选 U 盘 / 安全启动挡住 | 进 BIOS 关 Secure Boot、选 U 盘优先 |
| 改完 SSH 登不上 | 默认禁止 root 密码 SSH | 改 `PermitRootLogin yes` 并重启 ssh |

---

## 安全提醒

- 只用于**你自己忘了密码的机器**，别用在别人的机器上。
- 改完建议设强密码。
- 别把真实密码提交到公开仓库。
