# qbittorrent-nox 配置


## 配置方式

配置源，并apt安装`qbittorrent-nox`。不使用docker的原因是避开docker环境下wireguard的端口转发问题，防止下载走VPN。

- 需新建专属用户 qbittorrent ，属于jellyfin组，方便下载文件时自动归到jellyfin目录下。先参阅`jellyfin.md`。

- 可选：在traefik里配置默认转发端口号8080以访问web页面。

- 默认密码在启动后的日志里`systemctl status qbittorrent-nox`，登录web后可在设置里修改。

## 注意事项

- 注意：本次设计上，jellyfin使用 `/storage/jellyfin-data/media/`下（movie/ 和 tvshow/）作为媒体库，qbittorrent-nox使用 `/storage/jellyfin-data/download/`下作为下载目录（也建立movie/ 和 tvshow/，保持子目录结构一致）。下载完成后通过硬链接，自动链接到jellyfin的媒体库目录下。

- 设置分类，到`/storage/jellyfin-data/download`下的`/movies`或`/tvshow`，需保障和jellyfin设置的 的目录一致，否则自动链接脚本会产生问题。

  - jellyfin媒体库，例：`/storage/jellyfin-data/media/movies`

  - qbittorrent下载目录，例：`/storage/jellyfin-data/download/movies`，需保持一致。

  - jellyfin媒体库，例：`/storage/jellyfin-data/media/tvshow`

  - qbittorrent下载目录，例：`/storage/jellyfin-data/download/tvshow`，需保持一致。

- 下载媒体时，记得勾选分类，并设置下载位置到`/storage/jellyfin-data/download/`，设置`torrent管理模式：自动`。后续调整分类时，就会自动转移目录。需在web页面设置里设置：`当 Torrent 分类修改时：重新定位 Torrent`。

- 设置/下载 里，勾选 为所有文件预分配磁盘空间。
  - 勾选 torrent 完成时运行外部程序 `/usr/bin/python3 /storage/jellyfin-data/link_qb2jellyfin.py "%F"` *配合下方工具*。
- 设置/链接 里，设置传输端口号，之后配合iptables打mark标记，避开wireguard VPN。
  - **重要**：设置/高级 里，指定网络接口为实际的本地端口！！！否则打了mark也会走wg0 网卡。

## 避开VPN下载(无代理请忽略）

有的情况下，仅设置qt web里的网络接口就可以避免，如果测试可以，就忽略下面内容。

*注：以下内容通过服务动态添加，移除服务后，重启即可清除规则*

1. 在iptables里打mark标记，避免走VPN下载。

     - `iptables -t mangle -A OUTPUT -p tcp --dport <qb下载的port> --set-mark 0x1234`
     - `...`等，具体操作请问AI（包括iptabes持久化）

    最终效果，`iptables -vL -t mangle`：
    ```
    Chain PREROUTING (policy ACCEPT 250M packets, 215G bytes)
    pkts bytes target     prot opt in     out     source               destination         
    91M  162G MARK       tcp  --  any    any     anywhere             anywhere             tcp dpt:10388 MARK set 0x1234
    135M   28G MARK       udp  --  any    any     anywhere             anywhere             udp dpt:10388 MARK set 0x1234


    Chain INPUT (policy ACCEPT 250M packets, 215G bytes)
    pkts bytes target     prot opt in     out     source               destination         

    Chain FORWARD (policy ACCEPT 135K packets, 102M bytes)
    pkts bytes target     prot opt in     out     source               destination         

    Chain OUTPUT (policy ACCEPT 282M packets, 210G bytes)
    pkts bytes target     prot opt in     out     source               destination         
    65M   51G MARK       tcp  --  any    any     anywhere             anywhere             tcp spt:10388 MARK set 0x1234
    192M  111G MARK       udp  --  any    any     anywhere             anywhere             

    Chain POSTROUTING (policy ACCEPT 283M packets, 210G bytes)
    pkts bytes target     prot opt in     out     source               destination
    ```

2. 新建持久化directout路由表：

    编辑 `/etc/iproute2/rt_tables` 文件：
    ```
    sudo vi /etc/iproute2/rt_tables
    ```
    在文件末尾添加以下内容：

    ```
    100 directout
    ```
    保存并退出。



3. 设置 ip rule:

    注意，建议使用systemd的服务添加rule,例`sudo vi /etc/systemd/system/persistent-rules.service`：
    ```
    [Unit]
    Description=Persistent IP Routing Rules

    # 重要，一定要在wg之后启动，不然wg后启动，优先级会变低
    After=wg-quick@wg0.service network.target   
    Requires=wg-quick@wg0.service network.target

    [Service]
    Type=oneshot

    # 添加查找规则
    ExecStart=/sbin/ip route add default via <网关> dev <网卡> table directout
    ExecStart=/sbin/ip rule add fwmark 0x1234 table directout pref 10000
    RemainAfterExit=yes

    [Install]
    WantedBy=multi-user.target
    ```
    启动服务：
    ```
    systemctl daemon-reload
    systemctl enable persistent-rules.service
    systemctl start persistent-rules.service
    ```

    结果：

    `ip rule show`:
    ```
    0:      from all lookup local
    10000:  from all fwmark 0x1234 lookup directout
    32764:  from all lookup main suppress_prefixlength 0    #wg
    32765:  not from all fwmark 0xca6c lookup 51820         #wg
    ...
    ```

---
---

##  自动链接脚本

### 配置


**注意需要配置jellyfin相关目录own为jellyfin及其组 ，并至少设置770权限** 


