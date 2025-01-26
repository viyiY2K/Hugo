---
url: a_record_of_the_new_burnout_worker_hiring_ai
title: 新晋社畜雇佣 AI 实录
toc: true
authors: viyi
tags:
  - AI
  - Python
categories:
  - 札记
description: 本文记录了一位绝望的社畜利用 AI 完成琐碎事项的自动化过程。
date: 2025-01-26
lastmod: 2025-01-26
---

> 我们觉得，没有必要因为世界表现得荒谬，我们就随着它一起来做荒谬之事。——斯蒂芬·茨威格「昨日的世界」

诚然，我知道一份工作里不可避免的会存在 Dirty work，但我确实没料到它占据了绝大部分。作为某个头部博主团队里的新媒体运营，我日常工作里最无聊的是，监测视频情况、检查是否达成 KPI 并且定时收录相关数据。

老板娘理直气壮地要求我，就算是周五发视频，周末也要“时不时”看一下数据。是的，就算是双休也并不代表我拥有离线权🙂。我的职业素养告诉我，这确实是应当的，视频发出并不代表这个项目结束了，运营维护和数据分析也很重要。

如果我有对应的职权，可以根据数据情况做进一步的应对，我想它不会显得那么无聊。事实上，我既没有预算用于投流，也无法参与内容制作，连视频的标封也无权过问。甚至，在我对视频评论区做文本情感分析的时候，她对着数据表评价了一句「没必要」。

因此，我想她只是需要我转播数据情况给她。

理论上来说，这件事并不需要耗费太多时间。它只是很琐碎，B 站、微博、公众号、抖音、小红书、视频号，光是简中就有 6 个平台，再加上海外的 YouTube——这意味着我需要汇总 7 个平台的播放量以及相关的互动数据（实际上分发总计是 12 个平台）。

与此同时，我也需要关注这些平台里的评论，是否有观众指出视频中的片源问题和事实性错误，抑或是对论证过程、测试细节的质疑。这些有损品牌形象的评论，只会占非常小的比例，但一旦出现又没有及时响应，很容易造成舆情危机。当然，考虑到在各个平台影响力不同，我通常也只关注 B 站，顺带看看微博和 YouTube。

