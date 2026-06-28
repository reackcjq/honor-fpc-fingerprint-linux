# 各发行版适配方案

## 补丁兼容性前提

补丁依赖 libfprint 版本 **≥ 1.94.0**（fpcmoc 驱动在此版本引入）。

| libfprint 版本 | fpcmoc 驱动 | 适用性 |
|----------------|-------------|--------|
| < 1.94 | ❌ 不存在 | 不可用，需升级系统或自行移植驱动 |
| 1.94 ~ 1.95 | ✅ | 补丁正常应用 |
| ≥ 1.96 | 可能已内置 | 先检查 `fprintd-list $USER` 是否已识别 |

---

## Ubuntu / Debian 系

**deb 包安装** — 仅限 **Ubuntu 26.04**（同版本 libfprint 1.95.1+tod1）。

其他版本用源码编译：

```bash
# Debian 13+ / Ubuntu 24.04+
sudo apt build-dep libfprint-2-2 -y
apt source libfprint-2-2
cd libfprint-*/
patch -p1 < ../fpc-10a5-9924.patch
DEB_BUILD_OPTIONS="nocheck" dpkg-buildpackage -b -uc -us
sudo dpkg -i ../libfprint-2-2_*.deb ../libfprint-2-tod1_*.deb
sudo systemctl restart fprintd
```

---

## Fedora / RHEL 系

```bash
# 安装编译依赖
sudo dnf install -y meson gcc glib2-devel libusb1-devel \
    nss-devel libgudev-devel polkit-devel pam-devel \
    cairo-devel gobject-introspection-devel gtk-doc rpm-build
sudo dnf builddep libfprint

# 下载源码
dnf download --source libfprint
rpm -ivh libfprint-*.src.rpm
cd ~/rpmbuild/SPECS/

# 方法 A：修改 spec 文件加入补丁
# 在 %prep 段添加: %patch -P 0 -p1 fpc-10a5-9924.patch
# cp fpc-10a5-9924.patch ~/rpmbuild/SOURCES/

# 方法 B：直接编译安装（不打包）
cd ~/rpmbuild/BUILD/libfprint-*/
patch -p1 < /path/to/fpc-10a5-9924.patch
meson setup build && ninja -C build
sudo ninja -C build install
sudo ldconfig
sudo systemctl restart fprintd
```

---

## Arch Linux

```bash
# 方法 A：用 AUR 的 libfprint-2-tod 并打补丁
git clone https://aur.archlinux.org/libfprint-2-tod.git
cd libfprint-2-tod
# 编辑 PKGBUILD，在 prepare() 中添加：
#   patch -p1 -i /path/to/fpc-10a5-9924.patch
# 并在 source=() 中添加补丁文件路径
makepkg -si

# 方法 B：直接编译
sudo pacman -S meson glib2 libusb nss gudev polkit pam cairo \
    gobject-introspection gtk-doc
git clone https://gitlab.freedesktop.org/libfprint/libfprint.git
cd libfprint
git checkout v1.95.1  # 或匹配系统版本
patch -p1 < /path/to/fpc-10a5-9924.patch
meson setup build -Dudev_rules_dir=/usr/lib/udev/rules.d
ninja -C build
sudo ninja -C build install
sudo systemctl restart fprintd
```

---

## openSUSE

```bash
# 安装依赖
sudo zypper install meson gcc glib2-devel libusb-1_0-devel \
    mozilla-nss-devel libgudev-1_0-devel polkit-devel pam-devel \
    cairo-devel gobject-introspection-devel gtk-doc
sudo zypper source-install libfprint-2-2

# 编译安装
cd ~/rpmbuild/BUILD/libfprint-*/
patch -p1 < /path/to/fpc-10a5-9924.patch
meson setup build && ninja -C build
sudo ninja -C build install
sudo systemctl restart fprintd
```

---

## NixOS

```nix
# 在 configuration.nix 中添加 overlay：
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

---

## 快速判断方法

在目标笔记本上先跑这两条命令：

```bash
# 1. 查传感器 USB ID
lsusb | grep -i FPC

# 2. 查 libfprint 版本
pkg-config --modversion libfprint-2 2>/dev/null || \
  dpkg -l libfprint-2-2 2>/dev/null | tail -1 || \
  rpm -q libfprint 2>/dev/null
```

- USB ID 不是 `10a5:9924` → 需改补丁中的 PID
- libfprint < 1.94 → 系统太旧，建议先升级系统
- `fprintd-list $USER` 已识别 → 不需要任何修改
