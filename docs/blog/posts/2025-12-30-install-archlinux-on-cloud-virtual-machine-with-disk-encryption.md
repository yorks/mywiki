---
title: 在云主机上加密磁盘来安装archlinux
slug: install-archlinux-on-cloud-virtual-machine-with-disk-encryption
date: 2025-12-30
tags:
  - luks
  - encrypt
  - vm
  - security
  - archlinux
  - cloud
  - disk
---

当前云主机提供了公网，家里没有公网IP，只能依赖云主机来做中转，
虽然当前[reinstall][1]脚本非常优秀，但是有些东西我想自定义，
比如加密某个分区，就要自己动手了。

## 先重装archlinux来支持grub引导ISO启动

直接使用 [reinstall][1] [国内脚本][2]

执行完脚本后直接reboot就自动安装好archlinnux了，很方便。
如果没有加密需求，这是一个很干净的系统，没有云服务器提供的
各种agent后门。

如果有加密需求继续往下。

- 下载 archlinux.iso
```bash
curl -L -o /archlinux.iso https://mirrors.nju.edu.cn/archlinux/iso/latest/archlinux-x86_64.iso
```
- 添加grub引导iso的配置
```bash
sd=$(findmnt / -n | awk '{print $2}')
[[ -z $uuid  ]] && { echo "input / /dev/sd? " && read sd 
uuid=$(blkid $sd | grep -Eo '\s+UUID="[^"]+' | awk -F'"' '{print $2}')

cat >>/etc/grub.d/40_custom<<_EOF
menuentry "Arch Linux ISO from HDD" {
    search --no-floppy --fs-uuid --set=root $uuid
    set iso_file="/archlinux.iso"
    loopback loop \$iso_file
    linux (loop)/arch/boot/x86_64/vmlinuz-linux img_dev=$sd img_loop=\$iso_file earlymodules=loop archisobase
dir=arch copytoram=y
    initrd (loop)/arch/boot/x86_64/initramfs-linux.img
}
_EOF
grub-mkconfig -o /boot/grub/grub.cfg
```
**注意** : copytoram=y 如果内存不够就要先分一个小分区用来放 iso 文件

- 设置引导iso启动为默认启动方式
```bash
index=$(grep -Ec  'menuentry\s+' /boot/grub/grub.cfg | awk '{print $1-1}')
grep -q GRUB_DEFAULT=$index /etc/default/grub || sed -i "s/^GRUB_DEFAULT=.*/GRUB_DEFAULT=$index/g" /etc/defau
lt/grub

grub-mkconfig -o /boot/grub/grub.cfg
```

- 重启进去ISO 按照系统界面


## 手工安装系统
这步可以ssh进去，因为archlinux.iso默认开了22的sshd链接
避免这个时候人连进来，可以利用 iptables 加一下防火墙
```bash
iptables -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A INPUT -s $YOUR-IP -j ACCEPT
iptables -A INPUT -j DROP
```
- 分区
类似这样
```bash
 vda       254:0    0   40G  0 disk  
├─vda1    254:1    0  100M  0 part  /boot
└─vda2    254:2    0 39.9G  0 part  
```
为什么要保留 /boot 不加密呢？也可以全盘加密，但是每次重启都需要通过
云服务商的 vnc 或者 console 之类进去输入解密磁盘的密码进行解密后才能进去系统。
如果服务商在vnc或者console里面做手脚这样很容易获取到加密密码。

这里让/boot不加密，放一些用来启动的系统的文件，比如efi, initrd.img vmlinuz。
看到这里，你也许会有疑问，那服务商也可能在这些文件上做手脚，等你启动的时候
获取你远程输入的密码不也一样可以破解了吗？是的，按理是，后面我再说这个<span id="q">防护手段</span>。

