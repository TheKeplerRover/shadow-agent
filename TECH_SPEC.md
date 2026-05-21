# Shadow Audio v0 TECH_SPEC

## 1. 这份文档是干嘛的

`PRD` 解决的是“做什么”。  
`TECH_SPEC` 解决的是“怎么做，先做哪一块，做到什么算跑通”。

这份文档会坚持一种写法：

- 先讲人话
- 再给一个小例子
- 最后给技术实现

原因很简单：这是个人项目，不需要靠黑话显得专业。需要的是把系统真的做出来。

## 2. 系统总览

### 人话

这个系统像一个“会自动记账的录音秘书”。

- iPhone 负责把你说过的话录下来
- 云端负责把录音变成文字
- 再从文字里提炼出“承诺、决定、偏好、关系、上下文”

### 例子

像你一天里收了很多快递：

- iPhone 是前台收件员，负责收包裹、贴单号、先放进仓库
- 云端是后仓，负责拆包、分类、记账
- 最后你可以问系统：“我昨天关于 X 说过什么？”

### 技术

v0 的主链路：

1. 用户在 iPhone 上点击 `Start Listening`
2. iPhone 开始一个 recording session
3. 音频输入进入本地 VAD 检测
4. 有语音时生成 chunk 文件，最长 60 秒切一段
5. chunk 先写本地磁盘，再写入本地 SQLite 状态
6. 后台上传器把 chunk 文件异步上传到云端
7. 云端接收 chunk，保存 raw audio，并创建 STT job
8. worker 执行 STT，产出 transcript
9. transcript 合并成 session transcript
10. memory processor 从 transcript 提炼 memory items
11. 用户后续按时间、主题、任务、偏好检索

## 3. 用户流程

### 人话

第一版不是“永远在听”，而是“你按一下开始，它开始记；你按一下停止，它结束整理”。

### 例子

像你开一个会议录音：

- 开会前点开始
- 会中系统自动切成小段
- 散会后系统慢慢上传、转写、整理
- 之后你能看到这次会说了什么、决定了什么

### 技术

用户流程：

1. 点击 `Start Listening`
2. app 创建 `recording_session`
3. session 状态变成 `active`
4. 音频流进入 VAD 和 chunk writer
5. 点击 `Stop`
6. 当前 chunk 收尾并写盘
7. session 状态变成 `ended`
8. 上传器继续把未上传 chunk 推到云端
9. 云端完成 STT 和 memory distill
10. 用户在 history / search 页面查看结果

## 4. iPhone 端设计

### 4.1 Recording Session

### 人话

`session` 就是一整次“我明确开始录音，到我明确停止录音”的大盒子。

### 例子

一次 session 就像一整场会议。  
chunk 是会议里的几段小录音文件。

### 技术

每次 `Start Listening` 创建一个 `session_id`。  
这个 session 记录：

- 开始时间
- 结束时间
- 当前状态
- 总 chunk 数
- 上传进度

建议状态：

- `active`
- `stopping`
- `ended`
- `processed`

### 4.2 Location Capture

### 人话

地点不是附加信息，而是语境。  
这次话是在家里、办公室、机场还是医院说的，会直接改变它的意义。

### 例子

同一句“我得把这个处理掉”，如果是在机场说，和在家里说，后续检索价值完全不同。

### 技术

v0 的地点策略：

- 只在 active recording session 期间采地点
- session 开始时抓一次位置
- 如果 session 很长，且位置发生明显变化，再补抓一次
- session 结束时可再抓一次收尾位置

不建议 v0 申请 `Always` 定位权限。  
更合理的是：

- 请求 `When In Use`
- 只在你主动开始录音后采集位置

这是更干净的权限边界，也更符合现在的产品形态。

建议 iOS 配置：

```xml
<key>NSLocationWhenInUseUsageDescription</key>
<string>Shadow records where a listening session happens so the transcript and memories keep real-world context.</string>
```

如果后面需要更精确地点，再补：

```xml
<key>NSLocationTemporaryUsageDescriptionDictionary</key>
<dict>
  <key>SessionLocationCapture</key>
  <string>Shadow temporarily uses precise location to label where a recording session happened.</string>
</dict>
```

v0 不做后台持续定位。  
地点只服务于“这段 session 在哪里发生”。

### 4.3 Chunking + VAD

### 人话

不要把整场录音塞成一个大文件。大文件难上传、难重试、难处理。  
要切成很多小块，但也不能机械地每秒都切，所以要先判断“有没有人在说话”。

### 例子

像抄笔记时，不会把一整天写成一段话。  
你会在“有内容”的地方分段，“没人说话”的空白就不记。

### 技术

v0 规则：

- 使用本地 VAD 检测语音活动
- 检测到语音时开始或继续写当前 chunk
- 连续静默超过 `2s`，结束当前 chunk
- 单个 chunk 最长 `60s`
- 纯静默不保留为长期 chunk

