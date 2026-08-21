---
type: source-note
title: "OpenAPI 规范 v3.1.0（OAS）— 简介、定义、版本与格式（第 1-3 节、4.1-4.7 节）"
description: "OpenAPI 规范（OAS，OpenAPI Specification）v3.1.0 中文版前半部分：简介、定义（OpenAPI 文档/路径模板/媒体类型/HTTP 状态码）、版本（major.minor.patch）、格式（JSON/YAML）、文档结构、数据类型（integer int32/int64、number float/double、string password）、富文本与相对引用规则。"
tags: [OpenAPI, API规范, HTTP, JSON Schema, 路径模板]
doc_type: 技术规范
authority: 5
resource: "https://spec.openapis.org.cn/oas/v3.1.0.html"
sources:
  - raw/converted/oas/v3.1.0.md
timestamp: "2021-02-15"
key_claims:
  - "OpenAPI 文档必须至少包含一个 paths 字段、一个 components 字段或一个 webhooks 字段；OpenAPI 文档是自包含或复合资源，用于定义或描述 API 或 API 的元素。"
  - "路径模板（Path Templating）使用花括号（{}）分隔的模板表达式标记 URL 路径的一部分；路径中的每个模板表达式必须对应于包含在 Path Item 本身和/或其每个 Operation 中的路径参数。"
  - "路径参数的值禁止包含未转义的通用语法字符：正斜杠（/）、问号（?）或井号（#），如 RFC3986 第 3 节所述。"
  - "媒体类型定义分散在多个资源中，应该符合 RFC6838；示例包括 text/plain; charset=utf-8、application/json、application/vnd.github+json 等。"
  - "HTTP 状态码用于指示执行的操作的状态，可用状态码由 RFC7231 第 6 节定义，注册的状态码列在 IANA 状态码注册表中。"
  - "OpenAPI 规范使用 major.minor.patch（主.次.补丁）版本控制方案，major.minor 部分（如 3.1）必须指定 OAS 功能集；.patch 版本只解决此文档中的错误或澄清，而不是功能集。"
  - "OAS 3.1 数据类型基于 JSON Schema 规范草案 2020-12 支持的类型，integer 被定义为没有小数部分或指数部分的 JSON 数字；OAS 定义了额外格式：int32（带符号 32 位）、int64（带符号 64 位，又名长整型）、float、double、password（提示 UI 隐藏输入）。"
  - "规范中的所有字段名称都区分大小写；架构公开两种类型的字段：固定字段（具有声明的名称）和模式字段（为字段名称声明正则表达式模式）。"
related_to: []
contradicts: []
supports: []
---

# OpenAPI 规范 v3.1.0 — 简介、定义、版本与格式

## Source

- Path: `raw/converted/oas/v3.1.0.md`
- Type: 技术规范（第 1-3 节、4.1-4.7 节）
- Author: Darrel Miller, Jeremy Whitlock, Marsh Gardiner, Mike Ralphson, Ron Ratovsky, Uri Sarid（编辑）
- Published: 2021-02-15
- Imported: 由 PieKBS 从 raw 文件修改日期设置

## Summary

本节覆盖规范的引言部分与规范主体的前半部分。第 1-3 节交代文档状态、简介与核心定义：OpenAPI 文档（Document）、路径模板（Path Templating）、媒体类型（Media Types）与 HTTP 状态码（HTTP Status Codes）。第 4.1-4.7 节说明版本控制方案、文档格式（JSON 或 YAML）、文档结构、数据类型、富文本格式化以及 URI/URL 中的相对引用规则。

关键词"必须、禁止、需要、应、不应、应该、不应该、推荐、不推荐、可以、可选"按 BCP 14（RFC2119、RFC8174）解释，且仅当以全部大写字母出现时生效。文档本身以 Apache 许可证 2.0 版许可，版权归 Linux 基金会。

## Key Facts

### 1. OpenAPI 规范（第 1 节）

- 本文档中的关键词"必须"、"禁止"、"需要"、"应"、"不应"、"应该"、"不应该"、"推荐"、"不推荐"、"可以"和"可选"应按 BCP 14 [RFC2119] [RFC8174] 中所述进行解释，当且仅当它们以全部大写字母出现时，如这里所示。
- 本文档根据 Apache 许可证 2.0 版获得许可。

