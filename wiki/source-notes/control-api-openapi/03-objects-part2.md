---
type: source-note
title: "OpenAPI 规范 v3.1.0 — 对象定义（二）：4.8.16-4.8.30（Responses 至 Security Requirement）"
description: "OpenAPI 规范（OAS）v3.1.0 对象定义后半：Responses/Response/Callback/Example/Link/Header/Tag/Reference/Schema/Discriminator/XML/Security Scheme/OAuth Flows/OAuth Flow/Security Requirement 对象的字段目录与约束（运行时表达式 ABNF、JSON Schema 2020-12 超集、OAS 方言模式 ID、多态 discriminator）。"
tags: [OpenAPI, API规范, JSON Schema, OAuth2, 运行时表达式]
doc_type: 技术规范
authority: 5
resource: "https://spec.openapis.org.cn/oas/v3.1.0.html"
sources:
  - raw/converted/oas/v3.1.0.md
timestamp: "2021-02-15"
key_claims:
  - "Responses 对象把 HTTP 状态代码映射到预期响应；模式化字段可用大写通配符 X 定义范围（仅 1XX/2XX/3XX/4XX/5XX），显式代码优先于范围定义；default 字段覆盖未声明代码。"
  - "Response 对象固定字段：description（必需）、headers（Map[string, 头部对象|引用对象]）、content（Map[string, 媒体类型对象]）、links（Map[string, 链接对象|引用对象]）；RFC7230 规定标头名不区分大小写，名为 Content-Type 的响应标头必须忽略。"
  - "Callback 对象的键是运行时表达式（如 $request.body#/url），值是被引用的路径项对象；运行时表达式 ABNF：expression = ( $url / $method / $statusCode / $request.source / $response.source )。"
  - "Link 对象用 operationRef（相对/绝对 URI）或 operationId（文档内唯一可解析）标识目标操作，两字段互斥；参数值可用常量或 {表达式}，参数名可用 [{in}.]{name} 限定。"
  - "Reference 对象固定字段 $ref（必需，URI 形式，RFC3986）、summary、description；不能扩展其他属性——这是它与含 $ref 关键字的 Schema 对象的区别。"
  - "Schema 对象是 JSON Schema 草案 2020-12 的超集，OAS 方言模式 ID 为 https://spec.openapis.org.cn/oas/3.1/dialect/base；固定字段 discriminator/xml/externalDocs/example（example 已弃用，改用 JSON Schema examples 关键字）。"
  - "Discriminator 对象（propertyName 必需 + mapping）仅在组合关键字 oneOf/anyOf/allOf 之一使用时合法；鉴别器值不匹配时验证应失败，映射键可为隐式模式名或显式 mapping 覆盖。"
  - "Security Scheme 对象 type 必填，有效值 apiKey/http/mutualTLS/oauth2/openIdConnect；http 的 scheme 值应在 IANA 身份验证方案注册表登记；openIdConnectUrl 必须使用 TLS。"
  - "OAuth Flows 对象：implicit/password/clientCredentials（2.0 曾名 application）/authorizationCode（2.0 曾名 accessCode）；OAuth Flow 对象 authorizationUrl（implicit/authorizationCode）、tokenUrl（password/clientCredentials/authorizationCode）、refreshUrl、scopes（必需）。"
  - "Security Requirement 对象：多个方案在同一对象内为 AND（都须满足），对象列表之间为 OR（满足其一即可授权）；oauth2/openIdConnect 类型的值是该流程所需范围名列表，可为空列表；空对象 {} 表示可选安全。"
related_to: []
contradicts: []
supports: []
---

# OpenAPI 规范 v3.1.0 — 对象定义（二）：4.8.16-4.8.30

## Source

- Path: `raw/converted/oas/v3.1.0.md`
- Type: 技术规范（第 4.8.16-4.8.30 节）
- Author: Darrel Miller, Jeremy Whitlock, Marsh Gardiner, Mike Ralphson, Ron Ratovsky, Uri Sarid（编辑）
- Published: 2021-02-15
- Imported: 由 PieKBS 从 raw 文件修改日期设置

## Summary

本节覆盖 OAS 3.1.0 规范对象定义的后半部分（4.8.16-4.8.30）。Responses/Response 描述
操作响应容器与单条响应；Callback 与 Link 引入运行时表达式（runtime expression）实现
带外回调与响应驱动链接；Example 提供示例；Header 是 Parameter 的 header 位置特化；
Tag 提供标签元数据；Reference 提供 $ref 引用机制。Schema 对象是规范中最重要的类型定义
设施——JSON Schema 草案 2020-12 的超集，含 OAS 方言、多态 discriminator 与 XML 建模。
安全对象（Security Scheme/OAuth Flows/OAuth Flow/Security Requirement）定义
apiKey/http/mutualTLS/oauth2/openIdConnect 五种方案与 OAuth2 四流程。

## Key Facts

