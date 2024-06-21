之前经常在不同的博客站信息源上会看到博主每天发布`Github Trending`，也不是很清楚这样做是否能吸引来访问流量。最近稍微有空，也会点python，想想看看能不能自己每天抓取这些信息。

## 抓取Github Trending
> 访问地址：https://github.com/trending

![image-20240621221956705](https://github.com/junglehxj/junglehxj.github.io/assets/38659409/abdb1db6-35a3-4a50-9b27-1d189b37f59a)
上图所示即是`Github Trending`的展示情况，首先我网上搜索了下是否有大佬已经做过类似的工作了（脚本或者开源项目），无奈的是并没有找到合适开源方案。思来想去，自己恰好会一点python，并且借助Chat-GPT的能力，应该可以写出我想要的结果。

## Chat-GPT使用
> https://chatgpt.com/

![image-20240621222759997](https://github.com/junglehxj/junglehxj.github.io/assets/38659409/e68c8fb5-71b1-4839-9520-57a27e55e935)
⛅大模型的应用是近年来的热门，大小公司都推出了自家的大语言模型，但据我的使用体验，还是Chat-GPT最顺手，唯一需要注意的就是科学上网。结合GPT的生成能力，可以很好的辅助我们编码，达到想要的效果。

首先，我直接问出了自己想达到的目标，问法也比较简单：
![image-20240621223102805](https://github.com/junglehxj/junglehxj.github.io/assets/38659409/d071ef75-41c7-43e8-8b4b-8e18a0fb3208)
结合GPT给出的答案，我们在打开自己的IDE工具，我这边用`pycharm`来调试代码，经过自己的微调和异常处理，最后额外将结果导出到json文件。这里我加了一个`Frequency`来区分每天/每周/每月的`Trending`情况。
```python
from bs4 import BeautifulSoup
import requests
import json
from datetime import datetime
import time
import os
import logging
from openai import OpenAI
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware


# 设置代理
os.environ["http_proxy"] = "http://127.0.0.1:7890"
os.environ["https_proxy"] = "http://127.0.0.1:7890"

client = OpenAI(
        api_key="your_api_key",
        base_url="https://api.openai.com/v1"
    )

app = FastAPI()

OPEN_CROSS_DOMAIN = True
if OPEN_CROSS_DOMAIN:
    app.add_middleware(
        CORSMiddleware,
        allow_origins=["*"],
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )

logger = logging.getLogger()
logger.setLevel(logging.INFO)  # 设置打印级别
formatter = logging.Formatter('%(asctime)s %(filename)s %(funcName)s [line:%(lineno)d] %(levelname)s %(message)s')

# 设置屏幕打印的格式
sh = logging.StreamHandler()
sh.setFormatter(formatter)
logger.addHandler(sh)

# 设置log保存
fh = logging.FileHandler("logGithubTrendings.log", encoding='utf8')
fh.setFormatter(formatter)
logger.addHandler(fh)


def get_github_trending(frequency):
    url = f"https://github.com/trending?since={frequency}"
    response = requests.get(url)
    if response.status_code != 200:
        logger.error("error: [请求调用失败]-[{}]".format(url))
        return []

    soup = BeautifulSoup(response.content, 'html.parser')
    repo_list = []

    for index, repo in enumerate(soup.find_all('article', class_='Box-row')):
        try:
            repo_name = "/".join(part.strip() for part in repo.h2.a.text.strip().replace('\n', ' ').split("/"))
            repo_url = f"https://github.com{repo.h2.a.get('href')}"
            repo_description_tag = repo.find('p', class_='col-9 color-fg-muted my-1 pr-4')
            repo_description = repo_description_tag.text.strip() if repo_description_tag else 'No description'
            repo_code_language = repo.find('span', class_='d-inline-block ml-0 mr-3')
            repo_code_language = repo_code_language.text.strip() if repo_code_language else 'No code language'
            repo_stars = repo.find('span', class_='d-inline-block float-sm-right')
            repo_stars = repo_stars.text.strip().split(" ")[0] if repo_stars else 'No code language'

            repo_list.append({
                'name': repo_name,
                'url': repo_url,
                'description': repo_description,
                # 'description': summaryText(repo_description),
                "language": repo_code_language,
                "stars": int(repo_stars.replace(',', ''))
            })

            # time.sleep(1)
        except Exception as e:
            logger.error("error: [Frequency]-[{}] [Repo order]-[{}] 解析失败".format(frequency, index))
            logger.error("error: {}".format(e))

    logger.info("[frequency]-[{}] has already finished! ".format(frequency))
    return repo_list


# 英文 -- 译 --> 中文
def summaryText(content: str):
    try:
        completion = client.chat.completions.create(
            model="gpt-4o",
            messages=[
                {
                    "role": "user",
                    "content": "请你作为专业的自身翻译家，请将下面的内容翻译为中文：\n" + content,
                }
            ]
        )
        return completion.choices[0].message.content
    except Exception as e:
        logger.error("error: {}".format(e))
        return "调用大模型翻译失败！"


def main(frequency):
    trending_repos = get_github_trending(frequency)
    if not trending_repos:
        print("No trending repositories found")
        logger.error("error: [Frequency]-[{}] No trending repositories found".format(frequency))
        return

    # 按stars排序
    trending_repos = sorted(trending_repos, key=lambda x: x["stars"], reverse=True)

    # 生成文件. 获取当前日期并格式化为字符串
    current_time = datetime.now().strftime("%Y-%m-%d_%H-%M-%S")
    file_name = f"GithubTrending_{frequency}_{current_time}.json"
    with open(file_name, 'w', encoding='utf-8') as json_file:
        json.dump(trending_repos, json_file, ensure_ascii=False, indent=4)

    logger.info("Handled - [Frequency]-[{}] has already finished! \n".format(frequency))


if __name__ == "__main__":
    frequencies = ["daily", "weekly", "monthly"]
    for index, frequency in enumerate(frequencies):
        main(frequency)

```
上面的代码中额外加入了大模型进行翻译，并且引入了FastAPI，便于以后直接做成一个接口使用。之后计划搞个服务器或者某种自动化的方式🤔，每天同步一份当日`Trending`在github或其他平台。
[githubTrending.zip](https://github.com/user-attachments/files/15929325/githubTrending.zip)