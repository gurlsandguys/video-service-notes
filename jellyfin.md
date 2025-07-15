# jellyfin 配置


## 配置方式

使用docker compose。

- 需新建专属用户 
- 提前建立相关volumes目录，并设置权限


### docker compose 配置

不走traefik网关的原因是，需要使用host模式，使得jellyfin可以正常访问非docker的xray，以刮削电影信息。


```yaml
#networks:
#  traefik:
#    external: true

services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    user: "996:986"  # jellyfin 用户的 UID:GID
    group_add:    #非root运行需加入render组,自行查找render组ID，解决调用失败问题
      - "993"
    network_mode: 'host'
    #networks:
    #  - traefik
    volumes:
      - /opt/jellyfin/config:/config
      - /home/jellyfin_cache:/cache   # 缓存目录,建议设置在SSD
      - /storage/jellyfin-data/media:/media
      - /storage/jellyfin-data/media2:/media2:ro
      - /usr/share/fonts/:/usr/share/fonts/
      - /storage/jellyfin-data/fonts:/usr/local/share/fonts/custom:ro
    restart: unless-stopped
    runtime: nvidia   #详见下方GPU加速节
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    environment:
      - JELLYFIN_PublishedServerUrl=https://movie.tanzmiao.com
      - http_proxy=http://172.17.0.1:1080
      - https_proxy=http://172.17.0.1:1080
    #labels:
    #  - "traefik.enable=true"
    #  - "traefik.http.routers.jellyfin.rule=Host(`movie.tanzmiao.com`)"
    #  - "traefik.http.routers.jellyfin.entrypoints=websecure"
    #  - "traefik.http.routers.jellyfin.tls=true"
    #  - "traefik.http.routers.jellyfin.tls.certresolver=zerossl"
    #  - "traefik.http.services.jellyfin.loadbalancer.server.port=8096"
```

### GPU加速

启用GPU加速，需安装 nvidia 驱动和 nvidia-docker-container ，jellyfin 的 WEB 界面可跳转官网文档。
 
nvdia runtime 需在docker config`vim /etc/docker/daemon.json`中添加启用
```
{
...
"runtimes": {
    "nvidia": {
      "path": "/usr/bin/nvidia-container-runtime",
      "runtimeArgs": []
    }
  }
}
```
在需要使用nvidia runtime 的docker compose中添加`runtime: nvidia` 

进入容器`docker exec -it jellyfin /bin/bash`，运行`nvidia-smi`查看是否正常。

## 其他自用工具

### 字幕文件自动监视、修复、重命名

**注意需要配置jellyfin相关目录own为jellyfin及其组 ，并至少设置770权限** 

#### 配置

在`/storage/jellyfin-data`（dcoker compose设置的主媒体文件目录）下，新建python脚本`subtitle_monitor.py`，内容如下(来自deepseak，已测试，需先安装`pip install watchdog chardet`)：


