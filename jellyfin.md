# jellyfin 配置


## 配置方式

使用docker compose。

- 需新建专属用户 jellyfin，无登录、无目录：`useradd -M -s /bin/false jellyfin`
- 提前建立相关volumes目录，并设置权限


### docker compose 配置

不走traefik网桥的原因是，需要使用host模式，使得jellyfin可以正常访问非docker的xray，以刮削themoviedb等数据源的电影信息。


```yaml
#networks:
#  traefik:
#    external: true

services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    user: "996:986"  # jellyfin 用户的 UID:GID
    group_add:    #非root运行需加入render组,getent group render | cut -d: -f3，解决调用失败问题
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
      - JELLYFIN_PublishedServerUrl=https://movie.example.com
      - http_proxy=http://172.17.0.1:1080   #自行配置代理，或不使用代理（无法刮削国外数据库，可搜索刮削豆瓣插件）
      - https_proxy=http://172.17.0.1:1080  #自行配置代理，或不使用代理（无法刮削国外数据库，可搜索刮削豆瓣插件）
    #labels:
    #  - "traefik.enable=true"
    #  - "traefik.http.routers.jellyfin.rule=Host(`movie.example.com`)"
    #  - "traefik.http.routers.jellyfin.entrypoints=websecure"
    #  - "traefik.http.routers.jellyfin.tls=true"
    #  - "traefik.http.routers.jellyfin.tls.certresolver=zerossl"
    #  - "traefik.http.services.jellyfin.loadbalancer.server.port=8096"
```

### GPU加速

启用GPU加速，需安装 nvidia 驱动和 `nvidia-docker-container` ，jellyfin 的 WEB 界面-控制台-播放-转码-硬件加速帮助可跳转jellyfin官网文档，按照教程安装，或问AI。
 
**nvdia runtime 需在docker config`vim /etc/docker/daemon.json`中添加启用**
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
重启docker：`systemctl restart docker`

然后在需要使用nvidia runtime 的docker compose中添加`runtime: nvidia` 

进入容器`docker exec -it jellyfin /bin/bash` 显示无用户,exit退出docker容器后变为执行：`docker exec -u root -it jellyfin /bin/bash`，运行`nvidia-smi`查看是否正常。

测试播放，客户端播放器调低码率，触发jellyfin转码，然后服务器运行`nvidia-smi`查看是否有ffmpeg工作，检查jellyfin的日志`docker logs -f jellyfin`，排查是否有报错、权限等问题。（常见ffmepg对cache等无写入权限问题，exit code: 187）

---
---

## 其他自用工具

### 字幕文件自动监视、修复、重命名

**注意需要配置jellyfin相关目录own为jellyfin及其组 ，并至少设置770权限** 

#### 配置

在`/storage/jellyfin-data`（dcoker compose设置的主媒体文件目录）下，新建python脚本`subtitle_monitor.py`，内容如下(来自deepseek，已测试，需先安装`apt install python3-watchdog python3-chardet`)：


