# Shadow Audio v0 PRD

## Product Goal

Shadow 的第一版不是做完整用户镜像系统，而是先做一条最小可验证主链路：

`iPhone 录音 -> 异步上传 -> STT -> 记忆整理 -> 可检索回看`

目标不是“理解整个人格”，而是先证明系统可以稳定保留真实语音内容，并把其中有用的信息转成长期可用的记忆。

## Target User

- 单用户
- 完全自用
- 不考虑多用户、分享、权限体系、合规产品化包装

## Core Use Case

用户在 iPhone 上主动点击开始录音，系统持续记录这一段 listening session。录音数据先在本地切片保存，再在设备空闲、连网、最好充电时异步上传到云端。云端完成 STT 和后续记忆整理，最终让用户可以按时间、主题、人物、地点、承诺、偏好等方式回看和检索。

## Product Decisions

### 1. Recording Mode

- 采用 `push-to-start`
- 只有用户明确点击开始后才进入录音状态
- 用户点击停止后结束本次 session
- 第一版不做默认 24/7 常驻监听

### 2. Primary Device

- iPhone 是第一采集端
- iPhone 第一版主要负责录音、切片、排队、上传
- STT 与记忆处理不要求在 iPhone 实时完成

### 3. Upload Strategy

- 音频 chunk 先本地保存
- 使用后台上传能力做异步上传
- 优先在空闲、连网、最好充电时上传
- “睡觉时上传”是目标场景，但不依赖精确时间触发

### 4. Chunking Strategy

- 采用 `VAD + max 60s chunk`
- 有语音时才持续生成有效 chunk
- 连续语音最长切到 60 秒一个 chunk
- 静默时间过长时结束当前 chunk 或 session

说明：
- `VAD` 用来判断是否有人声，避免把大量沉默和无效背景噪音送去 STT
- 固定最大 chunk 时长用来简化上传、重试、状态管理和后端处理

### 5. Retention Strategy

第一版长期保留：

- 有语音内容的 raw audio
- transcript
- memory items

第一版不要求长期保留：

- 纯静默 chunk

### 6. Location Capture

- 每次 listening session 需要记录地点
- 地点优先从 iPhone 的地理位置能力获取
- 第一版只在 active session 期间采集地点
- 第一版不做后台持续定位

### 7. Speaker Capture

- 每次 session 需要记录说话人数
- 需要尽量识别分别是谁
- 说话人解析可以使用大模型做后处理
- 但人物身份应保留置信度，不能把猜测直接当成确定事实

### 8. Summary And Derived Data Discipline

- `session_summary` 是可选的，不要求每次都有
- summary 应该保持很短
- transcript 之外的总结性内容不应无限膨胀
- 值得长期记住的内容应优先进入结构化 memory，而不是堆在 summary 里

## v0 System Scope

### In Scope

- iPhone 端录音
- iPhone 端地点采集
- 本地音频切片
- 本地上传队列
- 异步上传到云端
- 云端 STT
- 云端说话人解析
- transcript -> session -> memory 的处理链路
- 基础检索与回看

### Out of Scope

- 多用户
- 复杂权限控制
- 产品化隐私体系
- 多设备同步
- 重型知识图谱
- 自动月报 / 年报
- 完整人格建模

## Proposed Architecture

### iPhone App

负责：

- 开始 / 停止 recording session
- 麦克风采集
- VAD 判断与切片
- 本地文件存储
- SQLite 记录 chunk 与上传状态
- 后台异步上传

### Cloud Backend

负责：

- 接收上传的音频 chunk
- 存储 raw audio
- 创建 STT job
- 异步执行转写
- 合并 transcript 为 session
- 从 transcript 中提炼 memory
- 提供后续检索接口

## Memory Scope for v0

第一版优先提炼以下信息：

- commitments：用户明确承担的承诺、义务、要兑现的事
- decisions：用户做出的决定及理由
- preferences：稳定偏好与厌恶
- relationships：和重要人物相关的持续关系信息
- people / project context：和人、项目、主题相关的持续上下文

第一版不追求直接提炼“人格”。

## Infra Direction

推荐方向：

- iPhone 只做 capture 与 upload
- 云端做 STT 和 memory processing
- 使用便宜云平台搭建自用后端

候选组合：

- `Railway + Cloudflare R2`
- `DigitalOcean Droplet + Cloudflare R2`

原则：

- 计算节点便宜、可持续在线
- 对象存储成本低
- 适合异步 worker 处理音频与 STT 任务

## Success Criteria

v0 的成功标准是：

1. 用户可以在 iPhone 上稳定开始和结束一段 listening session
2. 音频 chunk 能可靠上传到云端
3. 云端能稳定生成 transcript
4. transcript 能产出有用的 commitments / decisions / preferences / relationships / context
5. 用户之后可以检索回昨天或更早说过的重要内容

如果做不到以上五点，就不进入更复杂的 Shadow 能力开发。

## Next Planning Topics

下一步要具体化的内容：

- 本地 SQLite schema
- audio chunk 文件结构与命名规则
- 上传状态机
- 后端 job schema
- STT pipeline
- transcript / session / memory schema
- 检索 API 与最小 UI
