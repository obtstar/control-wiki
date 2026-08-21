---
type: source-note
title: "OpenAPI 规范 v3.1.0 — 规范扩展、安全过滤与附录（4.9-4.10、附录 A/B）"
description: "OpenAPI 规范（OAS）v3.1.0 收尾部分：4.9 规范扩展（x- 前缀模式化字段，x-oai-/x-oas- 保留）、4.10 安全过滤（路径对象/路径项对象可空置）、附录 A 修订历史（2011-2021，Swagger 1.0 → OAS 3.1.0）、附录 B 规范性引用（RFC2119/3986/6749/6901/7230-7235、JSON Schema 2020-12 等）。"
tags: [OpenAPI, API规范, 修订历史, RFC, 规范扩展]
doc_type: 技术规范
authority: 5
resource: "https://spec.openapis.org.cn/oas/v3.1.0.html"
sources:
  - raw/converted/oas/v3.1.0.md
timestamp: "2021-02-15"
key_claims:
  - "规范扩展属性实现为模式化字段，始终以 x- 为前缀（如 x-internal-id）；以 x-oai- 和 x-oas- 开头的字段名保留给 OpenAPI Initiative 定义使用。"
  - "安全过滤：路径对象可以存在但为空（查看者到达正确位置但无法访问任何文档，仍可访问 Info 对象中的认证信息）；路径项对象可以为空（知道路径存在但看不到操作/参数）。"
  - "修订历史：OAS 3.1.0 于 2021-02-15 发布；Swagger 2.0 于 2015-12-31 捐赠给 OpenAPI Initiative；规范首版 Swagger 1.0 发布于 2011-08-10。"
  - "规范性引用包括：RFC2119/RFC8174（需求关键词）、RFC3986（URI）、RFC6570（URI 模板）、RFC6749（OAuth 2.0）、RFC6901（JSON Pointer）、RFC7159（JSON）、RFC7230/7231/7235（HTTP/1.1）、RFC7578（multipart/form-data）、JSON Schema 草案 2020-12、ABNF（RFC5234）、CommonMark、YAML 1.2、SPDX 许可证列表。"
related_to:
  - "wiki/source-notes/control-api-openapi/03-objects-part2.md"
  - "wiki/source-notes/control-api-openapi/01-overview.md"
contradicts: []
supports: []
---

# OpenAPI 规范 v3.1.0 — 规范扩展、安全过滤与附录

## Source

- Path: `raw/converted/oas/v3.1.0.md`
- Type: 技术规范（第 4.9-4.10 节、附录 A/B）
- Author: Darrel Miller, Jeremy Whitlock, Marsh Gardiner, Mike Ralphson, Ron Ratovsky, Uri Sarid（编辑）
- Published: 2021-02-15
- Imported: 由 PieKBS 从 raw 文件修改日期设置

## Summary

本节覆盖 OAS 3.1.0 的收尾部分。4.9 规范扩展说明如何在规范允许的点位追加 x- 前缀的
模式化字段扩展规范；4.10 安全过滤说明路径对象/路径项对象可空置的访问控制层机制。
附录 A 列出 2011 年 Swagger 1.0 至 2021 年 OAS 3.1.0 的完整修订历史；附录 B 汇总全部
规范性引用（RFC、JSON Schema 2020-12、ABNF、CommonMark、YAML、SPDX）。

## Key Facts

### 4.9 规范扩展

扩展属性实现为模式化字段，字段名称**必须以 `x-` 开头**（如 `x-internal-id`）。
`x-oai-` 和 `x-oas-` 开头的字段名保留给 OpenAPI Initiative 定义使用。
值可以为 null、基本类型、数组或对象。工具可支持或不支持扩展，也可自行扩展。

| 字段模式 | 类型 | 描述 |
|---|---|---|
| ^x- | 任何 | 允许扩展 OpenAPI 模式；x-oai-/x-oas- 前缀保留 |

### 4.10 安全过滤

部分对象可声明为空或完全删除（即使本质上是 API 文档核心），以支持基于身份认证/授权的
额外访问控制层：

- **路径对象（Paths Object）可以存在但为空**：查看者到达正确位置但无法访问任何文档，
  仍可访问至少一个 Info 对象（其中可能含认证相关信息）。