推荐实现：

- iOS 原生 app：`SwiftUI + AVAudioEngine`
- 用 input tap 获取音频 buffer
- 先做轻量 VAD
- 有语音时写入 chunk 文件

v0 不追求“学术级 VAD 精度”。  
目标只是不要把大量沉默送去 STT。

### 4.4 Audio File Format

### 人话

第一版音频格式要选“够用、稳定、体积别太大”的，不要为了完美折腾太久。

### 例子

像拍照存档，不一定非要原始无压缩电影级素材。  
先保证“看得清、存得下、传得动”。

### 技术

v0 建议：

- 单声道
- `.m4a` 容器
- AAC 编码

理由：

- 文件体积比 WAV 小很多
- iPhone 端更适合长期落盘
- 足够支持后续 STT

如果实现上先用 PCM/WAV 更快，也可以先这样做，但那是临时方案，不建议作为长期默认。

### 4.5 Local Storage

### 人话

iPhone 端要像一个小仓库，先把东西存好，再决定什么时候发货。  
不能一边录一边假设网络永远稳定。

### 例子

像快递站先把包裹放到货架，再等快递车来拉走。  
没有货架，车一晚点，包裹就丢了。

### 技术

本地需要两层存储：

1. 文件系统  
保存 chunk 音频文件

2. SQLite  
保存 session、chunk、地点快照、上传状态

建议目录：

- `AppData/recordings/{session_id}/{chunk_id}.m4a`

建议本地表：

#### `recording_sessions`

- `session_id`
- `started_at`
- `ended_at`
- `status`
- `chunk_count`
- `primary_location_snapshot_id`
- `created_at`
- `updated_at`

#### `audio_chunks`

- `chunk_id`
- `session_id`
- `seq_no`
- `file_path`
- `started_at`
- `ended_at`
- `duration_ms`
- `has_speech`
- `size_bytes`
- `sha256`
- `upload_status`
- `upload_attempts`
- `last_error`
- `remote_object_key`
- `created_at`
- `updated_at`

#### `location_snapshots`

- `snapshot_id`
- `session_id`
- `captured_at`
- `capture_reason`
- `latitude`
- `longitude`
- `horizontal_accuracy_m`
- `place_label_short`
- `locality`
- `admin_area`
- `country_code`
- `timezone_id`

## 5. 上传设计

### 人话

上传器的工作不是“赶紧传”，而是“别丢、别重、别乱”。

### 例子

像寄快递：

- 每个包裹都有单号
- 没寄出去就继续留在货架
- 重寄也不能变成两个包裹

### 技术

v0 上传原则：

- chunk 落盘成功后才允许进入上传队列
- 上传以 chunk 为单位
- 每个 chunk 有稳定的 `chunk_id`
- 服务端按 `chunk_id` 做幂等处理

本地 `upload_status`：

- `pending`
- `uploading`
- `uploaded`
- `failed`

服务端处理状态：

- `ingested`
- `stt_queued`
- `transcribed`
- `distilled`

推荐上传策略：

- 默认前台可上传
- 后台使用 `URLSession` background upload
- 文件上传必须基于本地文件，不基于内存 buffer

失败重试：

- 网络失败回到 `pending`
- 指数退避重试
- 永久失败写 `last_error`

## 6. 云端设计

### 6.1 Ingest API

### 人话

云端第一站像“收货口”，只负责确认收到、记账、放进仓库。  
不要在这个入口上做太重的计算。

### 例子

仓库收货员不会当场拆开每个包裹研究内容。  
他先登记入库，再交给后面的人处理。

### 技术

建议接口：

- `POST /v1/chunks/upload`
- `POST /v1/sessions/{session_id}/complete`
- `GET /v1/sessions`
- `GET /v1/sessions/{session_id}`
- `GET /v1/search?q=...`

`POST /v1/chunks/upload` 负责：

- 校验 `chunk_id`
- 校验 `session_id`
- 存 raw audio 到 object storage
- 写数据库记录
- 创建 `stt_job`
- 返回成功

### 6.2 Storage

### 人话

数据库负责记“索引卡”。  
对象存储负责放“大件货物”。

### 例子

图书馆不会把书塞进借书卡。  
借书卡只记编号、作者、位置。书本身放书架。

### 技术

推荐：

- Object storage：`Cloudflare R2`
- App database：`Postgres + pgvector`
- API framework：`FastAPI`

原因：

- raw audio 是大文件，适合 object storage
- session / transcript / memory 是结构化数据，适合数据库
- voiceprint embeddings 和后续相似度查询适合先放在 Postgres + pgvector
- Python 生态更适合后续 STT、diarization、voiceprint worker

### 6.3 STT Worker

