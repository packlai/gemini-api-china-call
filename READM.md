# Gemini API 国内调用教程：接口地址、API Key 与 Python 示例

想在自己的项目里使用 Gemini，第一步通常就是搜索“**Gemini API 国内调用**”或“**Gemini API 调用教程**”。

但真正开始接入时，很多开发者会卡在几个具体问题上：

- API Key 从哪里获取？
- 请求地址应该填什么？
- Gemini 模型名称怎么写？
- 已有的 OpenAI 接口代码能不能复用？
- 为什么请求返回 401、404 或 429？

如果你计划通过 **大模型 API 中转站** 接入 Gemini，最小调用流程通常只需要三个核心配置：**API Key、请求地址和模型 ID**。

本文以支持 OpenAI Chat Completions 兼容格式的中转接口为例，带你完成一次 Gemini API 请求，再说明如何接入实际业务。

**大模型 API 中转服务入口：**

> 平台地址：<https://quanzil.com>

> 平台地址：<https://quanzil.net>

具体接口地址、可用模型、价格与功能支持范围，请以平台后台和开发文档为准。本文代码中的占位内容需要替换后再运行。

---

## Gemini API 国内调用有哪些方式

接入 Gemini 时，可以根据自己的项目环境和需求选择接口方式。

| 接入方式 | 需要准备的内容 | 适合的情况 |
| --- | --- | --- |
| Gemini 官方接口 | 官方凭据、对应服务地址和可用访问环境 | 需要直接使用官方服务与原生功能 |
| Gemini 中转 API | 中转平台 Key、接口地址和支持的模型 ID | 希望通过中转服务接入 Gemini |
| OpenAI 兼容中转接口 | 平台 Key、Base URL 和兼容模型配置 | 已有 OpenAI 格式代码，或需要统一接入多个模型 |

选择官方服务时，需要确认其当前支持的地区、账户条件和使用要求。

选择中转服务时，需要确认平台是否支持你需要的 Gemini 模型和功能。**中转平台的可用性、功能和计费规则由该平台决定，不能直接套用官方服务的说明。**

本文重点介绍第三种方式：通过 OpenAI 兼容中转接口调用 Gemini。

如果你还不熟悉 API 中转，可以先阅读：

- [API中转站是什么？一篇看懂大模型 API 中转的作用与适用场景](/blog/api-middleman/what-is-api-middleman.html)
- [大模型 API 中转和官方 API 有什么区别？开发者如何选择](/blog/api-middleman/api-middleman-vs-official-api.html)

---

## 调用前先确认三个配置

很多 Gemini API 调用失败，并不是代码逻辑有问题，而是配置没有对应起来。

### API Key：确认使用哪个平台的凭据

使用中转接口时，应填写中转平台签发的 API Key。

不要把官方 Key 与中转地址混用，也不要把其他平台的 Key 填进当前平台的接口配置。

通过本文使用的兼容接口发送请求时，鉴权通常采用：

```text
Authorization: Bearer YOUR_API_KEY
```

API Key 建议放在服务端环境变量中，避免出现在浏览器前端或公开代码仓库里。

### 请求地址：区分 Base URL 和完整接口地址

平台文档可能同时提供基础地址和具体接口路径。

例如：

```text
Base URL：
https://your-api-domain.com/v1

完整聊天接口地址：
https://your-api-domain.com/v1/chat/completions
```

以上域名仅用于说明结构，不是真实调用地址。

通常：

- 用 cURL 或 Python `requests` 直接发请求，需要完整接口地址。
- 用 OpenAI SDK，需要按平台文档填写 Base URL。

如果地址路径重复，或者缺少必要路径，请求可能返回 `404`。

### 模型 ID：从平台支持列表复制

`model` 字段决定请求使用哪个模型。

模型 ID 应从平台当前支持列表中复制，不建议根据产品名称自行猜测。

同一个系列可能存在不同版本，平台还可能使用自己的模型别名。接入时应同时确认：

