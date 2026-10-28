# 实体机 Linux 忘密码 / 开 root

> 实体机（物理机）Linux 重置 root 密码。虚拟机请看 [vmware-root-reset](https://github.com/DaisySG297/vmware-root-reset)。

按情况选一种：

| 你的情况 | 方法 |
|---|---|
| 还能登录 | [方法一](#方法一还能登录直接开-root) `sudo passwd root` |
| 能看到 GRUB | [方法二](#方法二单用户模式) 改内核参数进单用户 |
| 看不到 GRUB | [方法三](#方法三u-盘应急盘) U 盘启动应急盘 |

> 提示：Ubuntu 24.04（autoinstall/cloud-init 装的）root 默认**被锁、没密码**，日常靠 sudo 用户管理。所以「改密码」通常 =「解锁 + 设密码」。

---

## 方法一：还能登录，直接开 root

```bash
sudo passwd root        # 输 sudo 密码，再给 root 设新密码（输两遍）
```

一条命令完成「解锁 + 设密码」。报「用户不存在」则先补建：

```bash
sudo useradd -m -s /bin/bash root && sudo passwd root
```

只是想要 root 权限、不一定要真开 root 账号，用 `sudo -i` 更省事也更安全。

---

## 方法二：单用户模式

重启，在 GRUB 菜单**快速按方向键/Esc**停住 → 选中启动项按 `e` 编辑 → 找到 `linux` 开头那行，末尾加上 `init=/bin/bash` → `Ctrl+X` 启动，直接进 root shell。

然后：

```bash
mount -o remount,rw /     # 根分区默认只读，先改成可写
passwd root               # 设新密码（报「用户不存在」就 useradd -m -s /bin/bash root 再 passwd）
exec /sbin/init           # 改完正常启动
```

各发行版要加的参数：

| 发行版 | 参数 |
|---|---|
| Ubuntu / Debian / Arch（systemd） | `init=/bin/bash` |
| CentOS 7 / RHEL 7 | `rd.break` |
| CentOS 8+ / Rocky / Alma | `rd.break` |

> CentOS/RHEL 用 `rd.break` 停在了 `/sysroot`，改法不同：
> ```bash
> mount -o remount,rw /sysroot
> chroot /sysroot
> passwd root
> touch /.autorelabel      # SELinux 必做，否则起不来
> reboot -f
> ```

---

## 方法三：U 盘应急盘

1. 下载 SystemRescue：https://www.system-rescue.org/download/
2. 用 **Rufus** 或 **balenaEtcher** 把 ISO 刻进 U 盘
3. U 盘启动（开机进 BIOS 引导菜单，键位看开机画面，选 USB 项）
4. 进入命令行后：

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT   # 找根分区（一般 sda2 / nvme0n1p2）
mount /dev/sda2 /mnt
echo 'root:新密码' | chroot /mnt chpasswd     # 自动解锁+设密码
grep '^root:' /mnt/etc/shadow                # 看到 $6$... / $y$... 即成功
umount /mnt
poweroff
```

> `grep '^root:'` 没输出 = root 不存在，先 `chroot /mnt useradd -m -s /bin/bash root` 再 chpasswd。

**LVM / LUKS 加密盘**多一步：

```bash
# LVM
vgchange -ay && mount /dev/mapper/<卷组>-root /mnt
# LUKS（需加密密码，没有则无解）
cryptsetup open /dev/sda2 rootfs && mount /dev/mapper/rootfs /mnt
```

---

## 改完密码，SSH 可能还是登不上

Ubuntu 默认禁止 root 密码 SSH，密码对了也登不上。要开的话：

```bash
sudo sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
sudo sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
sudo systemctl restart ssh        # CentOS/RHEL 用 sshd
```

> ⚠️ root + 密码 + 公网 22 端口易被爆破，建议设强密码或只用内网。

---

## 安全提醒

只用于你自己忘了密码的机器；改完设强密码；别把真实密码提交到公开仓库。
