# qbittorrent-nox 配置


## 配置方式

配置源，并apt安装`qbittorrent-nox`。不使用docker的原因是避开docker环境下wireguard的端口转发问题，防止下载走VPN。

- 需新建专属用户 qbittorrent ，属于jellyfin组，方便下载文件时自动归到jellyfin目录下。

- 可选，在traefik里配置默认转发端口号8080。

- 默认密码在启动后的日志里`systemctl status qbittorrent-nox`，登录web后可修改。

## 注意事项

- 注意：本次设计上，jellyfin使用 `/storage/jellyfin-data/media/`下作为媒体库，qbittorrent-nox使用 `/storage/jellyfin-data/download/`下作为下载目录。下载完成后通过硬链接，自动链接到jellyfin的媒体库目录下。

- 设置分类，到`/storage/jellyfin-data/download`下的`/movies`或`/tvshow`，需保障和jellyfin设置的 的目录一致，否则自动链接脚本会产生问题。

  - jellyfin媒体库，例：`/storage/jellyfin-data/media/movies`

  - qbittorrent下载目录，例：`/storage/jellyfin-data/download/movies`，需保持一致。

  - jellyfin媒体库，例：`/storage/jellyfin-data/media/tvshow`

  - qbittorrent下载目录，例：`/storage/jellyfin-data/download/tvshow`，需保持一致。

- 下载媒体时，记得勾选分类。后续调整分类时，也会自动转移目录。需在web页面设置里设置：默认 Torrent 管理模式：自动。当 Torrent 分类修改时：重新定位 Torrent。

- 设置/下载 里，勾选 为所有文件预分配磁盘空间。
  - 勾选 torrent 完成时运行外部程序 `/usr/bin/python3 /storage/jellyfin-data/link_qb2jellyfin.py "%F"` *配合下方工具*。
- 设置/链接 里，设置传输端口号，记住，之后配合iptables打mark标记，避开wireguard VPN。
- **重要**：设置/高级 里，指定网络接口为实际的本地端口！！！否则打了mark也会走wg0 VPN。

## 避开VPN下载

- 在iptables里打mark标记，避免走VPN下载。

  - `iptables -t mangle -A OUTPUT -p tcp --dport 8080 -j MARK --set-mark 0x1234`等，请问AI

最终：
```
root@bings-hs:/storage/jellyfin-data#iptables -vL -t mangle
```
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

新建directout表，走本地网卡。自行查阅互联网

设置 ip rule。结果：

`ip rule show`:
```
0:      from all lookup local
10000:  from all fwmark 0x1234 lookup directout
32764:  from all lookup main suppress_prefixlength 0    #wg
32765:  not from all fwmark 0xca6c lookup 51820         #wg
...
```
`ip rule show table directout`:
```
10000:  from all fwmark 0x1234 lookup directout
```

##  自动链接脚本

### 配置


**注意需要配置jellyfin相关目录own为jellyfin及其组 ，并至少设置770权限** 


