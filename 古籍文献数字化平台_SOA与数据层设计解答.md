# 面向服务架构（SOA）系统设计——古籍文献数字化平台 课程设计解答

## 一、题目背景与设计约束

某高校数字人文研究中心拟搭建地方古籍文献数字化平台，覆盖古籍扫描件元数据著录、文本 OCR 识别、人物地名实体标注、文献检索、专题汇编与成果对外共享四类业务角色：

- **档案管理员**：上传影像、著录元数据、审核标注结果；
- **研究人员**：OCR 文本校对、实体标注、检索与专题数据集导出；
- **匿名访客**：仅可浏览公开古籍与简单检索，不可下载涉密或未开放文献；
- **上层系统对接方**：学校图书馆门户、学术成果系统通过接口调用本平台数据。

**业务约束（设计必须遵守）：**

1. 古籍影像（大图）、OCR 文本、结构化元数据分开存储；
2. 古籍分为开放（PUBLIC）、校内访问（CAMPUS）、涉密管控（CLASSIFIED）三类，权限隔离；
3. OCR 识别、实体标注为可被其他服务复用的独立能力；
4. 关键操作支持留痕：谁、何时、修改了哪条古籍记录；
5. 各环节不得紧耦合，未来可替换 OCR 引擎、可更换数据库；
6. 支持后续新增地方志、族谱等资源类型。

---

## 二、问题1：面向服务设计

### 2.1 服务边界划分与核心服务清单

按照「基础能力服务—业务服务—编排服务」三层划分：基础能力服务沉淀可复用的原子技术能力，不绑定具体业务流程；业务服务围绕具体业务场景组织数据与规则；编排服务只负责流程组合与状态流转，不重复实现底层能力。各服务通过标准接口或消息契约通信，不共享业务数据库。

**表 2-1 核心服务清单**

| 服务分层 | 服务名称 | 主要职责 |
|---|---|---|
| 基础能力服务 | 用户与权限服务 | 维护账号、角色与三级密级权限校验，发放访问凭证；向上层提供统一鉴权能力，各业务服务不自行判断密级。 |
| 基础能力服务 | 文件存储服务 | 对接对象存储/文件系统，负责影像大图与附件的存取、切片与句柄（URI）管理，对上层屏蔽存储介质细节。 |
| 基础能力服务 | OCR 识别服务 | 封装可插拔 OCR 引擎，输入影像句柄，输出文本块、坐标与置信度；通过适配层隔离具体厂商引擎，便于替换。 |
| 基础能力服务 | 实体标注服务 | 提供人物、地名、机构、书名等实体的抽取、标注落库与校验能力，规则或模型可替换，可供多业务复用。 |
| 基础能力服务 | 元数据服务 | 管理结构化元数据的著录、查询与版本；资源类型采用可扩展模型，支持古籍、地方志、族谱等新增类型。 |
| 基础能力服务 | 检索服务 | 封装全文检索与条件过滤、聚合能力，屏蔽底层搜索引擎实现，支持后续更换检索引擎。 |
| 基础能力服务 | 操作日志服务 | 记录操作人、操作时间、操作对象与变更摘要，统一提供审计留痕查询。 |
| 业务服务 | 古籍资源管理服务 | 负责古籍著录、上下架、密级设置、影像挂接与资源状态管理。 |
| 业务服务 | 标注审核服务 | 承接校对与标注提交，组织「待审—通过—驳回」审核流转及审核结果回写。 |
| 业务服务 | 专题汇编服务 | 维护专题选题、文献归集条件与数据集清单，支撑研究人员专题整理。 |
| 业务服务 | 开放浏览服务 | 面向匿名访客提供公开古籍浏览与简单检索，强制密级过滤并拦截涉密、未开放文献的下载。 |
| 业务服务 | 对外集成服务 | 面向图书馆门户、学术成果系统提供鉴权后的数据查询与开放接口。 |
| 编排服务 | 古籍入库编排服务 | 串接文件存储、OCR、元数据与日志服务，完成从影像上传到可检索资源的全流程编排与状态补偿。 |
| 编排服务 | 校对标注编排服务 | 组合取影像与 OCR 文本、人工校对、实体标注、提交审核等环节，编排单篇文献的加工流程。 |
| 编排服务 | 检索聚合编排服务 | 作为统一检索入口，先按调用方身份做权限过滤，再聚合元数据、文本与影像线索返回。 |
| 编排服务 | 专题导出编排服务 | 按专题归集文献、做权限校验后打包导出数据集（文本、标注与引用清单）。 |

