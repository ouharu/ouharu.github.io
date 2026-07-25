---
title: 用iPhone快捷指令 + AWS Lambda + Bedrock Claude,搭建一个AI自动记账系统
tags:
  - aws
  - claude
  - 快捷指令
categories:
  - Diary
category_bar: true
math: true
date: 2026-07-13 02:28:12
---

{% note info %}
每次买完东西,票据要么随手丢了,要么攒成一沓,想统计一下每月花在餐饮、超市、数码上的钱,最后往往不了了之。这篇文章记录我如何用一套几乎全自动化的流程,把"拍照"这一个动作,变成一条结构化的Notion账单记录。

整体思路很简单:
`
iPhone拍照 → 压缩 → 传给AWS Lambda → Lambda调用Bedrock上的Claude视觉模型识别小票 → 结构化结果写入Notion数据库
`
全程不需要手动录入任何数字,拍完照几秒钟后,Notion里就多了一条带图片、带商家名、带金额分类的完整记录。
{% endnote %}


## 一、整体架构

先上一张架构图,把整条链路的每一环都画出来:

```mermaid
graph LR
    A[ iPhone拍照] --> B[快捷指令:压缩图片]
    B --> C[快捷指令:Base64编码]
    C --> D[POST请求]
    D --> E[AWS Lambda]
    E --> F[Bedrock Claude视觉模型]
    F -->|识别出结构化字段| E
    E -->|1.创建上传任务| G[Notion File Upload API]
    E -->|2.发送图片字节| G
    E -->|3.创建页面写入数据| H[(Notion数据库)]
    G -. 图片存储在 .-> H

    style A fill:#e8f4fd
    style E fill:#fff4e0
    style F fill:#fce4ec
    style H fill:#e8f5e9
```

图中可以看到,整条链路只有一次网络请求从手机发出(拍照→压缩→编码→POST),剩下所有的复杂逻辑(调用Bedrock、三段式上传Notion)全部封装在Lambda内部,手机端不需要感知这些细节。

在设计之初,有几个关键决策点:

### 1. 用AI视觉识别,而非传统OCR

传统OCR(如Textract的AnalyzeExpense)对规整的票据识别效果不错,但字段是固定的,遇到手写、模糊、非标准格式的小票容错率较低。直接用Bedrock上的Claude视觉模型,可以让AI"理解"小票内容,自己判断分类、拼接商品清单,灵活性更高。

### 2. 图片不落地S3,直接在Lambda内存中流转

图片全程走向是:iPhone → Lambda(内存)→ Notion自己的文件存储。不经过S3这一层,减少了一个存储环节和相关的管理成本。

### 3. 手机端只做"拍照+发送",逻辑全部收敛到Lambda

早期我在快捷指令里直接调用Notion的三段式File Upload API(创建上传任务→发送文件→创建页面),验证链路可行后,把这部分逻辑整体迁移到Lambda里,手机端只需要一次网络请求。

---

## 二、Notion数据库设计

数据库包含以下字段:

| 字段名 | 类型 | 说明 |
|---|---|---|
| 商家(Title) | 标题 | AI识别出的店名 |
| 日期 | Date | 小票上的消费时间 |
| 总金额 | Number | 含税总额 |
| 分类 | Select | 餐饮/超市/数码/缴费/其他 |
| 支付方式 | Select | 信用卡/現金/手机支付/其他 |
| 商品明细 | Text | AI拼接的"商品名 价格 数量 JANコード"列表 |
| 小票原图 | Files & media | 直传Notion自己的存储,不经S3 |
| 商品图片 | Files & media | 由JAN码查乐天API得到的商品图(见第八章,后期新增) |
| 记录创建时间 | Created time | Notion自动记录,不需要代码写入 |

**几个设计上的取舍:**

- 商品条目数量不固定(有的小票只有一件商品,有的有十几件),所以用一个多行文本字段拼接展示,而不是拆成"商品1/商品2..."这种固定列
- "记录创建时间"用Notion原生的Created time类型,创建页面时自动生成,不需要在代码里手动传时间戳,零维护成本
- 「商品图片」列是后期新增的功能(详见第八章),同样利用了`Files & media`一格可放多张图的特性,避免为此新建一个关联数据库

---

## 三、Notion API配置