在`/storage/jellyfin-data`（dcoker compose设置的主媒体文件目录）下，新建python脚本`link_qb2jellyfin.py`，内容如下(来自deepseak，已测试)：

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
qBittorrent 下载后自动硬链接脚本
功能：当qBittorrent种子下载完成后，将文件硬链接到Jellyfin媒体库
"""

import os
import sys
import argparse
import logging
import stat
import shutil
from collections import deque
from typing import Tuple, Optional, List

# 配置区域 - 根据您的实际设置修改
MEDIA_BASE_DIR = "/storage/jellyfin-data/media"
DOWNLOAD_BASE_DIR = "/storage/jellyfin-data/download"
MOVIES_SOURCE_DIR = os.path.join(DOWNLOAD_BASE_DIR, "movies")
TVSHOWS_SOURCE_DIR = os.path.join(DOWNLOAD_BASE_DIR, "tvshow")
MOVIES_DEST_DIR = os.path.join(MEDIA_BASE_DIR, "movies")
TVSHOWS_DEST_DIR = os.path.join(MEDIA_BASE_DIR, "tvshow")

# 配置日志
def setup_logging():
    """配置日志系统"""
    log_file = "/storage/jellyfin-data/qblogs/qb_hardlink.log"
    logging.basicConfig(
        level=logging.INFO,
        format='%(asctime)s [%(levelname)s] %(message)s',
        datefmt='%Y-%m-%d %H:%M:%S',
        handlers=[
            logging.FileHandler(log_file),
            logging.StreamHandler(sys.stdout)
        ]
    )
    logging.info(f"日志文件: {log_file}")

def determine_category(content_path: str) -> Tuple[str, str, str]:
    """
    根据内容路径确定是电影还是电视剧
    
    返回:
        Tuple[类型, 源目录, 目标目录]
    """
    # 获取内容路径的绝对路径
    content_path = os.path.abspath(content_path)
    
    # 检查路径是否在电影目录下
    if content_path.startswith(MOVIES_SOURCE_DIR):
        rel_path = os.path.relpath(content_path, MOVIES_SOURCE_DIR)
        return "movie", MOVIES_SOURCE_DIR, MOVIES_DEST_DIR, rel_path
    
    # 检查路径是否在电视剧目录下
    if content_path.startswith(TVSHOWS_SOURCE_DIR):
        rel_path = os.path.relpath(content_path, TVSHOWS_SOURCE_DIR)
        return "tvshow", TVSHOWS_SOURCE_DIR, TVSHOWS_DEST_DIR, rel_path
    
    # 无法确定类别
    return None, None, None, None

def validate_directories():
    """验证所有必要的目录都存在"""
    required_dirs = [
        MOVIES_SOURCE_DIR, TVSHOWS_SOURCE_DIR,
        MOVIES_DEST_DIR, TVSHOWS_DEST_DIR
    ]
    
    for dir_path in required_dirs:
        if not os.path.exists(dir_path):
            logging.error(f"目录不存在: {dir_path}")
            return False
        if not os.path.isdir(dir_path):
            logging.error(f"路径不是目录: {dir_path}")
            return False
    
    return True

def is_same_filesystem(path1: str, path2: str) -> bool:
    """检查两个路径是否在同一文件系统"""
    try:
        dev1 = os.stat(path1).st_dev
        dev2 = os.stat(path2).st_dev
        return dev1 == dev2
    except OSError as e:
        logging.error(f"无法获取文件系统信息: {e}")
        return False

def create_hardlinks(source_path: str, dest_path: str, rel_path: str):
    """
    创建硬链接结构
    
    参数:
        source_path: 源文件/目录路径
        dest_path: 目标目录根路径
        rel_path: 相对路径
    """
    # 创建目标目录
    target_dir = os.path.join(dest_path, rel_path)
    if not os.path.exists(target_dir):
        os.makedirs(target_dir, exist_ok=True)
        logging.info(f"创建目标目录: {target_dir}")
    
    # 处理文件或目录
    if os.path.isfile(source_path):
        # 处理单个文件
        target_file = os.path.join(target_dir, os.path.basename(source_path))
        create_file_hardlink(source_path, target_file)
    elif os.path.isdir(source_path):
        # 处理整个目录
        process_directory(source_path, target_dir)
    else:
        logging.warning(f"跳过非常规文件: {source_path}")

def create_file_hardlink(source_file: str, target_file: str):
    """为单个文件创建硬链接"""
    # 检查目标文件是否已存在
    if os.path.exists(target_file):
        # 如果是硬链接则跳过
        if os.path.samefile(source_file, target_file):
            logging.info(f"已存在硬链接: {target_file}")
            return
        
        # 删除现有文件（如果不是硬链接）
        try:
            os.remove(target_file)
            logging.info(f"删除现有文件: {target_file}")
        except OSError as e:
            logging.error(f"删除文件失败: {target_file} - {e}")
            return
    
    # 创建硬链接
    try:
        os.link(source_file, target_file)
        logging.info(f"创建硬链接: {source_file} -> {target_file}")
    except OSError as e:
        logging.error(f"创建硬链接失败: {source_file} -> {target_file} - {e}")

def process_directory(source_dir: str, target_dir: str):
    """处理整个目录结构"""
    # 使用队列进行广度优先遍历
    queue = deque([(source_dir, target_dir)])
    
    while queue:
        src_path, dest_path = queue.popleft()
        
        # 确保目标目录存在
        if not os.path.exists(dest_path):
            os.makedirs(dest_path, exist_ok=True)
            logging.info(f"创建目录: {dest_path}")
        
        # 遍历源目录中的所有项目
        try:
            items = os.listdir(src_path)
        except PermissionError as e:
            logging.warning(f"无法访问目录 {src_path}: {e}")
            continue
        except OSError as e:
            logging.error(f"读取目录 {src_path} 时出错: {e}")
            continue
            
        for item in items:
            src_item = os.path.join(src_path, item)
            dest_item = os.path.join(dest_path, item)
            
            # 跳过符号链接
            if os.path.islink(src_item):
                logging.debug(f"跳过符号链接: {src_item}")
                continue
                
            # 处理子目录
            if os.path.isdir(src_item):
                queue.append((src_item, dest_item))
                continue
                
            # 处理文件
            if os.path.isfile(src_item):
                create_file_hardlink(src_item, dest_item)
            else:
                logging.warning(f"跳过非常规文件: {src_item}")

def main():
    """主函数"""
    # 设置日志
    setup_logging()
    
    # 验证目录
    if not validate_directories():
        logging.error("目录验证失败，脚本终止")
        sys.exit(1)
    
    # 解析命令行参数
    parser = argparse.ArgumentParser(
        description="qBittorrent 下载后自动硬链接脚本",
        formatter_class=argparse.ArgumentDefaultsHelpFormatter
    )
    parser.add_argument("content_path", help="qBittorrent传递的内容路径 (%F)")
    args = parser.parse_args()
    
    logging.info(f"开始处理: {args.content_path}")
    
    # 确定内容类别
    category, source_base, dest_base, rel_path = determine_category(args.content_path)
    
    if not category:
        logging.error(f"无法确定内容类别: {args.content_path}")
        logging.error(f"路径应位于: {MOVIES_SOURCE_DIR} 或 {TVSHOWS_SOURCE_DIR}")
        sys.exit(1)
    
    logging.info(f"检测到内容类型: {category}")
    logging.info(f"源目录: {source_base}")
    logging.info(f"目标目录: {dest_base}")
    logging.info(f"相对路径: {rel_path}")
    
    # 检查文件系统
    if not is_same_filesystem(args.content_path, dest_base):
        logging.error("源和目标不在同一文件系统，无法创建硬链接")
        sys.exit(1)
    
    # 创建硬链接结构
    create_hardlinks(args.content_path, dest_base, rel_path)
    
    # 完成
    logging.info("处理完成！")

if __name__ == "__main__":
    main()
```

在qbittorrent-nox的web界面中配置：设置/下载 里：
  - 勾选 torrent 完成时运行外部程序 `/usr/bin/python3 /storage/jellyfin-data/link_qb2jellyfin.py "%F"` 。

**注意**：使用方法：找到资源链接，新建任务后，在下载开始后，目录/storage/jellyfin-data/download/movies(或tvshows)下生成资源的目录后，进入，并将字幕文件放入其中。在文件下载完成后，字幕文件会自动移动到jellyfin的媒体库目录`/storage/jellyfin-data/download/media/movies(或tvshow)`下。


### 辅助链接脚本

运行后，将download下目录及文件整理、硬链接到media下。

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
目录结构复刻与硬链接脚本
功能：将源目录结构完整复制到目标目录，所有文件使用硬链接方式创建
要求：源目录和目标目录必须在同一个文件系统上
"""