### 2.2 服务接口契约（JSON 风格）

#### 2.2.1 OCR 识别服务（基础能力服务）

**服务职责：** 接收影像文件句柄，调用可插拔 OCR 引擎完成文本识别，返回文本行/文本块及其坐标、置信度；服务不感知古籍业务语义，不直接写入业务库，识别结果交由上游编排服务落库。

**入参（Request）：**

```json
{
  "requestId": "string，幂等请求号，用于重试去重",
  "imageRef": {
    "fileId": "string，影像文件标识",
    "storageUri": "string，对象存储路径/句柄",
    "pageNo": "int，页码"
  },
  "engineConfig": {
    "engineType": "string，可选，如 tesseract / paddle / 第三方，缺省走适配层默认引擎",
    "lang": "string，可选，如 chi_sim / en",
    "options": { "detectLayout": "boolean，是否做版面分析" }
  },
  "callbackUrl": "string，可选；异步识别完成后回调地址，同步调用可为空"
}
```

**出参（同步成功，HTTP 200）：**

```json
{
  "requestId": "string",
  "imageRef": { "fileId": "string", "pageNo": "int" },
  "engineUsed": "string，实际执行的引擎名称",
  "status": "SUCCESS | PARTIAL | FAILED",
  "textBlocks": [
    {
      "blockId": "string",
      "text": "string，识别文本",
      "bbox": { "x": 0, "y": 0, "width": 0, "height": 0 },
      "confidence": 0.00,
      "lineNo": 1
    }
  ],
  "charCount": 0,
  "processedAt": "datetime，ISO 8601"
}
```

**异常返回（Error Contract）：**

```json
{
  "code": "IMAGE_NOT_FOUND | UNSUPPORTED_IMAGE_TYPE | OCR_ENGINE_UNAVAILABLE | REQUEST_TIMEOUT | INTERNAL_ERROR",
  "message": "string，可读的错误说明",
  "requestId": "string",
  "retryable": true
}
```

状态码约定：`400` 表示影像格式或参数不合法；`404` 表示影像不存在；`503` 表示 OCR 引擎不可用（可重试）；`500` 表示内部错误。

#### 2.2.2 古籍入库编排服务（编排服务）

**服务职责：** 串接文件存储、OCR、元数据与操作日志等服务，完成一册古籍从影像上传到可检索资源的入库流程；负责流程状态记录与失败补偿，本身不重复实现 OCR、标注等底层能力。

**入参（Request）：**

```json
{
  "resourceId": "string，可选，缺省由编排服务生成",
  "operatorId": "string，操作人标识",
  "resourceType": "string，GUJI 古籍 / LOCAL_CHRONICLE 地方志 / GENEALOGY 族谱，可扩展",
  "metadata": {
    "title": "string，题名",
    "author": "string，责任者，可选",
    "dynasty": "string，朝代，可选",
    "collection": "string，馆藏/来源，可选",
    "securityLevel": "PUBLIC | CAMPUS | CLASSIFIED"
  },
  "images": [
    { "fileId": "string", "storageUri": "string", "pageNo": 1 }
  ],
  "ocrEnabled": true
}
```

**出参（受理成功，HTTP 202；流程完成后状态推进至 COMPLETED）：**

```json
{
  "resourceId": "string",
  "ingestStatus": "ACCEPTED | METADATA_SAVED | IMAGE_LINKED | OCR_PENDING | OCR_DONE | COMPLETED | FAILED",
  "steps": [
    { "step": "SAVE_METADATA", "status": "SUCCESS", "timestamp": "datetime" },
    { "step": "LINK_IMAGES", "status": "SUCCESS", "timestamp": "datetime" },
    { "step": "TRIGGER_OCR", "status": "SUCCESS", "timestamp": "datetime" }
  ],
  "ocrTaskId": "string，可选，异步 OCR 任务号"
}
```

**异常返回（Error Contract）：**

```json
{
  "code": "PERMISSION_DENIED | METADATA_VALIDATION_FAILED | IMAGE_EMPTY | DUPLICATE_RESOURCE | DOWNSTREAM_UNAVAILABLE | INTERNAL_ERROR",
  "message": "string，可读的错误说明",
  "resourceId": "string",
  "failedStep": "string，失败环节标识",
  "retryable": true
}
```

### 2.3 服务解耦策略与同步/异步调用选型

#### 2.3.1 解耦策略