```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
import os
import re
import time
import logging
import argparse
import sys
import sqlite3
import threading
import queue
from chardet import detect
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

# ----------------- 日志配置 -----------------
logging.basicConfig(
    level=logging.INFO,
    format='[%(levelname)s] %(message)s'
)
logger = logging.getLogger('SubMonitor')

# ----------------- 常量 -----------------
MOVIE_ROOT = '/storage/jellyfin-data/media/movie'   #对应web页面添加的媒体库目录
TVSHOW_ROOT = '/storage/jellyfin-data/media/tvshow' #对应web页面添加的媒体库目录
VIDEO_EXTENSIONS = ['.mp4', '.mkv', '.avi', '.mov', '.wmv', '.flv', '.m4v', '.ts', '.webm']
SUBTITLE_EXTENSIONS = ['.ass', '.srt', '.ssa', '.sub']

# ----------------- SQLite 管理 -----------------
class ProcessedDB:
    def __init__(self, db_path='/var/lib/sub_monitor.db'):
        os.makedirs(os.path.dirname(db_path), exist_ok=True)
        self.conn = sqlite3.connect(db_path, check_same_thread=False)
        self.lock = threading.Lock()
        self._init_db()

    def _init_db(self):
        with self.lock, self.conn:
            self.conn.execute("""
                CREATE TABLE IF NOT EXISTS processed_files (
                    path TEXT PRIMARY KEY,
                    mtime REAL
                )
            """)

    def is_processed(self, path):
        try:
            mtime = os.path.getmtime(path)
        except FileNotFoundError:
            return True
        with self.lock:
            cur = self.conn.execute(
                "SELECT mtime FROM processed_files WHERE path=?",
                (path,)
            ).fetchone()
            return cur is not None and cur[0] == mtime

    def mark_processed(self, path):
        try:
            mtime = os.path.getmtime(path)
        except FileNotFoundError:
            return
        with self.lock, self.conn:
            self.conn.execute(
                "REPLACE INTO processed_files (path, mtime) VALUES (?, ?)",
                (path, mtime)
            )


# ----------------- 字幕处理类 -----------------
class SubtitleProcessor:
    def __init__(self, db: ProcessedDB):
        self.db = db

    def safe_exists(self, filepath):
        try:
            return os.path.exists(filepath)
        except:
            return False

    def process_subtitle(self, filepath):
        """主处理函数"""
        if not any(filepath.lower().endswith(ext) for ext in SUBTITLE_EXTENSIONS):
            return
        if self.db.is_processed(filepath):
            return

        logger.info(f"检测到字幕文件: {filepath}")

        try:
            if self.fix_subtitle_file(filepath):
                logger.info(f"修复完成: {os.path.basename(filepath)}")
            if self.safe_exists(filepath):
                self.rename_subtitle_file(filepath)
            self.db.mark_processed(filepath)
            logger.info(f"字幕处理完成: {filepath}")
        except Exception as e:
            logger.error(f"处理失败 {filepath}: {str(e)}")

    def fix_subtitle_file(self, filepath):
        """修复字幕编码/头部"""
        if not self.safe_exists(filepath):
            return False
        modified = False
        try:
            with open(filepath, 'rb') as f:
                raw = f.read()

            encoding = detect(raw)['encoding'] or 'utf-8'
            try:
                content = raw.decode(encoding, errors='replace')
            except:
                for enc in ['utf-8', 'gbk', 'latin1']:
                    try:
                        content = raw.decode(enc, errors='replace')
                        break
                    except:
                        continue

            if content.startswith('\ufeff'):
                content = content[1:]
                modified = True

            if filepath.lower().endswith('.ass'):
                if not re.search(r'^\[Script Info\]', content, re.I | re.M):
                    content = '[Script Info]\n' + content
                    modified = True
                if not re.search(r'^\[Events\]', content, re.I | re.M):
                    content += '\n\n[Events]\nFormat: Layer, Start, End, Style, Name, MarginL, MarginR, MarginV, Effect, Text'
                    modified = True

            if '\r\n' in content:
                content = content.replace('\r\n', '\n')
                modified = True

            if modified or encoding.lower() not in ['utf-8', 'ascii']:
                with open(filepath, 'w', encoding='utf-8') as f:
                    f.write(content)
                return True
        except Exception as e:
            logger.error(f"修复字幕失败: {filepath} - {str(e)}")
        return False

    def rename_subtitle_file(self, filepath):
        """根据媒体类型重命名字幕"""
        if filepath.startswith(MOVIE_ROOT):
            self.rename_movie_subtitle(filepath)
        elif filepath.startswith(TVSHOW_ROOT):
            self.rename_tvshow_subtitle(filepath)
        else:
            logger.warning(f"无法识别媒体类型: {filepath}")

    def get_video_file(self, directory):
        files = []
        try:
            for entry in os.scandir(directory):
                if entry.is_file() and any(entry.name.lower().endswith(ext) for ext in VIDEO_EXTENSIONS):
                    if 'sample' in entry.name.lower() or 'trailer' in entry.name.lower():
                        continue
                    try:
                        size = entry.stat().st_size
                        files.append((entry.path, size))
                    except:
                        continue
        except:
            return None
        if not files:
            return None
        files.sort(key=lambda x: x[1], reverse=True)
        return files[0][0]

    def rename_movie_subtitle(self, filepath):
        directory = os.path.dirname(filepath)
        video_file = self.get_video_file(directory)
        if not video_file:
            return
        video_name = os.path.splitext(os.path.basename(video_file))[0]
        sub_ext = os.path.splitext(filepath)[1]
        new_path = os.path.join(directory, f"{video_name}{sub_ext}")
        if filepath != new_path and not self.safe_exists(new_path):
            os.rename(filepath, new_path)
            logger.info(f"电影字幕重命名: {os.path.basename(filepath)} -> {os.path.basename(new_path)}")

    def extract_season_episode(self, filename):
        patterns = [
            r'[sS](\d{1,2})[eE](\d{1,2})',
            r'(\d{1,2})[xX](\d{1,2})',
            r'第(\d{1,2})季第(\d{1,2})集'
        ]
        for p in patterns:
            m = re.search(p, filename)
            if m:
                return m.group(1), m.group(2)
        return None, None

    def rename_tvshow_subtitle(self, filepath):
        directory = os.path.dirname(filepath)
        season, episode = self.extract_season_episode(os.path.basename(filepath))
        if not season or not episode:
            return
        sxxexx = f"S{season.zfill(2)}E{episode.zfill(2)}"
        video_file = None
        try:
            for entry in os.scandir(directory):
                if entry.is_file() and any(entry.name.lower().endswith(ext) for ext in VIDEO_EXTENSIONS):
                    if sxxexx.lower() in entry.name.lower():
                        video_file = entry.path
                        break
        except:
            return
        if not video_file:
            return
        video_name = os.path.splitext(os.path.basename(video_file))[0]
        sub_ext = os.path.splitext(filepath)[1]
        new_path = os.path.join(directory, f"{video_name}{sub_ext}")
        if filepath != new_path and not self.safe_exists(new_path):
            os.rename(filepath, new_path)
            logger.info(f"电视剧字幕重命名: {os.path.basename(filepath)} -> {os.path.basename(new_path)}")


# ----------------- 文件事件处理 -----------------
class SubtitleHandler(FileSystemEventHandler):
    def __init__(self, task_queue):
        self.task_queue = task_queue

    def on_created(self, event):
        if not event.is_directory:
            self.task_queue.put(event.src_path)


# ----------------- 主监控类 -----------------
class SubtitleMonitor:
    def __init__(self, watch_dir, db_path='/var/lib/sub_monitor.db', workers=4):
        self.watch_dir = watch_dir
        self.db = ProcessedDB(db_path)
        self.task_queue = queue.Queue()
        self.processor = SubtitleProcessor(self.db)
        self.workers = workers
        self.stop_event = threading.Event()

    def scan_existing(self):
        logger.info(f"启动字幕监控服务，监控目录：{self.watch_dir}")
        logger.info("正在扫描已有字幕文件...")
        total = 0
        added = 0
        start = time.time()
        for root, dirs, files in self._walk(self.watch_dir):
            for f in files:
                if any(f.lower().endswith(ext) for ext in SUBTITLE_EXTENSIONS):
                    total += 1
                    path = os.path.join(root, f)
                    if not self.db.is_processed(path):
                        self.task_queue.put(path)
                        added += 1
                        if added % 50 == 0:
                            logger.info(f"扫描进度：已加入 {added} / {total} 个文件")
        logger.info(f"初始扫描完成，总计 {total} 个字幕文件，其中 {added} 个待处理。耗时 {time.time()-start:.1f} 秒")
        logger.info("启动实时监控...")

    def _walk(self, directory):
        """高效目录遍历"""
        dirs, files = [], []
        try:
            for entry in os.scandir(directory):
                if entry.is_dir(follow_symlinks=False):
                    dirs.append(entry.path)
                elif entry.is_file():
                    files.append(entry.name)
        except PermissionError:
            return [], []
        yield directory, dirs, files
        for d in dirs:
            yield from self._walk(d)

    def worker(self, worker_id=0):
        
        last_idle_log = 0
        while not self.stop_event.is_set():
            try:
                filepath = self.task_queue.get(timeout=3)
                self.processor.process_subtitle(filepath)
                self.task_queue.task_done()
            except queue.Empty:
                # 仅第 0 号 worker 打印，且间隔 30 秒
                if worker_id == 0 and time.time() - last_idle_log > 30:
                    logger.info("处理队列为空，继续监控中...")
                    last_idle_log = time.time()

    def start(self):
        self.scan_existing()

        # 启动线程
        for i in range(self.workers):
            t = threading.Thread(target=self.worker, args=(i,), daemon=True)
            t.start()

        # 启动 watchdog
        event_handler = SubtitleHandler(self.task_queue)
        observer = Observer()
        observer.schedule(event_handler, self.watch_dir, recursive=True)
        observer.start()

        try:
            while True:
                time.sleep(1)
        except KeyboardInterrupt:
            self.stop_event.set()
            observer.stop()
        observer.join()


# ----------------- 启动入口 -----------------
if __name__ == "__main__":
    parser = argparse.ArgumentParser(description='字幕监控与修复服务')
    parser.add_argument('directory', help='要监控的目录路径')
    parser.add_argument('--db', default='/var/lib/sub_monitor.db', help='SQLite 数据库路径')
    args = parser.parse_args()

    if not os.path.exists(args.directory):
        logger.error(f"目录不存在: {args.directory}")
        sys.exit(1)

    monitor = SubtitleMonitor(args.directory, db_path=args.db)
    monitor.start()
```

