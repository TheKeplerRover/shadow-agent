# Shadow Audio v0 DATA_SCHEMA

## TODO

下次继续这个项目时，先提醒用户：

`我们要不要先一起过一遍 DATA_SCHEMA，再继续往下做？`

原因：

- 这份 schema 已经够开始开发
- 但它仍然是 v0 版本
- 一旦代码先跑太远，后面再改表结构会很脏

## 1. 这份文档是干嘛的

### 人话

这份文档回答的是：

- 系统里到底要存哪些东西
- 每种东西之间是什么关系
- 每种东西会经历哪些状态

### 例子

像开一家很小的仓库，你得先决定：

- 有哪些货架
- 每种货放哪
- 每个包裹怎么编号
- 怎么知道一个包裹现在是“刚收到”还是“已经处理完”

### 技术

这份 schema 分两部分：

1. iPhone 本地 `SQLite`
2. 云端 `Postgres`

原因很直接：

- 本地库负责“先稳稳存住，不丢”
- 云端库负责“处理、检索、回看”

## 2. 设计原则

### 2.1 先支持 v0，不支持未来幻想

只为以下流程建模：

- `push-to-start` 录音
- 本地切 chunk
- 本地排队上传
- 云端 STT
- transcript 整理
- memory 提炼
- 基础检索

不为这些建模：

- 多用户
- 团队协作
- 权限系统
- embedding 平台化
- 知识图谱

### 2.2 原始音频和结构化数据分开

### 人话

大文件不要塞进数据库。

### 例子

像图书馆不会把整本书抄进借书卡。  
借书卡只记编号和位置。

### 技术

- raw audio 存对象存储
- 数据库只存 metadata、状态、文本结果、memory 结果

### 2.3 每条数据都要能追溯来源

### 人话

以后你看到一条 memory，要知道它是从哪段录音里来的。

### 例子

像老师批注一句“这里是重点”，你得知道这句重点是课本第几页来的。

### 技术

每条 transcript 和 memory 都至少要能追到：

- `session_id`
- `chunk_id`
- 原始时间范围或证据文本

### 2.4 状态必须显式

### 人话

不要靠猜来判断一段录音现在处理到哪了。

### 技术

所有关键对象都用明确状态字段，不用隐式推断。

## 3. 业务对象总览

你这次提的 8 点会把 schema 从“录音数据库”推向“人中心数据库”。  
这件事是对的，但要做法干净。

### 人话

以后系统里最值钱的，不是 raw 音频，而是这些结构化对象：

- 这段话是在哪说的
- 当时有哪些人
- 哪些人真的重要
- 这件事为什么重要

### v0 核心对象

1. `recording_session`
2. `audio_chunk`
3. `location_snapshot`
4. `person_entity`
5. `session_speaker`
6. `voiceprint_profile`
7. `voiceprint_enrollment_sample`
8. `review_item`
9. `review_item_audio_example`
10. `stt_job`
11. `analysis_run`
12. `transcript_segment`
13. `session_summary`
14. `memory_item`
15. `memory_score`

### 关系摘要

- 一个 `recording_session` 有多个 `audio_chunk`
- 一个 `recording_session` 可有多个 `location_snapshot`
- 一个 `recording_session` 可有多个 `session_speaker`
- 一个 `session_speaker` 可映射到零个或一个 `person_entity`
- 一个 `person_entity` 可有零个或一个 active `voiceprint_profile`
- 一个 `voiceprint_profile` 可有多个 `voiceprint_enrollment_sample`
- 一个 `review_item` 可有最多 3 个 `review_item_audio_example`
- 一个 `audio_chunk` 对应零个或一个 `stt_job`
- 一个 `audio_chunk` 产出多个 `transcript_segment`
- 一个 `analysis_run` 可产出 `session_summary`、`session_speaker`、`memory_item`、`memory_score`
- 一个 `memory_item` 对应一个 `memory_score`

## 4. 分层原则

### 人话

数据库干净的关键，不是字段少，而是层次清楚。

### 例子

像厨房：

- 生肉放生肉区
- 切好的菜放备料区
- 成品放出餐区

不能把这些混在同一个盆里。

### 技术

v0 分三层：

1. `capture layer`
本地采集和上传状态  
表：`recording_sessions`、`audio_chunks`、`location_snapshots`

2. `canonical layer`
真正给产品用的干净主数据  
表：`sessions`、`chunks`、`person_entities`、`session_speakers`、`voiceprint_profiles`、`voiceprint_enrollment_samples`、`review_items`、`review_item_audio_examples`、`transcript_segments`、`session_summaries`、`memory_items`、`memory_scores`

3. `pipeline layer`
记录 AI / STT 跑过什么、用的什么模型、结果来自哪次分析  
表：`stt_jobs`、`analysis_runs`

结论：

- raw 层可以粗一点
- canonical 层必须窄、稳、可追溯
- AI 的长篇废话不要直接进 canonical 表

## 5. iPhone 本地 SQLite Schema

## 5.1 本地为什么新增地点表

### 人话

“地点”不是一个装饰字段，它是语境。

### 例子

同一句“我得处理这个事”，如果是在家里、公司、机场、医院，说话意义完全不同。