- 模型是否可用
- 当前 Key 是否具有调用权限
- 是否支持 Chat Completions 兼容接口
- 是否支持所需的文本、图片或其他功能

---

## 第一步：设置 Gemini API 调用配置

下面通过环境变量管理 Key、接口地址和模型名称。

在 macOS 或 Linux 终端中执行：

```bash
export AI_API_KEY="替换为你的中转平台API Key"
export AI_CHAT_URL="替换为平台提供的完整聊天接口地址"
export GEMINI_MODEL="替换为平台支持的Gemini模型ID"
```

如果使用 Windows PowerShell：

```powershell
$env:AI_API_KEY="替换为你的中转平台API Key"
$env:AI_CHAT_URL="替换为平台提供的完整聊天接口地址"
$env:GEMINI_MODEL="替换为平台支持的Gemini模型ID"
```

这里的 `AI_CHAT_URL` 应填写完整接口地址，而不是只有域名。

环境变量名称由项目自行约定。后续代码使用什么名称，配置时就需要保持一致。

---

## 第二步：用 cURL 测试 Gemini API

在编写业务代码之前，建议先用 cURL 发起一次最小请求。

下面的示例适用于 Bash，并使用 Python 生成 JSON 请求体，避免手动拼接模型名称时出现格式问题：

```bash
python - <<'PY' | curl --silent --show-error \
  --connect-timeout 10 \
  --max-time 90 \
  "$AI_CHAT_URL" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $AI_API_KEY" \
  --data-binary @-
import json
import os

payload = {
    "model": os.environ["GEMINI_MODEL"],
    "messages": [
        {
            "role": "user",
            "content": "请用一句话说明 Gemini API 可以做什么。"
        }
    ]
}

print(json.dumps(payload, ensure_ascii=False))
PY
```

如果接口正常，通常会返回包含模型回复的 JSON。

在 Chat Completions 兼容响应中，文本内容一般位于：

```text
choices[0].message.content
```

具体响应结构仍需以平台文档为准。

第一次测试时，建议只发送 `model` 和 `messages`。基础请求成功后，再添加流式输出、生成参数或其他功能。

---

## 第三步：用 Python 调用 Gemini API

如果你想把 Gemini 接入网站后端、机器人或自动化脚本，可以先使用 Python `requests`。

安装依赖：

```bash
python -m pip install requests
```

创建 `gemini_api_demo.py`：

```python
import os
import requests

required_vars = [
    "AI_API_KEY",
    "AI_CHAT_URL",
    "GEMINI_MODEL",
]

missing_vars = [
    name for name in required_vars
    if not os.environ.get(name)
]

if missing_vars:
    raise RuntimeError(
        "请先配置环境变量：" + ", ".join(missing_vars)
    )

headers = {
    "Content-Type": "application/json",
    "Authorization": f"Bearer {os.environ['AI_API_KEY']}",
}

payload = {
    "model": os.environ["GEMINI_MODEL"],
    "messages": [
        {
            "role": "user",
            "content": "请列出 Gemini API 的三个常见应用场景。"
        }
    ],
}

try:
    response = requests.post(
        os.environ["AI_CHAT_URL"],
        headers=headers,
        json=payload,
        timeout=(10, 90),
    )

    response.raise_for_status()
    data = response.json()

    choices = data.get("choices")
    if not choices:
        raise RuntimeError(
            "响应中没有 choices，请检查平台的返回格式。"
        )

    print(choices[0]["message"].get("content"))

except requests.exceptions.Timeout:
    print("请求超时，请检查网络、服务状态和任务长度。")

except requests.exceptions.HTTPError as exc:
    print(f"接口返回 HTTP 错误：{exc.response.status_code}")
    print("请结合平台返回的错误详情排查配置、权限或额度。")

except requests.exceptions.RequestException as exc:
    print(f"请求失败：{exc}")

except ValueError:
    print("接口未返回有效 JSON，请检查响应格式。")
```