- **分层解耦**：基础能力原子化并被业务服务复用，编排服务只做流程组合；服务之间通过标准化接口契约或消息交互，不共享数据库、不直接引用对方内部实现。
- **技术适配隔离**：OCR 引擎、检索引擎、数据库均通过适配接口封装在基础能力服务内部，业务侧只依赖抽象契约，满足「未来可替换 OCR 引擎、可更换数据库」的约束。
- **身份权限下沉**：鉴权与密级判断统一收敛到用户与权限服务，业务服务只调用校验结果，避免权限规则分散在各服务中。
- **数据分区存储**：影像进入对象存储、OCR 文本进入文本/检索存储、结构化元数据进入关系库，三类数据相互独立、各自优化。
- **类型可扩展**：资源类型以类型字段加可扩展元数据模型承载，新增地方志、族谱等资源类型时无需改动既有服务契约。

#### 2.3.2 同步调用与异步调用的场景选择

**同步调用**适用于请求方需要立即拿到结果、才能继续后续操作的短链路场景：

- 在线浏览与文献详情查看、简单检索、提交标注后即时返回审核状态、实时权限校验等；这类场景延迟敏感、链路短、下游可用，采用 REST/RPC 同步调用以保证即时响应与一致体验。

**异步调用**适用于耗时较长、需要削峰填谷或允许最终一致的场景：

- 批量 OCR 识别、实体抽取、大图切片压缩、大批量专题数据集导出、入库后异步建立检索索引等；这类任务通过消息队列发布事件（如 `ResourceIngested → OCR 消费 → TextExtracted → 索引消费`），消费方独立扩展、失败可重试，上游不被长耗时任务阻塞。

**选型理由：** OCR、标注等任务计算量大且可能失败，若同步等待会长期占用连接、放大超时风险；事件驱动的异步方式使能力可独立扩缩容、故障可隔离与补偿。而面向访客和研究人员的在线查询链路必须同步返回，以保证交互体验。两者按「是否需要即时结果、任务是否耗时、是否允许最终一致」划分边界。

---

## 三、问题2：数据层概念与逻辑模型设计

### 3.1 E-R 模型（实体—属性—关系清单）

依据业务识别出用户、用户权限、古籍资源、影像文件、OCR 文本、标注记录、操作日志等核心实体；专题汇编作为可选扩展实体，本节先列出题目要求的核心实体。

**表 3-1 核心实体、属性与键清单**

| 实体（表） | 主要属性 | 主键 | 外键 |
|---|---|---|---|
| 用户 sys_user | 用户ID、账号、姓名、邮箱、状态、创建时间 | user_id | — |
| 用户权限 user_permission | 授权ID、用户ID、角色编码、范围类型、限定资源、授权时间、授权人 | permission_id | user_id → sys_user.user_id；resource_id → resource.resource_id |
| 古籍资源 resource | 资源ID、资源类型、题名、责任者、朝代、馆藏、密级、状态、档号、摘要、著录人、创建/更新时间 | resource_id | created_by → sys_user.user_id |
| 影像文件 image_file | 影像ID、资源ID、页码、文件名、存储句柄、类型、大小、宽高、切片状态、创建时间 | image_id | resource_id → resource.resource_id |
| OCR 文本 ocr_text | 文本ID、资源ID、影像ID、页码、块号、文本内容、坐标、置信度、校对状态、校对人、引擎、时间 | text_id | resource_id → resource.resource_id；image_id → image_file.image_id；proofreader_id → sys_user.user_id |
| 标注记录 annotation_record | 标注ID、资源ID、文本ID、实体类型、实体值、起止位置、标注人、标注方式、审核状态、审核人、备注、时间 | annotation_id | resource_id → resource.resource_id；text_id → ocr_text.text_id；annotator_id / reviewer_id → sys_user.user_id |
| 操作日志 operation_log | 日志ID、操作人ID、操作人快照名、动作类型、对象类型、对象ID、资源ID、变更摘要、IP、操作时间 | log_id | operator_id → sys_user.user_id；resource_id → resource.resource_id |

**表 3-2 实体间关系清单**

