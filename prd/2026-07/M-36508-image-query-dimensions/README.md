# M-36508【AI生图】[运营]图片查询接口扩展支持站点+ASIN及纯ASIN查询（责任人：徐强）

禅道需求：[M-36508](https://pm.zhcxkj.com/zentao/story-view-36508.html)

GitHub PRD：[ai-image-prd-hub/prd/2026-07/M-36508-image-query-dimensions/README.md](https://github.com/xuqiang97/ai-image-prd-hub/blob/main/prd/2026-07/M-36508-image-query-dimensions/README.md)

## 修改纪要

| 日期 | 版本 | 修改人 | 主要修改点 |
|---|---|---|---|
| 2026-07-15 | V1.0 | 徐强 | 新建 PRD，明确运营版非 A+ 商品图片查询接口新增站点+ASIN及纯 ASIN 查询能力、查询范围、兼容规则和验收标准。 |

## 调研纪要

| 日期 | 业务部门 | 角色 | 调研对象（干系人） | 调研内容和结论 |
|---|---|---|---|---|
| 2026-07-15 | 运营 | Listing 管理使用人员 | 相关运营业务人员 | 原店铺被封后，业务会通过跟卖同一 ASIN 的方式将商品转移至新店铺；新店铺需要复用老店铺已审核通过的 AI 商品图。业务确认站点+ASIN及纯 ASIN 查询可覆盖公司全部运营生图数据，不受当前运营人员组织和店铺权限限制。 |

## 业务背景和价值

运营版非 A+ 商品图审核通过后不主动推送至 Listing，而是沉淀在 AI 生图系统中，由 Listing 管理调用图片查询接口按需获取。现有接口只能按“店铺 ID 集合 + ASIN”查询，当原店铺被封并通过跟卖 ASIN 转移至新店铺后，新店铺无法查询到老店铺此前生成并审核通过的 AI 商品图。

本需求扩展站点+ASIN和纯 ASIN两种查询维度，使下游能够跨店铺、跨站点复用已有可用图片，减少重复生图和重复审核，同时保留原店铺+ASIN查询方式及现有非 A+ 下游自取链路。

## 需求概述

扩展现有接口：

`POST /open/aiImageOperation/image/query`

接口根据请求参数自动识别以下查询模式：

1. 店铺+ASIN：传入非空 `orderSourceIds` 和 `asin`，保持现有逻辑。
2. 站点+ASIN：不传有效 `orderSourceIds`，传入 `site` 和 `asin`。
3. 纯 ASIN：仅传入 `asin`，跨全部 Amazon 站点查询。

本期只调整运营版非 A+ 商品图片查询接口，不新增页面，不调整生图、审核、A+ 推送及非 A+ 下游自取流程。

## 功能需求

### 1. 接口入参与查询模式识别

| 参数 | 类型 | 必填规则 | 处理口径 |
|---|---|---|---|
| `asin` | `String` | 必填 | 去除前后空格后不能为空，继续按 ASIN 精确查询。 |
| `orderSourceIds` | `List<Integer>` | 条件必填 | 非空时按店铺+ASIN查询；空数组按未传处理。该字段为 Listing 系统使用的店铺唯一 ID。 |
| `site` | `String` | 条件必填 | `orderSourceIds` 未传或为空且 `site` 非空时，按站点+ASIN查询；站点值使用任务表已保存的 `site`，如 `US`、`DE`。 |
| `isWhiteBackGround` | `Boolean` | 否 | 现有白底首图/主图预留参数，本期不调整其含义和处理逻辑。 |

查询模式按以下优先级自动识别：

1. `orderSourceIds` 非空：按店铺+ASIN查询；即使同时传入 `site`，也以 `orderSourceIds` 为准，`site` 不参与本次查询。
2. `orderSourceIds` 未传或为空，且 `site` 非空：按站点+ASIN查询。
3. `orderSourceIds` 未传或为空，且 `site` 未传或为空：按纯 ASIN查询。
4. `asin` 缺失或去除前后空格后为空：返回参数错误，不查询业务数据。

### 2. 三种查询范围

| 查询模式 | 查询范围 |
|---|---|
| 店铺+ASIN | 查询 `orderSourceIds` 指定店铺下，与 `asin` 匹配的全部历史可用图片，保持现有规则。 |
| 站点+ASIN | 查询公司全部运营生图数据中，任务 `site` 与入参一致且 ASIN 匹配的全部历史可用图片，可跨该站点下不同店铺。 |
| 纯 ASIN | 查询公司全部运营生图数据中，与 ASIN 匹配的全部历史可用图片，可跨全部 Amazon 站点和店铺。 |

站点+ASIN及纯 ASIN查询不受当前运营人员所属组织、部门、组或当前可管理店铺范围限制，但仍需通过接口现有鉴权。被封禁或停用老店铺的历史生图数据只要仍然存在且满足图片可用条件，应正常返回，不得按店铺当前状态过滤。

### 3. 可返回图片范围

三种查询模式统一遵循以下规则：

- 仅查询运营版 AI 生图数据。
- 仅返回业务审核通过的可用图片。
- 同一查询范围内存在多个历史任务时，返回全部历史任务中的可用图片，不只取最新任务。
- 全量返回，不新增分页、时间范围或最大张数限制。
- 返回顺序沿用现有接口逻辑，本期不调整排序规则。
- 仅返回以下 `imageType`：

| `imageType` | 图片类型 |
|---:|---|
| 1 | 场景图 |
| 2 | 特写图白底 |
| 3 | 特写图非白底 |
| 4 | 卖点图 |

`imageType=5` 的标准普通 A+ 图继续固定排除，本需求不涉及任何 A+ 查询或推送逻辑。

`imageType=2`“特写图白底”属于特写图，不等同于公司所称的白底首图/主图；不得使用 `isWhiteBackGround` 将 `imageType=2` 排除。`isWhiteBackGround` 继续作为白底首图/主图预留参数，当前无对应图片数据，本期不扩展该能力。

### 4. 返回结构扩展

现有成功结构和图片字段保持兼容，在每张图片明细中新增来源信息：

| 字段 | 类型 | 说明 |
|---|---|---|
| `orderSourceId` | `Integer` | 该图片对应的来源店铺 ID。 |
| `site` | `String` | 该图片对应任务保存的来源站点。 |

新增字段在三种查询模式下均返回，便于 Listing 识别图片来源并灵活展示或使用。现有图片字段 `id`、`imageType`、`imageUrl`、`width`、`height`、`size`、`format`、`sellPoint`、`isWhiteBackground` 保持不变。

顶层 `data.orderSourceIds` 处理规则：

- 店铺+ASIN模式：继续回显本次请求的 `orderSourceIds`。
- 站点+ASIN及纯 ASIN模式：返回本次查询结果实际命中的来源店铺 ID 去重集合；无可用图片时返回空数组。

`data.asin` 继续回显查询 ASIN。`data.hasAvailableImages` 根据 `images` 是否非空判断：有图片为 `true`，无图片为 `false`。

### 5. 无数据、异常和兼容规则

- 查询成功但没有符合条件的图片时，返回 `success=true`、`images=[]`、`hasAvailableImages=false`，不作为接口异常。
- 老调用方继续使用 `orderSourceIds + asin` 请求时，查询范围和现有业务结果保持不变；新增返回字段不得影响原字段解析。
- `orderSourceIds` 中店铺 ID 的合法性、接口鉴权失败及其他通用异常继续沿用现有接口处理。
- 本期不新增数据权限配置，不修改 AI 生图任务表中的历史数据，不新增图片去重规则。

## 风险描述

| 风险 | 处理方式 |
|---|---|
| 纯 ASIN查询跨站点，可能返回不同站点来源图片 | 每张图片返回来源 `orderSourceId` 和 `site`，由下游结合业务场景选择使用。 |
| 查询范围扩大后结果数量增加 | 业务已确认继续返回全部历史可用图片，本期不增加分页和数量限制。 |
| 跨店铺查询与原组织权限规则冲突 | 业务已确认站点+ASIN及纯 ASIN查询覆盖公司全部运营生图数据，仅保留接口现有鉴权。 |
| “特写图白底”被误判为白底首图/主图 | 明确 `imageType=2` 与 `isWhiteBackGround` 为不同业务概念，本期保持现有图片类型范围。 |

## 验收标准

| # | 验收场景 | 预期结果 |
|---:|---|---|
| 1 | 传入非空 `orderSourceIds + asin` | 按指定店铺+ASIN查询，结果范围与原接口一致，并在每张图片中返回准确的 `orderSourceId` 和 `site`。 |
| 2 | 不传有效 `orderSourceIds`，传入 `site + asin` | 返回公司全部运营生图数据中该站点、该 ASIN的全部历史可用图片，可命中不同店铺及被封禁或停用老店铺。 |
| 3 | 仅传入 `asin` | 跨全部 Amazon 站点和店铺返回该 ASIN的全部历史可用图片。 |
| 4 | 同时传入非空 `orderSourceIds`、`site` 和 `asin` | 以 `orderSourceIds` 为准按店铺+ASIN查询，`site` 不参与查询。 |
| 5 | 同一查询范围存在多个历史任务 | 返回所有历史任务中业务审核通过的可用图片，不只返回最新任务。 |
| 6 | 查询结果同时存在 `imageType=1、2、3、4、5` | 只返回 1～4；标准普通 A+ 图 `imageType=5` 不返回。`imageType=2` 不因 `isWhiteBackGround` 预留逻辑被错误排除。 |
| 7 | 当前登录用户与来源店铺不属于同一组织、部门或店铺权限范围 | 站点+ASIN及纯 ASIN模式仍可返回符合条件的图片；无有效鉴权时仍按现有规则拒绝访问。 |
| 8 | 查询无符合条件的图片 | 返回 `success=true`、`images=[]`、`hasAvailableImages=false`；站点+ASIN及纯 ASIN模式下 `orderSourceIds=[]`。 |
| 9 | 查询有图片 | `hasAvailableImages=true`；每张图片的来源店铺和来源站点与任务数据一致，顶层 `orderSourceIds` 符合对应模式规则。 |
| 10 | `asin` 缺失或仅包含空格 | 返回参数错误，不返回业务数据。 |
| 11 | 原 Listing 调用方继续按原参数调用 | 原有请求可正常使用，查询排序、全量返回、图片字段及 `isWhiteBackGround` 现有行为不变。 |

## 关联需求与参考资料

- 现有接口：`POST /open/aiImageOperation/image/query`
- 内网接口文档：[运营版生图记录查询接口](http://showdoc.zhcxkj.com/web/#/124/6536)
- 影响模块：运营版 AI 生图开放接口、Listing 管理图片查询链路