运行：

```bash
python gemini_api_demo.py
```

如果输出了模型回复，说明这次 Gemini API 调用已经成功。

示例中的超时设置分别限制连接等待和响应读取等待。正式项目应根据业务延迟要求调整，而不是一味增加超时时间。

---

## 已有 OpenAI SDK 项目怎么接入 Gemini

如果你的项目已经使用 OpenAI SDK，可以在平台支持兼容调用的情况下复用基础请求逻辑。

先安装依赖：

```bash
python -m pip install --upgrade openai
```

另外配置平台提供的 Base URL：

```bash
export AI_BASE_URL="替换为平台提供的Base URL"
```

调用示例：

```python
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["AI_API_KEY"],
    base_url=os.environ["AI_BASE_URL"],
    timeout=90.0,
    max_retries=2,
)

response = client.chat.completions.create(
    model=os.environ["GEMINI_MODEL"],
    messages=[
        {
            "role": "user",
            "content": "请介绍如何把 Gemini 接入智能客服。"
        }
    ],
)

print(response.choices[0].message.content)
```

注意：这里使用的是 `AI_BASE_URL`，不是前面用于直接 HTTP 请求的 `AI_CHAT_URL`。

基础文本请求可以复用，并不意味着所有高级功能都能直接迁移。工具调用、结构化输出、图片输入等能力，需要按平台和具体模型的支持情况验证。

更详细的 SDK 接入说明可以阅读：

- [Gemini API 怎么用 OpenAI SDK 调用？Python 接入与中转配置教程](/blog/gemini-api/gemini-openai-sdk-python.html)

---

## Gemini API 国内调用失败怎么排查

排查时建议同时查看 HTTP 状态码和接口返回的错误内容。

### 401：鉴权失败

常见原因包括：

- API Key 填写错误
- Key 已失效
- 请求头缺少鉴权信息
- 程序没有读取到环境变量
- Key 与请求平台不匹配

优先检查 Key 和请求地址的对应关系，再检查 Key 的状态。

### 403：权限或访问受到限制

可能与模型权限、账户状态、访问策略或地区要求有关。

如果是模型权限问题，需要确认当前账户是否可以调用该模型；如果涉及服务访问要求，应按服务提供商的说明处理。

### 404：地址或资源不存在

先检查完整接口地址、Base URL 和模型 ID。

尤其要注意：

- SDK 配置是否重复包含接口路径
- 直接 HTTP 请求是否缺少接口路径
- 平台是否支持当前 endpoint
- 模型名称是否已经变更

### 400：请求格式或参数不受支持

可以先删除额外参数，只保留 `model` 和最简单的文本 `messages`。

如果最小请求成功，再逐个添加其他参数，确认是哪一项不兼容。

输入过长、消息格式错误，以及模型不支持的功能，也可能触发此类错误。

### 429：限流或配额不足

如果错误详情显示请求频率过高，可以降低并发、使用队列，并设置有限次数的退避重试。

如果提示余额不足或配额耗尽，需要先处理账户额度，持续重试通常无效。

### 超时、500 或 502：检查网络与服务状态

这类问题可能来自连接异常、服务端故障或上游响应异常。

建议记录请求时间、状态码和平台提供的请求 ID，便于定位问题。对可重试的临时错误设置有限重试，并关注重复请求可能产生的费用。

---

## 接入业务时，Gemini 模型怎么选

跑通接口后，下一步是选择适合业务的模型。

| 任务 | 可以优先测试的模型方向 | 重点比较 |
| --- | --- | --- |
| 简单问答、分类、短摘要 | Flash 类模型 | 速度、成本、准确率 |
| 复杂分析、代码和长文理解 | Pro 类模型 | 任务完成质量、响应耗时 |
| 图片问答、截图分析 | 支持图片输入的模型 | 输入格式、识别效果 |
| 大量批处理 | 满足质量要求的低成本模型 | 单位任务成本、限流情况 |