### 人话

STT worker 是“听录音写字幕的人”。  
它不需要实时在线盯着用户，只需要把待处理队列慢慢做完。

### 例子

像会议结束后，助理回头去听录音整理纪要，不需要在会议进行时同步敲完整稿。

### 技术

`stt_jobs` 表负责排队。  
worker 循环取任务：

1. 拉取一个 `queued` job
2. 读取 raw audio
3. 调 STT 引擎
4. 写入 transcript segments
5. 更新 job 状态

建议 `stt_jobs.status`：

- `queued`
- `processing`
- `done`
- `failed`

### v0 Worker Strategy

第一阶段先用 thin mock worker，不直接接真实 STT。

mock worker 负责：

- 把上传成功的 chunk 标成 `done`
- 写入少量假 transcript segments
- 生成一条 identity review item
- 给 review item 关联最多 3 条 audio examples

这样可以先验证：

- 上传链路
- R2 存储
- Postgres 状态机
- transcript 查询
- review queue 卡片流

真实 STT、diarization、voiceprint worker 放到第二阶段接入。

### 6.4 Speaker Resolution And Analysis

### 人话

“这段话是谁说的”不能靠一个字符串糊过去。  
但也不能让大模型随便认亲。

### 例子

更合理的做法是先记：

- 这次对话里有 `SPK_0`
- 还有 `SPK_1`

然后再慢慢判断：

- `SPK_0` 很可能是你自己
- `SPK_1` 很可能是妈妈

### 技术

说话人处理分两步：

1. `session_speakers`
先解析这次 session 里有几个说话槽位、每个人说了多久

2. `person_entities`
再尝试把说话槽位映射到跨 session 的稳定人物实体

确认身份后，再建立或更新 `voiceprint_profiles`。  
这一步是为了让系统以后靠声音自动认人，减少重复问用户。

身份解析可以用大模型辅助，但结果必须带：

- `identity_confidence`
- `source_run_id`

不要让模型直接把“猜测身份”写死成无证据的真相。

### 声纹策略

人话：  
用户确认“这个人是妈妈”之后，系统应该开始记住妈妈的声音模板。以后听到相似声音时，先用 voiceprint 匹配，而不是每次都重新问。

技术：

- `voiceprint_profile` 绑定到 `person_entity`
- 主模板只允许由 `trusted` enrollment samples 更新
- `usable` 样本只能辅助排序和解释
- `rejected` 样本只保留引用和拒绝原因
- profile 要有健康状态，避免脏音频越学越脏

`profile_health`：

- `insufficient`
- `usable`
- `strong`
- `contaminated`
- `frozen`

一段音频进入 `trusted enrollment` 的最低条件：

- 身份来源可靠，来自用户确认或高置信自动识别
- 主要是单人发言
- 没有明显重叠说话
- 背景媒体声不是主导
- 单段至少 `3-5s`
- 初建 profile 最好累计 `20-30s` trusted audio
- 和当前 profile 接近，并明显领先第二候选

### 身份判定策略

v0 先不用精确数值阈值，先用三档策略：

- `auto_resolve`
  strong profile、高相似度、明显领先第二名、没有冲突信号

- `candidate_only`
  usable profile 或中等相似度，只作为候选，不打扰用户

- `review_queue`
  高影响身份问题、相似度不错但不够稳、或两个已知人物容易混淆

如果 profile 疑似被脏音频污染，进入 `frozen`，不参与自动认人。

### Review Queue

v0 review queue 只问两类问题：

- `identify_person`
- `merge_person`

不问：

- 这个人重不重要
- 这段关系是否长期重要
- 偏好是否应该升级

身份确认卡片必须提供原音频播放条。  
如果是多段音频需要用户判断，最多展示 3 条播放条。

播放条规则：

- 每条目标 `3-10s`
- 优先选清晰、单人、代表性强的片段
- 可以附一个很短的 transcript hint
- 用户可以听完再确认、否定、稍后或手动输入身份

## 7. Transcript / People / Memory

### 人话

这几层不是一回事：

- transcript：原话
- people：这次是谁在说
- session summary：这段大概讲了什么
- memory：以后值得记住的东西

### 例子

像上课：

- transcript 是老师每句话
- people 是谁在发言
- summary 是这节课讲了什么
- memory 是考试前真要背的重点

### 技术

建议服务端表：

#### `sessions`

- `session_id`
- `started_at`
- `ended_at`
- `status`
- `chunk_count`
- `primary_location_snapshot_id`
- `speaker_count_estimate`
- `created_at`
- `updated_at`

#### `location_snapshots`

- `snapshot_id`
- `session_id`
- `captured_at`
- `capture_reason`
- `place_label_short`

#### `person_entities`

- `person_id`
- `canonical_name`
- `person_kind`

