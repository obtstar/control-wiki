---
type: source-note
title: "OpenAPI 规范 v3.1.0 — 对象定义（一）：4.8.1-4.8.15（OpenAPI 至 Encoding 对象）"
description: "OpenAPI 规范（OAS）v3.1.0 对象定义前半：OpenAPI/Info/Contact/License/Server/Server Variable/Components/Paths/Path Item/Operation/External Documentation/Parameter/Request Body/Media Type/Encoding 对象的字段目录、参数位置与 style 序列化矩阵、多文件上传与 multipart 默认 Content-Type 规则。"
tags: [OpenAPI, API规范, 参数序列化, multipart, 服务器变量]
doc_type: 技术规范
authority: 5
resource: "https://spec.openapis.org.cn/oas/v3.1.0.html"
sources:
  - raw/converted/oas/v3.1.0.md
timestamp: "2021-02-15"
key_claims:
  - "OpenAPI 对象（文档根对象）固定字段：openapi（必需，规范版本号）、info（必需）、jsonSchemaDialect（$schema 默认值 URI）、servers（默认 url 为 /）、paths、webhooks、components、security（空对象 {} 使安全可选）、tags（名称唯一）、externalDocs。"
  - "Info 对象固定字段：title（必需）、summary、description、termsOfService（URL 形式）、contact、license、version（必需，与 OAS 规范版本无关）。License 对象：name（必需）、identifier（SPDX 表达式，与 url 互斥）、url（与 identifier 互斥）。"
  - "Server 对象：url（必需，支持 {变量} 模板替换）、description、variables；Server Variable 对象：enum（数组不得为空）、default（必需）、description。"
  - "Components 对象十类可复用对象（schemas/responses/parameters/examples/requestBodies/headers/securitySchemes/links/callbacks/pathItems），键必须匹配正则 ^[a-zA-Z0-9\\.\\-_]+$。"
  - "Paths 对象字段名必须以 / 开头，具体路径先于模板化路径匹配，同层次不同模板名非法；Path Item 固定字段 $ref/summary/description/get/put/post/delete/options/head/patch/trace/servers/parameters；Operation 对象 operationId 全 API 唯一、requestBody 语义见 RFC7231 §4.3.1、deprecated 默认 false。"
  - "Parameter 对象：name+in 组合定义唯一参数；in=path 时 required 必须为 true；header 的 Accept/Content-Type/Authorization 参数定义应忽略；schema 与 content 二选一；style 默认 query/cookie=form、path/header=simple。"
  - "Request Body 对象：content（必需）、description、required（默认 false）；Media Type 对象：schema、example（与 examples 互斥）、examples、encoding（仅 multipart 或 x-www-form-urlencoded 的 requestBody）。"
  - "文件上传：文件 I/O 用与任何其他 schema 相同的语义描述；contentEncoding 支持 RFC4648 全部编码（base64/base64url）与 quoted-printable；multipart 默认 Content-Type：原始类型 text/plain、复杂类型 application/json、带 contentEncoding 的 string 为 application/octet-stream。"
related_to: []
contradicts: []
supports: []
---

# OpenAPI 规范 v3.1.0 — 对象定义（一）：4.8.1-4.8.15

## Source

- Path: `raw/converted/oas/v3.1.0.md`
- Type: 技术规范（第 4.8.1-4.8.15 节）
- Author: Darrel Miller, Jeremy Whitlock, Marsh Gardiner, Mike Ralphson, Ron Ratovsky, Uri Sarid（编辑）
- Published: 2021-02-15
- Imported: 由 PieKBS 从 raw 文件修改日期设置

## Summary

本节覆盖 OAS 3.1.0 对象定义前半部分。OpenAPI 对象是文档根对象；Info/Contact/License
提供 API 元数据；Server/Server Variable 定义服务器 URL 模板与变量替换；Components
提供可复用对象；Paths/Path Item/Operation 定义路径与操作；Parameter 定义参数位置
（path/query/header/cookie）、style 序列化与展开规则；Request Body/Media Type/Encoding
定义请求体、媒体类型、文件上传与 multipart 默认 Content-Type。参数序列化 style 矩阵
与文件上传注意事项为本节重点。

## Key Facts

### 4.8.1 OpenAPI 对象（文档根对象）

| 字段名称 | 类型 | 描述 |
|---|---|---|
| openapi | 字符串 | **必需**。OpenAPI 规范版本号（与 info.version 无关） |
| info | Info 对象 | **必需**。API 元数据 |
| jsonSchemaDialect | 字符串 | 文档内 Schema 对象 $schema 的默认值，必须为 URI 形式 |
| servers | [Server 对象] | 缺省时默认 url 值为 / |
| paths | Paths 对象 | API 可用的路径和操作 |
| webhooks | Map[string, Path Item 对象\|引用对象] | 传入 Webhook（带外注册发起） |
| components | Components 对象 | 可复用元素 |
| security | [Security Requirement 对象] | 空对象 {} 使安全可选；操作可覆盖 |
| tags | [Tag 对象] | 名称必须唯一 |
| externalDocs | External Documentation 对象 | 其他外部文档 |

### 4.8.2 Info 对象

title（**必需**）/ summary / description / termsOfService（URL 形式）/ contact /
license / version（**必需**，与 OAS 规范版本不同）。

### 4.8.3 Contact 对象

name（识别名称）/ url（URL 形式）/ email（电子邮件格式）。

### 4.8.4 License 对象

name（**必需**）/ identifier（SPDX 表达式，与 url **互斥**）/ url（URL 格式，与 identifier **互斥**）。

### 4.8.5 Server 对象

url（**必需**，支持 {变量} 模板替换，可相对）/ description / variables
（Map[string, Server Variable 对象]）。