### 技术

v0 本地核心表改成三张：

- `recording_sessions`
- `audio_chunks`
- `location_snapshots`

本地依然不做 transcript / memory。

## 5.2 `recording_sessions`

### 字段

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `session_id` | TEXT PK | session 唯一 ID |
| `status` | TEXT | 当前状态 |
| `started_at` | INTEGER | 开始时间，Unix ms |
| `ended_at` | INTEGER NULL | 结束时间，Unix ms |
| `chunk_count` | INTEGER | 当前 session 下的 chunk 数 |
| `speech_chunk_count` | INTEGER | 有语音的 chunk 数 |
| `upload_progress` | REAL | 上传进度，0 到 1 |
| `primary_location_snapshot_id` | TEXT NULL | 主地点快照 |
| `created_at` | INTEGER | 本地创建时间 |
| `updated_at` | INTEGER | 最近更新时间 |

### 状态枚举

- `active`
- `stopping`
- `ended`
- `processed`

### 索引

- `idx_recording_sessions_status(status)`
- `idx_recording_sessions_started_at(started_at DESC)`

## 5.3 `audio_chunks`

### 字段

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `chunk_id` | TEXT PK | chunk 唯一 ID |
| `session_id` | TEXT FK | 属于哪个 session |
| `seq_no` | INTEGER | 在 session 内的顺序编号 |
| `file_path` | TEXT | 本地文件路径 |
| `started_at` | INTEGER | chunk 开始时间，Unix ms |
| `ended_at` | INTEGER | chunk 结束时间，Unix ms |
| `duration_ms` | INTEGER | chunk 时长 |
| `has_speech` | INTEGER | 是否检测到语音，0/1 |
| `vad_score` | REAL NULL | 粗粒度语音分数 |
| `size_bytes` | INTEGER | 文件大小 |
| `sha256` | TEXT | 文件内容 hash |
| `upload_status` | TEXT | 本地上传状态 |
| `upload_attempts` | INTEGER | 上传尝试次数 |
| `last_error` | TEXT NULL | 最近一次错误 |
| `remote_object_key` | TEXT NULL | 云端对象存储 key |
| `uploaded_at` | INTEGER NULL | 上传成功时间 |
| `created_at` | INTEGER | 本地创建时间 |
| `updated_at` | INTEGER | 最近更新时间 |

### 上传状态枚举

- `recording`
- `sealed`
- `pending`
- `uploading`
- `uploaded`
- `failed`

### 索引

- `idx_audio_chunks_session_id(session_id, seq_no)`
- `idx_audio_chunks_upload_status(upload_status, created_at)`
- `idx_audio_chunks_sha256(sha256)`

## 5.4 `location_snapshots`

### 人话

不要把地点粗暴地塞到 session 一列字符串里。  
地点需要同时保留“机器坐标”和“人类可读名字”。

### 例子

像寄快递时既要有 GPS 定位，也要有“温哥华机场”这种你看得懂的名字。

### 字段

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `snapshot_id` | TEXT PK | 地点快照 ID |
| `session_id` | TEXT FK | 所属 session |
| `captured_at` | INTEGER | 采样时间，Unix ms |
| `capture_reason` | TEXT | 为什么采样 |
| `latitude` | REAL | 纬度 |
| `longitude` | REAL | 经度 |
| `horizontal_accuracy_m` | REAL NULL | 水平精度 |
| `place_label_short` | TEXT NULL | 短地点名，例如 `Home`、`YVR` |
| `locality` | TEXT NULL | 城市 / 区域 |
| `admin_area` | TEXT NULL | 省州 |
| `country_code` | TEXT NULL | 国家码 |
| `timezone_id` | TEXT NULL | 时区 |
| `created_at` | INTEGER | 创建时间 |

### `capture_reason`

- `session_start`
- `significant_change`
- `session_end`

### 设计判断

v0 不按每个 chunk 强绑地点。  
更干净的做法是：

- 先记录 session 期间的地点快照
- 再把 `primary_location_snapshot_id` 指到最代表这次 session 的地点

### 索引

- `idx_location_snapshots_session(session_id, captured_at)`

## 5.5 本地不建什么表

v0 本地不建：

- transcript 表
- memory 表
- speaker 表
- 搜索索引表

理由：

- 第一版智能处理主要在云端
- 本地先专注采集、地点、上传

## 6. 云端 Postgres Schema

## 6.1 `sessions`

### 字段

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `session_id` | TEXT PK | session 唯一 ID |
| `status` | TEXT | 云端处理状态 |
| `started_at` | TIMESTAMPTZ | session 开始时间 |
| `ended_at` | TIMESTAMPTZ NULL | session 结束时间 |
| `chunk_count` | INTEGER | chunk 数 |
| `primary_location_snapshot_id` | TEXT NULL | 主地点快照 |
| `speaker_count_estimate` | INTEGER NULL | 说话人数估计 |
| `transcript_ready` | BOOLEAN | transcript 是否已生成 |
| `memory_ready` | BOOLEAN | memory 是否已生成 |
| `created_at` | TIMESTAMPTZ | 创建时间 |
| `updated_at` | TIMESTAMPTZ | 更新时间 |

