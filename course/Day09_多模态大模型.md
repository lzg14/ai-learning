# Day 10：多模态大模型

> 本课程介绍多模态大模型的原理、架构，以及如何调用国产多模态 API。

## Part 1：什么是多模态？

### 模态的定义

**"人类的五种感官"类比：**

| 模态 | 对应人类感官 | 例子 |
|------|-------------|------|
| 文本 | 读、写 | 文章、对话、代码 |
| 图像 | 视觉 | 图片、照片、图表 |
| 音频 | 听觉 | 语音、音乐 |
| 视频 | 视觉 + 听觉 | 电影、短视频 |

> 💡 多模态 = 模型能同时理解和生成多种模态的内容，而不只是纯文本。

### 多模态为什么重要？

| 纯文本 LLM 的限制 | 多模态模型的能力 |
|---|---|
| 只能处理文字 | 能理解图片、视频、音频 |
| 得用文字描述图片，它才懂 | 直接拍张照给它，它直接看懂 |
| 无法处理图表、流程图 | 看一眼图表就知道数据含义 |

---

## Part 2：多模态大模型架构

### 三种架构路线

| 架构 | 原理 | 代表 | 优点 | 缺点 |
|------|------|------|------|------|
| **拼接式** | 图像编码器 + LLM，中间加投影层 | GPT-4V | 简单、便宜 | 融合浅 |
| **统一表示式** | 所有模态转成同一种 token | Gemini | 融合深 | 训练难度大 |
| **专家混合式** | 每个模态有专门的专家模型 | GPT-4o | 效果最好 | 架构复杂 |

### 最主流的拼接式架构

```
输入图片 → 图像编码器 → 投影层 → 和文本 token 拼接 → LLM → 输出文字
```

**为什么流行？**
- ✅ 简单：现成的 LLM + 现成的图像编码器
- ✅ 便宜：只需要训练投影层
- ✅ 见效快：几万张图文对就能训出效果

---

## Part 3：国产多模态模型 API 调用

### GLM 多模态 API

```python
from zhipuai import ZhipuAI
import base64
import os

client = ZhipuAI(api_key=os.getenv("ZHIPU_API_KEY"))

# 图片转 base64
def encode_image(image_path):
    with open(image_path, "rb") as f:
        return base64.b64encode(f.read()).decode("utf-8")

# 多模态对话
response = client.chat.completions.create(
    model="glm-4v-plus",
    messages=[
        {
            "role": "user",
            "content": [
                {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{encode_image('test.jpg')}"}},
                {"type": "text", "text": "这张图片里有什么？"}
            ]
        }
    ]
)
print(response.choices[0].message.content)
```

### Qwen VL API

```python
from openai import OpenAI
import os

client = OpenAI(
    api_key=os.getenv("DASHSCOPE_API_KEY"),
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1"
)

# 多模态对话
response = client.chat.completions.create(
    model="qwen-vl-plus",
    messages=[
        {
            "role": "user",
            "content": [
                {"type": "image_url", "image_url": {"url": "https://example.com/image.jpg"}},
                {"type": "text", "text": "描述这张图片"}
            ]
        }
    ]
)
print(response.choices[0].message.content)
```

### Kimi 多模态 API

```python
from openai import OpenAI
import os

client = OpenAI(
    api_key=os.getenv("MOONSHOT_API_KEY"),
    base_url="https://api.moonshot.cn/v1"
)

response = client.chat.completions.create(
    model="moonshot-v1-multimodal",
    messages=[
        {
            "role": "user",
            "content": [
                {"type": "image_url", "image_url": {"url": "https://example.com/image.jpg"}},
                {"type": "text", "text": "这张图片的主要内容是什么？"}
            ]
        }
    ]
)
print(response.choices[0].message.content)
```

---

## Part 4：实战——图像理解应用

### 图像问答助手