#### `session_speakers`

- `session_speaker_id`
- `session_id`
- `speaker_label`
- `person_id`
- `identity_confidence`
- `speech_duration_ms`

#### `voiceprint_profiles`

- `voiceprint_profile_id`
- `person_id`
- `profile_status`
- `profile_health`
- `embedding_ref`
- `trusted_sample_count`
- `quality_score`
- `contamination_risk`

#### `voiceprint_enrollment_samples`

- `sample_id`
- `voiceprint_profile_id`
- `person_id`
- `session_id`
- `chunk_id`
- `sample_status`
- `identity_source`
- `duration_ms`
- `speech_quality_score`
- `overlap_score`
- `background_media_score`

#### `review_items`

- `review_item_id`
- `review_type`
- `status`
- `target_session_speaker_id`
- `target_person_id`
- `candidate_person_id`
- `candidate_confidence`

#### `review_item_audio_examples`

- `example_id`
- `review_item_id`
- `seq_no`
- `session_id`
- `chunk_id`
- `start_ms`
- `end_ms`
- `audio_object_key`

#### `transcript_segments`

- `segment_id`
- `session_id`
- `chunk_id`
- `session_speaker_id`
- `start_ms`
- `end_ms`
- `text`
- `confidence`
- `created_at`

#### `session_summaries`

- `session_id`
- `summary_short`
- `source_run_id`

#### `memory_items`

- `memory_id`
- `session_id`
- `memory_type`
- `related_person_id`
- `location_snapshot_id`
- `source_run_id`
- `title`
- `text`
- `importance_score`
- `confidence`
- `source_chunk_id`
- `evidence_text`
- `created_at`

#### `memory_scores`

- `memory_id`
- `financial_score`
- `emotional_score`
- `relational_score`
- `identity_score`
- `interest_score`
- `commitment_score`
- `importance_score`

`memory_type` v0 改成更人中心：

- `commitment`
- `decision`
- `preference`
- `relationship`
- `context`
- `fact`

### 设计收紧

- `session_summary` 改成可选且短，不再默认塞主题列表和 open loops
- 值得长期保留的主题和关系，应该进入 `memory_items`
- 重要性不再只是一列拍脑袋分数，而是拆成 6 维评分

## 8. 检索设计

### 人话

第一版检索不要做成“超级智能助手”。  
先做到“你能找回东西，而且找回来的内容靠谱”。

### 例子

像先把家里东西按抽屉分好，再谈是不是要一个会猜你想找什么的机器人。

### 技术

v0 支持三种检索：

1. 按时间看 session
2. 按关键词搜 transcript
3. 按 memory type 过滤
4. 按人物看历史
5. 按地点看历史

v0 不要求：

- 复杂 agentic search
- 全自动跨时间推理
- 完整人格画像

## 9. 状态机

### 人话

状态机的作用就是防止系统“脑子乱掉”。  
每段录音必须知道自己现在处在哪一步。

### 例子

像外卖订单：

- 已下单
- 已接单
- 配送中
- 已送达

不能一会儿显示“已送达”，一会儿又跳回“商家备餐”。

### 技术

chunk 生命周期：

`recording -> sealed -> pending_upload -> uploading -> uploaded -> stt_queued -> transcribed -> distilled`

异常分支：

`uploading -> failed -> pending_upload`

`processing -> failed -> queued`

这条状态机是 v0 的核心。  
如果状态机不清楚，后面所有重试、恢复、去重都会变脏。

## 10. v0 非目标

以下内容明确不做：

- 多用户
- 权限系统
- 分享能力
- 多设备同步
- 复杂知识图谱
- 完美说话人识别
- 24/7 默认常驻监听
- 高级推理型搜索

## 11. 推荐实施顺序

### 人话

不要一口气把全系统写完。  
先把最短闭环跑通，再逐层加能力。

### 技术顺序

1. iPhone 前台 `push-to-start` 录音
2. 本地 chunk 落盘
3. 本地 SQLite 记录 session / chunk
4. 手动上传单个 chunk
5. 后台批量上传
6. FastAPI ingest API + R2 object storage
7. Postgres + pgvector schema
8. thin mock worker
9. transcript 存储
10. review queue
11. 真实 STT / diarization / voiceprint worker
12. memory distill
13. session/history/search UI

## 12. 开发完成的最低标准

当满足以下条件，就算 v0 技术闭环成立：

1. iPhone 可以稳定开始和结束一次 recording session
2. 录音能被切成合理 chunk
3. chunk 能在网络不好时继续留在本地，不会丢
4. 网络恢复后 chunk 能继续上传
5. 云端能把 chunk 变成 transcript
6. transcript 能产出至少 4 类 memory
7. 用户能通过时间和关键词找回内容

做到这里，才有资格继续讨论更大的 Shadow。