### 状态枚举

- `ingesting`
- `transcribing`
- `analyzing`
- `ready`
- `failed`

### 索引

- `idx_sessions_started_at(started_at DESC)`
- `idx_sessions_status(status)`

## 6.2 `chunks`

### 字段

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `chunk_id` | TEXT PK | chunk 唯一 ID |
| `session_id` | TEXT FK | 所属 session |
| `seq_no` | INTEGER | 顺序号 |
| `object_key` | TEXT | 对象存储 key |
| `sha256` | TEXT | 文件 hash |
| `mime_type` | TEXT | 音频 MIME 类型 |
| `duration_ms` | INTEGER | 时长 |
| `size_bytes` | BIGINT | 文件大小 |
| `has_speech` | BOOLEAN | 是否有语音 |
| `ingest_status` | TEXT | 云端接收状态 |
| `stt_status` | TEXT | 转写状态 |
| `uploaded_at` | TIMESTAMPTZ | 上传成功时间 |
| `created_at` | TIMESTAMPTZ | 创建时间 |
| `updated_at` | TIMESTAMPTZ | 更新时间 |

### `ingest_status`

- `received`
- `stored`
- `verified`
- `failed`

### `stt_status`

- `pending`
- `queued`
- `processing`
- `done`
- `failed`

### 索引

- `idx_chunks_session_id(session_id, seq_no)`
- `idx_chunks_stt_status(stt_status, created_at)`
- `idx_chunks_sha256(sha256)`
- `unique_chunks_session_seq(session_id, seq_no)`

## 6.3 `location_snapshots`

### 人话

云端保留地点快照，是为了以后从“人在哪里”这个维度检索。

### 字段

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `snapshot_id` | TEXT PK | 地点快照 ID |
| `session_id` | TEXT FK | 所属 session |
| `captured_at` | TIMESTAMPTZ | 采样时间 |
| `capture_reason` | TEXT | 采样原因 |
| `latitude` | DOUBLE PRECISION | 纬度 |
| `longitude` | DOUBLE PRECISION | 经度 |
| `horizontal_accuracy_m` | REAL NULL | 精度 |
| `place_label_short` | VARCHAR(120) NULL | 短地点名 |
| `locality` | VARCHAR(120) NULL | 城市 / 区域 |
| `admin_area` | VARCHAR(120) NULL | 省州 |
| `country_code` | VARCHAR(8) NULL | 国家码 |
| `timezone_id` | VARCHAR(64) NULL | 时区 |
| `created_at` | TIMESTAMPTZ | 创建时间 |

### 索引

- `idx_cloud_location_snapshots_session(session_id, captured_at)`
- `idx_cloud_location_snapshots_place(place_label_short, locality)`

## 6.4 `person_entities`

### 人话

“说话人”不能只存成一段字符串。  
要把“这是谁”做成一个可复用的人物实体。

### 例子

如果你今天和妈妈说话，明天又和妈妈说话，这两个 session 应该都能连到同一个 `person_entity`。

### 字段

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `person_id` | TEXT PK | 人物 ID |
| `canonical_name` | VARCHAR(120) | 规范名字 |
| `display_name` | VARCHAR(120) NULL | UI 显示名 |
| `person_kind` | TEXT | 人物类型 |
| `created_at` | TIMESTAMPTZ | 创建时间 |
| `updated_at` | TIMESTAMPTZ | 更新时间 |

### `person_kind`

- `self`
- `known`
- `unknown`

### 设计判断

大模型可以帮你猜“这是谁”，但不能直接把猜测当真相。  
所以 v0 要分开：

- `session_speaker`：这次 session 里出现的说话槽位
- `person_entity`：跨 session 的稳定人物实体

## 6.5 `session_speakers`

### 人话

它记录的是“这次 session 里有哪些说话人”，不是“全世界有哪些联系人”。

### 字段

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `session_speaker_id` | TEXT PK | 本次 session 的说话人 ID |
| `session_id` | TEXT FK | 所属 session |
| `speaker_label` | VARCHAR(32) | `SPK_0`、`SPK_1` 之类的槽位 |
| `person_id` | TEXT FK NULL | 如果已识别到是谁，连到人物实体 |
| `identity_confidence` | REAL NULL | 身份识别置信度 |
| `is_primary_user` | BOOLEAN | 是否为用户本人 |
| `segment_count` | INTEGER | 该说话人片段数 |
| `speech_duration_ms` | INTEGER | 该说话人总发言时长 |
| `source_run_id` | TEXT NULL | 来自哪次分析 |
| `created_at` | TIMESTAMPTZ | 创建时间 |
| `updated_at` | TIMESTAMPTZ | 更新时间 |

### 索引

- `idx_session_speakers_session(session_id, speaker_label)`
- `idx_session_speakers_person(person_id, created_at DESC)`

## 6.6 `voiceprint_profiles`

### 人话

`person_entity` 解决“这是谁”。  
`voiceprint_profile` 解决“系统以后怎么靠声音认出这个人”。

### 例子

你确认 `SPK_1` 是妈妈后，系统不只是记住“妈妈”这个名字，还要建立一个“妈妈的声音模板”。以后再听到相似声音时，先用模板判断，减少重复问你。