```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import os
import re
import time
import logging
import argparse
import shutil
import sys
from chardet import detect
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

# 配置日志
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(message)s',
    datefmt='%Y-%m-%d %H:%M:%S'
)
logger = logging.getLogger('SubMonitor')

# 常量定义
MOVIE_ROOT = '/storage/jellyfin-data/media/movie'
TVSHOW_ROOT = '/storage/jellyfin-data/media/tvshow'
VIDEO_EXTENSIONS = ['.mp4', '.mkv', '.avi', '.mov', '.wmv', '.flv', '.m4v', '.ts', '.webm']
SUBTITLE_EXTENSIONS = ['.ass', '.srt', '.ssa', '.sub']

class SubtitleHandler(FileSystemEventHandler):
    def __init__(self, watch_dir):
        self.watch_dir = watch_dir
        self.processed_files = set()
        self.processing_lock = set()
        self.max_retries = 5
        self.retry_delay = 1  # 秒
        
    def on_created(self, event):
        if not event.is_directory:
            self.process_file(event.src_path)
    
    def on_modified(self, event):
        if not event.is_directory:
            self.process_file(event.src_path)
    
    def safe_file_exists(self, filepath):
        """安全检查文件是否存在，解决特殊字符问题"""
        try:
            return os.path.exists(filepath)
        except UnicodeEncodeError:
            # 尝试使用原始字节路径
            try:
                return os.path.exists(filepath.encode('utf-8'))
            except:
                return False
        except:
            return False
    
    def process_file(self, filepath):
        """处理字幕文件的主函数"""
        # 只处理字幕文件
        if not any(filepath.lower().endswith(ext) for ext in SUBTITLE_EXTENSIONS):
            return
            
        # 避免重复处理和重入
        if filepath in self.processing_lock:
            return
            
        # 避免重复处理
        file_key = (filepath, os.path.getmtime(filepath) if self.safe_file_exists(filepath) else 0)
        if file_key in self.processed_files:
            return
            
        logger.info(f"检测到字幕文件: {filepath}")
        self.processing_lock.add(filepath)
        
        try:
            # 尝试多次检查文件是否存在
            for attempt in range(self.max_retries):
                if self.safe_file_exists(filepath):
                    break
                logger.info(f"文件不存在，尝试 {attempt+1}/{self.max_retries}: {filepath}")
                time.sleep(self.retry_delay)
            else:
                logger.warning(f"文件多次检查仍不存在，跳过处理: {filepath}")
                return
                
            # 等待文件传输完成
            initial_size = os.path.getsize(filepath)
            time.sleep(0.5)  # 缩短等待时间
            if self.safe_file_exists(filepath) and os.path.getsize(filepath) != initial_size:
                logger.info("文件仍在传输中，等待完成...")
                time.sleep(2)
            
            # 修复字幕
            if self.fix_subtitle_file(filepath):
                logger.info(f"成功修复字幕: {filepath}")
            
            # 重命名字幕 - 仅当文件存在时
            if self.safe_file_exists(filepath):
                self.rename_subtitle_file(filepath)
            
            # 记录已处理
            if self.safe_file_exists(filepath):
                new_key = (filepath, os.path.getmtime(filepath))
                self.processed_files.add(new_key)
            
        except Exception as e:
            logger.error(f"处理失败 {filepath}: {str(e)}")
        finally:
            if filepath in self.processing_lock:
                self.processing_lock.remove(filepath)
    
    def fix_subtitle_file(self, filepath):
        """修复字幕文件并返回是否修改"""
        if not self.safe_file_exists(filepath):
            return False
            
        modified = False
        
        try:
            # 读取文件内容
            with open(filepath, 'rb') as f:
                raw_content = f.read()
            
            # 检测编码
            encoding = detect(raw_content)['encoding'] or 'utf-8'
            
            try:
                # 尝试解码
                content = raw_content.decode(encoding, errors='replace')
            except UnicodeDecodeError:
                # 尝试常见编码
                for enc in ['gbk', 'big5', 'latin1']:
                    try:
                        content = raw_content.decode(enc, errors='replace')
                        encoding = enc
                        break
                    except:
                        continue
                else:
                    content = raw_content.decode('utf-8', errors='replace')
            
            # 移除UTF-8 BOM
            if content.startswith('\ufeff'):
                content = content[1:]
                modified = True
            
            # ASS文件特殊处理 - 仅当是ASS文件时
            if filepath.lower().endswith('.ass'):
                # 确保有[Script Info]头部
                if not re.search(r'^\[Script Info\]', content, re.IGNORECASE | re.MULTILINE):
                    content = '[Script Info]\n' + content
                    modified = True
                
                # 确保有[Events]部分
                if not re.search(r'^\[Events\]', content, re.IGNORECASE | re.MULTILINE):
                    content += '\n\n[Events]\nFormat: Layer, Start, End, Style, Name, MarginL, MarginR, MarginV, Effect, Text'
                    modified = True
            
            # 标准化换行符
            if '\r\n' in content:
                content = content.replace('\r\n', '\n')
                modified = True
            
            # 如果有修改则保存
            if modified:
                with open(filepath, 'w', encoding='utf-8') as f:
                    f.write(content)
                return True
            
            # 检查是否需要转换编码
            if encoding.lower() not in ['utf-8', 'ascii']:
                with open(filepath, 'w', encoding='utf-8') as f:
                    f.write(content)
                return True
                
        except Exception as e:
            logger.error(f"修复字幕失败: {filepath} - {str(e)}")
            
        return False
    
    def rename_subtitle_file(self, filepath):
        """根据媒体类型智能重命名字幕文件"""
        if not self.safe_file_exists(filepath):
            return
            
        try:
            # 确定媒体类型
            if filepath.startswith(MOVIE_ROOT):
                self.rename_movie_subtitle(filepath)
            elif filepath.startswith(TVSHOW_ROOT):
                self.rename_tvshow_subtitle(filepath)
            else:
                logger.warning(f"无法识别媒体类型: {filepath}")
        except Exception as e:
            logger.error(f"重命名字幕失败: {filepath} - {str(e)}")
    
    def get_video_file(self, directory):
        """获取目录中最可能的主视频文件"""
        video_files = []
        for file in os.listdir(directory):
            file_path = os.path.join(directory, file)
            if not self.safe_file_exists(file_path):
                continue
                
            if any(file.lower().endswith(ext) for ext in VIDEO_EXTENSIONS):
                # 排除样本文件和预告片
                if 'sample' in file.lower() or 'trailer' in file.lower():
                    continue
                    
                try:
                    # 优先选择最大的文件
                    size = os.path.getsize(file_path)
                    # 优先选择包含主剧集标识的文件
                    priority = 0
                    if 'main' in file.lower() or 'feature' in file.lower():
                        priority += 100
                    if 'extras' not in file.lower():
                        priority += 50
                    
                    video_files.append((file_path, size, priority))
                except:
                    continue
        
        if not video_files:
            return None
            
        # 按优先级和大小排序
        video_files.sort(key=lambda x: (x[2], x[1]), reverse=True)
        return video_files[0][0]
    
    def rename_movie_subtitle(self, filepath):
        """重命名电影字幕文件"""
        directory = os.path.dirname(filepath)
        video_file = self.get_video_file(directory)
        
        if not video_file:
            logger.warning(f"未找到视频文件: {directory}")
            return
        
        video_name, video_ext = os.path.splitext(os.path.basename(video_file))
        sub_ext = os.path.splitext(filepath)[1]
        new_subtitle_path = os.path.join(directory, f"{video_name}{sub_ext}")
        
        # 如果已经是最佳名称，跳过
        if filepath == new_subtitle_path:
            return
        
        # 如果目标文件已存在，跳过
        if self.safe_file_exists(new_subtitle_path):
            logger.info(f"目标字幕文件已存在: {os.path.basename(new_subtitle_path)}")
            return
        
        # 重命名文件
        os.rename(filepath, new_subtitle_path)
        logger.info(f"电影字幕重命名: {os.path.basename(filepath)} -> {os.path.basename(new_subtitle_path)}")
        
        # 更新处理集合
        if self.safe_file_exists(new_subtitle_path):
            new_key = (new_subtitle_path, os.path.getmtime(new_subtitle_path))
            self.processed_files.add(new_key)
    
    def extract_season_episode(self, filename):
        """更健壮地从文件名中提取季集信息"""
        # 尝试多种常见的季集格式
        patterns = [
            r'[sS](\d{1,2})[eE](\d{1,2})',  # S01E02
            r'(\d{1,2})[xX](\d{1,2})',      # 01x02
            r'第(\d{1,2})季第(\d{1,2})集',   # 第1季第2集
            r'\.(\d)(\d{2})\.',             # .102. (S1E02)
            r'\[(\d{1,2})(\d{2})\]',        # [102] (S1E02)
        ]
        
        for pattern in patterns:
            match = re.search(pattern, filename)
            if match:
                season = match.group(1)
                episode = match.group(2)
                
                # 对于 .102. 格式，season是1，episode是02
                if pattern == r'\.(\d)(\d{2})\.' or pattern == r'\[(\d{1,2})(\d{2})\]':
                    season = str(int(season))
                    episode = str(int(episode))
                
                return season, episode
        
        # 尝试从文件名中提取纯数字季集
        digits = re.findall(r'\d+', filename)
        if len(digits) >= 2:
            return digits[0], digits[1]
        
        return None, None
    
    def rename_tvshow_subtitle(self, filepath):
        """重命名电视剧字幕文件"""
        directory = os.path.dirname(filepath)
        current_sub = os.path.basename(filepath)
        
        # 1. 从字幕文件名提取季集信息
        season, episode = self.extract_season_episode(current_sub)
        
        if not season or not episode:
            logger.warning(f"无法提取季集信息: {current_sub}")
            return
        
        sxxexx = f"S{season.zfill(2)}E{episode.zfill(2)}"
        
        # 2. 查找匹配的视频文件
        video_file = None
        for file in os.listdir(directory):
            file_path = os.path.join(directory, file)
            if not self.safe_file_exists(file_path):
                continue
                
            if any(file.lower().endswith(ext) for ext in VIDEO_EXTENSIONS):
                # 排除样本文件和预告片
                if 'sample' in file.lower() or 'trailer' in file.lower():
                    continue
                
                # 检查是否匹配季集
                if sxxexx.lower() in file.lower():
                    video_file = file_path
                    break
                elif f"{int(season)}{int(episode):02d}" in file:  # 匹配102格式
                    video_file = file_path
                    break
        
        if not video_file:
            logger.warning(f"未找到匹配的视频文件: {sxxexx} in {directory}")
            return
        
        # 3. 重命名字幕
        video_name, video_ext = os.path.splitext(os.path.basename(video_file))
        sub_ext = os.path.splitext(filepath)[1]
        new_subtitle_path = os.path.join(directory, f"{video_name}{sub_ext}")
        
        # 如果已经是最佳名称，跳过
        if filepath == new_subtitle_path:
            return
        
        # 如果目标文件已存在，跳过
        if self.safe_file_exists(new_subtitle_path):
            logger.info(f"目标字幕文件已存在: {os.path.basename(new_subtitle_path)}")
            return
        
        # 重命名文件
        os.rename(filepath, new_subtitle_path)
        logger.info(f"电视剧字幕重命名: {current_sub} -> {os.path.basename(new_subtitle_path)}")
        
        # 更新处理集合
        if self.safe_file_exists(new_subtitle_path):
            new_key = (new_subtitle_path, os.path.getmtime(new_subtitle_path))
            self.processed_files.add(new_key)
    
    def process_existing_files(self, directory):
        """处理目录中已有的字幕文件"""
        logger.info(f"扫描现有文件: {directory}")
        for root, _, files in os.walk(directory):
            for file in files:
                filepath = os.path.join(root, file)
                # 检查文件是否存在
                if not self.safe_file_exists(filepath):
                    continue
                    
                if any(file.lower().endswith(ext) for ext in SUBTITLE_EXTENSIONS):
                    self.process_file(filepath)

def start_monitoring(path):
    logger.info(f"开始监控目录: {path}")
    event_handler = SubtitleHandler(path)
    observer = Observer()
    observer.schedule(event_handler, path, recursive=True)
    
    # 启动前先处理现有文件
    event_handler.process_existing_files(path)
    
    observer.start()
    
    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        observer.stop()
    observer.join()

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description='字幕文件监控修复服务')
    parser.add_argument('directory', help='要监控的目录路径')
    args = parser.parse_args()
    
    # 确保目录存在
    if not os.path.exists(args.directory):
        logger.error(f"目录不存在: {args.directory}")
        sys.exit(1)
    
    # 启动监控
    start_monitoring(args.directory)
```