| 关系（实体 A — 动作 — 实体 B） | 基数 | 说明 |
|---|---|---|
| 用户 — 著录 — 古籍资源 | 1 : N | 一名管理员可著录多部古籍，一部古籍对应一名著录人。 |
| 用户 — 提交 — 标注记录 | 1 : N | 一名研究人员可提交多条标注，一条标注归属一名标注人。 |
| 用户 — 产生 — 操作日志 | 1 : N | 一名用户可产生多条日志；匿名访客或系统对接方操作时操作人可空。 |
| 古籍资源 — 包含 — 影像文件 | 1 : N | 一部古籍由多页影像组成，影像页按页码排序。 |
| 古籍资源 — 包含 — OCR 文本 | 1 : N | 一部古籍对应多条文本块记录。 |
| 古籍资源 — 拥有 — 标注记录 | 1 : N | 一部古籍下可挂多条实体标注。 |
| 影像文件 — 对应 — OCR 文本 | 1 : N | 一页影像可被识别为多个文本块。 |
| OCR 文本 — 被标注 — 标注记录 | 1 : N | 一条文本可被标注多个实体。 |
| 用户 — 授予 — 权限（经用户权限表） | N : M | 一名用户可拥有多个角色，同一角色可授予多名用户。 |
| 古籍资源 — 被记录 — 操作日志 | 1 : N | 一部古籍的每次关键变更均留下一条审计日志。 |

### 3.2 核心数据表设计

以下以 MySQL 8.0 类型为例给出逻辑表结构；约束列标注主键（PK）、外键（FK）、非空（NOT NULL）、唯一（UNIQUE）与默认值，各表索引在表后单列说明。

#### 3.2.1 古籍主表 resource

| 字段名 | 类型 | 约束 | 说明 |
|---|---|---|---|
| resource_id | BIGINT UNSIGNED | PK, AUTO_INCREMENT, NOT NULL | 资源主键 |
| resource_type | VARCHAR(32) | NOT NULL, DEFAULT 'GUJI' | 资源类型：GUJI / LOCAL_CHRONICLE / GENEALOGY，可扩展 |
| title | VARCHAR(255) | NOT NULL | 题名 |
| author | VARCHAR(128) | NULL | 责任者 |
| dynasty | VARCHAR(64) | NULL | 朝代 |
| collection_name | VARCHAR(128) | NULL | 馆藏/来源 |
| security_level | ENUM('PUBLIC','CAMPUS','CLASSIFIED') | NOT NULL, DEFAULT 'PUBLIC' | 密级：开放 / 校内 / 涉密 |
| status | ENUM('DRAFT','INGESTING','PUBLISHED','REVIEWING','ARCHIVED') | NOT NULL, DEFAULT 'DRAFT' | 资源状态 |
| catalog_no | VARCHAR(64) | NULL, UNIQUE | 索书号/档号 |
| description | TEXT | NULL | 内容摘要 |
| created_by | BIGINT UNSIGNED | NOT NULL, FK → sys_user.user_id | 著录人 |
| created_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 创建时间 |
| updated_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP | 最近更新时间 |

**索引：** 主键 `PK(resource_id)`；普通索引 `idx_title(title)`、`idx_security(security_level)`、`idx_type(resource_type)`、`idx_status(status)`；唯一索引 `uk_catalog(catalog_no)`。

#### 3.2.2 影像文件表 image_file

| 字段名 | 类型 | 约束 | 说明 |
|---|---|---|---|
| image_id | BIGINT UNSIGNED | PK, AUTO_INCREMENT, NOT NULL | 影像主键 |
| resource_id | BIGINT UNSIGNED | NOT NULL, FK → resource.resource_id | 所属古籍 |
| page_no | INT | NOT NULL | 页码 |
| file_name | VARCHAR(255) | NOT NULL | 原始文件名 |
| storage_uri | VARCHAR(512) | NOT NULL | 对象存储句柄/路径，不保存二进制 |
| mime_type | VARCHAR(32) | NOT NULL | 影像类型，如 image/jpeg、image/tiff |
| file_size | BIGINT UNSIGNED | NOT NULL | 文件大小（字节） |
| width | INT | NULL | 像素宽 |
| height | INT | NULL | 像素高 |
| slice_status | ENUM('NONE','SLICED','FAILED') | NOT NULL, DEFAULT 'NONE' | 大图切片状态 |
| created_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 上传时间 |

**索引：** 主键 `PK(image_id)`；唯一索引 `uk_resource_page(resource_id, page_no)`；普通索引 `idx_resource(resource_id)`。

#### 3.2.3 OCR 文本表 ocr_text