### 字段

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `voiceprint_profile_id` | TEXT PK | 声纹 profile ID |
| `person_id` | TEXT FK UNIQUE | 对应人物 |
| `profile_status` | TEXT | profile 状态 |
| `profile_health` | TEXT | 模板健康度 |
| `embedding_model` | VARCHAR(120) | 生成 embedding 的模型 |
| `embedding_model_version` | VARCHAR(64) | 模型版本 |
| `embedding_ref` | TEXT | 主声纹向量引用，可指向对象存储或向量存储 |
| `embedding_dim` | INTEGER | 向量维度 |
| `trusted_sample_count` | INTEGER | trusted 样本数 |
| `usable_sample_count` | INTEGER | usable 样本数 |
| `rejected_sample_count` | INTEGER | rejected 样本数 |
| `total_trusted_duration_ms` | INTEGER | trusted 样本累计时长 |
| `quality_score` | REAL | 模板质量，0 到 1 |
| `contamination_risk` | REAL | 污染风险，0 到 1 |
| `last_verified_at` | TIMESTAMPTZ NULL | 最近一次用户确认时间 |
| `last_matched_at` | TIMESTAMPTZ NULL | 最近一次自动匹配时间 |
| `frozen_reason` | VARCHAR(280) NULL | 冻结原因 |
| `created_at` | TIMESTAMPTZ | 创建时间 |
| `updated_at` | TIMESTAMPTZ | 更新时间 |

### `profile_status`

- `candidate`
- `active`
- `frozen`
- `retired`

### `profile_health`

- `insufficient`
- `usable`
- `strong`
- `contaminated`
- `frozen`

### 设计判断

主模板只能由 `trusted` enrollment samples 更新。  
`usable` 样本可以辅助解释和排序，但不能改主模板。  
`rejected` 样本只保留引用和拒绝原因，避免以后重复踩坑。

### 索引

- `idx_voiceprint_profiles_person(person_id)`
- `idx_voiceprint_profiles_health(profile_health, quality_score DESC)`

## 6.7 `voiceprint_enrollment_samples`

### 人话

这张表记录“哪些音频片段被拿来训练或辅助某个人的声纹”。  
它是防污染的关键。

### 例子

不是所有听起来像妈妈的片段都能更新妈妈的声音模板。只有清晰、单人、身份可靠的片段才可以进 `trusted`。

### 字段

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `sample_id` | TEXT PK | 样本 ID |
| `voiceprint_profile_id` | TEXT FK | 所属 voiceprint profile |
| `person_id` | TEXT FK | 冗余人物 ID，方便查询 |
| `session_id` | TEXT FK | 来源 session |
| `chunk_id` | TEXT FK | 来源 chunk |
| `session_speaker_id` | TEXT FK NULL | 来源说话人槽位 |
| `source_run_id` | TEXT FK NULL | 来自哪次分析 |
| `sample_status` | TEXT | 样本分级 |
| `identity_source` | TEXT | 身份来源 |
| `start_ms` | INTEGER | 在 chunk 内开始时间 |
| `end_ms` | INTEGER | 在 chunk 内结束时间 |
| `duration_ms` | INTEGER | 样本时长 |
| `embedding_ref` | TEXT NULL | 样本 embedding 引用 |
| `speech_quality_score` | REAL | 人声质量分 |
| `overlap_score` | REAL | 重叠说话风险 |
| `background_media_score` | REAL | 背景媒体污染风险 |
| `noise_score` | REAL | 环境噪音风险 |
| `distance_to_profile` | REAL NULL | 和当前主模板距离 |
| `distance_to_second_best` | REAL NULL | 和第二候选距离 |
| `rejection_reason` | VARCHAR(280) NULL | 拒绝原因 |
| `created_at` | TIMESTAMPTZ | 创建时间 |
| `updated_at` | TIMESTAMPTZ | 更新时间 |

### `sample_status`

- `trusted`
- `usable`
- `rejected`

### `identity_source`

- `user_confirmed`
- `high_confidence_auto`
- `historical_profile_match`

### `trusted` 标准

一段样本要进入 `trusted`，必须同时满足：

- 身份来源可靠：用户确认，或高置信自动识别且无冲突信号
- 主要是单人发言
- 没有明显多人重叠
- 外放视频、音乐、TikTok 声音不是主导
- 单段至少 `3-5s`
- profile 初建最好累计 `20-30s` trusted audio
- 和当前 profile 接近，并且明显领先第二候选

### 索引

- `idx_voiceprint_samples_profile(voiceprint_profile_id, sample_status, created_at DESC)`
- `idx_voiceprint_samples_person(person_id, created_at DESC)`
- `idx_voiceprint_samples_session(session_id, chunk_id)`

## 6.8 `review_items`

### 人话

review queue 只放需要你确认的身份问题。  
v0 不问“这个人重不重要”，也不问“这个偏好要不要升级”。

### v0 问题类型

- `identify_person`
- `merge_person`

