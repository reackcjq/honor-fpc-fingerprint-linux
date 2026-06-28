# HONOR MagicBook Pro (FMB-P) 指纹传感器 Linux 适配教程

## 适用机型

| 设备 | 传感器 | USB ID |
|------|--------|--------|
| HONOR MagicBook Pro FMB-P | FPC L:2407 FW:3334151 | `10a5:9924` |

其他同款传感器(FPC L:2407)的笔记本也可参考。

---

## 方法一：直接安装 deb 包（推荐，最快）

适用于 **Ubuntu 26.04 LTS**，其他版本需验证 libfprint 版本匹配。

### 步骤

```bash
# 1. 进入 deb 包所在目录
cd fingerprint-export/

# 2. 安装修改后的 libfprint
sudo dpkg -i libfprint-2-2_1.95.1+tod1-0ubuntu1_amd64.deb \
               libfprint-2-tod1_1.95.1+tod1-0ubuntu1_amd64.deb

# 3. 重启 fprintd 服务
sudo systemctl restart fprintd

# 4. 验证传感器已被识别
fprintd-list $USER
# 输出应显示: FPC L:2407 FW:3334151

# 5. 录入指纹
fprintd-enroll -f "right-index-finger" $USER

# 6. 测试验证
fprintd-verify $USER
# 输出应显示: verify-match (done)

# 7. 启用 PAM 指纹认证 (sudo/登录/解锁)
sudo pam-auth-update --enable fprintd
```

---

## 方法二：从源码编译（适用于其他发行版/版本）

### 前置依赖

```bash
# Ubuntu/Debian
sudo apt build-dep libfprint-2-2 -y
sudo apt install -y build-essential devscripts

# 如果 apt build-dep 失败，手动安装:
sudo apt install -y meson pkg-config libglib2.0-dev libusb-1.0-0-dev \
    libnss3-dev libgudev-1.0-dev libpolkit-gobject-1-dev \
    libpam0g-dev libxml2-dev libcairo2-dev libgirepository1.0-dev \
    gtk-doc-tools
```

### 编译安装

```bash
# 1. 获取源码
apt source libfprint-2-2
cd libfprint-1.95.1+tod1/

# 2. 应用补丁
patch -p1 < ../fpc-10a5-9924.patch

# 3. 编译 (跳过测试避免干扰)
DEB_BUILD_OPTIONS="nocheck" dpkg-buildpackage -b -uc -us

# 4. 安装
cd ..
sudo dpkg -i libfprint-2-2_*.deb libfprint-2-tod1_*.deb

# 5. 重启服务并录入
sudo systemctl restart fprintd
fprintd-enroll -f "right-index-finger" $USER
```

---

## 补丁说明

补丁文件 `fpc-10a5-9924.patch` 对 libfprint 做了三处修改：

### 改动 1: 添加设备 ID 到驱动支持列表

文件 `libfprint/drivers/fpcmoc/fpc.c`，函数 `id_table[]`：

```c
{ .vid = 0x10A5,  .pid = 0xC844,  },
{ .vid = 0x10A5,  .pid = 0x9924,  },  // <-- 新增
```

### 改动 2: 添加设备 ID 到 probe 探测函数

同一文件，函数 `fpc_dev_probe()`，switch 语句：

```c
case 0xC844:
case 0x9924:   // <-- 新增
    self->max_enroll_stage = MAX_ENROLL_SAMPLES;
    break;
```

### 改动 3: 修复验证回调，兼容非标准身份格式

同一文件，函数 `fpc_verify_cb()`：

FPC L:2407 传感器在 IDENTIFY 命令后返回 `status=0`（匹配成功），但身份数据字段（subfactor、identity_type、identity_size）全部为零，而不是其他 FPC 传感器使用的 `0xF5/0x3` 格式。

在标准身份匹配逻辑之后添加了回退处理：

```c
/* Sensor returned match with zero identity fields */
else if (presp->status == 0 && current_action == FPI_DEVICE_ACTION_VERIFY)
{
    FpPrint *print = NULL;
    fpi_device_get_verify_data(device, &print);
    if (print) {
        fpi_device_verify_report(device, FPI_MATCH_SUCCESS, print, error);
        fpi_ssm_mark_completed(self->task_ssm);
        return;
    }
}
```

---

## 验证与使用

```bash
# 查看已录入指纹
fprintd-list $USER

# 录入更多手指
fprintd-enroll -f "left-index-finger" $USER   # 左食指
fprintd-enroll -f "right-middle-finger" $USER  # 右中指
fprintd-enroll -f "right-thumb" $USER          # 右拇指

# 删除指纹
fprintd-delete $USER "right-index-finger"

# 测试指纹验证
fprintd-verify $USER

# 测试 sudo 指纹认证
sudo -k true    # (-k 清除密码缓存, 强制要求认证)

# 指纹失败时自动回退密码, 无需额外操作
```

---

## 自定义补丁指南

如果你的传感器 USB ID 不同（同样是 FPC 但 PID 不同），只需修改补丁中的 PID：

```bash
# 1. 查看你的传感器 USB ID
lsusb | grep FPC
# 输出类似: Bus 003 Device 003: ID 10a5:XXXX FPC ...

# 2. 在补丁中替换 0x9924 为你的 PID
sed -i 's/0x9924/0xXXXX/g' fpc-10a5-9924.patch

# 3. 同样替换 deb 包 (如需在其他 PID 上使用)
```

> **注意**: 不同 PID 的传感器协议可能不同。如果替换 PID 后验证失败，请检查 IDENTIFY 返回的身份格式。

### 调试方法

```bash
# 停止服务，前台运行并查看完整日志
sudo systemctl stop fprintd
sudo G_MESSAGES_DEBUG=all /usr/libexec/fprintd -t 2>&1 | tee /tmp/fprintd.log

# 另一个终端中测试
fprintd-verify $USER 2>&1

# 查看关键日志
grep -i "IDENTIFY response\|No driver\|error\|fail" /tmp/fprintd.log
```

关键判断：
- `No driver found for USB device 10A5:XXXX` → PID 未加入 id_table
- `IDENTIFY response: status=0, subfactor=0x0, identity_type=0x0` → 和本机同款传感器，需应用完整补丁
- `IDENTIFY response: status=0, subfactor=0xF5, identity_type=0x3` → 标准身份格式，只需添加 PID 即可

---

## 文件清单

```
fingerprint-export/
├── fpc-10a5-9924.patch                          # 补丁文件
├── libfprint-2-2_1.95.1+tod1-0ubuntu1_amd64.deb  # 主库 deb
├── libfprint-2-tod1_1.95.1+tod1-0ubuntu1_amd64.deb # TOD 驱动库 deb
├── fpc-modified.c.bak                              # 修改后的完整源文件(备用)
└── HONOR-MagicBook-FPC-指纹适配教程.md              # 本教程
```

---

## 注意事项

1. **系统更新会覆盖**: `sudo apt upgrade` 更新 libfprint 时需重新安装修改版 deb
2. **内核更新不受影响**: libfprint 是用户态驱动，内核升级不影响
3. **仅限同系统版本**: deb 包编译于 Ubuntu 26.04 LTS，其他版本请用源码编译方式
4. **传感器清洁**: 指纹传感器表面保持清洁可获得更好识别率
