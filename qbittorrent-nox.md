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
qBittorrent 下载后自动硬链接脚本（修复特殊字符问题）
"""

import os
import sys
import argparse
import logging
import stat
import shutil
import grp
import pwd
import time
import glob
from collections import deque
from typing import Tuple, Optional, List

# 配置区域 - 根据您的实际设置修改
MEDIA_BASE_DIR = "/storage/jellyfin-data/media"
DOWNLOAD_BASE_DIR = "/storage/jellyfin-data/download"
MOVIES_SOURCE_DIR = os.path.join(DOWNLOAD_BASE_DIR, "movies")
TVSHOWS_SOURCE_DIR = os.path.join(DOWNLOAD_BASE_DIR, "tvshow")
MOVIES_DEST_DIR = os.path.join(MEDIA_BASE_DIR, "movies")
TVSHOWS_DEST_DIR = os.path.join(MEDIA_BASE_DIR, "tvshow")
LOG_DIR = "/storage/jellyfin-data/qblogs"

# 设置文件所有者和组（根据您的系统配置修改）
FILE_OWNER = "jellyfin"
FILE_GROUP = "jellyfin"

# 配置日志
def setup_logging():
    """配置日志系统"""
    if not os.path.exists(LOG_DIR):
        os.makedirs(LOG_DIR, exist_ok=True)
    
    log_file = os.path.join(LOG_DIR, "qb_hardlink.log")
    
    # 设置日志目录权限
    try:
        uid = pwd.getpwnam(FILE_OWNER).pw_uid
        gid = grp.getgrnam(FILE_GROUP).gr_gid
        os.chown(LOG_DIR, uid, gid)
        os.chmod(LOG_DIR, 0o775)
    except Exception as e:
        logging.error(f"设置日志目录权限失败: {e}")
    
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
    """根据内容路径确定是电影还是电视剧"""
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
        MOVIES_DEST_DIR, TVSHOWS_DEST_DIR,
        LOG_DIR
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

def set_file_permissions(path: str):
    """设置文件权限和所有权"""
    try:
        # 设置权限为 rw-rw-r-- (664) 或目录为 rwxrwxr-x (775)
        if os.path.isdir(path):
            os.chmod(path, 0o775)
        else:
            os.chmod(path, 0o664)
        
        # 设置所有者和组
        uid = pwd.getpwnam(FILE_OWNER).pw_uid
        gid = grp.getgrnam(FILE_GROUP).gr_gid
        os.chown(path, uid, gid)
        logging.debug(f"设置权限: {path}")
        return True
    except Exception as e:
        logging.error(f"设置权限失败: {path} - {e}")
        return False

def create_hardlinks(source_path: str, dest_path: str, rel_path: str):
    """创建硬链接结构"""
    # 创建目标目录
    target_dir = os.path.join(dest_path, rel_path)
    if not os.path.exists(target_dir):
        os.makedirs(target_dir, exist_ok=True)
        set_file_permissions(target_dir)
        logging.info(f"创建目标目录: {target_dir}")
    
    # 处理文件或目录
    if os.path.isfile(source_path):
        # 确保源文件有正确权限
        if not set_file_permissions(source_path):
            logging.error(f"无法设置源文件权限: {source_path}")
            return
        
        # 处理单个文件
        target_file = os.path.join(target_dir, os.path.basename(source_path))
        create_file_hardlink(source_path, target_file)
    elif os.path.isdir(source_path):
        # 确保源目录有正确权限
        set_file_permissions(source_path)
        
        # 处理整个目录
        process_directory(source_path, target_dir)
    else:
        logging.warning(f"跳过非常规文件: {source_path}")

def create_file_hardlink(source_file: str, target_file: str):
    """为单个文件创建硬链接"""
    # 正确处理带特殊字符的文件名
    safe_source = source_file
    safe_target = target_file
    
    # 确保源文件可读
    if not os.access(safe_source, os.R_OK):
        logging.error(f"源文件不可读: {safe_source}")
        # 尝试修复权限
        try:
            os.chmod(safe_source, 0o664)
            logging.info(f"已修复源文件权限: {safe_source}")
        except Exception as e:
            logging.error(f"修复源文件权限失败: {safe_source} - {e}")
            return
    
    # 检查目标文件是否已存在
    if os.path.exists(safe_target):
        # 如果是硬链接则跳过
        try:
            if os.path.samefile(safe_source, safe_target):
                logging.info(f"已存在硬链接: {safe_target}")
                return
        except OSError as e:
            logging.warning(f"检查相同文件时出错: {e} - 继续处理")
        
        # 尝试删除现有文件（如果不是硬链接）
        try:
            # 确保有删除权限
            if not os.access(safe_target, os.W_OK):
                os.chmod(safe_target, 0o666)  # 临时添加写权限
            os.remove(safe_target)
            logging.info(f"删除现有文件: {safe_target}")
        except OSError as e:
            logging.error(f"删除文件失败: {safe_target} - {e}")
            return
    
    # 创建硬链接 - 添加重试逻辑
    max_retries = 3
    for attempt in range(max_retries):
        try:
            os.link(safe_source, safe_target)
            set_file_permissions(safe_target)  # 设置正确权限
            logging.info(f"创建硬链接: {safe_source} -> {safe_target}")
            return  # 成功则退出函数
        except OSError as e:
            if e.errno == 1:  # Operation not permitted
                logging.warning(f"尝试 {attempt+1}/{max_retries}: 创建硬链接被拒绝，等待后重试...")
                
                # 检查并修复目标目录权限
                target_dir = os.path.dirname(safe_target)
                if not os.access(target_dir, os.W_OK):
                    try:
                        os.chmod(target_dir, 0o775)
                        logging.info(f"已修复目标目录权限: {target_dir}")
                    except Exception as e:
                        logging.error(f"修复目录权限失败: {target_dir} - {e}")
                
                time.sleep(1)  # 等待1秒后重试
            else:
                logging.error(f"创建硬链接失败: {safe_source} -> {safe_target} - {e}")
                # 尝试使用引号包裹文件名
                if "'" in safe_source or "[" in safe_source:
                    logging.warning("文件名包含特殊字符，尝试使用glob处理")
                    try:
                        # 使用glob匹配带特殊字符的文件
                        matched_files = glob.glob(safe_source)
                        if matched_files:
                            actual_source = matched_files[0]
                            os.link(actual_source, safe_target)
                            set_file_permissions(safe_target)
                            logging.info(f"使用glob成功创建硬链接: {actual_source} -> {safe_target}")
                            return
                        else:
                            logging.error(f"未找到匹配文件: {safe_source}")
                    except Exception as e:
                        logging.error(f"使用glob创建硬链接失败: {e}")
                return
    
    # 所有重试都失败
    logging.error(f"创建硬链接失败（重试{max_retries}次）: {safe_source} -> {safe_target}")

def process_directory(source_dir: str, target_dir: str):
    """处理整个目录结构"""
    # 使用队列进行广度优先遍历
    queue = deque([(source_dir, target_dir)])
    
    while queue:
        src_path, dest_path = queue.popleft()
        
        # 确保目标目录存在并有正确权限
        if not os.path.exists(dest_path):
            os.makedirs(dest_path, exist_ok=True)
            set_file_permissions(dest_path)
            logging.info(f"创建目录: {dest_path}")
        else:
            # 确保目标目录有写权限
            if not os.access(dest_path, os.W_OK):
                try:
                    os.chmod(dest_path, 0o775)
                    logging.info(f"修复目录写权限: {dest_path}")
                except Exception as e:
                    logging.error(f"修复目录权限失败: {dest_path} - {e}")
        
        # 遍历源目录中的所有项目
        try:
            # 使用glob正确处理带特殊字符的文件名
            items = glob.glob(os.path.join(src_path, '*'))
            # 只保留实际存在的文件/目录
            items = [item for item in items if os.path.exists(item)]
        except PermissionError as e:
            logging.warning(f"无法访问目录 {src_path}: {e}")
            continue
        except OSError as e:
            logging.error(f"读取目录 {src_path} 时出错: {e}")
            continue
            
        for item in items:
            # 获取基本文件名（不带路径）
            base_name = os.path.basename(item)
            dest_item = os.path.join(dest_path, base_name)
            
            # 跳过符号链接
            if os.path.islink(item):
                logging.debug(f"跳过符号链接: {item}")
                continue
                
            # 处理子目录
            if os.path.isdir(item):
                queue.append((item, dest_item))
                continue
                
            # 处理文件
            if os.path.isfile(item):
                # 设置源文件权限
                set_file_permissions(item)
                create_file_hardlink(item, dest_item)
            else:
                logging.warning(f"跳过非常规文件: {item}")

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
    
    # 确保源路径有正确权限
    if os.path.isdir(args.content_path):
        set_file_permissions(args.content_path)
    elif os.path.isfile(args.content_path):
        set_file_permissions(args.content_path)
    
    # 创建硬链接结构
    create_hardlinks(args.content_path, dest_base, rel_path)
    
    # 完成
    logging.info("处理完成！")

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