### 字段

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `review_item_id` | TEXT PK | review item ID |
| `review_type` | TEXT | 问题类型 |
| `status` | TEXT | 当前状态 |
| `priority` | INTEGER | 优先级 |
| `target_session_speaker_id` | TEXT FK NULL | 要确认的 session speaker |
| `target_person_id` | TEXT FK NULL | 候选人物 |
| `candidate_person_id` | TEXT FK NULL | merge 或识别候选 |
| `candidate_confidence` | REAL NULL | 候选置信度 |
| `question_payload_json` | JSONB | UI 问题载荷 |
| `answer_payload_json` | JSONB NULL | 用户回答 |
| `created_from_run_id` | TEXT FK NULL | 来源分析 |
| `snooze_until` | TIMESTAMPTZ NULL | 稍后提醒时间 |
| `resolved_at` | TIMESTAMPTZ NULL | 解决时间 |
| `created_at` | TIMESTAMPTZ | 创建时间 |
| `updated_at` | TIMESTAMPTZ | 更新时间 |

### `status`

- `pending`
- `snoozed`
- `resolved`
- `dismissed`

### 设计判断

review queue 可以长期积压。  
系统不应该因为用户暂时没空确认就阻断主处理链路。

### 索引

- `idx_review_items_status_priority(status, priority, created_at)`
- `idx_review_items_target_speaker(target_session_speaker_id)`
- `idx_review_items_target_person(target_person_id)`

## 6.9 `review_item_audio_examples`

### 人话

身份确认卡片必须让用户知道自己在确认哪段声音。  
每张卡最多提供 3 个原音频播放条。

### 例子

如果系统问“这个说话人是不是妈妈？”，卡片里应该能播放 1-3 段代表性原声，而不是只显示一个抽象问题。

### 字段

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `example_id` | TEXT PK | 音频例子 ID |
| `review_item_id` | TEXT FK | 所属 review item |
| `seq_no` | INTEGER | 展示顺序，最多 1-3 |
| `session_id` | TEXT FK | 来源 session |
| `chunk_id` | TEXT FK | 来源 chunk |
| `session_speaker_id` | TEXT FK NULL | 来源说话人 |
| `start_ms` | INTEGER | 播放起点 |
| `end_ms` | INTEGER | 播放终点 |
| `audio_object_key` | TEXT | 原音频对象 key |
| `transcript_hint` | VARCHAR(280) NULL | 可选短转写提示 |
| `quality_score` | REAL NULL | 片段质量 |
| `created_at` | TIMESTAMPTZ | 创建时间 |

### 约束

- 同一个 `review_item_id` 最多 3 条 example
- 每条 example 应尽量短，目标 `3-10s`
- 优先选清晰、单人、能代表候选声纹的片段

### 索引

- `idx_review_audio_examples_item(review_item_id, seq_no)`

## 6.10 `stt_jobs`

### 字段

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `job_id` | TEXT PK | job 唯一 ID |
| `chunk_id` | TEXT FK UNIQUE | 对应哪个 chunk |
| `provider` | VARCHAR(64) | STT 提供方 |
| `status` | TEXT | job 状态 |
| `attempts` | INTEGER | 重试次数 |
| `priority` | INTEGER | 优先级，默认 100 |
| `leased_at` | TIMESTAMPTZ NULL | worker 领走时间 |
| `finished_at` | TIMESTAMPTZ NULL | 完成时间 |
| `last_error` | VARCHAR(1000) NULL | 最近错误 |
| `created_at` | TIMESTAMPTZ | 创建时间 |
| `updated_at` | TIMESTAMPTZ | 更新时间 |

### 状态枚举

- `queued`
- `processing`
- `done`
- `failed`

### 索引

- `idx_stt_jobs_status_priority(status, priority, created_at)`

## 6.11 `analysis_runs`

### 人话

AI 派生结果很有用，但不能直接往主表里乱写。  
要知道“这条结论是谁算出来的、用的哪个模型、哪版 prompt”。

### 例子

像化验单。你不能只记结论，还得记是哪次化验、用的什么方法。

### 字段

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `run_id` | TEXT PK | 分析运行 ID |
| `session_id` | TEXT FK | 所属 session |
| `run_type` | TEXT | 分析类型 |
| `model_name` | VARCHAR(120) | 模型名 |
| `prompt_version` | VARCHAR(64) | prompt / pipeline 版本 |
| `status` | TEXT | 运行状态 |
| `input_ref` | TEXT NULL | 输入引用，可指向对象存储 |
| `output_ref` | TEXT NULL | 输出引用，可指向对象存储 |
| `last_error` | VARCHAR(1000) NULL | 最近错误 |
| `created_at` | TIMESTAMPTZ | 创建时间 |
| `updated_at` | TIMESTAMPTZ | 更新时间 |

### `run_type`

- `speaker_resolution`
- `session_summary`
- `memory_extraction`
- `memory_scoring`

### `status`

- `queued`
- `processing`
- `done`
- `failed`

## 6.12 `transcript_segments`

### 人话

转录原文要尽量原子化，但不要太长。  
太长以后既难读，也难标说话人。