创建系统服务文件：`vim /etc/systemd/system/subtitle-monitor.service`

```bash
[Unit]
Description=Subtitle Monitor Service
After=network.target

[Service]
# /storage/jellyfin-data/media 为需要监视的总目录，设置为web上总的媒体库
ExecStart=/usr/bin/python3 /storage/jellyfin-data/subtitle_monitor.py /storage/jellyfin-data/media
WorkingDirectory=/storage/jellyfin-data
Restart=always
User=root

[Install]
WantedBy=multi-user.target
```

添加执行权限：`chmod +x /storage/jellyfin-data/subtitle_monitor.py`

启动服务：`systemctl start subtitle-monitor`

设置开机自启：`systemctl enable subtitle-monitor`

查看运行状态：`systemctl status subtitle-monitor`

查看日志命令：`journalctl -u subtitle-monitor -f`

#### 功能

1. 监控指定目录下的字幕文件，当有新字幕文件添加时(包括硬链接产生)，自动检查编码、修复、重命名。
2. 支持电视剧和电影字幕的自动重命名，确保字幕文件与视频文件名一致。需将字幕放到视频文件的同目录。电影：寻找此目录下最大的视频文件名。电视剧：根据当前目录下文件的`S01E01`等文件名匹配。
3. 支持自动跳过已处理过的文件，避免重复处理。
4. 支持自动跳过样本文件和预告片文件，避免误处理。