在`/storage/jellyfin-data`（dcoker compose设置的主媒体文件目录）下，新建python脚本`link_qb2jellyfin.py`，内容如下(来自deepseek，已测试)：

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
qBittorrent 下载后自动硬链接脚本（简化检查 + dry-run 模式）
"""
import os
import sys
import argparse
import logging
import pwd
import grp
from collections import deque

MEDIA_BASE_DIR = "/storage/jellyfin-data/media"
DOWNLOAD_BASE_DIR = "/storage/jellyfin-data/download"
MOVIES_SOURCE_DIR = os.path.join(DOWNLOAD_BASE_DIR, "movies")
TVSHOWS_SOURCE_DIR = os.path.join(DOWNLOAD_BASE_DIR, "tvshow")
MOVIES_DEST_DIR = os.path.join(MEDIA_BASE_DIR, "movies")
TVSHOWS_DEST_DIR = os.path.join(MEDIA_BASE_DIR, "tvshow")
LOG_DIR = "/storage/jellyfin-data/qblogs"
FILE_OWNER = "jellyfin"
FILE_GROUP = "jellyfin"

DRY_RUN = False  # 全局 dry-run 标志


def setup_logging():
    if not os.path.exists(LOG_DIR):
        os.makedirs(LOG_DIR, exist_ok=True)
    log_file = os.path.join(LOG_DIR, "qb_hardlink.log")
    logging.basicConfig(
        level=logging.INFO,
        format="%(asctime)s [%(levelname)s] %(message)s",
        datefmt="%Y-%m-%d %H:%M:%S",
        handlers=[logging.FileHandler(log_file), logging.StreamHandler(sys.stdout)]
    )
    logging.info(f"日志文件: {log_file}")


def set_owner_group(path: str):
    try:
        uid = pwd.getpwnam(FILE_OWNER).pw_uid
        gid = grp.getgrnam(FILE_GROUP).gr_gid
        os.chown(path, uid, gid)
    except Exception as e:
        logging.warning(f"无法设置所有者: {path} - {e}")


def ensure_permissions(path: str, is_dir: bool = False):
    mode = 0o775 if is_dir else 0o664
    try:
        os.chmod(path, mode)
        set_owner_group(path)
    except Exception as e:
        logging.warning(f"无法设置权限: {path} - {e}")


def determine_category(content_path: str):
    content_path = os.path.abspath(content_path)
    if content_path.startswith(MOVIES_SOURCE_DIR):
        return "movie", MOVIES_DEST_DIR, os.path.relpath(content_path, MOVIES_SOURCE_DIR)
    if content_path.startswith(TVSHOWS_SOURCE_DIR):
        return "tvshow", TVSHOWS_DEST_DIR, os.path.relpath(content_path, TVSHOWS_SOURCE_DIR)
    return None, None, None


def create_file_hardlink(source_file: str, target_file: str):
    if DRY_RUN:
        logging.info(f"[dry-run] 链接: {source_file} -> {target_file}")
        return
    try:
        os.link(source_file, target_file)
        ensure_permissions(target_file)
        logging.info(f"创建硬链接: {source_file} -> {target_file}")
    except FileExistsError:
        logging.info(f"目标文件已存在: {target_file}")
    except OSError as e:
        logging.error(f"创建硬链接失败: {source_file} -> {target_file} - {e}")


def process_directory(source_dir: str, target_dir: str):
    queue = deque([(source_dir, target_dir)])
    while queue:
        src, dst = queue.popleft()
        logging.info(f"遍历目录: {src}")
        if not os.path.exists(dst) and not DRY_RUN:
            os.makedirs(dst, exist_ok=True)
            ensure_permissions(dst, is_dir=True)
        try:
            for item in os.listdir(src):
                src_item = os.path.join(src, item)
                dst_item = os.path.join(dst, item)
                if os.path.isdir(src_item):
                    queue.append((src_item, dst_item))
                elif os.path.isfile(src_item):
                    create_file_hardlink(src_item, dst_item)
        except Exception as e:
            logging.error(f"读取目录失败: {src} - {e}")


def main():
    global DRY_RUN
    setup_logging()

    parser = argparse.ArgumentParser(description="qBittorrent 下载后自动硬链接脚本")
    parser.add_argument("content_path", help="qBittorrent 传递的内容路径 (%F)")
    parser.add_argument("--dry-run", action="store_true", help="只显示将要硬链接的文件，不执行")
    args = parser.parse_args()
    DRY_RUN = args.dry_run

    category, dest_base, rel_path = determine_category(args.content_path)
    if not category:
        logging.error(f"无法确定内容类别: {args.content_path}")
        sys.exit(1)

    target_dir = os.path.join(dest_base, rel_path)
    logging.info(f"检测到类型: {category}, 目标目录: {target_dir}")

    if os.path.isfile(args.content_path):
        if not os.path.exists(target_dir) and not DRY_RUN:
            os.makedirs(target_dir, exist_ok=True)
        create_file_hardlink(args.content_path, os.path.join(target_dir, os.path.basename(args.content_path)))
    else:
        process_directory(args.content_path, target_dir)

    logging.info("处理完成！")


if __name__ == "__main__":
    main()

```

#### 使用

手动：
```bash
python3 link_qb2jellyfin.py /path/to/source
```

自动：

配合上面提到的qbittorren web界面的下载完成执行脚本，即可实现自动执行。

#### 注意事项
- 请确保你有足够的权限来访问和修改源目录和目标目录。
- 如果目标目录中已经存在与源目录中相同的文件，脚本会删除这些文件并改为创建硬链接。