### 4.8.16 Responses 对象

操作预期响应的容器：HTTP 响应代码 → 预期响应。`default` 可作未单独涵盖代码的默认。
至少包含一个响应代码；仅一个时应为成功操作响应。

| 字段名称 | 类型 | 描述 |
|---|---|---|
| default | 响应对象\|引用对象 | 除为特定 HTTP 响应代码声明的响应之外的响应文档 |
| HTTP 状态代码（模式化字段） | 响应对象\|引用对象 | 任何 HTTP 状态代码可作属性名；为 JSON/YAML 兼容必须引号括起（如 "200"）；可含大写通配符 X 定义范围，仅允许 1XX/2XX/3XX/4XX/5XX；显式代码优先于范围定义 |

示例：`"200"` 返回 Pet（`$ref: '#/components/schemas/Pet'`），`default` 返回
ErrorModel（`$ref: '#/components/schemas/ErrorModel'`）。

### 4.8.17 Response 对象

| 字段名称 | 类型 | 描述 |
|---|---|---|
| description | 字符串 | **必需**。响应描述，[CommonMark] 可作富文本 |
| headers | Map[string, 头部对象\|引用对象] | 标头名 → 定义；标头名不区分大小写；名为 "Content-Type" 的响应标头必须忽略 |
| content | Map[string, 媒体类型对象] | 潜在响应有效负载描述；多键命中时最具体键适用 |
| links | Map[string, 链接对象\|引用对象] | 可从响应跟随的操作链接映射 |

### 4.8.18 Callback 对象

可能的带外回调映射，每个值是一个路径项对象。键为运行时计算的表达式。

| 字段模式 | 类型 | 描述 |
|---|---|---|
| {表达式} | 路径项对象\|引用对象 | 定义回调请求和预期响应 |

键表达式示例：`$request.body#/url`；可计算 `$url`、`$method`、`$request.path.eventType`、
`$request.query.queryUrl`、`$request.header.content-Type`、`$request.body#/failedUrl`、
`$response.header.Location`。独立于其他 API 调用的传入请求用 `webhooks` 字段。

### 4.8.19 Example 对象

| 字段名称 | 类型 | 描述 |
|---|---|---|
| summary | 字符串 | 简短描述 |
| description | 字符串 | 详细描述 |
| value | 任何 | 嵌入文字示例；与 externalValue **互斥** |
| externalValue | 字符串 | 指向文字示例的 URI；与 value **互斥** |

### 4.8.20 Link 对象

| 字段名称 | 类型 | 描述 |
|---|---|---|
| operationRef | 字符串 | 到 OAS 操作的相对/绝对 URI 引用；与 operationId **互斥** |
| operationId | 字符串 | 现有可解析 OAS 操作名称；与 operationRef **互斥** |
| parameters | Map[string, Any \| {表达式}] | 键为参数名（可 [{in}.]{name} 限定如 path.id），值可为常量或表达式 |
| requestBody | Any \| {表达式} | 调用目标操作时的请求体 |
| description | 字符串 | 链接描述 |
| server | 服务器对象 | 目标操作要使用的服务器 |

operationRef 使用 JSON 引用时需转义正斜杠（如 `#/paths/~12.0~1repositories~1{username}/get`）。
运行时表达式计算失败时不传递参数值。

**运行时表达式 ABNF**（链接与回调共用）：`expression = ( "$url" / "$method" / "$statusCode"
/ "$request." source / "$response." source )`；source = header-reference / query-reference /
path-reference / body-reference；body-reference = "body" ["#" json-pointer ]；name 区分
大小写，token 不区分。

### 4.8.21 Header 对象

遵循 Parameter 对象结构，两点更改：`name` 不得指定（在 headers 映射中给出）；`in` 不得
指定（隐式为 header）。受位置影响的特征须适用 header 位置（如 style）。

### 4.8.22 Tag 对象

name（**必需**）/ description / externalDocs（外部文档对象）。

### 4.8.23 Reference 对象

| 字段名称 | 类型 | 描述 |
|---|---|---|
| $ref | 字符串 | **必需**。引用标识符，必须是 URI 形式（RFC3986） |
| summary | 字符串 | 默认覆盖被引用组件的摘要 |
| description | 字符串 | 默认覆盖被引用组件的描述 |

**不能扩展其他属性**（任何添加属性被忽略）——这是与含 $ref 的 Schema 对象的区别。
示例：`$ref: '#/components/schemas/Pet'`；相对文档 `Pet.json`；嵌入式 `definitions.json#/Pet`。

### 4.8.24 Schema 对象

允许定义输入/输出数据类型，是 **JSON Schema 草案 2020-12 的超集**。OAS Schema 对象方言
模式 ID：`https://spec.openapis.org.cn/oas/3.1/dialect/base`。