import os
import sys
import argparse
import logging
import stat
from collections import deque
from typing import Tuple, Optional

# 配置日志
def setup_logging(verbose: bool = False):
    """配置日志系统"""
    level = logging.DEBUG if verbose else logging.INFO
    logging.basicConfig(
        level=level,
        format='%(asctime)s [%(levelname)s] %(message)s',
        datefmt='%Y-%m-%d %H:%M:%S',
        handlers=[
            logging.StreamHandler(sys.stdout)
        ]
    )

def check_directories(source: str, dest: str) -> Tuple[str, str]:
    """
    验证源目录和目标目录
    
    返回:
        Tuple[源目录绝对路径, 目标目录绝对路径]
    """
    # 获取绝对路径
    source = os.path.abspath(source)
    dest = os.path.abspath(dest)
    
    # 检查源目录是否存在
    if not os.path.isdir(source):
        logging.error(f"源目录不存在: {source}")
        sys.exit(1)
    
    # 检查目标目录是否存在
    if not os.path.isdir(dest):
        logging.info(f"目标目录不存在，正在创建: {dest}")
        os.makedirs(dest, exist_ok=True)
    
    # 检查是否相同目录
    if os.path.samefile(source, dest):
        logging.error("源目录和目标目录是同一个目录，操作已取消")
        sys.exit(1)
    
    # 检查是否同一文件系统
    source_dev = os.stat(source).st_dev
    dest_dev = os.stat(dest).st_dev
    if source_dev != dest_dev:
        logging.error("源目录和目标目录不在同一文件系统，无法创建硬链接")
        sys.exit(1)
    
    return source, dest

