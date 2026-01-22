# AI漫剧代理API接入制作教程：推荐神马中转API接入Gemini3+NanoBanana+Sora2一键生成AI漫剧


过去，做一部动画漫剧，需要编剧、美术、分镜、剪辑，

而现在——

**你只需要一个主题。******

AI 正在重构“内容生产”的底层逻辑：

剧本不再从空白开始，

角色不再一张张手绘，

分镜不再靠经验堆出来，

动画，也不再是专业团队的专属能力。

真正的门槛，

已经从「创作能力」变成了 **「API 调用能力」**。

***

在本教程中，你将完成一件过去几乎不可能由个人完成的事：

> **通过一个神马中转API，一键生成完整 AI 漫剧流程：剧本 → 角色三视图 → 场景分镜 → AI 漫剧视频******

整个过程**不依赖本地算力、不需要动画基础**，

只通过 **Gemini3 + NanoBanana Pro + Sora2** 三个模型的协同调用完成。

![AI漫剧代理API接入制作教程：推荐神马中转API接入Gemini3+NanoBanana+Sora2一键生成AI漫剧](https://pic.imgdd.cc/item/6971c38ac94292c21eefbf71.webp)

## **为什么AI漫剧推荐使用“神马中转API”？**

因为真实的AI应用，并不是单模型炫技，而是**流程化生产**：

* Gemini3：负责**思考与结构******

* NanoBanana：负责**视觉落地******

* Sora2：负责**时间与动态叙事**

***

**准备工作：BaseURL 与 API Key**

* **BaseURL 通常是网站域名**（也可在工作台查看）

* **API Key 在令牌页面获取** 

下面教程用统一写法：

* BASE\_URL = 你的中转站域名（例如神马中转API：api.whatai.cc）

* Header：Authorization: Bearer {{YOUR\_API\_KEY}}

***

**三段式流水线总览** **Gemini3 生成“可直接生产用”的结构化剧本**

使用 **Chat Completions** 接口：POST /v1/chat/completions 

模型指定为：gemini-3-flash-preview-nothinking

**NanoBanana 生成三视图/场景图/分镜图**

使用 DALL·E 兼容生图接口：POST /v1/images/generations 

模型可用：nano-banana-2

**Sora2 用“故事板格式 prompt”生成 AI 漫剧视频**

统一视频生成接口：POST /v2/videos/generations 

* 文生视频示例参数：prompt/model/aspect\_ratio/hd/duration 

* 故事板启用方式：把 prompt 按 Shot 1: ... 格式传入即可 

* 查询任务：GET /v2/videos/generations/{task\_id}，状态枚举与返回示例见文档 

***

## **Step 1：Gemini3 生成“剧本 + 场景 + 分镜规划”（结构化输出）**

**关键点：让模型输出可解析 JSON**

你要后续自动化（生图/拼故事板 prompt），强烈建议让 Gemini3 **只输出 JSON**。Chat 接口支持 response\_format 字段（文档示例里给出了该字段）

### **Python示例代码（requests）**

```
import requests

BASE_URL = "https://api.whatai.cc"
API_KEY = "YOUR_API_KEY"

url = f"{BASE_URL}/v1/chat/completions"
headers = {
    "Accept": "application/json",
    "Authorization": f"Bearer {API_KEY}",
    "Content-Type": "application/json",
}

payload = {
    "model": "gemini-3-flash-preview-nothinking",
    "messages": [
        {
            "role": "system",
            "content": "你是AI漫剧编剧与分镜导演。你必须只输出严格JSON，不要输出任何多余文字。"
        },
        {
            "role": "user",
            "content": "主题：赛博校园侦探。请生成：1) 角色设定（含外观特征与服饰关键点，方便三视图绘制）2) 场景列表（每场景的光线/色调/道具）3) 分镜规划（每镜头时长、景别、机位、动作、台词/旁白、画面要点）。要求：9-12个镜头，总时长10秒；镜头可用于Sora2故事板。输出JSON结构：{title,logline,characters[],scenes[],shots[]}" 
        }
    ],
    "temperature": 0.7,
    "top_p": 1,
    "stream": false
}

resp = requests.post(url, headers=headers, json=payload, timeout=120)
resp.raise_for_status()
data = resp.json()

script_json_text = data["choices"][0]["message"]["content"]
print(script_json_text)
```

***

## **Step 2：NanoBananaPRO生成「角色三视图 / 场景图 / 分镜草图」**

**神马中转API **NanoBanana 生图使用：POST /v1/images/generations，Header 为 Authorization: Bearer ... + Content-Type: application/json，Body 至少包含 prompt 与 model 

### **三视图 prompt 模板（推荐）**

把 Gemini 输出的角色外观字段填进去，确保**同一角色一致性**：发色、瞳色、服装标志、配饰、鞋子、体型比例等写死。

示例 prompt（你可以程序化拼接）：

* “同一角色，三视图：正面/侧面/背面，白底，线稿+上色，角色名标注，保持比例一致”

* “禁止：真人写真、摄影、复杂背景”

![AI漫剧代理API接入制作教程：推荐神马中转API接入Gemini3+NanoBanana+Sora2一键生成AI漫剧](https://pic.imgdd.cc/item/6971bf8ac94292c21eefa836.png)  

### **Python：封装一个通用生图函数（角色/场景/分镜都能用）**

```
import requests

def nano_image_generate(base_url: str, api_key: str, prompt: str, model: str = "nano-banana"):
    url = f"{base_url}/v1/images/generations"
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json",
    }
    payload = {
        "prompt": prompt,
        "model": model
    }
    r = requests.post(url, headers=headers, json=payload, timeout=300)
    r.raise_for_status()
    return r.json()

# 用法示例
img_resp = nano_image_generate(
    base_url=BASE_URL,
    api_key=API_KEY,
    prompt="你是专业二次元漫剧分镜导演。

请基于【同一角色 / 同一场景 / 同一光照】生成一个完整的 3x3 电影 Contact Sheet（电影印样），用于 AI 漫剧分镜图制作。",
    model="nano-banana-2"
)
print(img_resp)
```

> 文档对返回体示例展示为空对象（Apifox页面“示例 {}”）

> 所以你在代码里建议**打印真实返回**，按你中转站实际返回字段取图（常见做法是取 url/base64/data\[] 之类，但以你实际响应为准）。

![AI漫剧代理API接入制作教程：推荐神马中转API接入Gemini3+NanoBanana+Sora2一键生成AI漫剧](https://pic.imgdd.cc/item/6971c0a7c94292c21eefab32.png)  

***

## **Step 3：把分镜规划转成 Sora2「故事板格式 prompt」**

Sora2 故事板视频的关键是：prompt 按文档要求写成多段 Shot，并包含每段的 duration 与 Scene 

**故事板 prompt 拼接规则（建议）**

从 Gemini 的 shots\[] 自动拼：

* Shot N:

* duration: Xsec

* Scene: 画面描述（包含动作、景别、机位、光线、风格、角色一致性要点）

* （可加）Dialogue/VO: 台词或旁白（如果你希望视频包含字幕/口型，可在画面描述中提示）

示例（符合文档格式的骨架） ：

```
Shot 1:
duration: 1.0sec
Scene: ...

Shot 2:
duration: 1.0sec
Scene: ...
```

***

## **Step 4：提交Sora2生成AI漫剧（故事板视频）**

**神马中转API统一生成接口：POST /v2/videos/generations**

文档给出请求示例：prompt/model/aspect\_ratio/hd/duration 

* model: "sora-2"

* aspect\_ratio: "16:9"

* hd: false/true

* duration: "10"（字符串形式示例中就是 "10"）

**cURL（故事板）**

```
curl --location --request POST '{{BASE_URL}}/v2/videos/generations' \
--header 'Authorization: Bearer {{YOUR_API_KEY}}' \
--header 'Content-Type: application/json' \
--data-raw '{
  "prompt": "Shot 1:\nduration: 1.0sec\nScene: 赛博校园走廊，霓虹公告栏闪烁，雨夜地面反光。镜头：推轨近景，林澈从暗处抬头，手指划过发光校徽臂章。\n\nShot 2:\nduration: 1.0sec\nScene: 特写，数据卡扣弹出全息投影，蓝色UI光照在脸上，背景虚化。\n\nShot 3:\nduration: 1.0sec\nScene: ...（此处继续拼到10秒总时长）",
  "model": "sora-2",
  "aspect_ratio": "16:9",
  "hd": false,
  "duration": "10"
}'
```

**返回 task\_id**

生成接口返回示例中给出：

```
{ "task_id": "..." }
```

***

## **Step 5：轮询查询任务，拿到最终视频链接**

查询接口：GET /v2/videos/generations/{task\_id} 

状态枚举：NOT\_START / IN\_PROGRESS / SUCCESS / FAILURE 

成功返回示例里 data.output 给出了 mp4 链接 

**cURL：查询任务**

```
curl --location --request GET '{{BASE_URL}}/v2/videos/generations/{{task_id}}' \
--header 'Authorization: Bearer {{YOUR_API_KEY}}'
```

***

## **让“一键生成 AI 漫剧”更稳的 5 个实战技巧**

1. **Gemini 输出 JSON 一定要“严格”**：系统提示里写死“只输出 JSON”，否则后续自动拼接很容易崩。

2. **角色一致性**：三视图 prompt 里把“不可变特征”写成清单（发色、挑染、臂章、卡扣、鞋子、身形比例）。

3. **场景一致性**：每个 Scene 都带上“色调 + 光源方向 + 标志性道具”（霓虹公告栏/售货机/雨夜反光）。

4. **分镜时长总和**：Sora2 例子里 duration 是 "10"，你的 shots 总时长也建议凑到 10 秒（或略小于 10 秒）。

5. **失败要可重试**：查询状态如果 FAILURE，记录失败原因字段（示例里有 fail\_reason）并降级：缩短 prompt、减少敏感元素、关掉 hd。

***

## **你可以直接复用的“最小闭环”接口清单**

* 剧本：POST {{BASE\_URL}}/v1/chat/completions（model=gemini-3-flash-preview-nothinking）

* 生图：POST {{BASE\_URL}}/v1/images/generations（model=nano-banana-2 或 nano-banana）

* 视频提交：POST {{BASE\_URL}}/v2/videos/generations（model=sora-2，prompt=故事板格式）

* 视频查询：GET {{BASE\_URL}}/v2/videos/generations/{task\_id}

神马聚合中转API（api.whatai.cc）是一个高效的Open AI、Midjourney API代理、Claude代理、Suno代理等供应商
我们致力于提供优质的 API 接入服务，让您可以轻松集成先进的AI模型至您的产品和服务。通过 API 综合管理平台，无缝整合OpenAl最尖端的人工智能模型。借助我们可靠且易于使用的API解决方案，升级您的产品与服务。

[![神马聚合中转API_低价gpt_中转api_好用稳定的GPT代理_claude中转api_Midjourney代理_Suno代理_Luma代理](https://pic2.imgdd.cc/item/68c78cabfcdff65483faea2a.jpg "神马聚合中转API_低价gpt_中转api_好用稳定的GPT代理_claude中转api_Midjourney代理_Suno代理_Luma代理")](https://api.whatai.cc)