```python
import os
from dotenv import load_dotenv
from openai import OpenAI

load_dotenv()

class ImageAssistant:
    def __init__(self, provider="zhipuai"):
        if provider == "zhipuai":
            self.client = OpenAI(
                api_key=os.getenv("ZHIPU_API_KEY"),
                base_url="https://open.bigmodel.cn/api/paas/v4"
            )
            self.model = "glm-4v-plus"
        elif provider == "qwen":
            self.client = OpenAI(
                api_key=os.getenv("DASHSCOPE_API_KEY"),
                base_url="https://dashscope.aliyuncs.com/compatible-mode/v1"
            )
            self.model = "qwen-vl-plus"
        else:
            self.client = OpenAI(
                api_key=os.getenv("MOONSHOT_API_KEY"),
                base_url="https://api.moonshot.cn/v1"
            )
            self.model = "moonshot-v1-multimodal"

    def encode_image(self, image_path):
        import base64
        with open(image_path, "rb") as f:
            return base64.b64encode(f.read()).decode("utf-8")

    def ask(self, image_path, question):
        ext = image_path.split(".")[-1].lower()
        mime_type = f"image/{ext}" if ext in ["jpg", "jpeg", "png", "gif", "webp"] else "image/jpeg"

        response = self.client.chat.completions.create(
            model=self.model,
            messages=[
                {
                    "role": "user",
                    "content": [
                        {"type": "image_url", "image_url": {"url": f"data:{mime_type};base64,{self.encode_image(image_path)}"}},
                        {"type": "text", "text": question}
                    ]
                }
            ]
        )
        return response.choices[0].message.content

# 使用
assistant = ImageAssistant(provider="zhipuai")
result = assistant.ask("invoice.jpg", "这张发票的总金额是多少？")
print(result)
```

### 文档OCR提取

```python
class DocumentOCR:
    def __init__(self):
        from openai import OpenAI
        import os
        from dotenv import load_dotenv
        load_dotenv()

        self.client = OpenAI(
            api_key=os.getenv("ZHIPU_API_KEY"),
            base_url="https://open.bigmodel.cn/api/paas/v4"
        )

    def extract_text(self, image_path):
        """提取图片中的所有文字"""
        import base64
        with open(image_path, "rb") as f:
            img_data = base64.b64encode(f.read()).decode("utf-8")

        response = self.client.chat.completions.create(
            model="glm-4v-plus",
            messages=[
                {
                    "role": "user",
                    "content": [
                        {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{img_data}"}},
                        {"type": "text", "text": "请提取图片中的所有文字，保持原有格式。"}
                    ]
                }
            ]
        )
        return response.choices[0].message.content

# 使用
ocr = DocumentOCR()
text = ocr.extract_text("document.jpg")
print(text)
```

---

## Part 5：国产多模态模型对比

| 模型 | 公司 | 特点 | 适用场景 |
|------|------|------|----------|
| GLM-4V | 智谱 | 中文理解强 | 文档理解、图像问答 |
| Qwen-VL | 阿里 | 开源最强 | 通用视觉理解 |
| Kimi Vision | 月之暗面 | 长文本+多模态 | 复杂文档分析 |
| 讯飞星火 | 科大讯飞 | 语音+视觉 | 语音交互场景 |

---

## 今日要点

- ✅ 理解了多模态的定义：模型能同时处理多种模态
- ✅ 掌握了三种多模态架构的优缺点
- ✅ 学会了国产多模态 API 的调用方法（GLM、Qwen、Kimi）
- ✅ 实现了图像问答助手和 OCR 提取功能

---

## 自测题

1. 什么是多模态大模型？
2. 拼接式架构的三个步骤是什么？
3. 国产有哪些多模态模型可用？
4. 如何调用 GLM 的多模态 API？

---

## 动手作业

1. 注册国产大模型平台，申请多模态 API Key
2. 找一张图片，测试图像问答功能
3. 实现一个发票信息提取应用