def is_symlink(path: str) -> bool:
    """检查路径是否是符号链接"""
    try:
        return os.path.islink(path)
    except OSError:
        return False

def process_directory(source: str, dest: str, dry_run: bool = False):
    """
    处理目录结构并创建硬链接
    
    参数:
        source: 源目录路径
        dest: 目标目录路径
        dry_run: 仅模拟运行，不实际修改文件系统
    """
    # 使用队列进行广度优先遍历
    queue = deque([(source, dest)])
    processed_items = 0
    created_links = 0
    skipped_items = 0
    
    while queue:
        src_path, dest_path = queue.popleft()
        
        # 确保目标目录存在
        if not os.path.exists(dest_path) and not dry_run:
            os.makedirs(dest_path, exist_ok=True)
            logging.info(f"创建目录: {dest_path}")
        
        # 遍历源目录中的所有项目
        try:
            items = os.listdir(src_path)
        except PermissionError as e:
            logging.warning(f"无法访问目录 {src_path}: {e}")
            continue
        except OSError as e:
            logging.error(f"读取目录 {src_path} 时出错: {e}")
            continue
            
        for item in items:
            src_item = os.path.join(src_path, item)
            dest_item = os.path.join(dest_path, item)
            processed_items += 1
            
            # 跳过符号链接
            if is_symlink(src_item):
                logging.debug(f"跳过符号链接: {src_item}")
                skipped_items += 1
                continue
                
            # 处理目录
            if os.path.isdir(src_item):
                queue.append((src_item, dest_item))
                continue
                
            # 处理文件
            if os.path.isfile(src_item):
                try:
                    # 检查目标文件是否存在
                    if os.path.exists(dest_item):
                        # 如果是硬链接，跳过
                        if os.path.samefile(src_item, dest_item):
                            logging.debug(f"已存在硬链接: {dest_item}")
                            skipped_items += 1
                            continue
                            
                        # 删除现有文件（如果不是硬链接）
                        if not dry_run:
                            os.remove(dest_item)
                            logging.info(f"删除现有文件: {dest_item}")
                    
                    # 创建硬链接
                    if not dry_run:
                        os.link(src_item, dest_item)
                        created_links += 1
                        logging.info(f"创建硬链接: {src_item} -> {dest_item}")
                    else:
                        logging.info(f"[模拟] 将创建硬链接: {src_item} -> {dest_item}")
                    
                except OSError as e:
                    logging.error(f"处理文件 {src_item} 时出错: {e}")
            else:
                logging.warning(f"跳过非常规文件: {src_item}")
                skipped_items += 1
    
    # 输出统计信息
    logging.info(f"\n处理完成:")
    logging.info(f"  处理项目总数: {processed_items}")
    logging.info(f"  创建硬链接数: {created_links}")
    logging.info(f"  跳过项目数: {skipped_items}")

def main():
    """主函数"""
    # 解析命令行参数
    parser = argparse.ArgumentParser(
        description="目录结构复刻与硬链接脚本",
        formatter_class=argparse.ArgumentDefaultsHelpFormatter
    )
    parser.add_argument("source", help="源目录路径")
    parser.add_argument("dest", help="目标目录路径")
    parser.add_argument("--verbose", "-v", action="store_true", help="显示详细输出")
    parser.add_argument("--dry-run", "-n", action="store_true", help="模拟运行，不实际修改文件系统")
    args = parser.parse_args()
    
    # 配置日志
    setup_logging(args.verbose)
    
    # 验证目录
    logging.info("开始目录验证...")
    source, dest = check_directories(args.source, args.dest)
    logging.info(f"源目录: {source}")
    logging.info(f"目标目录: {dest}")
    
    # 处理目录
    logging.info("\n开始处理目录和文件...")
    process_directory(source, dest, args.dry_run)
    
    # 显示磁盘使用情况
    if not args.dry_run:
        logging.info("\n磁盘使用情况:")
        source_size = os.popen(f"du -sh {source}").read().strip()
        dest_size = os.popen(f"du -sh {dest}").read().strip()
        logging.info(f"  源目录: {source_size}")
        logging.info(f"  目标目录: {dest_size}")
    
    logging.info("\n操作成功完成！")

if __name__ == "__main__":
    main()

```

#### 使用

```bash
python3 link-all.py /path/to/source /path/to/dest
```

#### 注意事项
- 请确保你有足够的权限来访问和修改源目录和目标目录。
- 如果目标目录中已经存在与源目录中相同的文件，脚本会删除这些文件并创建硬链接。