这两件事让这份工作很大程度上变成了 [Bullshit Jobs](https://www.marxists.org/chinese/david-graeber/2018/02.htm)，转播数据情况是在充当老板娘的 flunky，而紧盯评论区通常都是在给制作团队当 duct taper。

我们大多数时候换源是因为评论区指出画面标注错误、剪辑存在误导性、字幕错误等片源问题，而如果这个团队有合理的审片流程，这些显然都可以避免。很不幸，我目睹了排期紧张导致无法预留出内审时间的项目管理惨状，再加上曾经因为推动内审成为那个被 Shooting 的 The Messenger，现在只能表示无能为力。

这篇文章并不是为了抱怨这份工作，尽管我对这些琐碎的工作内容感到厌烦，但还是试图做点什么让它不那么无聊，比如利用自动化工具。

## 定时转播视频数据

### 自动化获取数据

为了避免重复造轮子，先做点简单的调研。如果在搜索引擎查询「社媒矩阵管理」，能够查询到许多工具。这些都是 To B 的服务商，很显然，被雇佣的我大概率比这些工具便宜(。

所幸我只需要监测视频数据，听起来很简单——只需要定时获取各个视频平台的数据，再转播到飞书就可以了。实现起来也不难，如果平台足够开放的话。

虽然这点小需求不涉及什么复杂的技术栈，但起码得选定某种编程语言实现。考虑到 Python 的库支持足够丰富……从零开始学 Python，启动(bushi。如果真的从零开始学，大概这篇文章要再难产久一点。事实上，后文出现的所有代码 ，基本都是 AI 完成的，感谢 Monica 的黑五折扣🙏。

长视频平台中，最重要的莫过于 bilibili 和 YouTube。所幸这俩都很友好，再加上 [bilibili-api](https://github.com/Nemo2011/bilibili-api) 和 [youtube-dl](https://github.com/ytdl-org/youtube-dl) 也是很完善的库，很容易获取到特定视频的播放量和点赞等数据。

B 站特定视频数据抓取的示例代码如下：

```python
import asyncio
from bilibili_api import video

async def main() -> None:
    ## 提示用户输入 BV 号
    bvid = input("请输入 bilibili 视频的 BV 号：")
    
    ## 实例化 Video 类
    v = video.Video(bvid=bvid)
    ## 获取信息
    info = await v.get_info()
    
    ## 筛选并输出所需信息
    output = {
        '标题'：info['title']，
        'BV 号': info['bvid'],
        '视频作者': info['owner']['name'],
        '发布时间': info['pubdate'],
        '统计信息': {
            '播放量': info['stat']['view'],
            '点赞数': info['stat']['like'],
            '评论数': info['stat']['reply'],
            '投币数': info['stat']['coin'],
            '收藏数': info['stat']['favorite'],
            '弹幕数': info['stat']['danmaku']
        }
    }
    
    print(output)

if __name__ == "__main__":
    asyncio.get_event_loop().run_until_complete(main())
```

通过 pip 安装的 youtube-dl 是 2021.12.17 的版本，该版过旧不能顺利提取数据，且无法更新版本（原因未知）。因此这里改用了 yt-dlp，这是 youtube-dl 的一个分支，通常更新更快，功能更强大。YouTube 特定视频数据抓取的示例代码如下：

```python
import yt_dlp

def get_video_info(video_url):
    ydl_opts = {
        'quiet': True,  ## 关闭输出
        'format': 'best',  ## 获取最佳格式
        'noplaylist': True,  ## 不下载播放列表
    }

    with yt_dlp.YoutubeDL(ydl_opts) as ydl:
        info_dict = ydl.extract_info(video_url, download=False)
        title = info_dict.get('title', None)
        view_count = info_dict.get('view_count', None)
        like_count = info_dict.get('like_count', None)
        comment_count = info_dict.get('comment_count', None)

        return title, view_count, like_count, comment_count

video_url = "<https://www.youtube.com/watch?v=>"  ## 替换为需要的视频链接
title, views, likes, comments = get_video_info(video_url)

print(f"标题: {title}")
print(f"观看次数: {views}")
print(f"点赞数: {likes}")
print(f"评论数: {comments}")
```

接下来是微博，至于为什么是它，因为 [Weibo Spider](https://github.com/dataabc/weiboSpider) 是现成的好工具。官方的文档只提供了[转发评论数的读取接口](https://open.weibo.com/wiki/%E5%BE%AE%E5%8D%9AAPI)，还缺少点赞和视频播放量。这个第三方的工具可以获取点赞数，但目前也无法获取微博视频的播放量。

因此在此基础上，我还修改了源码中页面解析相关的内容，比如获取和解析视频信息的函数、修改写入结果文件的表头，并且更新写入方法，确保该项数据被写入文件。

这里容易翻车的是第一步，解决方法是：通过添加调试信息，打印出抓取的原始数据，确认播放量的字段是否存在、在哪个字段中。打印出 Weibo Info 后，可以看到播放量 play_count 的信息是在 page_info 的结构中。这个字段的值是「5 万次播放」，而不是一个数字。相关调试代码如下：

```python
def get_video_url(self, weibo_info):
    """获取微博视频url和播放量"""
    video_url = ""
    play_count = 0  ## 新增播放量变量

    if weibo_info.get("page_info"):
        media_info = weibo_info["page_info"].get("urls") or weibo_info["page_info"].get("media_info")

        ## print("Weibo Info:", weibo_info)  
        ## print("Media Info:", media_info)  
        
        if media_info and weibo_info["page_info"].get("type") == "video":
            video_url = media_info.get("mp4_720p_mp4")

            ## 从 page_info 中获取播放量
            play_count_str = weibo_info["page_info"].get("play_count", "0次播放")
            ## 提取数字部分
            ## 把字符串转换为整数 play_count = int(play_count_str.replace("次播放", "").replace("万", "0000").replace("万次播放", "0000").replace("次", ""))
            match = re.search(r'\\d+', play_count_str)
            play_count = int(match.group()) if match else 0  ## 如果找到数字，则转换为整数，否则为 0

            print("Video URL:", video_url)  
            print("Play Count:", play_count)  
```

Weibo Spider 是根据微博用户 id 访问主页并依序逐条提取微博数据，最终存储到 csv 文件的爬虫程序。它本质上是一种轮查（Polling），数据更新的频率取决于爬取的间隔时间。

按照一开始的需求，我只需要获取单条微博的数据。本来犯懒不想仔细看是怎么实现的，直接从 csv 文件中筛选出对应的单条微博凑合用来着，后来还是忍受不了……因为如果设定的内容爬取周期比较长，加之该期间账号发布的内容较多，这种方法会比较浪费资源。

其实实现起来也不难，移动端的微博数据比较规整也不会有登录限制，解析抓取到的内容就可以了。稍微不太顺手的一点是单条微博 id 的获取，网页端直接分享的短链并不带有微博唯一 id，必须从移动端分享取得。相关示例代码如下：

```python
#!/usr/bin/env python
## -*- coding: utf-8 -*-

import argparse
import requests
import json
from datetime import datetime
import logging
import urllib3

## 禁用 SSL 警告
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

def get_single_weibo(weibo_id, headers=None):
    """获取指定ID的单条微博信息"""
    if headers is None:
        headers = {
            'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/86.0.4240.111 Safari/537.36'
        }
    
    try:
        url = f"<https://m.weibo.cn/detail/{weibo_id}>"
        response = requests.get(url, headers=headers, verify=False)
        html = response.text
        html = html[html.find('"status":'):]
        html = html[:html.rfind('"call"')]
        html = html[:html.rfind(",")]
        html = "{" + html + "}"
        js = json.loads(html, strict=False)
        weibo_info = js.get("status")
        
        if weibo_info:
            ## 提取基本信息
            weibo = {
                'id': weibo_info['id'],
                'screen_name': weibo_info['user']['screen_name'],
                'text': weibo_info['text'],
                'attitudes_count': weibo_info['attitudes_count'],
                'comments_count': weibo_info['comments_count'],
                'reposts_count': weibo_info['reposts_count'],
                'created_at': weibo_info['created_at']
            }
            
            ## 获取视频信息
            if weibo_info.get("page_info"):
                page_info = weibo_info["page_info"]
                if page_info.get("type") == "video":
                    weibo['play_count'] = page_info.get("play_count", "0")
                    weibo['video_title'] = page_info.get("title", "")
            
            return weibo
            
        return None
            
    except Exception as e:
        logger.error(f"获取微博信息出错: {e}")
        return None

def main():
    parser = argparse.ArgumentParser(description='获取微博数据并输出到命令行')
    parser.add_argument('--id', type=str, required=True, help='微博ID')
    args = parser.parse_args()
    
    weibo_info = get_single_weibo(args.id)
    if weibo_info:
        ## 输出微博信息到命令行
        print(f"用户: {weibo_info['screen_name']}")
        print(f"创建时间: {weibo_info['created_at']}")
        print(f"内容: {weibo_info['text']}")
        print(f"点赞数: {weibo_info['attitudes_count']}")
        print(f"评论数: {weibo_info['comments_count']}")
        print(f"转发数: {weibo_info['reposts_count']}")
        if 'play_count' in weibo_info:
            print(f"播放量: {weibo_info['play_count']}")
            print(f"视频标题: {weibo_info['video_title']}")
    else:
        logger.error("获取微博信息失败")

if __name__ == "__main__":
    main()
```

至此，和简中互不联网平台的斗智斗勇现在才刚刚开始。实际上一开始称之为半成品的原因在于，剩下的平台我并未没有成功用较为简单的方式获取到播放数据。

[小红书开放平台](https://open.xiaohongshu.com/home)只提供店铺相关的 API，倒是[有人使用 Appium+Mitmproxy+Fiddler+夜神模拟器](https://github.com/Big-Buffer/XiaohongshuSpider)实现，看起来有点过于麻烦。Github 上还有个项目 [xhs](https://github.com/ReaJason/xhs?tab=readme-ov-file) 似乎实现了，但目前的可用性无法验证。考虑到最重要的播放量数据，在小红书上并不公开可见，这本来就仅限发布者本人可以查看，非公开数据本身获取难度会更大一些。

抖音开放平台提供[查询特定视频数据的 API](https://developer.open-douyin.com/docs/resource/zh-CN/dop/develop/openapi/video-management/douyin/search-video/video-data)，坏消息是个人身份仅支持创建小游戏和直播小玩法，这个接口需要企业身份认证，来开启移动/网站应用的视频权限。好消息是 [Douyin_TikTok_Download_API](https://github.com/Evil0ctal/Douyin_TikTok_Download_API) 提供了类似的功能 ，坏消息 again 是它并没有提供单个视频播放、互动数据的接口。

公众号的[获取图文群发每日数据接口](https://developers.weixin.qq.com/doc/offiaccount/Analytics/Graphic_Analysis_Data_Interface.html)可以获取到实时的阅读数据，但需要公众号管理员权限开通验证 access_token，我的操作权限不够，此外点赞、在看和转发等互动数据也无法获取。第三方实现通常需要通过代理抓包，但关键参数也需要手动获取，因为 key 会定时刷新……详见 [https://github.com/wnma3mz/wechat_articles_spider。同为腾讯系的视频号状况也类似，作为麻瓜和腾讯](https://github.com/wnma3mz/wechat_articles_spider%E3%80%82%E5%90%8C%E4%B8%BA%E8%85%BE%E8%AE%AF%E7%B3%BB%E7%9A%84%E8%A7%86%E9%A2%91%E5%8F%B7%E7%8A%B6%E5%86%B5%E4%B9%9F%E7%B1%BB%E4%BC%BC%EF%BC%8C%E4%BD%9C%E4%B8%BA%E9%BA%BB%E7%93%9C%E5%92%8C%E8%85%BE%E8%AE%AF) battle 的成本过高，遂放弃。

你看，世上无难事，只要肯放弃(。本想偷个懒以项目为单位获取全平台的播放量数据，结果转了一圈下来四处撞墙。事已至此，还是先把基本流程跑通，给这件事画个句号。

### 飞书 WebHook

bilibili、YouTube 和微博的数据获取已经解决，主要的长视频平台左右也就这俩(仨)，接下来解决定时转播数据的需求，再拆解其实是两个问题——信息传递和定时任务。如果有需要，还可以带上个数据存储，毕竟阿 B 后台并不提供固定周期的数据对比功能。

公司用的 IM 软件是飞书，它的自定义机器人支持 WebHook，可以将其他平台的消息推送至该群组中，完美的提醒工具。

> WebHook 是一种基于 HTTP 的回调机制，允许一个系统在特定事件发生时自动向另一个系统发送实时通知。当触发预设条件时，源系统会将相关数据以 POST 请求的方式推送到目标系统的指定 URL，接收方可以立即处理这些信息。这种方式类似于 " 事件通知 "，避免了传统轮询方式的低效，广泛应用于支付通知、代码托管平台的变更提醒、消息推送等场景，实现了不同系统间快速、实时的数据交互。

好的，显然上面这个解释太掉书袋了。简单来说，在飞书里设定好一个自定义机器人，我们会获得一个链接。利用这个链接传输数据，机器人就会在群组里弹消息了！

这样我就不需要“时不时”戳开视频看一眼数据，然后播报给老板娘了，好耶！基本的设置流程，可以参见飞书的[自定义机器人使用指南](https://open.feishu.cn/document/client-docs/bot-v3/add-custom-bot?lang=zh-CN)。获取到 WebHook URL 后，只需要简单地修改一下脚本，写好消息发送的格式就可以了。

发布时间也是衡量数据表现的维度之一，因此加上了这个数据，还顺手计算了互动总数。下例为 bilibili，其他平台类似：

```python
import asyncio
import logging
import argparse
from datetime import datetime
from bilibili_api import video
import requests

## 配置日志
logging.basicConfig(
    level=logging.INFO,  ## 日志级别
    format='%(asctime)s - %(levelname)s - %(message)s',  ## 日志格式
    handlers=[
        logging.FileHandler("video_monitor.log"),  ## 日志文件
        logging.StreamHandler()  ## 控制台输出
    ]
)

## 飞书 Webhook URL
FEISHU_WEBHOOK_URL = '<https://open.feishu.cn/open-apis/bot/>'  ## 输入飞书 Bot url

## 设置命令行参数
parser = argparse.ArgumentParser(description='Monitor Bilibili video.')
parser.add_argument('bv_id', type=str, help='The BV ID of the video to monitor.')
args = parser.parse_args()

async def fetch_video_data(bvid: str) -> None:
    ## 实例化 Video 类
    v = video.Video(bvid=bvid)  ## 使用传入的 BV 号
    ## 获取信息
    info = await v.get_info()

    ## 获取视频发布时间
    pub_time_str = info['pubdate']  
    pub_time = datetime.fromtimestamp(pub_time_str)  

    ## 计算当前时间和视频发布时间的差值
    now = datetime.now()
    time_diff = now - pub_time

    ## 获取统计信息
    name = info['owner']['name']
    views = info['stat']['view']
    likes = info['stat']['like']
    replies = info['stat']['reply']
    coins = info['stat']['coin']
    favorites = info['stat']['favorite']
    shares = info['stat']['share']
    danmaku_count = info['stat']['danmaku']

    ## 计算互动总数、互动占比和投币占比
    interaction_total = likes + replies + coins + favorites + shares + danmaku_count
    interaction_ratio = (interaction_total / views) * 100 if views > 0 else 0  ## 避免除以零
    coin_ratio = (coins / interaction_total) * 100 if interaction_total > 0 else 0  ## 避免除以零

    ## 构建飞书卡片消息格式
    card_message = {
        "msg_type": "interactive",
        "card": {
            "config": {
                "wide_screen_mode": True
            },
            "header": {
                "title": {
                    "tag": "plain_text",
                    "content": f"{info['owner']['name']} | {info['title']}"  ## 修改消息头
                },
                "template": "orange"  ## 设置标题主题颜色
            },
            "elements": [
                {
                    "tag": "div",
                    "text": {
                        "tag": "lark_md",
                        "content": f"[该视频](<https://www.bilibili.com/video/{bvid}>)距今已发布 {time_diff.days} 天 {time_diff.seconds // 3600} 小时，当前 B 站播放量为 **{views / 10000:.1f} 万**，\\n\\n互动总数为 **{interaction_total / 10000:.1f} 万**，互动占比为 **{interaction_ratio:.2f}%**，投币占比为 **{coin_ratio:.2f}%**。\\n\\n详细数据 👉 播放量: {views}  互动总数: {interaction_total} 点赞数: {likes}  评论数: {replies}  投币数: {coins}  收藏数: {favorites}  转发数: {shares}  弹幕数: {danmaku_count}"
                    }
                }
            ]
        }
    }

    ## 发送消息到飞书
    await send_to_feishu(card_message)

async def send_to_feishu(data):
    requests.post(FEISHU_WEBHOOK_URL, json=data)

## 主函数
async def main():
    await fetch_video_data(args.bv_id)  ## 使用命令行参数中的 BV ID

## 执行主函数
if __name__ == "__main__":
    asyncio.run(main())
```

实际接收到消息的效果是这样的：

![1737881422761.png](https://zyin-1309341307.cos.ap-nanjing.myqcloud.com/note/1737881422761.png)

如果你像我一样要运营多个矩阵号，还可以顺手修改下颜色方便辨识，加上个变量即可：

```python
#如果是账号1，则是紫色；账号 2 则是粉色；除了这俩，默认是橙色。
template_color = "purple" if info['owner']['name'] == "账号名1" else "carmine" if info['owner']['name'] == "账号名2" else "orange"

#记得把消息格式里的 template 也改下。
"template": template_color  ## 设置标题主题颜色
```

如果希望以 7 天为周期横向对比各个视频的数据表现，那还可以加入下面这段代码，这样它就会在第七天保存当时的数据进入 bilibili_7-Day.csv 的文件中。

```python
## 在文件开头添加
from pathlib import Path
import csv

## 在 fetch_video_data 函数开始处添加数据目录设置
def ensure_data_dir():
    ## 在脚本所在目录创建 data 文件夹
    data_dir = Path(__file__).parent / 'data'
    data_dir.mkdir(exist_ok=True)
    return data_dir

## 在 fetch_video_data 函数中修改CSV相关代码
    ## 检查是否为第7天
    days_diff = time_diff.days
    if days_diff == 7:
        ## 准备CSV文件
        data_dir = ensure_data_dir()
        csv_file = data_dir / 'bilibili_7-Day.csv'
        file_exists = csv_file.exists()
        
        ## 准备要写入的数据
        headers = ['视频标题', '发布时间', '当前时间', '播放量', '互动总数', 
                  '点赞数', '评论数', '投币数', '收藏数', '转发数', '弹幕数']
        row_data = [
            info['title'],
            pub_time.strftime('%Y-%m-%d %H:%M:%S'),
            now.strftime('%Y-%m-%d %H:%M:%S'),
            views,
            interaction_total,
            likes,
            replies,
            coins,
            favorites,
            shares,
            danmaku_count
        ]
        
        ## 写入CSV文件
        with open(csv_file, 'a', newline='', encoding='utf-8-sig') as f:
            writer = csv.writer(f)
            if not file_exists:
                writer.writerow(headers)
            writer.writerow(row_data)
            
        logging.info(f"已将第7天数据写入 {csv_file}: {info['title']}")
```

### 定时执行脚本任务

能成功写入第 7 天数据的前提是，我们需要在第 7 天的时候运行它，这就涉及到定时任务了。

很可惜我家没有装宽带，也没有可以玩的小主机之类的东西。考虑到我们这种随时 on call 的岗，为了方便远控访问 NAS，电脑几乎不关机……那就直接用公司电脑来设置定时任务好了。

最简单的实现是 Windows 自带的「任务计划程序」，理论上这种系统工具应该[逐步按引导操作](https://learn.microsoft.com/zh-cn/troubleshoot/windows-server/system-management-components/schedule-server-process)就可以实现，GUI 就应当让人看到就明白对吗？可惜这个世界上，应然和实然是两回事。

常规的教程只说了”将所需要运行的脚本路径填入即可“，但它一直无法按预期运行定时任务。几经周折后终于在某个角落（一篇名为「[windows设置定时执行脚本](https://www.cnblogs.com/sui776265233/p/13602893.html)」的文章）里查到，其实需要在启动程序的那部分，先填入 Python.exe 的绝对路径，并在可选的两个参数中，填入 Python 脚本路径和解释器路径。

![1737881461733.png](https://zyin-1309341307.cos.ap-nanjing.myqcloud.com/note/1737881461733.png)

尽管这位[迎风而来的小随](https://www.cnblogs.com/sui776265233)在 2021 年停止了博客的更新，但还是由衷地感谢 ta 的文章解决了我的小问题。这就是我现在依旧更喜欢文字、更喜欢网页，并且无比痛恨死链的原因。

不过我依旧无法接受需要为每次的定时任务逐次点击这么多次鼠标，我的腕管综合征对此表示强烈的不满。于是又让 AI 写了一段使用 Schtasks 命令行完成操作的命令：

```python
C:\\Windows\\System32\\schtasks.exe /create /tn "测试" `
    /tr "cmd.exe /c cd /d C:\\Users\\wl747\\AppData\\Local\\Programs\\Python\\Python313 && C:\\Users\\wl747\\AppData\\Local\\Programs\\Python\\Python313\\pythonw.exe G:\\zdh\\bili.py BV号" `
    /sc HOURLY /mo 6 /sd (Get-Date).ToString("yyyy/MM/dd") /st (Get-Date).ToString("HH:mm") `
    /du 360 /ed (Get-Date).AddDays(16).ToString("yyyy/MM/dd") `
    /f /ru "SYSTEM"
```

它可以实现每 6 小时运行一次 Python 脚本的计划任务，持续 16 天。这里需要注意，每次的定时任务最好不重名。如果你需要更详细的解释以便修改这段命令，可以戳[这里让 Monica 为您服务](https://monica.im/share/chat?shareId=q4cNbhNWIcmox9wq):p。

至此，看似简单实则并没那么简单的转播数据需求，算是完成了。

## 数据汇总录入

当我向老板娘表示这个 Bot 可以实现定时播报数据情况，可以把它配置到群聊的时候，新的需求又来了。同理可得，AI 并不会让人失业，只会让预期可实现的需求越变越多。

出于交付达标的考虑，她依旧要求我在每周五更新一份全平台数据的表单（事实上只有 4 个平台，而且我发现她上次看这个文档已经是两周前了🚬）。加上本身时隔 7 天就需要收录的长、短视频表单，以及 2025 年绩效标准更新之后对内部公开的视频数据表单（是的，很难想象视频内容制作团队在此之前是无法看到平台非公开数据的……）。另外，甲方也经常提出需要汇总全平台数据的需求。

于是我需要频繁填写的表单从两份变成了五份，表头项要求各不相同。数据汇总这件事，也变成了费时费力且毫无成就感的 Dirty Work。没有人可以忍受需要花一个小时只是为了填一份该死的表格，为此需要跳转 12 个平台找到对应视频、一项项确认数据、依次对比表头填入固定格子里，如此重复成百上千次。

这太愚蠢了，但很显然，各平台的封闭性并没有提供足够简单好用的接口让这种琐碎事消失，当然有更复杂的方式可以实现，这让那些社媒管理平台得以盈利。只不过本麻瓜被拒之门外，好消息是我们依旧可以曲线救国。

大多数（7/12）平台提供了详细数据导出的功能，还有 2 个（YouTube 和微博）可以通过上述的脚本获取到，剩下 3 个无关紧要的平台，只有甲方会需要，因此手动收录也显得不那么不可接受。那么，导出 7 个平台的数据并筛选标题关键词，同时给到 YouTube 和微博的视频链接，理论上是能实现一键汇总 9 个平台的数据的。

只是手动下载 7 个平台的数据再提供 2 个链接，显然比之前逐个汇总那么多表项来得可以接受。于是对外汇总数据的 1.0 版本非常简单粗暴，筛选固定路径里的文件，根据平台名找到对应文件，然后匹配和检索关键词一致的行，复制表头和该行数据到新文件中。

![1737881485152.png](https://zyin-1309341307.cos.ap-nanjing.myqcloud.com/note/1737881485152.png)

至于为什么区分出了 1.0 版本，因为总是会有新的要求，[Scope Creep](https://en.wikipedia.org/wiki/Scope_creep) 永远正确，被困住的只有牛马🚬。有些甲方对表头的要求是指定的，于是 2.0 版本用了更通用的匹配方式，也让输出更加规整。

最终的输出将会是：

![1737881509051.png](https://zyin-1309341307.cos.ap-nanjing.myqcloud.com/note/1737881509051.png)

具体的实现如下：

```python
import os
import sys
import re
import pandas as pd
import traceback
from datetime import datetime
import urllib3
import yt_dlp
import requests
import json
import logging

## 配置日志
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

## 禁用 SSL 警告
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

def format_youtube_date(date_str):
    try:
        if date_str and len(date_str) == 8:
            return f"{date_str[:4]}-{date_str[4:6]}-{date_str[6:]}"
        return date_str
    except Exception as e:
        print(f"日期格式转换错误: {e}")
        return date_str

def format_weibo_date(date_str):
    try:
        date_obj = datetime.strptime(date_str, "%a %b %d %H:%M:%S %z %Y")
        return date_obj.strftime("%Y-%m-%d %H:%M")
    except Exception as e:
        print(f"微博日期格式转换错误: {e}")
        return date_str

def get_video_info(video_url):
    ydl_opts = {
        'quiet': True,
        'format': 'best',
        'noplaylist': True,
    }

    try:
        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            info_dict = ydl.extract_info(video_url, download=False)
            title = info_dict.get('title', 'N/A')
            upload_date = format_youtube_date(info_dict.get('upload_date', 'N/A'))
            view_count = info_dict.get('view_count', 0)
            like_count = info_dict.get('like_count', 0)
            comment_count = info_dict.get('comment_count', 0) or 0

            return [
                'YouTube',
                title, 
                video_url, 
                upload_date, 
                str(view_count), 
                str(like_count), 
                str(comment_count)
            ]
    except Exception as e:
        print(f"获取YouTube视频信息失败: {e}")
        return None

def convert_play_count(play_count_str):
    """
    将微博播放量文字转换为数值
    例如：
    '10万次播放' -> 100000
    '2千次播放' -> 2000
    '892次播放' -> 892
    """
    if not play_count_str:
        return 0
    
    play_count_str = str(play_count_str).replace('次播放', '').replace(' ', '')
    
    multipliers = {
        '万': 10000,
        '千': 1000,
    }
    
    for unit, multiplier in multipliers.items():
        if unit in play_count_str:
            try:
                number = float(play_count_str.replace(unit, ''))
                return int(number * multiplier)
            except ValueError:
                return 0
    
    try:
        return int(play_count_str)
    except ValueError:
        return 0

def get_single_weibo(weibo_id, headers=None):
    """获取指定ID的单条微博信息"""
    if headers is None:
        headers = {
            'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/86.0.4240.111 Safari/537.36'
        }
    
    try:
        url = f"<https://m.weibo.cn/detail/{weibo_id}>"
        response = requests.get(url, headers=headers, verify=False)
        html = response.text
        html = html[html.find('"status":'):]
        html = html[:html.rfind('"call"')]
        html = html[:html.rfind(",")]
        html = "{" + html + "}"
        js = json.loads(html, strict=False)
        weibo_info = js.get("status")
        
        if weibo_info:
            ## 获取播放量
            play_count = 0
            if weibo_info.get("page_info"):
                page_info = weibo_info["page_info"]
                if page_info.get("type") == "video":
                    play_count_str = page_info.get("play_count", "0")
                    play_count = convert_play_count(play_count_str)
            
            ## 格式化返回数据，与原有输出列保持一致
            weibo = {
                '平台': '微博',
                '标题': weibo_info['text'],
                '链接': f"<https://weibo.com/{weibo_info['user']['id']}/{weibo_info['bid']}>",
                '发布时间': format_weibo_date(weibo_info['created_at']),
                '播放量': str(play_count),
                '点赞': str(weibo_info['attitudes_count']),
                '评论': str(weibo_info['comments_count']),
                '转发': str(weibo_info['reposts_count'])
            }
            
            return list(weibo.values())
            
        return None
    
    except Exception as e:
        logger.error(f"获取微博信息出错: {e}")
        return None

def extract_platform_name(filename):
    match = re.search(r'^([^\\-]+)', filename)
    return match.group(1) if match else filename

def match_column(df, target_columns):
    for col in df.columns:
        for target in target_columns.split('/'):
            if target in str(col):
                return col
    return None

def process_files(search_keyword, input_dir=None, sub_folder=None):
    ## 如果没有提供输入目录，使用默认路径
    if input_dir is None:
        today = datetime.now().strftime("%Y-%m-%d")
        input_dir = os.path.join('G:\\\\zdh\\\\platformdata', today)
        if sub_folder:
            input_dir = os.path.join(input_dir, sub_folder)
    
    ## 平台处理顺序
    platform_order = ['bilibili', '抖音', '小红书', '公众号', '视频号', '快手', '头条号']
    
    ## 输出列及其可能的列名
    output_columns = {
        '平台': '',
        '标题': '作品/标题/描述',
        '链接': '链接/url',
        '发布时间': '发布时间/发表时间',
        '播放量': '播放量/观看量/总阅读次数',
        '点赞': '点赞/喜欢',
        '评论': '评论',
        '转发': '转发/转发次数/分享',
        '收藏': '收藏'
    }
    
    ## 输出目录和文件名
    output_dir = 'G:\\\\zdh\\\\to_party_a'
    os.makedirs(output_dir, exist_ok=True)
    output_filename = f'output_{datetime.now().strftime("%Y%m%d%H%M%S")}.xlsx'
    output_file_path = os.path.join(output_dir, output_filename)
    
    ## 获取所有输入文件，按指定顺序排序
    input_files = [
        os.path.join(input_dir, f) 
        for f in os.listdir(input_dir)
        if f.endswith(('.csv', '.xlsx', '.xls', '.et')) 
        and not f.startswith('~$') 
        and not f.startswith('.')
    ]
    
    ## 按平台顺序排序
    input_files.sort(key=lambda x: next((i for i, p in enumerate(platform_order) if p in os.path.basename(x)), len(platform_order)))
    
    ## 初始化输出行
    output_rows = [list(output_columns.keys())]
    
    for file_path in input_files:
        filename = os.path.basename(file_path)
        platform = extract_platform_name(filename)
        print(f"\\n正在处理文件: {filename}")
        
        try:
            ## 读取文件
            if file_path.endswith('.csv'):
                df = pd.read_csv(file_path, encoding='utf-8-sig')
            elif file_path.endswith('.xls'):
                df = pd.read_excel(file_path, engine='xlrd')
            else:
                if '小红书' in filename:
                    df = pd.read_excel(file_path, header=1)
                else:
                    df = pd.read_excel(file_path, engine='openpyxl')
            
            print(f"文件列名: {list(df.columns)}")
            print(f"文件行数: {len(df)}")
            
            ## 在第一列中搜索关键词
            matched_rows = df[df.iloc[:, 0].astype(str).str.contains(search_keyword, case=False, na=False)]
            
            if not matched_rows.empty:
                print(f"找到 {len(matched_rows)} 行匹配数据")
                
                for _, row in matched_rows.iterrows():
                    ## 创建新行
                    new_row = [platform]
                    
                    ## 匹配其他列
                    for col_name, possible_names in list(output_columns.items())[1:]:
                        if possible_names:
                            matched_col = match_column(df, possible_names)
                            value = row[matched_col] if matched_col else ''
                            new_row.append(str(value))
                        else:
                            new_row.append('')
                    
                    output_rows.append(new_row)
            else:
                print(f"文件 {filename} 未找到匹配项")
        
        except Exception as e:
            print(f"处理文件 {filename} 时出错: {e}")
            traceback.print_exc()
    
    ## 询问微博和 YouTube 链接
    print("\\n请输入微博链接（以空格分隔，没有则直接回车）:")
    weibo_links = input().split()
    
    for link in weibo_links:
        weibo_id = link.split('/')[-1]
        weibo_data = get_single_weibo(weibo_id)
        if weibo_data:
            output_rows.append(weibo_data)
    
    print("\\n请输入 YouTube 链接（以空格分隔，没有则直接回车）:")
    youtube_links = input().split()
    
    for link in youtube_links:
        youtube_data = get_video_info(link)
        if youtube_data:
            output_rows.append(youtube_data)
    
    ## 保存输出文件
    output_df = pd.DataFrame(output_rows[1:], columns=output_rows[0])
    output_df.to_excel(output_file_path, index=False)
    print(f"\\n匹配结果已保存到: {output_file_path}")

## 主程序入口
if __name__ == '__main__':
    ## 检查命令行参数
    if len(sys.argv) < 2:
        print("用法: python 脚本.py <搜索关键词> [子文件夹]")
        sys.exit(1)
    
    ## 获取命令行参数
    search_keyword = sys.argv[1]
    sub_folder = sys.argv[2] if len(sys.argv) > 2 else None
    
    ## 调用处理函数
    process_files(search_keyword, sub_folder=sub_folder)
```

如果需要增删输出中的列，只需要修改这里的代码：

```python
    ## 输出列及其可能的列名
    output_columns = {
        '平台': '',
        '标题': '作品/标题/描述',
        '链接': '链接/url',
        '发布时间': '发布时间/发表时间',
        '播放量': '播放量/观看量/总阅读次数',
        '点赞': '点赞/喜欢',
        '评论': '评论',
        '转发': '转发/转发次数/分享',
        '收藏': '收藏'
    }
```

冒号前是输出的列名，冒号后是输入表中改数值项可能对应的列名，记得注意逗号。

除了给甲方的全平台汇总之外，内部要填的表单只是区分长短视频，需要把不同平台的数据对应起来，实现也是类似的，以短视频平台的汇总为例：

```python
import os
import pandas as pd
import glob
from datetime import datetime

def process_files(search_keyword, input_dir=None, sub_folder=None):
    ## 输出路径和文件名
    output_dir = r'G:\\zdh\\data'
    os.makedirs(output_dir, exist_ok=True)  ## 确保目录存在

    ## 如果没有提供输入目录，使用默认路径
    if input_dir is None:
        today = datetime.now().strftime("%Y-%m-%d")
        input_dir = os.path.join('G:\\\\zdh\\\\platformdata', today)
        if sub_folder:
            input_dir = os.path.join(input_dir, sub_folder)

    ## 输出文件
    output_file = os.path.join(output_dir, f'Douyin_Daily.xlsx')
    
    ## 创建或读取输出文件
    try:
        df_output = pd.read_excel(output_file)
    except FileNotFoundError:
        ## 如果文件不存在，创建一个带有预定义列的空DataFrame
        columns = [
            '作品名称', '发布时间', '体裁', '审核状态', '播放量', '完播率', '5s完播率', 
            '封面点击率', '2s跳出率', '平均播放时长', '点赞量', '分享量', '评论量', 
            '收藏量', '主页访问量', '粉丝增量', 
            '小红书-观看量', '小红书-点赞', '小红书-涨粉',
            '快手-播放量', '快手-点赞量', '快手-涨粉量',
            '视频号-播放量', 
            'bilibili2-播放量', 'bilibili2-涨粉量'
        ]
        df_output = pd.DataFrame(columns=columns)

    ## 按顺序处理不同平台文件
    platforms = [
        {'name': '抖音', 'filename': '*抖音*.xlsx', 'search_column': 0, 'header': 0, 'data_columns': [
            '作品名称', '发布时间', '体裁', '审核状态', '播放量', '完播率', '5s完播率', 
            '封面点击率', '2s跳出率', '平均播放时长', '点赞量', '分享量', '评论量', 
            '收藏量', '主页访问量', '粉丝增量'
        ]},
        {'name': '小红书', 'filename': '*小红书*.xlsx', 'search_column': 0, 'header': 1, 'data_columns': [
            {'column': '小红书-观看量', 'source_column': '观看量'},
            {'column': '小红书-点赞', 'source_column': '点赞'},
            {'column': '小红书-涨粉', 'source_column': '涨粉'}
        ]},
        {'name': '快手', 'filename': '*快手*.xlsx', 'search_column': 0, 'header': 0, 'data_columns': [
            {'column': '快手-播放量', 'source_column': '播放量'},
            {'column': '快手-点赞量', 'source_column': '点赞量'},
            {'column': '快手-涨粉量', 'source_column': '涨粉量'}
        ]},
        {'name': '视频号', 'filename': ['*视频号*.csv', '*视频号*.xlsx'], 'search_column': 0, 'header': 0, 'data_columns': [
            {'column': '视频号-播放量', 'source_column': '播放量'}
        ]},
        {'name': 'bilibili2', 'filename': ['*bilibili2*.csv', '*bilibili*.xlsx'], 'search_column': 0, 'header': 0, 'data_columns': [
            {'column': 'bilibili2-播放量', 'source_column': '播放量'},
            {'column': 'bilibili2-涨粉量', 'source_column': '涨粉量'}
        ]}
    ]

    ## 创建一个新的空行用于存储数据
    new_row = pd.DataFrame(columns=df_output.columns)

    for platform in platforms:
        ## 处理可能的多个文件名模式
        filenames = platform['filename'] if isinstance(platform['filename'], list) else [platform['filename']]
        
        matched_file = None
        for filename in filenames:
            search_pattern = os.path.join(input_dir, filename)
            files = [f for f in glob.glob(search_pattern) if not os.path.basename(f).startswith('~$')]
            
            if files:
                matched_file = files[0]
                break
        
        if matched_file:
            try:
                ## 根据文件类型读取
                if matched_file.endswith('.xlsx'):
                    df = pd.read_excel(matched_file, header=platform['header'])
                elif matched_file.endswith('.csv'):
                    df = pd.read_csv(matched_file, encoding='utf-8', header=platform['header'])
            except Exception as e:
                print(f"读取文件 {matched_file} 时发生错误：{e}")
                continue
            
            ## 搜索关键词
            try:
                matched_rows = df[df.iloc[:, platform['search_column']].str.contains(search_keyword, na=False)]
            except:
                matched_rows = df[df.apply(lambda row: row.astype(str).str.contains(search_keyword).any(), axis=1)]
            
            if not matched_rows.empty:
                row = matched_rows.iloc[0]
                
                ## 根据平台特定列名提取数据
                for col_info in platform['data_columns']:
                    if isinstance(col_info, dict):
                        source_column = col_info['source_column']
                        if source_column in df.columns:
                            new_row.loc[0, col_info['column']] = row[source_column]
                    else:
                        if col_info in df.columns:
                            col_index = df.columns.get_loc(col_info)
                            new_row.loc[0, col_info] = row.iloc[col_index]

    ## 如果找到了数据，添加到输出DataFrame
    if not new_row.empty and not new_row.loc[0].isnull().all():
        df_output = pd.concat([df_output, new_row], ignore_index=True)

    ## 保存结果
    df_output.to_excel(output_file, index=False)
    print(f"数据处理完成，结果已保存到 {output_file}")
    return df_output

## 使用示例
if __name__ == "__main__":
    keyword = input("请输入要检索的关键词：")
    result = process_files(keyword)
    print(result)

```

## 评论区监测

好的，如果你看到这里，估计已经忘记我一开始还有个需求是监测评论来着。只是筛选评论区并转播倒是不难，感谢 [bilibili-api](https://github.com/Nemo2011/bilibili-api)。

目前的进度是已经可以读取到所有的评论（包括副楼）保存到表格，还加了简单的筛选词，筛选出的评论可以通过带有 rpid 的直链跳转处理，再加个 WebHook 基本能凑合用，不过一直没空摸鱼把它加上……然后就放假了！

这个版本的缺点是如果定时任务设得比较紧凑，IP 会暂时被封禁无法 work，这个应该可以设置代理绕过。

现在「设置关键词筛选 -> 给直链跳转删除」比起之前「逐条翻评论 -> 定向删除」来得方便，不过理论上还可以在飞书消息[加个交互](https://open.feishu.cn/document/uAjLw4CM/ukzMukzMukzM/feishu-cards/configuring-card-interactions)，结合 [async def delete()](https://nemo2011.github.io/bilibili-api/#/modules/comment?id=async-def-delete) 就可以直接一键删除，不用二次跳转网页。

是的，这里又要有但是了，账号会需要在浏览器长期登录账户以便投稿，这样会导致 Cookies 被刷新……。总之，以下是再凑合一下的版本：

```python
from bilibili_api import comment, Credential
import asyncio
import json
import pandas as pd
from datetime import datetime, timedelta
import os
import sys
import logging
import argparse

## 配置日志
def setup_logging(bvid):
    log_dir = os.path.join(r'G:\\zdh\\data\\comments', bvid)
    os.makedirs(log_dir, exist_ok=True)
    
    log_file = os.path.join(log_dir, 'bilibili_comment_crawler.log')
    logging.basicConfig(
        level=logging.INFO, 
        format='%(asctime)s - %(levelname)s: %(message)s',
        filename=log_file,
        filemode='a'
    )

## 定义时间记录文件路径
def get_last_run_file(bvid):
    return os.path.join(r'G:\\zdh\\data\\comments', bvid, 'last_run_time.txt')

def read_last_run_time(bvid):
    """读取上次运行时间"""
    last_run_file = get_last_run_file(bvid)
    if os.path.exists(last_run_file):
        with open(last_run_file, 'r') as f:
            lines = f.readlines()
            ## 如果文件不为空，返回最后一行的时间戳
            return int(lines[-1].split()[0]) if lines else 0
    return 0  ## 如果文件不存在，返回0表示获取所有评论

def write_last_run_time(bvid, timestamp):
    """写入本次运行时间"""
    last_run_file = get_last_run_file(bvid)
    ## 转换时间戳为可读格式
    readable_time = datetime.fromtimestamp(timestamp).strftime('%Y/%m/%d %H:%M')
    
    with open(last_run_file, 'a') as f:
        ## 写入时间戳和可读时间
        f.write(f"{timestamp} ## {readable_time}\\n")

def flatten_comment(comment, bvid):
    """展平单个评论"""
    try:
        rpid = comment.get("rpid", 0)
        flattened_comment = {
            "uname": comment["member"]["uname"],
            "message": comment["content"]["message"],
            "like": comment.get("like", 0),
            "ctime": comment["ctime"],
            "ip_location": comment.get('reply_control', {}).get('location', '未知'),
            "rpid": rpid,
            "comment_url": f"<https://www.bilibili.com/video/{bvid}/#reply{rpid}>"
        }
        return flattened_comment
    except Exception as e:
        logging.error(f"解析评论时出错: {e}")
        return None

def extract_comments(comments, bvid, last_run_time):
    """
    递归提取评论，仅保留指定时间后的评论
    
    :param comments: 评论列表
    :param bvid: 视频ID
    :param last_run_time: 上次运行的时间戳
    :return: 过滤后的评论列表
    """
    all_comments = []
    for comment in comments:
        ## 检查评论时间是否在上次运行之后
        if comment["ctime"] > last_run_time:
            flattened = flatten_comment(comment, bvid)
            if flattened:
                all_comments.append(flattened)
        
        ## 递归处理子评论
        if "replies" in comment and comment["replies"]:
            ## 递归时传入 last_run_time
            child_comments = extract_comments(comment["replies"], bvid, last_run_time)
            all_comments.extend(child_comments)
    
    return all_comments

async def get_new_comments(bvid, last_run_time):
    """获取视频的新评论"""
    comments = []
    page = 1
    
    while True:
        try:
            c = await comment.get_comments(
                bvid, 
                comment.CommentResourceType.VIDEO, 
                page,
                credential=credential
            )
            
            replies = c.get('replies', [])
            if not replies:
                break
            
            ## 检查当前页是否还有新的评论
            new_comments = [r for r in replies if r["ctime"] > last_run_time]
            comments.extend(new_comments)
            
            ## 如果没有新评论或已经到最后一页，退出
            if not new_comments or page > c['page']['count'] // c['page']['size'] + 1:
                break
            
            page += 1
        
        except Exception as e:
            logging.error(f"获取评论时出错: {e}")
            break
    
    return comments

def append_to_excel(new_df, bvid):
    """
    追加新数据到现有Excel文件
    如果文件不存在，则创建新文件
    """
    filename = os.path.join(r'G:\\zdh\\data\\comments', bvid, f'{bvid}_comments.xlsx')
    
    try:
        ## 如果文件存在，读取现有数据
        if os.path.exists(filename):
            existing_df = pd.read_excel(filename)
            
            ## 去重
            combined_df = pd.concat([existing_df, new_df]).drop_duplicates(subset=['rpid'])
            combined_df.to_excel(filename, index=False)
        else:
            ## 如果文件不存在，直接保存
            new_df.to_excel(filename, index=False)
        
        logging.info(f"数据已成功保存到 {filename}")
    except Exception as e:
        logging.error(f"保存Excel文件时出错: {e}")

async def main(bvid):
    ## 设置日志
    setup_logging(bvid)
    
    try:
        ## 读取上次运行时间
        last_run_time = read_last_run_time(bvid)
        logging.info(f"上次运行时间: {last_run_time}")
        
        ## 获取新评论
        new_video_comments = await get_new_comments(bvid, last_run_time)
        
        ## 展平并提取新评论
        flattened_comments = extract_comments(new_video_comments, bvid, last_run_time)
        
        ## 打印总新评论数
        logging.info(f"共获取到 {len(flattened_comments)} 条新评论")
        
        ## 筛选包含关键词的评论
        filtered_comments = [
            comment for comment in flattened_comments 
            if any(keyword in comment['message'] for keyword in FILTER_KEYWORDS)
        ]
        
        ## 记录关键词评论
        logging.info(f"包含关键词的评论共 {len(filtered_comments)} 条")
        
        ## 转换为 DataFrame
        df = pd.DataFrame(flattened_comments)
        
        ## 导出到 xlsx 文件，使用追加模式
        append_to_excel(df, bvid)
        
        ## 记录本次运行时间
        current_timestamp = int(datetime.now().timestamp())
        write_last_run_time(bvid, current_timestamp)
        logging.info(f"本次运行时间已记录: {current_timestamp}")
        
    except Exception as e:
        logging.error(f"脚本执行出错: {e}", exc_info=True)

## 添加依赖检查
try:
    import bilibili_api
    import pandas as pd
except ImportError as e:
    print(f"缺少必要依赖: {e}")
    print("请运行 pip install bilibili-api-python pandas openpyxl")
    sys.exit(1)

if __name__ == "__main__":
    ## 设置命令行参数解析
    parser = argparse.ArgumentParser(description='bilibili评论爬取脚本')
    parser.add_argument('bvid', help='要爬取评论的视频BV号')
    
    ## 解析参数
    args = parser.parse_args()
    
    ## 实例化 Credential
    credential = Credential(
        sessdata="your_sessdata",
        bili_jct="your_bili_jct",
        buvid3="your_buvid3",
        dedeuserid="your_dedeuserid",
        ac_time_value="your_ac_time_value"
    )

    ## 定义关键词列表
    FILTER_KEYWORDS = ['恰饭', '恰', '广告', '推广', '剪辑', '调色', '字幕']

    ## 运行主程序
    asyncio.run(main(args.bvid))
```

## 写在最后

“如果我能明白这究竟是怎么回事就好了”，通常这个想法会出现在最后 30% 的实现上。诚然，如果没有 AI，我绝无可能把这些琐碎的工作用 Python 解决掉，但它的表现依旧差口气。

在想是不是用 cursor 的体验会好上更多……有空再写篇「麻瓜视角下的 AI 编程」聊聊，总之这篇真的断断续续写了好久，先这样，有空再填坑……