### 4.8.6 Server Variable 对象

enum（[string]，**数组不得为空**）/ default（**必需**，enum 定义时须在枚举中）/ description。

### 4.8.7 Components 对象

十类固定字段（键匹配 `^[a-zA-Z0-9\.\-_]+$`）：schemas / responses / parameters /
examples / requestBodies / headers / securitySchemes / links / callbacks / pathItems。
除非被外部显式引用，组件对 API 无影响。

### 4.8.8 Paths 对象

| 字段模式 | 类型 | 描述 |
|---|---|---|
| /{path} | Path Item 对象 | 必须以 / 开头；具体路径先于模板化路径匹配；同层次不同模板名的模板化路径非法；歧义由工具决定 |

因 ACL 约束，路径可以为空。

### 4.8.9 Path Item 对象

$ref / summary / description / get / put / post / delete / options / head / patch /
trace（Operation 对象）/ servers（备用数组）/ parameters（路径级参数，可覆盖不可删除，
唯一性 = 名称+位置）。因 ACL 约束可为空。

### 4.8.10 Operation 对象

tags / summary / description / externalDocs / operationId（**全 API 唯一**，区分大小写）/
parameters / requestBody（RFC7231 §4.3.1 明确语义的方法完全支持；GET/HEAD/DELETE 允许但
语义未明确定义，应避免）/ responses / callbacks / deprecated（默认 false）/ security
（空数组移除顶级声明）/ servers。

### 4.8.11 External Documentation 对象

description（可选）/ url（**必需**，URL 形式）。

### 4.8.12 Parameter 对象

**参数位置（in）**：path（如 /items/{itemId} 的 itemId）/ query / header（标头名不区分
大小写）/ cookie。

固定字段：name（**必需**，区分大小写；in=path 须对应路径模板表达式；in=header 且为
Accept/Content-Type/Authorization 时应忽略）/ in（**必需**）/ description / required
（**in=path 必须为 true**）/ deprecated / allowEmptyValue（仅 query，默认 false）。

序列化字段：style（默认 query/cookie=form、path/header=simple）/ explode（form 默认 true，
其余 false）/ allowReserved（仅 query）/ schema / example（与 examples 互斥）/ examples /
content（Map 必须仅一个条目）。**约束：schema 与 content 二选一。**

**style 值表**：matrix/label=path；form=query,cookie；simple=path,header；
spaceDelimited/pipeDelimited/deepObject=query。渲染差异（color=blue / [blue,black,brown] /
{R:100,G:200,B:150}）：matrix false → `;color=blue,black,brown`、`;color=R,100,G,200,B,150`；
form true → `color=blue&color=black&color=brown`、`R=100&G=200&B=150`；deepObject true →
`color[R]=100&color[G]=200&color[B]=150`。

### 4.8.13 Request Body 对象

description / content（**必需**，多键命中最具体键适用）/ required（默认 false）。

### 4.8.14 Media Type 对象

schema / example（与 examples **互斥**）/ examples / encoding（仅 multipart 或
application/x-www-form-urlencoded 的 requestBody 可用）。

**文件上传注意事项**：与 2.0 相反，文件 I/O 用与任何其他 schema 相同的语义描述；与 3.0
相反，format 对内容编码无影响——用 `contentEncoding`（支持 RFC4648 全部编码含
base64/base64url、quoted-printable）；`contentMediaType` 已被键或 encoding.contentType
指定时必须忽略。二进制传输可省略 schema（`image/png: {}`）；base64 传输用
`type: string + contentEncoding: base64`；多文件上传必须用 multipart。

**multipart 默认 Content-Type**：原始类型/原始值数组 → text/plain；复杂类型/复杂值数组 →
application/json；带 contentEncoding 的 type: string → application/octet-stream。

### 4.8.15 Encoding 对象

contentType（默认：object → application/json、array → 基于内部类型、其余 →
application/octet-stream；可具体/通配符/逗号分隔列表）/ headers（如 Content-Disposition；
非 multipart 必须忽略）/ style（行为同 query 参数）/ explode / allowReserved（后三项仅
application/x-www-form-urlencoded 或 multipart/form-data 有效）。

## Quotes

- "在匹配 URL 时，将先匹配具体的（非模板化的）路径，然后再匹配其模板化的对应路径。"
- "参数必须包含 schema 属性或 content 属性，但不能同时包含两者。"

## Terms

- **路径模板（Path Templating）**：{表达式} 标记 URL 路径部分，参数值禁止未转义 /?#。
- **style / explode**：参数序列化方式与展开标志（form/simple/matrix/label/spaceDelimited/pipeDelimited/deepObject）。
- **服务器变量（Server Variable）**：Server.url 中 {变量} 的替换定义。
- **operationId**：全文档唯一、区分大小写的操作标识。
- **contentEncoding / contentMediaType**：JSON Schema 2020-12 关键字，OAS 3.1 用于文件传输语义。

## Limitations

- 本文档为对象定义部分（4.8.1-4.8.15）字段目录与约束；完整 JSON/YAML 示例代码块见 raw
  原文 `raw/converted/oas/v3.1.0.md` 对应行段（369-1821）。
- 4.8.16-4.8.30 见 [03-objects-part2.md](03-objects-part2.md)；第 1-3、4.1-4.7 节见
  [01-overview.md](01-overview.md)。

## Related Pages

- [OpenAPI 规范 v3.1.0 — 简介、定义、版本与格式（01-overview.md）](01-overview.md)
- [OpenAPI 规范 v3.1.0 — 对象定义（二）（03-objects-part2.md）](03-objects-part2.md)
- [OpenAPI 规范 v3.1.0 — 索引（index.md）](index.md)