这些是测试方向，具体表现取决于模型版本和实际任务。

建议准备一组真实业务样例，记录回答质量、调用耗时和使用量，再决定使用哪个模型。不要只根据模型名称或宣传描述选择。

---

## Gemini API 调用费用怎么计算

文本调用的费用通常与输入、输出 token 和模型单价有关。

如果平台按百万 token 报价，可以按下面的方式估算：

```text
文本调用费用 ≈
输入 token ÷ 1,000,000 × 输入单价
+
输出 token ÷ 1,000,000 × 输出单价
```

实际计费还可能涉及缓存、推理 token、多模态内容或平台其他规则，应以对应价格说明为准。

控制成本时，可以优先做三件事：

- 只传当前任务需要的上下文，减少重复内容。
- 根据任务限制回答长度，避免简单问题生成长文。
- 用不同模型处理不同复杂度的任务。

多轮对话尤其需要注意：如果每次都发送完整历史，后续请求的输入量可能不断增长。

相关价格信息可以查看：

- [价格说明](/pricing.html)

---

## 国内开发者选择 Gemini 中转 API，要看哪些信息

选择服务时，建议结合自己的部署环境做小规模测试，并确认：

- 是否提供明确的接口文档和模型列表
- 是否支持你的 SDK 与所需功能
- 是否说明价格、限流和额度规则
- 是否提供调用记录与使用量统计
- 是否说明数据处理、日志保留和隐私政策
- 是否能够从你的实际服务器环境正常访问

基础文本调用成功之后，再验证流式输出、图片输入、工具调用等业务所需能力。

对于涉及客户资料或内部文档的应用，还应根据平台的数据处理说明，判断哪些内容适合发送。

---

## 常见问题

### Gemini API 国内可以直接调用吗？

是否能够使用官方服务，需要结合其当前地区支持、账户要求和实际网络环境判断。

通过中转平台调用时，则需要确认平台的服务条件、模型支持情况，以及你的服务器是否能够访问对应接口。

### Gemini API 调用一定要用 Python 吗？

不需要。

只要能够发送 HTTPS 请求，JavaScript、Java、Go、PHP 等语言都可以接入。本文使用 Python，是为了方便展示最小调用流程。

### Gemini 中转 API 能用于智能客服吗？

可以作为客服系统的模型能力来源。

实际应用还需要结合商品资料、业务规则、知识库检索和人工转接流程。API 提供模型能力，客服业务逻辑仍需要由你的系统实现。

### Gemini API 支持图片理解吗？

需要选择支持图片输入的模型，并确认中转接口提供对应输入格式。

本文的文本请求示例不能直接替代图片调用示例。

### 换 Gemini 模型需要重写代码吗？

在平台支持相同基础接口格式的情况下，简单文本请求通常可以通过修改模型 ID 进行测试。

但模型支持的参数、输出效果和功能范围可能不同，上线前仍需验证。

### 调用成功后，为什么答案不符合预期？

接口成功表示请求完成，不代表回答已经满足业务要求。

可以从提示词、输入资料、模型选择和输出格式要求入手改进，并用真实业务样例检查效果。

---

## 从最小请求开始接入 Gemini

完成一次 **Gemini API 国内调用**，可以按下面的顺序操作：

1. 获取中转平台提供的 API Key。
2. 确认完整接口地址或 SDK Base URL。
3. 从模型列表复制可用的 Gemini 模型 ID。
4. 使用 cURL 或 Python 发送最小文本请求。
5. 检查回复结构，再接入实际业务。
6. 增加超时处理、限流控制和费用记录。

如果你的项目后续还要接入其他大模型，OpenAI 兼容中转接口可以帮助复用基础调用代码。模型的特殊能力与参数，则需要分别确认。

你可以从以下入口查看大模型 API 中转服务：

> <https://quanzil.com>

> <https://quanzil.net>