### 字段

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `segment_id` | TEXT PK | segment 唯一 ID |
| `session_id` | TEXT FK | 所属 session |
| `chunk_id` | TEXT FK | 来源 chunk |
| `session_speaker_id` | TEXT FK NULL | 对应说话人槽位 |
| `seq_no` | INTEGER | 在 chunk 内顺序 |
| `start_ms` | INTEGER | 在 chunk 内开始时间 |
| `end_ms` | INTEGER | 在 chunk 内结束时间 |
| `text` | TEXT | 转写文本 |
| `confidence` | REAL NULL | 转写置信度 |
| `created_at` | TIMESTAMPTZ | 创建时间 |

### 索引

- `idx_transcript_segments_session(session_id, chunk_id, seq_no)`
- `idx_transcript_segments_speaker(session_speaker_id, created_at)`

## 6.13 `session_summaries`

### 人话

你这个判断是对的：`session_summary` 不该变成一个装各种附属 JSON 的垃圾抽屉。  
它应该是可选的、很短的、便于快速扫一眼的东西。

### 技术判断

- 不是每个 session 都必须有 summary
- 不再默认存 `key_topics_json`
- 不再默认存 `open_loops_json`
- 如果真有值得长期记住的主题、关系、承诺、偏好，应该进 `memory_items`

### 字段

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `session_id` | TEXT PK FK | 对应哪个 session |
| `source_run_id` | TEXT FK NULL | 来自哪次分析 |
| `summary_short` | VARCHAR(280) NULL | 短摘要 |
| `created_at` | TIMESTAMPTZ | 创建时间 |
| `updated_at` | TIMESTAMPTZ | 更新时间 |

## 6.14 `memory_items`

### 人话

这里存的是“值得留下来的结构化认知”，不是任何一次对话里的所有结论。

### 技术判断

为了避免系统思维被 TODO 带偏，v0 的 memory 类型改成更人中心：

- `commitment`
- `decision`
- `preference`
- `relationship`
- `context`
- `fact`

### 字段

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `memory_id` | TEXT PK | memory 唯一 ID |
| `session_id` | TEXT FK | 来源 session |
| `chunk_id` | TEXT FK NULL | 来源 chunk |
| `related_person_id` | TEXT FK NULL | 关联人物 |
| `location_snapshot_id` | TEXT FK NULL | 关联地点 |
| `source_run_id` | TEXT FK NULL | 来自哪次分析 |
| `memory_type` | TEXT | 记忆类型 |
| `title` | VARCHAR(120) NULL | 短标题 |
| `text` | VARCHAR(600) | 记忆正文 |
| `evidence_text` | VARCHAR(280) NULL | 证据摘录 |
| `importance_score` | REAL | 总重要度，0 到 1 |
| `confidence` | REAL | 置信度，0 到 1 |
| `valid_from` | TIMESTAMPTZ NULL | 生效时间 |
| `created_at` | TIMESTAMPTZ | 创建时间 |
| `updated_at` | TIMESTAMPTZ | 更新时间 |

### 索引

- `idx_memory_items_session(session_id, created_at DESC)`
- `idx_memory_items_type(memory_type, created_at DESC)`
- `idx_memory_items_person(related_person_id, created_at DESC)`
- `idx_memory_items_importance(importance_score DESC)`

## 6.15 `memory_scores`

### 人话

“重要性”不能只是一列拍脑袋分数。  
要能拆开看，知道它为什么重要。

### 6 维 MVP 公式

v0 用 6 个维度：

1. `financial_score`
这件事对钱、成本、收益、资源有多大影响

2. `emotional_score`
这件事引发了多强的情绪波动

3. `relational_score`
这件事和重要关系、人际义务、社会意义的关联有多强

4. `identity_score`
这件事和“我是谁、我重视什么、我的价值观和自我叙事”有多强关联

5. `interest_score`
这件事和长期兴趣、爱好、持续关注点有多强关联

6. `commitment_score`
这件事是否涉及承诺、责任、截止时间、必须兑现的后果

### v0 总分公式

`importance_score = 0.15 * financial + 0.20 * emotional + 0.20 * relational + 0.15 * identity + 0.10 * interest + 0.20 * commitment`

### 字段

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `memory_id` | TEXT PK FK | 对应哪条 memory |
| `source_run_id` | TEXT FK NULL | 来自哪次分析 |
| `scoring_version` | VARCHAR(32) | 评分公式版本 |
| `financial_score` | REAL | 财务维度分数 |
| `emotional_score` | REAL | 情感维度分数 |
| `relational_score` | REAL | 社会 / 关系维度分数 |
| `identity_score` | REAL | 自我认同维度分数 |
| `interest_score` | REAL | 兴趣维度分数 |
| `commitment_score` | REAL | 承诺维度分数 |
| `importance_score` | REAL | 总分 |
| `reason_short` | VARCHAR(200) NULL | 短解释 |
| `created_at` | TIMESTAMPTZ | 创建时间 |
| `updated_at` | TIMESTAMPTZ | 更新时间 |

### 设计判断

总分可以冗余存一份在 `memory_items.importance_score`。  
原因不是偷懒，而是：

- 列表排序更快
- 搜索更简单
- 同时仍保留可拆解的细分维度

## 7. 推荐 ID 规则

v0 建议全部用 `ULID` 或者时间有序 UUID。