#### 使用说明

- **首次启动**时，会扫描指定目录下的所有字幕文件，并自动修复、重命名。后续新增字幕文件时，会自动触发处理。*（耗时较久，100字幕文件约100s）*

- 启动服务后，将字幕文件放在对应视频目录内，即可自动整理，供jellyfin识别。

*建议：配合本note下的qbittorrent-nox版的自定义硬链接脚本使用，在下载完成后，自动将视频、字幕等链接到媒体库，触发监控脚本。*

---

### 缺失字幕检查工具

#### 配置

在`/storage/jellyfin-data`（dcoker compose设置的主媒体文件目录）下，新建python脚本`check_missing_sub.py`，内容如下(来自deepseak，已测试)：
```
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
字幕文件缺失检测工具
功能：检测目录中缺少同名字幕文件(.ass .srt)的视频文件(.mkv, .mp4)
"""

import os
import sys
import argparse
from collections import defaultdict
from typing import List, Dict, Tuple

def find_videos_without_subtitles(directory: str) -> Dict[str, List[Tuple[str, str]]]:
    """
    查找缺少字幕文件的视频文件
    
    参数:
        directory: 要检查的目录路径
        
    返回:
        字典: {目录路径: [(视频文件名, 期望的字幕文件名)]}
    """
    # 存储结果：目录 -> (视频文件, 期望字幕文件)
    results = defaultdict(list)
    
    # 支持的视频扩展名
    video_exts = {'.mkv', '.mp4'}
    
    # 遍历目录树
    for root, _, files in os.walk(directory):
        # 收集当前目录下的视频文件和字幕文件
        video_files = []
        sub_files = set()
        
        for file in files:
            name, ext = os.path.splitext(file)
            ext = ext.lower()
            
            if ext in video_exts:
                video_files.append((file, name))
            elif (ext == '.ass') or (ext == '.srt') :
                sub_files.add(name.lower())  # 使用小写进行比较，不区分大小写
        
        # 检查每个视频文件是否有对应的字幕
        for video_file, base_name in video_files:
            # 检查基本名称（不区分大小写）
            if base_name.lower() not in sub_files:
                expected_sub = f"{base_name}.ass"
                results[root].append((video_file, expected_sub))
    
    return results

def print_results(results: Dict[str, List[Tuple[str, str]]]):
    """
    打印检测结果
    
    参数:
        results: 检测结果字典
    """
    if not results:
        print("未发现缺少字幕的视频文件")
        return
        
    total_missing = sum(len(files) for files in results.values())
    print(f"发现 {len(results)} 个目录中有 {total_missing} 个视频文件缺少字幕:")
    print("=" * 80)
    
    for i, (directory, files) in enumerate(results.items(), 1):
        print(f"\n#{i} 目录: {directory}")
        print("-" * 60)
        
        for video, expected_sub in files:
            print(f"  视频文件: {video}")
            print(f"  缺少字幕: {expected_sub}")
            print(f"  建议添加: {os.path.join(directory, expected_sub)}")
            print("-" * 40)
    
    print("\n" + "=" * 80)
    print(f"总计: {total_missing} 个视频文件缺少字幕")

def main():
    """主函数"""
    # 解析命令行参数
    parser = argparse.ArgumentParser(
        description="字幕文件缺失检测工具",
        formatter_class=argparse.ArgumentDefaultsHelpFormatter
    )
    parser.add_argument("directory", nargs="?", default=".", 
                        help="要检查的目录路径 (默认为当前目录)")
    args = parser.parse_args()
    
    # 获取绝对路径
    directory = os.path.abspath(args.directory)
    
    # 验证目录
    if not os.path.isdir(directory):
        print(f"错误: 目录不存在 - {directory}")
        sys.exit(1)
    
    print(f"开始扫描目录: {directory}")
    results = find_videos_without_subtitles(directory)
    
    print_results(results)

if __name__ == "__main__":
    main()
```

#### 使用说明

设置权限：`chmod +x /storage/jellyfin-data/check_missing_sub.py`

直接运行`/storage/jellyfin-data/check_missing_sub.py`，默认扫描本目录及子目录下的视频文件，并检查缺少字幕的文件并输出。

如需指定目录，使用`/storage/jellyfin-data/check_missing_sub.py /path/to/directory`。

*注意：此工具仅检查缺少字幕的视频文件，不会自动下载字幕。并且不剔除可能的samle等样片。*