| 字段名 | 类型 | 约束 | 说明 |
|---|---|---|---|
| text_id | BIGINT UNSIGNED | PK, AUTO_INCREMENT, NOT NULL | 文本主键 |
| resource_id | BIGINT UNSIGNED | NOT NULL, FK → resource.resource_id | 所属古籍 |
| image_id | BIGINT UNSIGNED | NULL, FK → image_file.image_id | 来源影像页 |
| page_no | INT | NOT NULL | 页码 |
| block_no | INT | NOT NULL | 文本块序号 |
| content | TEXT | NOT NULL | 识别文本内容 |
| bbox_x | INT | NULL | 块左上角 X 坐标 |
| bbox_y | INT | NULL | 块左上角 Y 坐标 |
| bbox_w | INT | NULL | 块宽度 |
| bbox_h | INT | NULL | 块高度 |
| confidence | DECIMAL(5,4) | NULL | 识别置信度 0~1 |
| proofread_status | ENUM('UNPROOFREAD','PROOFREAD','CONFIRMED') | NOT NULL, DEFAULT 'UNPROOFREAD' | 校对状态 |
| proofreader_id | BIGINT UNSIGNED | NULL, FK → sys_user.user_id | 校对人 |
| engine_used | VARCHAR(64) | NULL | 识别所用引擎 |
| created_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 生成时间 |
| updated_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP | 最近更新时间 |

**索引：** 主键 `PK(text_id)`；普通索引 `idx_resource(resource_id)`、`idx_image(image_id)`、`idx_proofread(proofread_status)`；全文索引 `idx_content(content)` 支撑文本检索。

#### 3.2.4 标注记录表 annotation_record

| 字段名 | 类型 | 约束 | 说明 |
|---|---|---|---|
| annotation_id | BIGINT UNSIGNED | PK, AUTO_INCREMENT, NOT NULL | 标注主键 |
| resource_id | BIGINT UNSIGNED | NOT NULL, FK → resource.resource_id | 所属古籍 |
| text_id | BIGINT UNSIGNED | NULL, FK → ocr_text.text_id | 被标注文本 |
| entity_type | ENUM('PERSON','PLACE','ORGANIZATION','WORK','OTHER') | NOT NULL | 实体类型：人物/地名/机构/书名/其他 |
| entity_value | VARCHAR(255) | NOT NULL | 实体名称 |
| start_pos | INT | NULL | 在文本中的起始偏移 |
| end_pos | INT | NULL | 在文本中的结束偏移 |
| annotator_id | BIGINT UNSIGNED | NOT NULL, FK → sys_user.user_id | 标注人 |
| annotate_method | ENUM('MANUAL','MODEL','IMPORT') | NOT NULL, DEFAULT 'MANUAL' | 标注方式 |
| review_status | ENUM('PENDING','APPROVED','REJECTED') | NOT NULL, DEFAULT 'PENDING' | 审核状态 |
| reviewer_id | BIGINT UNSIGNED | NULL, FK → sys_user.user_id | 审核人 |
| remark | VARCHAR(500) | NULL | 审核备注 |
| created_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 标注时间 |
| updated_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP | 最近更新时间 |

**索引：** 主键 `PK(annotation_id)`；普通索引 `idx_resource(resource_id)`、`idx_text(text_id)`、`idx_review(review_status)`；组合索引 `idx_entity(entity_type, entity_value)` 支撑按实体类型与名称检索。

#### 3.2.5 用户表 sys_user

| 字段名 | 类型 | 约束 | 说明 |
|---|---|---|---|
| user_id | BIGINT UNSIGNED | PK, AUTO_INCREMENT, NOT NULL | 用户主键 |
| account | VARCHAR(64) | NOT NULL, UNIQUE | 登录账号 |
| user_name | VARCHAR(64) | NOT NULL | 姓名 |
| email | VARCHAR(128) | NULL | 联系邮箱 |
| status | ENUM('ACTIVE','DISABLED') | NOT NULL, DEFAULT 'ACTIVE' | 账号状态 |
| created_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 创建时间 |

**索引：** 主键 `PK(user_id)`；唯一索引 `uk_account(account)`。

#### 3.2.6 用户权限表 user_permission

| 字段名 | 类型 | 约束 | 说明 |
|---|---|---|---|
| permission_id | BIGINT UNSIGNED | PK, AUTO_INCREMENT, NOT NULL | 授权主键 |
| user_id | BIGINT UNSIGNED | NOT NULL, FK → sys_user.user_id | 被授权用户 |
| role_code | ENUM('ADMIN','RESEARCHER','VISITOR','API_CLIENT') | NOT NULL | 角色：管理员/研究员/访客/系统对接方 |
| scope_type | ENUM('GLOBAL','RESOURCE') | NULL | 授权范围：全局 / 指定资源 |
| resource_id | BIGINT UNSIGNED | NULL, FK → resource.resource_id | 限定可访问资源，空表示全局 |
| granted_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 授权时间 |
| granted_by | BIGINT UNSIGNED | NULL | 授权人 |

