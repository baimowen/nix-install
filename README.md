# Nixos

>   [!note]
>
>   安装前的准备：一个刷写了 NixosLive 的启动盘、一台主机
>
>   本文仅对 **Minimal ISO image** 配置，**Graphical ISO image** 根据桌面提示进行即可



修改root密码(可选)：`sudo passwd root`

>[!tip]
>
>小技巧：使用代码块组合命令
>
>```bash
># `{` 后必须有一个空格 `}` 必须单独占一行或前包含分号(;)
># 命令之间需要换行或分号隔开
>{ whoami; pwd; }
>root
>/root
>```

---

## 联网

查看默认网卡

```bash
ip r | awk 'NR==1{print $5}'
```

查看当前ip

```bash
ip r | awk 'NR==1{print $9}'
```

格式化显示默认网卡、当前ip以及网关信息

```bash
ip r | awk '/default/ {g=$3} /src/ {ip=$(NF-2)} END {print "interface: ens33\nipaddr:", ip, "\ngateway:", g}'
interface:
ipaddr:
gateway:

# 或

ifconfig ens33 | awk '/inet / {ip=$2} END {print "interface: ens33\nipaddr:", ip}'; ip r | awk '/default/ {print "gateway:", $3}'
interface:
ipaddr:
gateway:
```

使用 `ifconfig` 修改ip

```bash
ifconfig ens33 <ipaddr> netmask 255.255.255.0  # 设置ip和子网掩码
route add default gw <gateway>  # 设置网关
# 验证
ifconfig ens33 | grep "inet "
ip route | grep default
```

---

## 分区仪式

创建 GPT 分区表

```bash
parted /dev/nvme0n1 -- mklabel gpt
```

创建 EFI 分区（512MB，FAT32）

```bash
parted /dev/nvme0n1 -- mkpart ESP fat32 1MiB 513MiB
parted /dev/nvme0n1 -- set 1 esp on
```

创建 Btrfs 根分区

```
parted /dev/nvme0n1 -- mkpart primary btrfs 513MiB -1GiB
```

创建 Swap 分区（1GB）

```
parted /dev/nvme0n1 -- mkpart primary linux-swap -1GiB 100%
```

格式化 EFI 分区（FAT32）

```bash
mkfs.fat -F32 /dev/nvme0n1p1
```

格式化 Btrfs 分区

```bash
mkfs.btrfs /dev/nvme0n1p2
```

启用 Swap 分区

```bash
mkswap /dev/nvme0n1p3
swapon /dev/nvme0n1p3
```

挂载 Btrfs 分区

```bash
mount /dev/nvme0n1p2 /mnt
```

创建子卷

```bash
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@nix
btrfs subvolume list /mnt  # 查看子卷
```

卸载临时挂载

```bash
umount /mnt
```

重新挂载子卷

```bash
mount -o subvol=@ /dev/nvme0n1p2 /mnt
mkdir -p /mnt/{home,nix,boot}
mount -o subvol=@home /dev/nvme0n1p2 /mnt/home
mount -o subvol=@nix /dev/nvme0n1p2 /mnt/nix
mount /dev/nvme0n1p1 /mnt/boot
```

检查分区

```bash
lsblk -f
swapon --show
```

---

## 生成配置

为root生成默认配置

```bash
nixos-generate-config --root /mnt/
```

检查硬件配置文件 `/mnt/etc/nixos/hardware-configuration.nix` 确保挂载点正确

```bash
fileSystems."/" =
{ device = "/dev/disk/by-uuid/7de5e3d3-d2a1-4017-9eea-4d113910094c";
  fsType = "btrfs";
  options = [ "subvol=@" ];
};

fileSystems."/home" =
{ device = "/dev/disk/by-uuid/7de5e3d3-d2a1-4017-9eea-4d113910094c";
  fsType = "btrfs";
  options = [ "subvol=@home" ];
};

fileSystems."/nix" =
{ device = "/dev/disk/by-uuid/7de5e3d3-d2a1-4017-9eea-4d113910094c";
  fsType = "btrfs";
  options = [ "subvol=@nix" ];
};

fileSystems."/boot" =
{ device = "/dev/disk/by-uuid/BDE9-0613";
  fsType = "vfat";
  options = [ "fmask=0022" "dmask=0022" ];
};
```



## 修改主配置文件 

`/mnt/etc/nixos/configuration.nix`

### 修改引导

*   grub

```bash
# Use the GRUB 2 boot Loader.
boot.loader = {
    grub = {
      enable = true;
      device = "nodev";
      efiSupport = true;
      # timeout = 0;  # 设置等待时间为 0 秒
    };
    efi = {
      canTouchEfiVariables = true;
      efiSysMountPoint = "/boot";
    };
};
```