创建系统服务文件：`vim /etc/systemd/system/subtitle-monitor.service`

```bash
[Unit]
Description=Subtitle Monitor and Auto Fix Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /storage/jellyfin-data/subtitle_monitor.py /storage/jellyfin-data/media
WorkingDirectory=/storage/jellyfin-data
Restart=on-failure
RestartSec=5
# 限制最大重启次数，避免频繁重启（默认10次/10秒就被判定为失败）
StartLimitInterval=30
StartLimitBurst=5

# 日志直接写入 journald
StandardOutput=journal
StandardError=journal

User=root
Group=jellyfin

# 环境变量
Environment="PYTHONUNBUFFERED=1"
Environment="LANG=en_US.UTF-8"

# 文件句柄限制
LimitNOFILE=65535

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
   
   *为何修复？从win等其他设备，手动通过ftp等软件导入字幕到linux服务器，会因为编码不同导致ass等格式文件无法被jellyfin识别，有时ass转码还会丢失头部信息。*

2. 支持电视剧和电影字幕的自动重命名，确保字幕文件与视频文件名一致。需将字幕放到视频文件的同目录。电影：寻找此目录下最大的视频文件名。电视剧：根据当前目录下文件的`S01E01`等文件名匹配。
3. 支持自动跳过已处理过的文件，避免重复处理。
4. 支持自动跳过样本文件和预告片文件，避免误处理。

#### 使用说明

- **首次启动**时，会扫描指定目录下的所有字幕文件，并自动修复、重命名。**后续新增字幕文件时，会自动触发处理。**

- 启动服务后，将字幕文件放在对应视频目录内，即可自动检查并重命名，供jellyfin识别。

*建议：配合本note下的qbittorrent-nox版的自定义硬链接脚本使用，在下载完成后，qb自动将视频、手动导入的字幕等硬链接到媒体库，触发监控脚本。日常使用见qbittorrent-nox.md*

---

### 缺失字幕检查工具

#### 配置

在`/storage/jellyfin-data/media`（dcoker compose及jellyfin web设置的主媒体文件目录）下，新建python脚本`check_missing_sub.py`，内容如下(来自deepseek，已测试)：
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

设置权限：`chmod +x /storage/jellyfin-data/media/check_missing_sub.py`

直接运行`/storage/jellyfin-data/media/check_missing_sub.py`，默认扫描本目录及子目录下的视频文件，检查缺少字幕的文件并输出信息。

如需指定目录，使用`/storage/jellyfin-data/media/check_missing_sub.py /path/to/directory`。

*注意：此工具仅检查缺少字幕的视频文件，不会自动下载字幕。并且不剔除可能的sample等样片。*