**索引：** 主键 `PK(permission_id)`；唯一索引 `uk_user_role(user_id, role_code, scope_type, resource_id)`；普通索引 `idx_user(user_id)`、`idx_resource(resource_id)`。

#### 3.2.7 操作日志表 operation_log

| 字段名 | 类型 | 约束 | 说明 |
|---|---|---|---|
| log_id | BIGINT UNSIGNED | PK, AUTO_INCREMENT, NOT NULL | 日志主键 |
| operator_id | BIGINT UNSIGNED | NULL, FK → sys_user.user_id | 操作人，访客/系统可为空 |
| operator_name | VARCHAR(64) | NULL | 操作人姓名快照，防止用户删除后留痕丢失 |
| action_type | VARCHAR(32) | NOT NULL | 动作：UPLOAD / METADATA_UPDATE / PROOFREAD / ANNOTATE / REVIEW / EXPORT / DOWNLOAD / LOGIN |
| target_type | VARCHAR(32) | NOT NULL | 对象类型：RESOURCE / IMAGE / OCR_TEXT / ANNOTATION |
| target_id | BIGINT UNSIGNED | NULL | 被操作记录主键 |
| resource_id | BIGINT UNSIGNED | NULL, FK → resource.resource_id | 关联古籍 |
| change_summary | VARCHAR(1000) | NULL | 变更摘要/前后值 |
| ip_address | VARCHAR(45) | NULL | 操作来源 IP |
| operation_time | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 操作时间 |

**索引：** 主键 `PK(log_id)`；普通索引 `idx_operator(operator_id)`、`idx_resource(resource_id)`、`idx_target(target_type, target_id)`、`idx_time(operation_time)`、`idx_action(action_type)`。

#### 3.2.8 范式与冗余说明

各表属性尽量满足第三范式：可由其他字段推导的信息不重复存储，关联关系通过外键表达，避免题名、责任者等数据在影像、文本、标注表中大量冗余。唯一有意保留的反范式是操作日志表中的 `operator_name` 姓名快照：审计日志必须在账号被删除后仍能追溯「谁操作」，因此在写入时固化一份姓名副本。

### 3.3 古籍影像文件不直接存入数据库表的理由

- **容量与性能：** 古籍大图（TIFF/JPEG，单张可达数十 MB）若以 BLOB 字段入库，会迅速撑大表空间与索引，导致查询、事务、备份与恢复变慢；数据库单行长度有限，大字段读写还会占用缓冲池与连接资源。
- **存储介质分工不同：** 数据库擅长结构化数据的事务、索引与关联查询；影像属于大对象，更适合对象存储或文件系统，支持高吞吐读取、按 URI 访问、大图切片与 CDN 分发。二者分离后可分别按各自特征优化。
- **备份与水平伸缩：** 影像可在对象存储侧独立扩容、版本化与冗余备份，不必随业务库一起全量备份；结构化数据与影像解耦后，更换存储或数据库都不影响对方的表结构，符合可替换、可扩展的约束。
- **并发下载与在线浏览：** 图片浏览是高频读场景，直连数据库取二进制会占用数据库连接与内存；存入对象存储后可独立水平扩展，并通过临时签名 URL 或鉴权代理控制访问。
- **安全与权限隔离：** 涉密、未开放影像不进入业务库字段，访问时由开放浏览/对外集成服务先做密级校验，再发放有时效的影像访问地址，便于拦截越权下载。
- **分层存储规范：** 与「影像文件、OCR 文本、结构化元数据分开存储」的业务约束一致，数据库表中只保存影像的句柄（`storage_uri`）、元数据与索引信息，二进制文件本身交由存储层管理。

---

## 四、设计小结

本方案在服务层按基础能力、业务、编排三层划分边界，OCR 识别与实体标注沉淀为可复用的独立服务，并通过适配层隔离可替换的引擎与数据库；在数据层以古籍资源为核心，组织影像、文本、标注、权限与日志七张表，外键关系清晰、索引贴合检索与审计需求；影像二进制与结构化数据分离存储，兼顾性能、扩展与安全留痕。整体结构支持后续新增地方志、族谱等资源类型，也便于按服务独立演进与替换技术实现。