| 字段名称 | 类型 | 描述 |
|---|---|---|
| discriminator | Discriminator 对象 | 多态支持 |
| xml | XML 对象 | 仅可用于属性模式，对根模式无影响 |
| externalDocs | 外部文档对象 | 其他外部文档 |
| example | 任何 | **已弃用**：改用 JSON Schema `examples` 关键字 |

- **组合与继承（多态）**：`allOf` 组合模型；`discriminator` 必须是必需字段；鉴别器值可用
  模式名或 mapping 覆盖；无给定 ID 的内联模式不能用于多态。
- **方言指定**：`$schema` 可出现在根 Schema 对象；`OpenAPI 对象` 的 `jsonSchemaDialect`
  设置默认值；Schema 对象内 `$schema` 覆盖默认。
- **XML 建模**：`xml` 属性在 JSON→XML 转换时提供额外定义。

### 4.8.25 Discriminator 对象

propertyName（**必需**）/ mapping（Map[string, string]）。仅当使用 `oneOf`/`anyOf`/`allOf`
之一时合法；使用鉴别器时不考虑内联模式；值无匹配（隐式或显式 mapping）时验证失败。

### 4.8.26 XML 对象

name（替换元素/属性名；items 内影响单个元素名）/ namespace（**必须为绝对 URI**）/ prefix /
attribute（默认 false）/ wrapped（仅数组定义，默认 false）。

### 4.8.27 Security Scheme 对象

| 字段名称 | 类型 | 应用于 | 描述 |
|---|---|---|---|
| type | 字符串 | 任何 | **必需**。apiKey / http / mutualTLS / oauth2 / openIdConnect |
| description | 字符串 | 任何 | 方案描述 |
| name | 字符串 | apiKey | **必需**。标头/查询/cookie 参数名 |
| in | 字符串 | apiKey | **必需**。query / header / cookie |
| scheme | 字符串 | http | **必需**。Authorization 头中的 HTTP 授权方案名（RFC7235 §5.1），应注册于 IANA |
| bearerFormat | 字符串 | http ("bearer") | 提示 bearer 令牌格式（文档用途） |
| flows | OAuth Flows 对象 | oauth2 | **必需**。支持的流程类型配置 |
| openIdConnectUrl | 字符串 | openIdConnect | **必需**。OpenID Connect URL，要求 TLS |

注：截至 2020 年，OAuth2 隐式流程即将被弃用；大多数用例建议带 PKCE 的授权码流程。

### 4.8.28 OAuth Flows 对象

implicit / password / clientCredentials（2.0 曾名 application）/ authorizationCode（2.0
曾名 accessCode），均为 OAuth Flow 对象。

### 4.8.29 OAuth Flow 对象

authorizationUrl（implicit/authorizationCode，**必需**，要求 TLS）/ tokenUrl
（password/clientCredentials/authorizationCode，**必需**，要求 TLS）/ refreshUrl /
scopes（**必需**，Map[string, string]，可为空映射）。

### 4.8.30 Security Requirement 对象

| 字段模式 | 类型 | 描述 |
|---|---|---|
| {名称} | [string] | 名称须对应 components.securitySchemes 声明；oauth2/openIdConnect 类型值为所需范围名列表（可空）；其他类型可为角色名列表 |

同一对象内多方案为 AND；对象列表之间为 OR。可选安全示例：
`security: [ {}, { petstore_auth: [write:pets, read:pets] } ]`。

## Quotes

- "此对象是 JSON Schema 规范草案 2020-12 的超集。"
- "OpenAPI 规范允许使用 JSON Schema 的 allOf 属性组合和扩展模型定义，实际上提供了模型组合。"

## Terms

- **运行时表达式（runtime expression）**：基于实际 API 调用中 HTTP 消息信息计算的值，ABNF 定义见 4.8.20.4。
- **OAS 方言模式 ID**：`https://spec.openapis.org.cn/oas/3.1/dialect/base`。
- **鉴别器（discriminator）**：负载中用于选择多态模式的属性。
- **RFC7230/7231/7235**：HTTP 标头大小写、媒体类型范围、HTTP 认证方案定义来源。
- **RFC6901（JSON Pointer）**：运行时表达式 body-reference 的 #/json-pointer 语法来源。

## Limitations

- 本文档为对象定义部分（4.8.16-4.8.30）字段目录与约束；完整 JSON/YAML 示例见 raw 原文
  `raw/converted/oas/v3.1.0.md` 对应行段（1822-3488）。
- 4.8.1-4.8.15 见 [02-objects-part1.md](02-objects-part1.md)；第 1-3、4.1-4.7 节见
  [01-overview.md](01-overview.md)。

## Related Pages

- [OpenAPI 规范 v3.1.0 — 简介、定义、版本与格式（01-overview.md）](01-overview.md)
- [OpenAPI 规范 v3.1.0 — 对象定义（一）（02-objects-part1.md）](02-objects-part1.md)
- [OpenAPI 规范 v3.1.0 — 索引（index.md）](index.md)