### 1. 创建Integration,获取Token

Notion的自动化写入,依赖一个叫"Integration"(或者叫"Connection",不同版本UI叫法略有差异)的机制,本质上是一个专属的API访问凭证。

1. 打开 [notion.so/my-integrations](https://www.notion.so/my-integrations)
2. 点"+ New integration",填写名称,选择关联的workspace
3. 创建后,进入配置页面,找到"Internal Integration Secret"这一栏,点"Show"显示出来
4. 复制这个token——新建的token是`ntn_`开头(旧版本是`secret_`开头,仍然有效)

**安全提示**:这个token拥有对workspace的写入权限,绝对不能提交到代码仓库、不能贴在聊天记录或者任何公开场合。如果不小心泄露了,要立刻回到这个页面重置(Refresh)。

### 2. 把Integration连接到目标数据库

仅仅创建了Integration还不够,必须显式把它"连接"到你想操作的具体数据库,否则调用API时会收到权限错误。

1. 打开你要写入的Notion数据库页面(整页视图)
2. 右上角点"..."(三个点菜单)
3. 找到"Connect to"(部分版本叫"Add connections")
4. 搜索并选中你创建的Integration

### 3. 获取数据库ID

1. 打开这个数据库的整页视图
2. 复制浏览器地址栏的URL,形如:
   ```
   https://www.notion.so/xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx?v=yyyyyyyy
   ```
3. `notion.so/`后面到`?v=`前面那段32位字符,就是数据库ID(有没有横杠都可以)

### 4. 每次API请求必带的Headers

```
Authorization: Bearer {你的token}
Notion-Version: 2026-03-11
Content-Type: application/json
```

Notion的API是按版本迭代的,`Notion-Version`这个头必须显式声明,不同版本之间字段结构可能有差异(比如后期引入的"data source"概念,对多数据源的database有影响)。

---

## 四、Notion API踩坑记录

### 401 Unauthorized:token放错了位置

第一次调试时一直收到401报错,排查后发现:**Authorization和Notion-Version被错误地放进了"请求体(JSON)"里,而不是"请求头(Headers)"里**。这两个信息属于HTTP协议层面的元数据,必须放在Headers,放进body里服务器完全读不到。

### JSON嵌套结构:字典模式下容易迷路

Notion创建页面的请求体是深层嵌套的(parent→properties→商家→title→数组→词典→text→content),在快捷指令这种"字典模式"的界面里手动搭建,很容易把某个字段放错层级(比如误把properties塞进了parent里面)。建议每加一层,就退出来截图核对一次结构。

---

## 五、Lambda + Bedrock 核心代码

### 整体流程

```python
def lambda_handler(event, context):
    # 1. 解析请求体,拿到Base64图片
    image_bytes = base64.b64decode(image_b64)

    # 2. 调用Bedrock Claude视觉模型,识别小票内容
    extracted = _extract_receipt_fields(image_bytes)

    # 3. 三步上传到Notion:创建上传任务 → 发送图片 → 创建页面
    file_upload_info = _notion_create_file_upload()
    upload_result = _notion_send_file(file_upload_info, image_bytes)
    page = _notion_create_page(extracted, file_upload_info)

    return _response(200, {...})
```

### 自动检测图片格式,而非写死JPEG

考虑到以后可能遇到不同来源的图片(不只是iPhone直接拍照),加了一个基于文件头字节判断真实格式的函数:

```python
def _detect_media_type(image_bytes):
    """根据图片二进制头部字节,自动判断真实的图片格式"""
    if image_bytes[:8] == b"\x89PNG\r\n\x1a\n":
        return "image/png"
    if image_bytes[:3] == b"\xff\xd8\xff":
        return "image/jpeg"
    if image_bytes[:6] in (b"GIF87a", b"GIF89a"):
        return "image/gif"
    if image_bytes[:4] == b"RIFF" and image_bytes[8:12] == b"WEBP":
        return "image/webp"
    return "image/jpeg"
```

这样不管图片实际是什么格式,调Bedrock和传Notion时都会用正确的media_type,不会因为类型不匹配而报错。

### Prompt设计:明确告知每个字段的数据类型

调试中踩过一个坑:prompt里给`total_amount`的示例值加了引号,导致AI把它当成字符串返回(`"1617"`而非`1617`),写入Notion时因为类型不匹配直接报错。

修复方式是双重保险:

1. **JSON模板本身不给total_amount加引号**,暗示这里应该是数字
2. **额外用自然语言明确写出每个字段的类型要求**:

```
各字段的数据类型要求(务必严格遵守):
- merchant: 字符串(string),用双引号包裹
- total_amount: 数字(number),不加引号,不带货币符号或千分位逗号
...
```

3. **代码层面再加一层防御性转换**,即便AI偶尔还是返回字符串,也能自动纠正:

```python
if extracted.get("total_amount") is not None:
    try:
        properties["总金额"] = {"number": float(extracted["total_amount"])}
    except (TypeError, ValueError):
        pass
```

### 错误处理:让HTTP错误的详细信息不被吞掉

早期`except Exception as e: return _response(500, {"error": str(e)})`这种写法,对于`urllib.error.HTTPError`只会打印出"HTTP Error 400: Bad Request"这种毫无信息量的提示。改成专门捕获这个异常类型,把响应体读出来:

```python
except urllib.error.HTTPError as e:
    error_detail = e.read().decode("utf-8")
    return _response(500, {"error": f"HTTP {e.code}: {error_detail}"})
```

这样调试效率大幅提升,能直接看到Notion API返回的具体校验失败原因。

---

## 六、最终效果

拍一张真实的小票,几秒后Lambda返回:

```json
{
  "status": "success",
  "notion_page_url": "https://app.notion.com/p/...",
  "extracted": {
    "merchant": "はま寿司(Hamazushi)都島本通店",
    "date": "2018-04-03T20:36:00",
    "total_amount": 1617,
    "category": "餐饮",
    "payment_method": "現金",
    "items": "平日寿司皿90円 x15 ¥1,455\n寿司皿150円 x1 ¥162"
  },
  "image_upload": {
    "status": "uploaded",
    "filename": "receipt.jpeg",
    "content_type": "image/jpeg",
    "content_length": 524021
  }
}
```

对应地,Notion数据库里同步出现一条完整记录,商家名、金额、分类全部自动填好,还能点开查看原始小票图片。

---

## 七、经验总结

1. **先手动跑通,再自动化**:一开始在快捷指令里手动调三次Notion API,虽然繁琐,但让我彻底理解了File Upload API的完整流程,后续搬到Lambda时debug效率高很多
2. **AI的输出格式,要靠明确指令而非隐晦暗示**:prompt里的示例格式(带不带引号)会直接影响AI的输出类型,与其让AI"猜",不如直接用自然语言写清楚类型要求
3. **日期时间一定要显式规定时区**:AI从图片上识别出的时间字符串本身不带时区信息,如果不在prompt里明确要求(比如固定加上`+09:00`),Notion按UTC或默认时区解读时,存进去的时间可能会跟小票上写的对不上,产生几个小时的偏差
4. **代码里的防御性写法值得投入**:图片格式检测、数字类型转换,这些"看似多余"的代码,恰恰是应对真实世界数据不规整的关键
5. **错误信息要完整暴露,不要吞掉细节**:排查问题时,笼统的"HTTP 400"和具体的校验失败原因,debug效率天差地别

---

## 八、进阶更新:给每个商品配上真实商品图(2026-07-25)

> 本节是这套系统跑了一段时间后的一次功能升级,记录当天从需求到踩坑到跑通的完整过程。

### 8.1 需求由来

系统跑通后,Notion里每条账单已经有了商家、金额、分类和一段文本形式的商品明细。但翻看记录时总觉得少了点什么——**光看"ポケモン カクチョウインフェ 90点"这样一行文字,很难一眼想起当时到底买了什么**。于是冒出一个想法:能不能把每个商品的**条形码编号**和**商品图片**也展示出来?

### 8.2 两个绕不开的设计问题

动手前先想清楚两件事,否则方向一错就白忙。

**问题一:商品图片从哪来?**

这是最反直觉的一点:**小票照片里根本没有商品图**。小票上只有文字和(有时有的)条形码数字,不存在商品的实拍图。所以"展示商品图"这件事,必须从**外部数据源**去查。

最终方案:让Claude在识别小票时**顺便读出每个商品的JANコード(日本商品条形码,13位数字)**,再拿这个JAN码去调**乐天商品搜索API**,把返回的商品图URL填进Notion。

**问题二:Notion里怎么存多个商品的图?**

一张小票有好几个商品,一开始想到的是建一个独立的"商品DB",用Relation关联——这是最规范的关系型设计。但实际权衡后放弃了,原因是:

- 单DB方案下,Notion的`Files & media`类型属性**本身就支持一格放多张图**
- 我不需要"点开单个商品看它的独立属性",只要能在账单这一行里把几张商品图**平铺展示**出来就够了

所以最终选了**最轻量的方案**:

- **不建新DB**,只在现有账单DB上**加一列「商品图片」**(Files & media类型)
- **条形码不单独建列**,直接塞进原有的「商品明细」文本里
- 一格`files`属性里放多张外部图片URL

这个方案的"天花板"要认清:图片和商品是"一堆"而非"一一对应",Notion不会告诉你"第2张图对应哪个商品",它们只是靠顺序平铺在同一格里。对我的使用场景,这个取舍完全可以接受。

### 8.3 乐天商品搜索API的配置与踩坑

这一步是当天耗时最久的地方,坑很密集,单独记录下来。

**申请应用**

到[乐天开发者平台](https://webservice.rakuten.co.jp/)创建一个应用,拿到两样东西:

- `applicationId`:一串UUID(新版格式,不再是旧版的纯数字)
- `accessKey`:`pk_`开头的一串(publishable key)

创建时"アプリケーションタイプ"选了「Webアプリケーション」,并且要填一项「許可されたWebサイト」(允许的网站白名单)。我这里填的是 `*.amazonaws.com`。

**核心大坑:校验的是`Origin`头,不是`Referer`**

新版API域名是 `https://openapi.rakuten.co.jp/ichibams/api/IchibaItem/Search/20260701`,请求里带上`applicationId`和`accessKey`后,从命令行/后端直接调用,一直报:

```json
{"errors": {"errorCode": 403, "errorMessage": "REQUEST_CONTEXT_BODY_HTTP_REFERRER_MISSING"}}
```

`REFERRER_MISSING`这个错误名极具误导性——它写着"referrer",于是我在`Referer`请求头上换了各种域名反复试,**无论怎么带Referer,都还是报同样的missing**。

真正的突破口来自一个被忽略的对比信号:

| 请求带了什么 | 返回 |
|---|---|
| 不带来源头 / 只带`Referer` | 403 `REFERRER_MISSING` |
| 带`Origin`头(但值不匹配白名单) | **503 `Authentication service error`** |
| **带`Origin`头(值匹配白名单)** | **200 成功** ✅ |

关键在于:**带`Origin`时返回的是503(说明它真的去做认证校验了),带`Referer`时是403 missing(说明它压根没理会这个头)**。这说明乐天这个`pk_` key校验的是`Origin`头,而不是名字里写的那个`Referer`。

把三个条件凑齐后,终于返回200:

1. 带`Origin`请求头(不是`Referer`)
2. `Origin`的值要**匹配白名单**`*.amazonaws.com`,比如 `https://receipt.amazonaws.com`
3. `Origin`必须**带`https://`前缀**(去掉前缀会失败)

> 补充一点:这个`Origin`值和你Lambda的真实URL没有任何关系,乐天不会真去访问它,只做字符串匹配。它本质上就是一个"进门暗号",在代码里写死或走环境变量即可。之所以之前一直失败,还有一个原因是白名单一开始误填成了`*.rakuten.com`,和请求里的来源根本对不上。

**返回结果里图片的位置**

`format=json`时,商品图在 `Items[0].Item.mediumImageUrls[0].imageUrl`,URL常带`?_ex=128x128`的尺寸后缀,去掉后能拿到更清晰的原图。

### 8.4 代码改动(共4处)

整个升级只动了`lambda_function.py`一个文件,没有引入任何第三方库(依旧只用标准库`urllib`)。

**① 新增乐天相关配置和环境变量**

```python
import urllib.parse   # 新增,用于URL编码参数

RAKUTEN_APP_ID = os.environ["RAKUTEN_APP_ID"]          # applicationId(UUID)
RAKUTEN_ACCESS_KEY = os.environ["RAKUTEN_ACCESS_KEY"]  # accessKey(pk_开头)
RAKUTEN_ORIGIN = os.environ["RAKUTEN_ORIGIN"]          # 匹配白名单的Origin,带https://
RAKUTEN_SEARCH_URL = "https://openapi.rakuten.co.jp/ichibams/api/IchibaItem/Search/20260701"
```

**② 改Prompt,让商品明细带上JANコード**

原来`items`是"商品名 价格 数量"的格式,现在改成用**竖线`|`分隔**的四段式,多加了JANコード字段。用竖线而不是空格分隔,是为了避免商品名本身含空格时被拆错:

```
"items": "商品明细,每行一个商品,用换行符分隔,每行格式为「商品名 | 价格 | 数量 | JANコード」,
四个字段之间用竖线|分隔;如果小票上没有印JANコード(条形码数字),该位置留空,但竖线|要保留"
```

**③ 新增两个取图函数**

一个负责解析明细、遍历JAN码;一个负责调乐天API。单个商品取图失败(超时、查不到)时直接跳过,绝不影响主流程:

```python
def _lookup_product_images(items_text):
    """解析商品明细,遍历每行的JANコード,调乐天API查商品图片URL"""
    if not items_text:
        return []
    image_urls, seen = [], set()
    for line in items_text.splitlines():
        parts = [p.strip() for p in line.strip().split("|")]
        jan_code = parts[3] if len(parts) >= 4 else ""
        if not jan_code or not jan_code.isdigit():   # 没读到JAN码就跳过
            continue
        url = _rakuten_search_image(jan_code)
        if url and url not in seen:
            seen.add(url)
            image_urls.append(url)
    return image_urls


def _rakuten_search_image(keyword):
    """用JANコード调乐天API,返回第一条商品的代表图URL;失败返回None"""
    params = {
        "applicationId": RAKUTEN_APP_ID, "accessKey": RAKUTEN_ACCESS_KEY,
        "keyword": keyword, "hits": 1, "imageFlag": 1, "format": "json",
    }
    url = RAKUTEN_SEARCH_URL + "?" + urllib.parse.urlencode(params)
    req = urllib.request.Request(url, headers={
        "User-Agent": "receipt-processor/1.0",
        "Origin": RAKUTEN_ORIGIN,   # 关键:乐天校验的是Origin头,不是Referer!
    })
    try:
        with urllib.request.urlopen(req, timeout=5) as resp:
            data = json.loads(resp.read())
    except Exception:
        return None                 # 单个商品取图失败不中断整个流程
    items = data.get("Items", [])
    if not items:
        return None
    medium = items[0]["Item"].get("mediumImageUrls") or []
    return medium[0]["imageUrl"].split("?")[0] if medium else None
```

**④ 把图片URL写进Notion「商品图片」列**

Notion的`files`属性支持`external`类型,可以**直接贴外部图片URL,不需要像小票原图那样走三段式上传**。多个商品图组装成一个数组即可:

```python
image_urls = _lookup_product_images(extracted.get("items"))
if image_urls:
    properties["商品图片"] = {
        "files": [
            {"type": "external", "name": f"item_{i + 1}.jpg", "external": {"url": u}}
            for i, u in enumerate(image_urls)
        ]
    }
```

### 8.5 本地验证 + Remote Invoke实测

调乐天API的逻辑,先用一个独立的小脚本在本地反复试通(这也是定位`Origin`头那个坑的过程)。链路确认没问题后,再合进Lambda。

部署到Lambda并配好3个乐天环境变量后,直接用**AWS Toolkit的Remote Invoke**功能测试——它在本地VS Code里点一下,实际调用的是云上已部署的函数,好处是**Bedrock权限、环境变量、网络全部用真实运行环境,不用在本地装Docker或配AWS凭证**。

拿一张真实的ヨドバシカメラ(友都八喜)小票测试,返回:

```json
{
  "status": "success",
  "notion_page_url": "https://app.notion.com/p/...",
  "extracted": {
    "merchant": "ヨドバシカメラ 梅田店",
    "date": "2026-07-25T17:41:00+09:00",
    "total_amount": 20655,
    "category": "数码",
    "payment_method": "信用卡",
    "items": "ポケモン カクチョウインフェ 90点 | 16200 | 1 | 4521329431512\nポケモン スタートデッキ 5点 | 4455 | 1 | 4521329427270"
  }
}
```

最惊喜的是**Claude把两个13位JAN码一位不差地读了出来**,乐天API据此精准查到了对应商品图,Notion账单的「商品图片」列里同步出现了两张商品图。整条链路一次跑通。

### 8.6 本次升级的经验补充

1. **错误信息的名字未必可信**:`REFERRER_MISSING`实际指的是"缺少合法来源",而它认定来源的依据是`Origin`头。与其盯着字面意思死磕,不如对比"不同请求触发的不同错误码"来反推真实校验逻辑——503和403的区别,就是那个决定性线索
2. **设计要匹配真实需求,而非追求"标准"**:关系型的多DB方案更"正确",但我的需求只是"平铺展示几张图",单DB加一列就够了,过度设计反而增加维护成本
3. **外部依赖要做到"失败不扩散"**:取图是锦上添花的功能,所以`_rakuten_search_image`里任何异常都吞掉返回`None`,绝不能因为某个商品查不到图,就让整条记账链路失败
4. **Notion的`external`文件类型很省事**:相比小票原图那套三段式上传,外部URL直接贴进`files`属性,一行代码搞定,还不占用自己的存储

---

## 附:完整Lambda代码

```python
import json
import base64
import os
import urllib.request
import urllib.error
import urllib.parse
import uuid
import boto3

# ---- 配置 ----
NOTION_TOKEN = os.environ["NOTION_TOKEN"]
NOTION_DATABASE_ID = os.environ["NOTION_DATABASE_ID"]
NOTION_VERSION = "2026-03-11"
BEDROCK_MODEL_ID = os.environ["BEDROCK_MODEL_ID"]

# ---- 乐天商品搜索API配置(用条形码查商品图片) ----
RAKUTEN_APP_ID = os.environ["RAKUTEN_APP_ID"]          # applicationId(UUID)
RAKUTEN_ACCESS_KEY = os.environ["RAKUTEN_ACCESS_KEY"]  # accessKey(pk_开头)
# Origin头:必须匹配乐天后台「許可されたWebサイト」白名单,且带https://前缀
RAKUTEN_ORIGIN = os.environ["RAKUTEN_ORIGIN"]
RAKUTEN_SEARCH_URL = "https://openapi.rakuten.co.jp/ichibams/api/IchibaItem/Search/20260701"

bedrock = boto3.client("bedrock-runtime", region_name="ap-northeast-1")


def lambda_handler(event, context):
    try:
        # 1. 解析请求体,拿到Base64图片
        body = json.loads(event["body"]) if isinstance(event.get("body"), str) else event.get("body", {})
        image_b64 = body.get("image")
        if not image_b64:
            return _response(400, {"error": "缺少image字段"})

        image_bytes = base64.b64decode(image_b64)

        # 2. 调用Bedrock Claude视觉模型,识别小票内容
        extracted = _extract_receipt_fields(image_bytes)

        # 3. 三步上传到Notion:创建上传任务 → 发送图片 → 创建页面
        file_upload_info = _notion_create_file_upload()
        upload_result = _notion_send_file(file_upload_info, image_bytes)
        page = _notion_create_page(extracted, file_upload_info)

        return _response(200, {
            "status": "success",
            "notion_page_url": page.get("url"),
            "extracted": extracted,
            "image_upload": {
                "status": upload_result.get("status"),
                "filename": upload_result.get("filename"),
                "content_type": upload_result.get("content_type"),
                "content_length": upload_result.get("content_length")
            }
        })

    except urllib.error.HTTPError as e:
        error_detail = e.read().decode("utf-8")
        return _response(500, {"error": f"HTTP {e.code}: {error_detail}"})
    except Exception as e:
        return _response(500, {"error": str(e)})


def _detect_media_type(image_bytes):
    """根据图片二进制头部字节,自动判断真实的图片格式"""
    if image_bytes[:8] == b"\x89PNG\r\n\x1a\n":
        return "image/png"
    if image_bytes[:3] == b"\xff\xd8\xff":
        return "image/jpeg"
    if image_bytes[:6] in (b"GIF87a", b"GIF89a"):
        return "image/gif"
    if image_bytes[:4] == b"RIFF" and image_bytes[8:12] == b"WEBP":
        return "image/webp"
    # 默认按JPEG处理(iPhone快捷指令拍照后压缩,通常就是JPEG)
    return "image/jpeg"


def _extract_receipt_fields(image_bytes):
    """调用Bedrock Claude视觉模型,提取小票结构化字段"""
    image_b64 = base64.b64encode(image_bytes).decode("utf-8")
    media_type = _detect_media_type(image_bytes)

    prompt = """请分析这张购物小票图片,提取以下字段,严格按照JSON格式返回,不要包含任何其他文字说明或markdown标记:

{
  "merchant": "商家名称",
  "date": "YYYY-MM-DDTHH:MM:SS+09:00格式的日期时间,固定带上+09:00时区偏移,如果小票上没有具体时间只有日期,时间部分填00:00:00",
  "total_amount": 含税总金额的数字(不含货币符号,不加引号),
  "category": "根据商家和商品判断的分类,从这些选项里选一个:餐饮/超市/数码/缴费/其他",
  "payment_method": "支付方式,从这些选项里选一个:信用卡/現金/手机支付/其他",
  "items": "商品明细,每行一个商品,用换行符分隔,每行格式为「商品名 | 价格 | 数量 | JANコード」,四个字段之间用竖线|分隔;如果小票上没有印JANコード(条形码数字),该位置留空,但竖线|要保留"
}

各字段的数据类型要求(务必严格遵守):
- merchant: 字符串(string),用双引号包裹
- date: 字符串(string),用双引号包裹
- total_amount: 数字(number),不加引号,不带货币符号或千分位逗号,例如 1617 或 1617.5
- category: 字符串(string),用双引号包裹
- payment_method: 字符串(string),用双引号包裹
- items: 字符串(string),用双引号包裹,内部每行格式「商品名 | 价格 | 数量 | JANコード」,用竖线分隔,行与行之间用换行符

如果某个字段无法从图片中识别,对应值设为null。总金额如果看到多个数字,优先使用"合计"或"总计"这一行的最终金额。"""

    request_body = {
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 1024,
        "messages": [
            {
                "role": "user",
                "content": [
                    {
                        "type": "image",
                        "source": {
                            "type": "base64",
                            "media_type": media_type,
                            "data": image_b64
                        }
                    },
                    {
                        "type": "text",
                        "text": prompt
                    }
                ]
            }
        ]
    }

    response = bedrock.invoke_model(
        modelId=BEDROCK_MODEL_ID,
        body=json.dumps(request_body)
    )

    response_body = json.loads(response["body"].read())
    text_output = response_body["content"][0]["text"]

    # 清理可能的markdown代码块标记
    text_output = text_output.strip()
    if text_output.startswith("```"):
        text_output = text_output.split("```")[1]
        if text_output.startswith("json"):
            text_output = text_output[4:]
        text_output = text_output.strip()

    return json.loads(text_output)


def _lookup_product_images(items_text):
    """解析商品明细,遍历每行的JANコード,调乐天API查商品图片URL。
    返回图片URL列表(每个有JANコード且查到图的商品对应一张代表图)。
    """
    if not items_text:
        return []

    image_urls = []
    seen = set()
    for line in items_text.splitlines():
        line = line.strip()
        if not line:
            continue
        # 每行格式:商品名 | 价格 | 数量 | JANコード
        parts = [p.strip() for p in line.split("|")]
        jan_code = parts[3] if len(parts) >= 4 else ""
        # 只处理读到了JANコード(纯数字,通常8或13位)的商品
        if not jan_code or not jan_code.isdigit():
            continue

        url = _rakuten_search_image(jan_code)
        if url and url not in seen:
            seen.add(url)
            image_urls.append(url)

    return image_urls


def _rakuten_search_image(keyword):
    """用关键词(JANコード)调乐天商品搜索API,返回第一条商品的代表图URL;失败返回None"""
    params = {
        "applicationId": RAKUTEN_APP_ID,
        "accessKey": RAKUTEN_ACCESS_KEY,
        "keyword": keyword,
        "hits": 1,
        "imageFlag": 1,
        "format": "json",
    }
    url = RAKUTEN_SEARCH_URL + "?" + urllib.parse.urlencode(params)
    req = urllib.request.Request(url, headers={
        "User-Agent": "receipt-processor/1.0",
        "Origin": RAKUTEN_ORIGIN,  # 关键:乐天校验的是Origin头(必须匹配白名单)
    })
    try:
        with urllib.request.urlopen(req, timeout=5) as resp:
            data = json.loads(resp.read())
    except Exception:
        # 单个商品取图失败不应中断整个流程
        return None

    items = data.get("Items", [])
    if not items:
        return None
    item = items[0]["Item"]
    medium = item.get("mediumImageUrls") or []
    if not medium:
        return None
    # 去掉尺寸后缀(?_ex=128x128),拿更清晰的原图
    return medium[0]["imageUrl"].split("?")[0]


def _notion_create_file_upload():
    """第一步:创建Notion文件上传任务,返回file_upload_id"""
    req = urllib.request.Request(
        "https://api.notion.com/v1/file_uploads",
        data=json.dumps({}).encode("utf-8"),
        headers=_notion_headers(),
        method="POST"
    )
    with urllib.request.urlopen(req) as resp:
        result = json.loads(resp.read())
    return result["id"], result["upload_url"]


def _notion_send_file(file_upload_info, image_bytes):
    """第二步:把图片字节发送到upload_url(multipart/form-data)"""
    file_upload_id, upload_url = file_upload_info
    boundary = uuid.uuid4().hex
    media_type = _detect_media_type(image_bytes)
    extension = media_type.split("/")[1]

    body = (
        f"--{boundary}\r\n"
        f'Content-Disposition: form-data; name="file"; filename="receipt.{extension}"\r\n'
        f"Content-Type: {media_type}\r\n\r\n"
    ).encode("utf-8") + image_bytes + f"\r\n--{boundary}--\r\n".encode("utf-8")

    headers = {
        "Authorization": f"Bearer {NOTION_TOKEN}",
        "Notion-Version": NOTION_VERSION,
        "Content-Type": f"multipart/form-data; boundary={boundary}"
    }

    req = urllib.request.Request(upload_url, data=body, headers=headers, method="POST")
    with urllib.request.urlopen(req) as resp:
        return json.loads(resp.read())


def _notion_create_page(extracted, file_upload_info):
    """第三步:创建Notion页面,写入结构化数据+图片"""
    file_upload_id, _ = file_upload_info

    properties = {
        "商家": {
            "title": [{"text": {"content": extracted.get("merchant") or "未知商家"}}]
        },
        "小票原图": {
            "files": [{"type": "file_upload", "file_upload": {"id": file_upload_id}}]
        }
    }

    if extracted.get("date"):
        properties["日期"] = {"date": {"start": extracted["date"]}}

    if extracted.get("total_amount") is not None:
        try:
            properties["总金额"] = {"number": float(extracted["total_amount"])}
        except (TypeError, ValueError):
            pass

    if extracted.get("category"):
        properties["分类"] = {"select": {"name": extracted["category"]}}

    if extracted.get("payment_method"):
        properties["支付方式"] = {"select": {"name": extracted["payment_method"]}}

    if extracted.get("items"):
        properties["商品明细"] = {"rich_text": [{"text": {"content": extracted["items"]}}]}

    # 用商品的JANコード查乐天商品图,写入「商品图片」列(files类型,支持多张)
    image_urls = _lookup_product_images(extracted.get("items"))
    if image_urls:
        properties["商品图片"] = {
            "files": [
                {"type": "external", "name": f"item_{i + 1}.jpg", "external": {"url": u}}
                for i, u in enumerate(image_urls)
            ]
        }

    payload = {
        "parent": {"database_id": NOTION_DATABASE_ID},
        "properties": properties
    }

    req = urllib.request.Request(
        "https://api.notion.com/v1/pages",
        data=json.dumps(payload).encode("utf-8"),
        headers=_notion_headers(),
        method="POST"
    )
    with urllib.request.urlopen(req) as resp:
        return json.loads(resp.read())


def _notion_headers():
    return {
        "Authorization": f"Bearer {NOTION_TOKEN}",
        "Notion-Version": NOTION_VERSION,
        "Content-Type": "application/json"
    }


def _response(status_code, body_dict):
    return {
        "statusCode": status_code,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps(body_dict, ensure_ascii=False)
    }
```