当然你也可以使用 lvm[Encrypting_an_entire_system#LUKS_on_LVM][3], 我这里没有扩容
磁盘的需求就先不用lvm了。

- 加密 vda2
```bash
cryptsetup luksFormat --type luks1 --use-random -S 1 -s 512 -h sha512 -i 5000 /dev/vda2
cryptsetup open /dev/vda2 crypt
# 看到分区结构
lsblk
 vda       254:0    0   40G  0 disk  
├─vda1    254:1    0  100M  0 part  /boot
└─vda2    254:2    0 39.9G  0 part  
  └─crypt 253:0    0 39.9G  0 crypt /
```

- mount newOS目录
```bash
mkfs.ext4 /dev/mapper/crypt
mount /dev/mapper/crypt /mnt
mount --mkdir /dev/vda1 /mnt/boot
```

- 后面就跟正常安装archlinux一样了
[Install_Guide][4]
比如
```bash
timedatectl set-ntp true
echo 'Server = https://mirrors.nju.edu.cn/archlinux/$repo/os/$arch' > /etc/pacman.d/mirrorlist
pacstrap -K /mnt base vim grub openssh efibootmgr dosfstools sudo
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt
echo myarch > /etc/hostname
ln -sf /usr/share/zoneinfo/Asia/Hong_Kong /etc/localtime
cat>/etc/systemd/timesyncd.conf<<_EOF
[Time]
NTP=ntp.aliyun.com
FallbackNTP=0.arch.pool.ntp.org 1.arch.pool.ntp.org 2.arch.pool.ntp.org 3.arch.pool.ntp.org
_EOF

cat>/etc/systemd/network/10-wire.network<<_EOF
[Match]
Name=enp*
Name=ens*

[Network]
DHCP=ipv4
DNS=223.5.5.5
_EOF

echo 'nameserver 223.5.5.5' > /etc/resolv.conf
sed -i 's/^#en_US.UTF-8/en_US.UTF-8/g'  /etc/locale.gen
locale-gen
echo 'LANG=en_US.UTF-8' > /etc/locale.conf

echo 'myarch' > /etc/hostname

useradd yorks
usermod -G whell -a yorks
passwd yorks

mkdir -p /home/yorks/.ssh 
cp -f /etc/skel/.bash* /home/yorks/
echo 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICnGCMPoPA/CwHePhSwqwNH4dt3bARrb4sDIymMpbT2b yangyou' > /home/yorks
/.ssh/authorized_keys
chmod 0600 /home/yorks/.ssh/authorized_keys
chown -R yorks:yorks /home/yorks
chmod 0700 /home/yorks

cat>/etc/ssh/sshd_config.d/10-keyonly.conf<<_EOF
Port 22
Protocol 2
PermitRootLogin no
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
PasswordAuthentication no
ChallengeResponseAuthentication no
PermitEmptyPasswords no
UseDNS no
AllowUsers yorks
LogLevel Debug
_EOF

# 如果有其他软件需求一起安装了
# 比如我有nginx stream 模块利用tls来登录 sshd 等..
pacman -S nginx nginx-mod-stream wireguard-tools cronie
sed 's/^MAILTO=.*/MAILTO=""/g' /etc/cron.d/0hourly

```

- 安装支持加密的initrd工具
```bash
pacman -S linux mkinitcpio-netconf mkinitcpio-dropbear mkinitcpio-utils
```
1. `mkinitcpio-netconf` 是为了grub启动的时候支持配置网络
2. `mkinitcpio-dropbear` 是为了grub启动的时候支持配置网络
3. `mkinitcpio-utils` 配置initrd的工具包,包括`encrytssh`

- 把vda1的文件系统ko打进去initrd.img
```bash
# /mkinitcpio.conf
grep 'MODULES=(vfat fat)'
```

- 把encrypt相关工具打到initrd.img
```bash
# /etc/mkinitcpio.conf
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block netconf dropbear encryptssh filesystems fsck)'
```

- 修改encryptssh 让支持ssh登录
```bash
# /usr/lib/initcpio/install/encryptssh
make_etc_passwd() {
    #echo 'root:x:0:0:root:/root:/bin/cryptsetup_shell' > "${BUILDROOT}"/etc/passwd
    #echo '/bin/cryptsetup_shell' > "${BUILDROOT}"/etc/shells
    echo 'root:x:0:0:root:/root:/bin/sh' > "${BUILDROOT}"/etc/passwd
    echo '/bin/sh' > "${BUILDROOT}"/etc/shells
}
```

- 修复 dropbear的 dss问题
```bash
# /usr/lib/initcpio/install/dropbear
  for keytype in rsa ecdsa ; do
#                   ^ remove dss
```

- 修改grub 相关内核参数cmdline
```bash
blkid /dev/vda2
# /etc/default/grub
GRUB_CMDLINE_LINUX="ip=dhcp cryptdevice=UUID=<uuid of vda2>:crypt"
```

- 重新生成initrd.img 跟 grub.cfg
```bash
mkinitcpio -P
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB --recheck
grub-mkconfig -o /boot/grub/grub.cfg
```
- 确认sshd networkd等自启 
```bash
systemctl enable systemd-timesyncd
systemctl enable systemd-networkd
systemctl enable sshd

# 禁止用户用密码登录 也就是禁止用户从console ttyS0等终端登录
# 只有ssh才能登录  # 注意重要数据请定期备份走
passwd -l yorks
passwd -l root
# should we disable all user from tty ??
## cat /etc/securetty

cd /boot && sha256sum vmlinuz-linux initramfs-linux.img
exit 

# 记录上面两个sha256sum
```
**注意:** 每次有更新到initrd.img 都需执行这个命令或者可以写在pacman的hook里面 
然后复制出来保存好.



## 重启登录解密
```bash
#!/bin/bash

echo -n "🔍 正在挂载 boot 分区并校验..."

REMOTE_CMD="mkdir -p /tmp/boot_check && \
            mount -t vfat /dev/vda1 /tmp/boot_check && \
            cd /tmp/boot_check/ && \
            sha256sum vmlinuz-linux initramfs-linux.img && \
            umount /tmp/boot_check"

HOST="root@${YOUR-VM-IP"
PORT="22"
while sleep 1; do
    ssh -p $PORT $HOST pwd | grep -q ^/root && break
done
ssh -p $PORT $HOST "$REMOTE_CMD" 2>/dev/null > /tmp/hash.remote
vmhash="hash-of-vmlinuz" # 129a122f14f2398528e6585cae8301e479501cc8c41fb377356496c7edb85db6
inhash="hash-of-initramfs-linux.img" # 6dcf13fd3609f065ef1a85292a29e8fe95c4c13499a5468a7a350d24654e5a40

grep vmlinuz /tmp/hash.remote | grep -q $vmhash || {
	echo "🚨 严重警告：远程vmlinux哈希不匹配！停止操作!本地: $vmhash "
	cat /tmp/hash.remote
	exit 1
}

grep initramfs /tmp/hash.remote | grep -q $inhash || {
	echo "🚨 严重警告：远程initramfs哈希不匹配！停止操作！本地: $inhash"
	cat /tmp/hash.remote
	exit 2
}

echo "✅ 校验通过。"

echo "🔓 正在发送解锁指令..."
ssh -p $PORT $HOST "/bin/cryptsetup_shell" &&  echo "🚀 解锁指令发送成功！服务器正在启动..."
```
可以看到这步骤解决了之前的[疑问](#q).


[1]: https://github.com/bin456789/reinstall "reinstall"
[2]: https://cnb.cool/bin456789/reinstall/-/git/raw/main/reinstall.sh "reinstall-cn"
[3]: https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#LUKS_on_LVM "Encrypting_an_entire_system#LUKS_on_LVM"
[4]: https://wiki.archlinux.org/title/Installation_guide "Install_Guide"