建议前缀：

- `ses_...`
- `chk_...`
- `loc_...`
- `per_...`
- `ssp_...`
- `vpr_...`
- `ves_...`
- `rev_...`
- `rea_...`
- `job_...`
- `run_...`
- `seg_...`
- `sum_...`
- `mem_...`

## 8. 关键关系

### 关系摘要

- `recording_sessions.session_id -> audio_chunks.session_id`
- `recording_sessions.primary_location_snapshot_id -> location_snapshots.snapshot_id`
- `sessions.session_id -> chunks.session_id`
- `sessions.primary_location_snapshot_id -> location_snapshots.snapshot_id`
- `sessions.session_id -> session_speakers.session_id`
- `person_entities.person_id -> session_speakers.person_id`
- `person_entities.person_id -> voiceprint_profiles.person_id`
- `voiceprint_profiles.voiceprint_profile_id -> voiceprint_enrollment_samples.voiceprint_profile_id`
- `session_speakers.session_speaker_id -> voiceprint_enrollment_samples.session_speaker_id`
- `review_items.review_item_id -> review_item_audio_examples.review_item_id`
- `session_speakers.session_speaker_id -> review_items.target_session_speaker_id`
- `chunks.chunk_id -> stt_jobs.chunk_id`
- `chunks.chunk_id -> transcript_segments.chunk_id`
- `session_speakers.session_speaker_id -> transcript_segments.session_speaker_id`
- `analysis_runs.run_id -> session_summaries.source_run_id`
- `analysis_runs.run_id -> session_speakers.source_run_id`
- `analysis_runs.run_id -> memory_items.source_run_id`
- `analysis_runs.run_id -> memory_scores.source_run_id`
- `sessions.session_id -> memory_items.session_id`
- `memory_items.memory_id -> memory_scores.memory_id`

### 一个关键约束

`stt_jobs.chunk_id` 必须唯一。  
同一段录音不能同时排多个正式 STT 任务。

## 9. 状态流

## 9.1 本地 chunk 状态流

`recording -> sealed -> pending -> uploading -> uploaded`

失败分支：

`uploading -> failed -> pending`

## 9.2 云端 chunk / stt 状态流

`received -> stored -> verified -> queued -> processing -> done`

失败分支：

`processing -> failed -> queued`

## 9.3 session 状态流

`ingesting -> transcribing -> analyzing -> ready`

失败分支：

`* -> failed`

## 10. 哪些字段可以为空

### 人话

“可空”不是随便留口子。  
可空字段应该只出现在“这个信息现在合理地还不知道”的地方。

### 主要可空字段

- `recording_sessions.ended_at`
- `recording_sessions.primary_location_snapshot_id`
- `audio_chunks.vad_score`
- `audio_chunks.last_error`
- `audio_chunks.remote_object_key`
- `audio_chunks.uploaded_at`
- `location_snapshots.horizontal_accuracy_m`
- `location_snapshots.place_label_short`
- `location_snapshots.locality`
- `location_snapshots.admin_area`
- `location_snapshots.country_code`
- `location_snapshots.timezone_id`
- `sessions.ended_at`
- `sessions.primary_location_snapshot_id`
- `sessions.speaker_count_estimate`
- `session_speakers.person_id`
- `session_speakers.identity_confidence`
- `session_speakers.source_run_id`
- `voiceprint_profiles.last_verified_at`
- `voiceprint_profiles.last_matched_at`
- `voiceprint_profiles.frozen_reason`
- `voiceprint_enrollment_samples.session_speaker_id`
- `voiceprint_enrollment_samples.source_run_id`
- `voiceprint_enrollment_samples.embedding_ref`
- `voiceprint_enrollment_samples.distance_to_profile`
- `voiceprint_enrollment_samples.distance_to_second_best`
- `voiceprint_enrollment_samples.rejection_reason`
- `review_items.target_session_speaker_id`
- `review_items.target_person_id`
- `review_items.candidate_person_id`
- `review_items.candidate_confidence`
- `review_items.answer_payload_json`
- `review_items.created_from_run_id`
- `review_items.snooze_until`
- `review_items.resolved_at`
- `review_item_audio_examples.session_speaker_id`
- `review_item_audio_examples.transcript_hint`
- `review_item_audio_examples.quality_score`
- `stt_jobs.leased_at`
- `stt_jobs.finished_at`
- `stt_jobs.last_error`
- `analysis_runs.input_ref`
- `analysis_runs.output_ref`
- `analysis_runs.last_error`
- `transcript_segments.session_speaker_id`
- `transcript_segments.confidence`
- `session_summaries.source_run_id`
- `session_summaries.summary_short`
- `memory_items.chunk_id`
- `memory_items.related_person_id`
- `memory_items.location_snapshot_id`
- `memory_items.source_run_id`
- `memory_items.title`
- `memory_items.evidence_text`
- `memory_items.valid_from`
- `memory_scores.source_run_id`
- `memory_scores.reason_short`

## 11. 哪些字段需要限制字数

### 原则

- raw transcript 不做过度硬裁切，但要通过 segment 粒度控制长度
- summary 和 memory 类字段必须短
- 错误信息可以长一点，但不能无限长