### 2. 简介（第 2 节）

- OpenAPI 规范 (OAS) 定义了一种标准的、与语言无关的 HTTP API 接口，使人和计算机都能够发现和理解服务的的能力，而无需访问源代码、文档或通过网络流量检查。
- OpenAPI 定义可用于文档生成工具来显示 API、代码生成工具来使用各种编程语言生成服务器和客户端、测试工具以及许多其他用例。

### 3. 定义（第 3 节）

#### 3.1 OpenAPI 文档

一个自包含或复合资源，用于定义或描述 API 或 API 的元素。OpenAPI 文档必须至少包含一个 `paths` 字段、一个 `components` 字段或一个 `webhooks` 字段。OpenAPI 文档使用并符合 OpenAPI 规范。

#### 3.2 路径模板

路径模板指的是使用由花括号(`{}`)分隔的模板表达式来标记 URL 路径的一部分，以便使用路径参数替换。

- 路径中的每个模板表达式必须对应于包含在 Path Item 本身和/或其每个 Operation 中的路径参数。如果路径项为空，例如由于 ACL 约束，则不需要匹配路径参数。
- 这些路径参数的值禁止包含 [RFC3986] 第 3 节中描述的任何未转义的"通用语法"字符：正斜杠 (/)、问号 (?) 或井号 (#)。

#### 3.3 媒体类型

媒体类型定义分散在多个资源中。媒体类型定义应该符合 [RFC6838]。一些可能的媒体类型定义示例：

- `text/plain; charset=utf-8`
- `application/json`
- `application/vnd.github+json`
- `application/vnd.github.v3+json`
- `application/vnd.github.v3.raw+json`
- `application/vnd.github.v3.text+json`
- `application/vnd.github.v3.html+json`
- `application/vnd.github.v3.full+json`
- `application/vnd.github.v3.diff`
- `application/vnd.github.v3.patch`

#### 3.4 HTTP 状态码

HTTP 状态码用于指示执行的操作的状态。可用状态码由 [RFC7231] 第 6 节定义，注册的状态码列在 IANA 状态码注册表中。

### 4. 版本（4.1）

- OpenAPI 规范使用 major.minor.patch 版本控制方案。版本字符串的 major.minor 部分（例如 3.1）必须指定 OAS 功能集。.patch 版本解决此文档中的错误或提供澄清，而不是功能集。
- 支持 OAS 3.1 的工具应该与所有 OAS 3.1.* 版本兼容。补丁版本不应该被工具考虑，例如不区分 3.1.0 和 3.1.1。
- 偶尔，在 OAS 的 minor 版本中可能会进行不向后兼容的更改，如果认为更改对提供的好处的影响较小。
- 与 OAS 3.*.* 兼容的 OpenAPI 文档包含一个必需的 `openapi` 字段，该字段指定它使用的 OAS 版本。

### 5. 格式（4.2）

- 符合 OpenAPI 规范的 OpenAPI 文档本身是一个 JSON 对象，可以以 JSON 或 YAML 格式表示。
- 规范中的所有字段名称都区分大小写。这包括用作映射中键的所有字段，除非明确指出键不区分大小写。
- 架构公开两种类型的字段：**固定字段**，具有声明的名称；**模式字段**，为字段名称声明正则表达式模式。模式字段必须在包含的对象中具有唯一的名称。
- 为了保留在 YAML 和 JSON 格式之间进行双向转换的能力，YAML 1.2 版本建议与一些其他约束一起使用：标签必须限于 JSON Schema 规则集允许的标签；YAML 映射中使用的键必须限于标量字符串，如 YAML Failsafe 架构规则集定义的那样。
- 注意：虽然 API 可以通过 YAML 或 JSON 格式的 OpenAPI 文档来定义，但 API 请求和响应主体以及其他内容不需要是 JSON 或 YAML。

### 6. 文档结构（4.3）

- OpenAPI 文档可以由单个文档组成，也可以根据作者的意愿划分为多个连接的部分。在后一种情况下，将使用引用对象和架构对象 `$ref` 关键字。
- 建议将根 OpenAPI 文档命名为：`openapi.json` 或 `openapi.yaml`。

### 7. 数据类型（4.4）

OAS 中的数据类型基于 JSON 架构规范草案 2020-12 支持的类型。请注意，`integer` 作为一种类型也受支持，并且被定义为没有小数部分或指数部分的 JSON 数字。模型使用架构对象定义，它是 JSON 架构规范草案 2020-12 的超集。

如 JSON 架构验证词汇表所定义，数据类型可以具有可选的修饰符属性：`format`。OAS 定义了其他格式，以便为基本数据类型提供更详细的信息。OAS 定义的格式为：

| 类型    | 格式 | 注释                                   |
|-----------|--------|------------------------------------------|
| 整数    | int32  | 带符号的 32 位                      |
| 整数    | int64  | 带符号的 64 位（又名长整型） |
| 数字    | float  |                                          |
| 数字    | double |                                          |
| 字符串 | 密码 | 提示 UI 隐藏输入。                |

### 8. 富文本格式化（4.5）

- 在整个规范中，`description` 字段都被注释为支持 [CommonMark] markdown 格式。在 OpenAPI 工具呈现富文本时，必须至少支持 [CommonMark-0.27] 中描述的 markdown 语法。工具可以选择忽略某些 CommonMark 功能以解决安全问题。

### 9. URI 中的相对引用（4.6）

- 除非另有说明，否则所有作为 URI 的属性可以是 [RFC3986] 第 4.2 节中定义的相对引用。
- 相对引用，包括引用对象、路径项对象 `$ref` 字段、链接对象 `operationRef` 字段和示例对象 `externalValue` 字段中的相对引用，都使用引用文档作为根据 [RFC3986] 第 5.2 节的基本 URI 进行解析。
- 如果 URI 包含片段标识符，则应根据引用文档的片段解析机制解析片段。如果引用文档的表示形式为 JSON 或 YAML，则片段标识符应该被解释为根据 [RFC6901] 的 JSON 指针。
- 架构对象中的相对引用，包括任何作为 `$id` 值出现的相对引用，都使用最近的父级 `$id` 作为基本 URI，如 JSON 架构规范草案 2020-12 中所述。如果没有任何父架构包含 `$id`，则必须根据 [RFC3986] 第 5.1 节确定基本 URI。

### 10. URL 中的相对引用（4.7）

- 除非另有说明，否则所有作为 URL 的属性可以是 [RFC3986] 第 4.2 节中定义的相对引用。
- 除非另有说明，否则相对引用将使用服务器对象中定义的 URL 作为基本 URL 进行解析。请注意，这些 URL 本身可以相对于引用文档。

## Important Quotes Or Evidence

> "OpenAPI 规范 (OAS) 定义了一种标准的、与语言无关的 HTTP API 接口，使人和计算机都能够发现和理解服务的的能力，而无需访问源代码、文档或通过网络流量检查。"
> "路径模板指的是使用由花括号({})分隔的模板表达式来标记 URL 路径的一部分，以便使用路径参数替换。"
> "与 OAS 3.*.* 兼容的 OpenAPI 文档包含一个必需的 openapi 字段，该字段指定它使用的 OAS 版本。"

## Terms

- 路径模板（Path Templating）：用 `{}` 分隔的模板表达式标记 URL 路径部分，以路径参数替换。
- 固定字段（Fixed Field）：具有声明名称的字段；模式字段（Patterned Field）：为字段名称声明正则表达式模式的字段。
- BCP 14：RFC2119 与 RFC8174 合称，规范需求级别关键词定义。
- JSON Schema 草案 2020-12：OAS 3.1 数据类型的规范基础。
- CommonMark：description 字段的富文本标记语法标准（至少支持 CommonMark-0.27）。
- RFC3986：URI 通用语法；RFC6901：JSON 指针；RFC6838：媒体类型规范；RFC7231：HTTP/1.1 语义和内容；RFC1866（4.8.14 使用）：HTML 2.0。
- IANA 状态码注册表：已注册 HTTP 状态码的官方列表。

## Limitations

- 本页覆盖第 1-3 节与 4.1-4.7 节；4.8 节 30 个对象的固定字段表格见分节页 02-07。
- 数据类型表按原文逐字保留（表头"类型/格式/注释"与单元格原文一致）。
- 源为中文翻译版，标识符（`openapi`、`paths`、`webhooks`、`$ref`、`$id`、`operationRef`、`externalValue` 等）保留英文原文。

## Related Pages

- [00-index.md](00-index.md)（分页索引）