>   nodev: UEFI 模式专用，表示 GRUB 不直接安装到磁盘的 MBR/GPT，而是通过 EFI 变量管理启动项。

*   systemd-boot

```bash
# Use the systemd-boot EFI boot Loader.
boot.loader = {
    efi = {
      canTouchEfiVariables = true;
      efiSysMountPoint = "/boot";  # 确保与 EFI 分区挂载点一致
    };
    systemd-boot = {
      enable = true;
      timeout = 0;  # 设置等待时间为 0 秒
    };
};
```

### 修改时区

```bash
sed -i '/time\.timeZone/s/^.*$/  time.timeZone = "Asia\/Shanghai";/' /mnt/etc/nixos/configuration.nix
```

### 修改语言

```bash
sed -i '/i18n\.defaultLocale/s/^.*$/  i18n.defaultLocale = "en_US.UTF-8";/' /mnt/etc/nixos/configuration.nix
cat /mnt/etc/nixos/configuration.nix | grep -B 1 'i18n.defaultLocale'
  # Select internationalisation properties.
  i18n.defaultLocale = "en_US.UTF-8";
```

### 用户配置

```bash
# Define a user account. Don't forget to set a password with ‘passwd’.
users.mutableUsers = false;  # 强制使用配置文件管理用户，无法使用passwd修改密码
# users.users.root.hashedPassword = "<hashedPassword>"
users.users.<username> = {
  # hashedPassword = "<hashedPassword>"
  isNormalUser = true;
  extraGroups = [ "wheel" ]; # Enable ‘sudo’ for the user.
  packages = with pkgs; [
    neofetch
    tree
  ];
};
```

**hashedPassword** 是root用户密码的哈希值，使用 `mkpasswd` 生成

### 系统环境设置

```bash
sed -i '/^  # environment.systemPackages/,/^  # ];/ s/^  # /  /' /mnt/etc/nixos/configuration.nix
cat /mnt/etc/nixos/configuration.nix | grep -A6 "List packages installed in system profile"
  # List packages installed in system profile.
  # You can use https://search.nixos.org/ to find more packages (and options).
  environment.systemPackages = with pkgs; [
    vim # Do not forget to add an editor to edit configuration.nix! The Nano editor is also installed by default.
    wget
    curl
  ];
```



开启openssh：

```bash
sed -i '/^  # services.openssh.enable = true;/ s/^  # /  /' /mnt/etc/nixos/configuration.nix
# 或使用以下匹配
# sed -i '/services\.openssh\.enable/s/^[[:space:]]*#[[:space:]]*/  /' /mnt/etc/nixos/configuration.nix
cat /mnt/etc/nixos/configuration.nix | grep -B 1 'services.openssh'
  # Enable the OpenSSH daemon.
  services.openssh.enable = true;
```



放行指定端口

```bash
sed -i '/# Open ports in the firewall./a \  networking.firewall.allowedTCPPorts = [\n    22\n    80 443\n    9090\n    8807 9000\n  ];' /mnt/etc/nixos/configuration.nix
sed -i '/# Or disable the firewall altogether./i \  services.openssh.settings.PermitRootLogin = "yes";' /mnt/etc/nixos/configuration.nix  # 允许ssh root
cat /mnt/etc/nixos/configuration.nix | grep -A12 'Open ports in the firewall'
  # Open ports in the firewall.
  networking.firewall.allowedTCPPorts = [
    22
    80 443
    9090
    8807 9000
  ];
  networking.firewall.allowedTCPPorts = [ 22 80 443 9090 8807 9000 ];
  # networking.firewall.allowedTCPPorts = [ ... ];
  # networking.firewall.allowedUDPPorts = [ ... ];
  services.openssh.settings.PermitRootLogin = "yes";
  # Or disable the firewall altogether.
  # networking.firewall.enable = false;
```



添加国内源(ustc)

```bash
sed -i '/^[[:space:]]*imports[[:space:]]*=/i \  nix.settings.substituters = [ "https://mirrors.ustc.edu.cn/nix-channels/store" ];' /mnt/etc/nixos/configuration.nix
```

---

## 安装Nixos

```bash
nixos-install --option substituters https://mirrors.ustc.edu.cn/nix-channels/store
nixos-enter --root /mnt -c 'passwd nix'
reboot
```

安装过程中输入的密码为root密码，若使用nix登录需root进入系统后手动添加密码

---

## 修复系统

```bash
nixos-enter --root /mnt
```

**nixos-enter**: NixOS 提供的特殊命令，用于进入已安装的 NixOS 系统的环境。类似于 chroot，但比单纯的 chroot 更完善（它会挂载必要的虚拟文件系统，如 `/proc`, `/dev`, `/sys` 等）。



进入已安装的系统并执行命令

```bash
nixos-enter --root /mnt -c '<command>'
```