- **路径项对象（Path Item Object）可以为空**：查看者知道路径存在，但看不到任何操作或
  参数——与从路径对象隐藏路径本身不同（用户知道其存在），允许文档提供者精细控制可见性。

### 附录 A：修订历史

| 版本 | 日期 | 备注 |
|---|---|---|
| 3.1.0 | 2021-02-15 | OpenAPI 规范 3.1.0 版本 |
| 3.1.0-rc1 | 2020-10-08 | 3.1 规范的 rc1 版本 |
| 3.1.0-rc0 | 2020-06-18 | 3.1 规范的 rc0 版本 |
| 3.0.3 | 2020-02-20 | OpenAPI 规范 3.0.3 修订版 |
| 3.0.2 | 2018-10-08 | OpenAPI 规范 3.0.2 修订版 |
| 3.0.1 | 2017-12-06 | OpenAPI 规范 3.0.1 修订版 |
| 3.0.0 | 2017-07-26 | OpenAPI 规范 3.0.0 版本 |
| 3.0.0-rc2 | 2017-06-16 | 3.0 规范的 rc2 版本 |
| 3.0.0-rc1 | 2017-04-27 | 3.0 规范的 rc1 版本 |
| 3.0.0-rc0 | 2017-02-28 | 3.0 规范的实现者草稿 |
| 2.0 | 2015-12-31 | 将 Swagger 2.0 捐赠给 OpenAPI Initiative |
| 2.0 | 2014-09-08 | Swagger 2.0 版本 |
| 1.2 | 2014-03-14 | 正式文档的初始版本 |
| 1.1 | 2012-08-22 | Swagger 1.1 版本 |
| 1.0 | 2011-08-10 | Swagger 规范的第一个版本 |

### 附录 B：规范性引用（B.1）

- **RFC5234** — ABNF（D. Crocker；P. Overell；2008-01，互联网标准）
- **CommonMark** — CommonMark 规范（spec.commonmark.cn）
- **IANA** — HTTP 身份验证方案注册表、HTTP 状态代码注册表
- **JSON Schema 2020-12** — 描述 JSON 文档的媒体类型（草稿 2020-12，A. Wright 等，IETF 互联网草稿）
- **RFC1866** — HTML 2.0（历史）
- **RFC2045** — MIME 第一部分：消息体格式
- **RFC2119 / RFC8174** — 需求级别关键词（MUST/SHOULD 语义）
- **RFC3986** — URI 通用语法（互联网标准）
- **RFC4648** — Base16/Base32/Base64 编码
- **RFC6570** — URI 模板
- **RFC6749** — OAuth 2.0 授权框架
- **RFC6838** — 媒体类型规范与注册程序
- **RFC6901** — JSON Pointer
- **RFC7159** — JSON 数据交换格式
- **RFC7230 / RFC7231 / RFC7235** — HTTP/1.1 消息语法与路由 / 语义与内容 / 认证
- **RFC7578** — multipart/form-data 返回值
- **SPDX** — SPDX 许可证列表（Linux 基金会）
- **YAML 1.2** — YAML 不是标记语言（Oren Ben-Kiki 等，2009-10-01）

## Quotes

- "虽然 OpenAPI 规范试图适应大多数用例，但可以在某些点添加其他数据来扩展规范。"
- "路径对象可以存在但为空。这可能与直觉相反，但这可能会告诉查看者他们到达了正确的位置，但无法访问任何文档。"

## Terms

- **规范扩展（Specification Extensions）**：x- 前缀的模式化字段，扩展 OpenAPI 规范。
- **安全过滤（Security Filtering）**：通过空置路径/路径项对象实现的分层文档可见性控制。

## Limitations

- 附录 B 引用列表按原文顺序收录；个别内联子引用（如 CommonMark-0.27）已并入主条目说明。

## Related Pages

- [OpenAPI 规范 v3.1.0 — 简介、定义、版本与格式（01-overview.md）](01-overview.md)
- [OpenAPI 规范 v3.1.0 — 对象定义（一）（02-objects-part1.md）](02-objects-part1.md)
- [OpenAPI 规范 v3.1.0 — 对象定义（二）（03-objects-part2.md）](03-objects-part2.md)
- [OpenAPI 规范 v3.1.0 — 索引（index.md）](index.md)
