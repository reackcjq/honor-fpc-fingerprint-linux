# HONOR MagicBook Pro 14 2025 (FMB-P) 指纹传感器 Linux 适配教程

国内下载地址：[点我](https://www.reackcjq.top/d/%E8%B5%84%E6%BA%90/magicbook%20pro%2014%E6%8C%87%E7%BA%B9%E9%A9%B1%E5%8A%A8/honor-fpc-linux.zip)

### 适用硬件

| 设备 | 传感器 | USB ID |
|------|--------|--------|
| HONOR MagicBook Pro FMB-P | FPC L:2407 FW:3334151 | `10a5:9924` |

其他搭载同款 FPC L:2407 传感器的笔记本也可参考。

### 快速安装 (Ubuntu 26.04)

```bash
sudo dpkg -i deb/libfprint-2-2_1.95.1+tod1-0ubuntu1_amd64.deb \
               deb/libfprint-2-tod1_1.95.1+tod1-0ubuntu1_amd64.deb
sudo systemctl restart fprintd
fprintd-enroll -f "right-index-finger" $USER
fprintd-verify $USER                         # 应显示: verify-match
sudo pam-auth-update --enable fprintd        # 启用 sudo/登录/解锁指纹认证
```

### 源码编译 (通用，需 libfprint ≥ 1.94)

核心补丁 `fpc-10a5-9924.patch` 跨发行版通用。

#### Ubuntu / Debian

```bash
sudo apt build-dep libfprint-2-2 -y
apt source libfprint-2-2
cd libfprint-*/
patch -p1 < fpc-10a5-9924.patch
DEB_BUILD_OPTIONS="nocheck" dpkg-buildpackage -b -uc -us
sudo dpkg -i ../libfprint-2-2_*.deb ../libfprint-2-tod1_*.deb
sudo systemctl restart fprintd
```

#### Fedora / RHEL

```bash
sudo dnf builddep libfprint
dnf download --source libfprint
rpm -ivh libfprint-*.src.rpm
cd ~/rpmbuild/BUILD/libfprint-*/
patch -p1 < fpc-10a5-9924.patch
meson setup build && ninja -C build
sudo ninja -C build install
sudo ldconfig
sudo systemctl restart fprintd
```

#### Arch Linux

```bash
sudo pacman -S meson glib2 libusb nss gudev polkit pam
git clone https://gitlab.freedesktop.org/libfprint/libfprint.git
cd libfprint && git checkout v1.95.1
patch -p1 < fpc-10a5-9924.patch
meson setup build -Dudev_rules_dir=/usr/lib/udev/rules.d
ninja -C build && sudo ninja -C build install
sudo systemctl restart fprintd
```

#### openSUSE

```bash
sudo zypper source-install libfprint-2-2
cd ~/rpmbuild/BUILD/libfprint-*/
patch -p1 < fpc-10a5-9924.patch
meson setup build && ninja -C build
sudo ninja -C build install
sudo systemctl restart fprintd
```

#### NixOS

```nix
# configuration.nix 中添加 overlay:
{
  nixpkgs.overlays = [
    (final: prev: {
      libfprint = prev.libfprint.overrideAttrs (old: {
        patches = (old.patches or []) ++ [
          ./fpc-10a5-9924.patch
        ];
      });
    })
  ];
}
```

### 安装后使用

```bash
fprintd-list $USER                          # 查看已录入指纹
fprintd-enroll -f "left-index-finger" $USER # 录入更多手指
fprintd-delete $USER "right-index-finger"   # 删除指纹
fprintd-verify $USER                        # 测试验证
sudo -k true                                # 测试 sudo 指纹认证
```

### 补丁做了什么

| 改动 | 位置 | 说明 |
|------|------|------|
| 添加 USB ID | `fpc.c: id_table[]` | `{ .vid = 0x10A5, .pid = 0x9924 }` |
| 添加 probe 分支 | `fpc.c: fpc_dev_probe()` | `case 0x9924:` |
| 修复验证回调 | `fpc.c: fpc_verify_cb()` | 传感器 status=0 匹配成功但身份字段为零时的回退处理 |

FPC L:2407 传感器在 IDENTIFY 命令后返回 `status=0`（匹配成功），但身份字段全为零，而非其他 FPC 传感器使用的 `0xF5/0x3` 标准格式。

### 适配其他传感器

```bash
# 前台运行并查看调试日志
sudo systemctl stop fprintd
sudo G_MESSAGES_DEBUG=all /usr/libexec/fprintd -t 2>&1 | tee /tmp/fprintd.log &
# 另一个终端:
fprintd-verify $USER
grep -E "No driver|IDENTIFY response|error|fail" /tmp/fprintd.log
```

- `No driver found for USB device 10A5:XXXX` → 将 PID 加入 id_table
- `IDENTIFY response: status=0, subfactor=0x0` → 同款传感器，应用完整补丁
- `IDENTIFY response: status=0, subfactor=0xF5` → 标准格式，只需加 PID

---

## License

MIT — see [LICENSE](LICENSE)

## Note

- System updates (`apt upgrade`) may overwrite the patched libfprint — reinstall the deb if needed
- Kernel updates do not affect this userspace driver
- Prebuilt debs are for Ubuntu 26.04 LTS — use source build for other versions