### 建议硬限制

- `location_snapshots.place_label_short <= 120`
- `location_snapshots.locality <= 120`
- `location_snapshots.admin_area <= 120`
- `location_snapshots.country_code <= 8`
- `location_snapshots.timezone_id <= 64`
- `person_entities.canonical_name <= 120`
- `person_entities.display_name <= 120`
- `session_speakers.speaker_label <= 32`
- `voiceprint_profiles.embedding_model <= 120`
- `voiceprint_profiles.embedding_model_version <= 64`
- `voiceprint_profiles.frozen_reason <= 280`
- `voiceprint_enrollment_samples.rejection_reason <= 280`
- `review_item_audio_examples.transcript_hint <= 280`
- `stt_jobs.provider <= 64`
- `stt_jobs.last_error <= 1000`
- `analysis_runs.model_name <= 120`
- `analysis_runs.prompt_version <= 64`
- `analysis_runs.last_error <= 1000`
- `session_summaries.summary_short <= 280`
- `memory_items.title <= 120`
- `memory_items.text <= 600`
- `memory_items.evidence_text <= 280`
- `memory_scores.scoring_version <= 32`
- `memory_scores.reason_short <= 200`

### transcript 的长度处理

`transcript_segments.text` 不建议直接上硬限制。  
更好的做法是：

- 单段 transcript 目标控制在 `300-400` 字以内
- 如果一段太长，就继续切 segment

## 12. 可扩展策略

### 人话

可扩展不等于先塞一堆 JSON 黑箱。  
真正可扩展，是以后要改时不用把旧数据全推翻。

### v0 扩展策略

1. 人物、地点、声纹、评分拆成独立表  
以后加新字段，不会污染 transcript 和 memory 主表

2. 所有 AI 派生结果都挂 `source_run_id`  
以后换模型、换 prompt、换算法时可重跑

3. 评分公式要有 `scoring_version`  
以后 6 维变 8 维，也不会把旧分数搞乱

4. 尽量少在 canonical 表放 JSONB  
真正需要半结构化扩展时，优先新建表，不优先塞 blob

## 13. v0 必须支持的查询

### 本地 SQLite

1. 哪些 session 还在录？
2. 某个 session 下有哪些 chunk？
3. 哪些 chunk 还没上传？
4. 某个 session 下有哪些地点快照？

### 云端 Postgres

1. 最近有哪些 session？
2. 某个 session 的 transcript 是什么？
3. 这次 session 在哪说的？
4. 这次 session 有几个人说话，分别可能是谁？
5. 哪些 chunk 还没转写？
6. 最近有哪些 `commitment / decision / preference / relationship / context / fact`？
7. 某个关键词出现在哪些 transcript 里？
8. 最近最重要的 memory 是哪些，它们为什么重要？
9. 哪些人物已经有可自动识别的 strong voiceprint profile？
10. 哪些 voiceprint profile 样本不足或存在污染风险？
11. 当前有哪些 pending identity review items？
12. 某个 review item 需要播放哪些最多 3 条音频片段？

## 14. v0 刻意不建的字段和表

为了防止 schema 失控，以下内容先不建：

- `users`
- `accounts`
- `permissions`
- `devices`
- `knowledge_graph_edges`
- `agent_access_logs`

### 一个明确的判断

我把原来的 `speaker_profiles` 删除了，换成三层：

- `person_entities`
- `session_speakers`
- `voiceprint_profiles`

这是更干净的抽象。  
“人物是谁”“这次 session 里谁在说话”“如何靠声音认出这个人”应该分开。

## 15. 建表顺序建议

### 先建本地 SQLite

1. `recording_sessions`
2. `audio_chunks`
3. `location_snapshots`

### 再建云端 Postgres

1. `sessions`
2. `chunks`
3. `location_snapshots`
4. `person_entities`
5. `session_speakers`
6. `voiceprint_profiles`
7. `voiceprint_enrollment_samples`
8. `review_items`
9. `review_item_audio_examples`
10. `stt_jobs`
11. `analysis_runs`
12. `transcript_segments`
13. `session_summaries`
14. `memory_items`
15. `memory_scores`

## 16. 现在这份 schema 的完成度

### 已经定下来的

- 地点作为一等对象进入 schema
- 说话人和人物实体分开建模
- 声纹 profile 和 enrollment sample 分开建模
- review queue 只处理身份类问题，并支持最多 3 条原音频播放例子
- summary 改成可选且短
- 重要性改成 6 维可拆分评分
- 可空字段和字数限制被明确化
- schema 的扩展方式被收紧

### 还没彻底定死的

- `person_entities` 初始如何命名
- 说话人解析具体用哪条 pipeline
- voiceprint 自动识别的最终阈值策略
- transcript 全文检索具体怎么做
- 评分维度后续是否要加入 `health` 或 `novelty`
- 哪些 memory 需要人工确认再长期升权

## 17. 下一步最合理的动作

如果继续推进，下一份文档建议是：

- `API_SPEC`

里面要定：

- iPhone 上传接口
- session 完成接口
- 地点快照上传接口
- session 列表接口
- transcript 查询接口
- search 接口
