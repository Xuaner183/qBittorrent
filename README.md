# 如何编译？

## 前置要求

### 1. 下载软件包

```shell
sudo apt install build-essential cmake git ninja-build pkg-config libboost-dev libssl-dev zlib1g-dev
```

### 2. 下载 Qt 6.6

无图形化安装 Qt:

```python
# 安装 Qt 的命令行工具 aqtinstall（Python 包）
pip install aqtinstall
```

```python
# 安装 Qt 6.6
aqt install-qt linux desktop 6.6.0 gcc_64 -O ~/Qt
```

`linux` 表示平台

`desktop` 表示桌面版（不是嵌入式）

`6.6.0` 是版本号

`gcc_64` 是编译器版本

`-O ~/Qt` 指定安装路径

下载完成后，你的 Qt 就在 `~/Qt/6.6.0/gcc_64/bin/` 下。

### 3.设置 Qt 变量

```shell
export PATH=$HOME/Qt/6.6.2/gcc_64/bin:$PATH
export LD_LIBRARY_PATH=$HOME/Qt/6.6.2/gcc_64/lib:$LD_LIBRARY_PATH
export PKG_CONFIG_PATH=$HOME/Qt/6.6.2/gcc_64/lib/pkgconfig:$PKG_CONFIG_PATH
```

## 编译

```shell
cmake -B build -G Ninja   -DCMAKE_BUILD_TYPE=Release   -DGUI=OFF  # 关键，nox 版本要加这个
cmake --build build -j$(nproc)
```

## 打包deb

### 1. cd 项目根目录

```shell
# 创建打包文件夹
mkdir -p debian_pkg/DEBIAN
mkdir -p debian_pkg/usr/bin

# 复制编译好的二进制文件
cp build/qbittorrent-nox debian_pkg/usr/bin
```
### 2. `ldd build/qbittorrent-nox` 查看软件依赖哪些动态链

```shell
libQt6Network.so.6 => /home/zhang/Qt/6.6.2/gcc_64/lib/libQt6Network.so.6 (0x000070d883a57000)
libQt6Sql.so.6 => /home/zhang/Qt/6.6.2/gcc_64/lib/libQt6Sql.so.6 (0x000070d88530e000)
libQt6Xml.so.6 => /home/zhang/Qt/6.6.2/gcc_64/lib/libQt6Xml.so.6 (0x000070d8852e1000)
libQt6Core.so.6 => /home/zhang/Qt/6.6.2/gcc_64/lib/libQt6Core.so.6 (0x000070d883200000)
libicui18n.so.56 => /home/zhang/Qt/6.6.2/gcc_64/lib/libicui18n.so.56 (0x000070d882400000)
libicuuc.so.56 => /home/zhang/Qt/6.6.2/gcc_64/lib/libicuuc.so.56 (0x000070d882000000)
libicudata.so.56 => /home/zhang/Qt/6.6.2/gcc_64/lib/libicudata.so.56 (0x000070d880600000)

libtorrent-rasterbar.so.2.0 => /lib/x86_64-linux-gnu/libtorrent-rasterbar.so.2.0 (0x0000758365200000)
```
把以上文件复制至 `debian_pkg/usr/bin<br />`
还有 `/home/zhang/Qt/6.6.2/gcc_64/plugins/tls` 目录以前复制。

### 3. 在 `debian_pkg/DEBIAN` 创建 `postinst`

```shell
#!/bin/sh
set -e

# 创建数据目录
mkdir -p /var/lib/qbittorrent
chmod 755 /var/lib/qbittorrent

# 设置系统服务
cat > /etc/systemd/system/qbittorrent-nox.service << 'EOS'
[Unit]
Description=qBittorrent Enhanced Command Line Client
After=network.target

[Service]
Environment="LD_LIBRARY_PATH=/usr/lib/qbittorrent:${LD_LIBRARY_PATH}"
Environment="QT_PLUGIN_PATH=/usr/lib/qbittorrent/plugins"
ExecStart=/usr/bin/qbittorrent-nox --profile=/var/lib/qbittorrent
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOS

# 启用服务但不立即启动
systemctl daemon-reload
systemctl enable qbittorrent-nox.service

# 首次安装时启动服务
if [ "$1" = "configure" ] && [ -z "$2" ]; then
    systemctl start qbittorrent-nox.service
fi

ldconfig
```

### 4. 在 `debian_pkg/DEBIAN` 创建 `postrm`

```shell
#!/bin/sh
set -e

# 这是 qbittorrent-nox 包的 postrm 脚本
# 在删除包后执行清理操作

case "$1" in
  purge|remove|upgrade|failed-upgrade|abort-install|abort-upgrade|disappear)
    ;;
  *)
    echo "postrm called with unknown argument \`$1'" >&2
    exit 1
    ;;
esac

# 只有在完全清除 (purge) 时才删除数据目录
if [ "$1" = "purge" ] || [ "$1" = "disappear" ]; then
    # 停止服务 (如果还在运行)
    if systemctl is-active --quiet qbittorrent-nox.service; then
        systemctl stop qbittorrent-nox.service
    fi

    # 禁用服务
    systemctl disable qbittorrent-nox.service >/dev/null 2>&1 || true

    # 删除数据目录
    if [ -d /var/lib/qbittorrent ]; then
        rm -rf /var/lib/qbittorrent
    fi

    # 删除服务配置文件
    if [ -f /etc/systemd/system/qbittorrent-nox.service ]; then
        rm -f /etc/systemd/system/qbittorrent-nox.service
    fi

    # 重新加载 systemd
    systemctl daemon-reload >/dev/null 2>&1 || true
fi

# 在 remove 时只执行基本清理
if [ "$1" = "remove" ] || [ "$1" = "upgrade" ] || [ "$1" = "abort-upgrade" ] || [ "$1" = "failed-upgrade" ]; then
    # 停止服务
    if systemctl is-active --quiet qbittorrent-nox.service; then
        systemctl stop qbittorrent-nox.service && rm -f /etc/systemd/system/qbittorrent-nox.service
    fi

    # 重新加载 systemd
    systemctl daemon-reload >/dev/null 2>&1 || true
fi

exit 0
```

### 5. 在 `debian_pkg/DEBIAN` 创建 `control`

```shell
Package: qbittorrent-nox
Version: 5.1.2.10
Section: net
Priority: optional
Architecture: amd64
Depends: libc6, libgcc-s1, libstdc++6, libssl3 (>= 3.0.2), zlib1g (>= 1.2.11)
Maintainer: sledgehammer999 <sledgehammer999@qbittorrent.org>
Description: Enhanced qBittorrent command-line client
 This package provides the qBittorrent client with enhanced features
 without the GUI interface.

```
别忘了末尾的空行，是必须的。

### 6. 设置 `postinst`, `postrm` 权限为 `755`

```shell
chmod 755 debian_pkg/DEBIAN/postinst debian_pkg/DEBIAN/prerm
```

### 7. 打包生成 `debian_pkg.deb`

```shell
dpkg-deb --build debian_pkg
```
