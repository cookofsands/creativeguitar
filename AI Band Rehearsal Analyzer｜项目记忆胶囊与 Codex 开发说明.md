# AI Band Rehearsal Analyzer

## 0. 项目一句话定义

**AI Band Rehearsal Analyzer 是一个面向乐队排练录音的离线/准离线节奏分析与复盘系统。**

核心目标：

> 将一次乐队排练录音转换成可量化的节奏事件、Timing Error、BPM 稳定性和乐器间 Synchronization 数据，并通过可视化与自然语言报告告诉乐队成员“哪里不稳、什么时候出问题、谁与谁没有锁住”。

项目第一阶段**只分析节奏，不分析音准，不做实时识别，不做复杂音乐表现力评价**。

核心原则：

> **先测量事实，再让 LLM 解释事实。**

底层音频算法必须能够脱离 LLM 独立工作。

---

# 1. 为什么做这个项目

目标不是做一个普通的：

- AI Chatbot
- RAG
- 音乐播放器
- 音乐生成器
- LLM + 音频 API Demo

而是做一个真正解决乐队排练问题的工程项目：

实际排练结束后，成员通常只知道：

> “刚才这一段感觉不太稳。”

但不知道：

- 从什么时候开始不稳？
- 是整体速度变了还是某个成员偏了？
- 吉他和鼓究竟差多少？
- 哪个片段最值得重新练？
- 过去几次排练反复出现的问题是什么？

本项目希望把这些模糊感受转化为：

```text
客观数据
+
问题事件
+
时间轴
+
历史记录
+
排练建议
```

---

# 2. 第一阶段严格范围

## 必须实现

第一版只关注以下四类能力：

### 2.1 Beat / Tempo Analysis

分析：

- BPM
- Beat 时间位置
- Tempo Stability
- 局部 BPM 波动

例如：

```text
Average BPM: 158.2
Tempo Std: 1.8 BPM
Largest Local Drift: +4.2 BPM
```

---

## 2.2 Onset Detection

分析每个音频轨道或乐器轨道的事件发生时间。

例如：

```text
Drums:
0.381
0.759
1.140
1.519

Bass:
0.394
0.771
1.161
1.542

Guitar:
0.402
0.782
1.187
1.571
```

---

## 2.3 Timing Error

计算实际演奏事件相对于参考 Beat / 参考事件的时间偏移。

例如：

```text
Expected: 1.140 s
Actual:   1.187 s

Timing Error: +47 ms
```

约定：

```text
负值 = 提前
正值 = 延后
```

---

## 2.4 Synchronization Analysis

分析成员之间的同步关系。

重点分析：

```text
Guitar ↔ Drums
Bass   ↔ Drums
Guitar ↔ Bass
```

输出：

```text
mean absolute offset
median offset
std offset
percentage within threshold
synchronization score
```

例如：

```text
Guitar ↔ Drums
Mean Offset: 31 ms
Median Offset: 24 ms
P95: 67 ms
Sync Score: 78
```

---

# 3. 第一阶段明确不做什么

以下全部暂时禁止进入 MVP：

### 不做实时分析

不是：

```text
麦克风输入
↓
实时 AI
↓
实时提示
```

而是：

```text
录音结束
↓
上传
↓
后台分析
↓
结果展示
```

允许几十秒到几分钟的离线分析延迟。

---

### 不做音准分析

第一阶段不分析：

- Pitch Accuracy
- Note Accuracy
- Chord Accuracy
- 错音检测
- 推弦
- 滑音
- 颤音

这些属于 V2。

---

### 不做“艺术表现力评分”

禁止输出没有明确算法依据的：

> “你弹得很有感情。”

> “这次演奏很有力量。”

> “整体音乐性 83 分。”

第一阶段所有评价必须尽量基于客观可计算指标。

---

### 不做复杂自动编曲

不生成：

- MIDI
- Guitar Pro
- 新歌曲
- AI 作曲

---

### 不做视频分析

第一阶段不处理：

- 摄像头
- 演奏姿态
- 手部动作
- 人体关键点

以后可以做多模态 V3。

---

# 4. 推荐的输入模式

项目应该支持两种输入。

## 模式 A：多轨录音，优先支持

理想输入：

```text
session/
├── drums.wav
├── bass.wav
├── guitar.wav
└── vocal.wav
```

第一阶段实际上只需要：

```text
drums.wav
bass.wav
guitar.wav
```

多轨录音具有最大的分析可靠性。

优势：

- 不需要 Source Separation
- 乐器身份明确
- onset 检测更稳定
- Synchronization 更容易计算
- 更适合做算法实验

---

## 模式 B：单个混合录音

例如：

```text
rehearsal.wav
```

以后可选：

```text
rehearsal.wav
↓
Source Separation
↓
drums / bass / guitar / vocal
```

第一阶段可以保留接口，但不要求实现完整 Source Separation。

---

# 5. Reference 的定义

为了判断“弹早/弹晚”，系统需要一个参考。

第一版推荐三种 Reference：

### Reference A：参考 MIDI / Beat 时间轴

最佳。

例如：

```text
reference.json
```

保存：

```json
{
  "bpm": 158,
  "beats": [
    0.000,
    0.380,
    0.760,
    1.139
  ]
}
```

---

### Reference B：原曲音频

V2 支持。

原曲：

```text
reference.wav
```

经过 Beat Tracking / Onset Detection 后获得 Reference Timeline。

---

### Reference C：首次正确演奏作为 Baseline

如果没有原曲，可以让乐队第一次录一遍相对稳定的演奏：

```text
baseline.wav
```

以后排练与 baseline 对齐。

---

# 6. 核心算法流水线

完整流程：

```text
Input Audio
    ↓
Audio Validation
    ↓
Preprocessing
    ↓
Beat Tracking
    ↓
Onset Detection
    ↓
Event Cleaning
    ↓
Reference Alignment
    ↓
Timing Error Calculation
    ↓
Pairwise Synchronization
    ↓
Window / Section Aggregation
    ↓
Problem Event Detection
    ↓
Scoring
    ↓
Visualization Data
    ↓
LLM Report
```

---

# 7. Audio Preprocessing

输入后统一：

- Sample Rate
- Channel
- PCM format
- Loudness normalization
- Silence trimming（谨慎使用）

建议内部统一：

```text
Sample Rate: 44100 Hz
dtype: float32
mono/stereo 根据轨道类型处理
```

不要在预处理阶段过度降噪，因为过度处理可能损害 onset。

所有原始录音都必须保留。

---

# 8. Beat Tracking

目标：

得到：

```python
beat_times: np.ndarray
```

例如：

```python
[
    0.000,
    0.379,
    0.759,
    1.139,
    ...
]
```

同时得到：

```python
tempo_bpm
```

Beat tracking 可以使用成熟音频分析库/模型，不要求第一版自己训练模型。

优先保证：

1. 可重复
2. 运行稳定
3. 输出时间戳
4. 有离线测试

---

# 9. Onset Detection

对于每一个轨道：

```text
drums.wav
bass.wav
guitar.wav
```

得到：

```python
onset_times
```

示例：

```python
{
    "drums": [0.381, 0.759, 1.141],
    "bass": [0.394, 0.770, 1.160],
    "guitar": [0.402, 0.782, 1.187]
}
```

需要做：

- threshold
- minimum inter-onset interval
- peak picking
- false positive filtering

这些参数必须集中配置，不要散落在代码中。

---

# 10. Beat-to-Onset Matching

不能简单使用最近邻而不考虑音乐情况。

初版采用：

> **Nearest Beat Matching + 最大允许时间窗口**

例如：

```text
beat = 1.140
onset = 1.187
delta = +47ms
```

如果：

```text
abs(delta) > threshold
```

则判定为异常事件。

推荐初始阈值：

```text
25 ms   - 非常紧
40 ms   - 较好
60 ms   - 可接受
80 ms+  - 明显偏移
```

注意：

这些阈值不是音乐学上的绝对真理。

必须设计为：

```yaml
timing:
  excellent_ms: 20
  good_ms: 40
  warning_ms: 60
  bad_ms: 80
```

后续方便实验调整。

---

# 11. Timing Error

每个事件应该生成类似结构：

```json
{
  "event_id": "evt_000123",
  "instrument": "guitar",
  "expected_time": 102.340,
  "actual_time": 102.387,
  "offset_ms": 47,
  "abs_offset_ms": 47,
  "direction": "late"
}
```

定义：

```text
offset_ms =
(actual_time - expected_time) * 1000
```

---

# 12. Synchronization Analysis

单独判断：

```text
“这个人相对于 Beat 是否准确”
```

还不够。

还需要：

```text
“这个人和其他成员是否同步”
```

例如：

```text
Drums onset = 1.140
Bass onset  = 1.158
Guitar onset = 1.187
```

则：

```text
Bass ↔ Drums = +18 ms
Guitar ↔ Drums = +47 ms
Guitar ↔ Bass = +29 ms
```

这三个数据应该同时保存。

---

# 13. 不要简单地把 BPM 当成节奏表现

例如：

```text
158 BPM
```

本身没有意义。

真正需要分析：

```text
local tempo
tempo drift
beat-to-beat deviation
```

建议按滑动窗口：

```text
window = 4 bars
hop = 1 bar
```

输出：

```text
00:00–00:08  158.1 BPM
00:08–00:16  158.4 BPM
00:16–00:24  159.7 BPM
00:24–00:32  163.2 BPM   ← abnormal
```

于是可以发现：

> 从 24 秒开始整体开始加速。

---

# 14. Event Model

不要只保存最终分数。

系统必须保存原始事件。

统一 Event：

```python
@dataclass
class PerformanceEvent:
    event_id: str
    timestamp: float
    instrument: str
    event_type: str
    expected_timestamp: float | None
    offset_ms: float | None
    severity: float | None
    confidence: float
    metadata: dict
```

event_type 第一阶段：

```text
onset
timing_error
tempo_drift
sync_error
```

未来可以扩展：

```text
pitch_error
missed_note
extra_note
section_transition
```

---

# 15. Problem Event Detection

不要把每一个 +10ms 都显示给用户。

需要将大量低级事件聚合成高级问题。

例如：

```text
102.3s +47ms
102.8s +44ms
103.2s +51ms
103.6s +49ms
104.0s +46ms
```

聚合为：

```text
Problem Event
102.3–104.0s
instrument: guitar
type: timing_instability
avg_offset: +47ms
severity: high
```

---

# 16. Window Aggregation

推荐按：

```text
4 bars
```

或者：

```text
8 bars
```

聚合。

每个窗口输出：

```json
{
  "start": 102.3,
  "end": 110.4,
  "tempo_mean": 158.2,
  "tempo_std": 2.1,
  "guitar_timing_mae": 31.4,
  "bass_timing_mae": 18.2,
  "drum_timing_mae": 12.7,
  "guitar_drum_sync": 47.2,
  "problem_score": 0.81
}
```

这样前端非常容易绘制时间轴。

---

# 17. Scoring

第一阶段评分只是辅助展示，不应该宣称是“音乐能力评分”。

建议：

## Timing Score

根据 MAE 映射。

例如：

```text
MAE <= 20ms  → 95+
MAE <= 40ms  → 80+
MAE <= 60ms  → 65+
MAE <= 80ms  → 50+
>80ms        → lower
```

但所有映射必须配置化。

---

## Synchronization Score

根据两个乐器之间的平均绝对时间差计算。

例如：

```text
<20ms   excellent
20–40   good
40–60   warning
>60     poor
```

---

## Tempo Stability Score

根据局部 BPM 偏差计算。

---

# 18. 项目第一阶段最终输出

一个排练 Session：

```text
Session
├── Basic Info
│   ├── date
│   ├── song
│   ├── duration
│   └── source
│
├── Global Metrics
│   ├── avg_bpm
│   ├── tempo_stability
│   ├── guitar_timing_score
│   ├── bass_timing_score
│   ├── drum_timing_score
│   └── synchronization_score
│
├── Timeline
│   ├── events
│   └── problem_windows
│
└── Report
    ├── summary
    ├── key_problems
    └── suggestions
```

---

# 19. 前端应该长什么样

第一版不要做花哨 UI。

重点是：

## 页面 1：Session Upload

```text
[ Upload Reference ]

[ Upload Guitar ]
[ Upload Bass ]
[ Upload Drums ]

[ Analyze ]
```

---

## 页面 2：Analysis Dashboard

顶部：

```text
BPM        158.2
Tempo      93
Timing     82
Sync       76
```

中间：

```text
Timeline
─────────────────────────────
       ⚠       ⚠⚠
───────┼───────┼─────────────
       01:32   02:14
```

下面：

```text
Guitar ↔ Drums    31 ms
Bass   ↔ Drums    18 ms
Guitar ↔ Bass     26 ms
```

---

## 页面 3：Problem Detail

点击：

```text
02:14–02:23
```

显示：

```text
Problem: Guitar Timing

Average Offset: +47ms
Direction: Late
Confidence: 0.91

[▶ Play Segment]

Related:
Guitar ↔ Drums +47ms
Bass ↔ Drums    +14ms
```

---

## 页面 4：AI Report

LLM 输出：

```text
本次排练整体速度较稳定。

最主要的问题出现在 02:14–02:23 的 Breakdown：
吉他平均比鼓晚约 47ms，而贝斯与鼓的偏差约为 14ms。

因此这段更可能是吉他进入点不稳定，而不是全员速度失控。

建议下一次：
1. 单独循环该段；
2. 先降低到 80% BPM；
3. 优先让吉他与 Kick 锁定；
4. 再逐渐恢复原速。
```

LLM 不允许凭空创建数据。

---

# 20. LLM 使用原则

LLM 只接收结构化数据。

禁止：

```text
把整首 wav 直接发给 LLM，让它自己判断。
```

正确：

```text
Audio
↓
Deterministic / ML Analysis
↓
Structured JSON
↓
LLM
```

LLM 的职责：

- 总结
- 解释
- 排练建议
- 历史比较
- 自然语言问答

LLM 不负责：

- 核心 Beat Tracking
- Onset Detection
- Timing Error Measurement
- 原始数据计算

---

# 21. LLM Prompt 原则

System Prompt 应强调：

```text
你是一个乐队排练分析助手。

所有事实必须来自提供的结构化分析结果。
不得虚构不存在的错误。
不得修改原始数值。
不得把统计相关性说成因果关系。
如果数据不足以判断责任归属，应明确说明“不确定”。
```

例如：

错误：

> 吉他手导致了整支乐队变慢。

正确：

> 吉他的 timing deviation 明显高于鼓和贝斯，因此当前数据更支持“吉他同步性较差”的判断；仅凭本次数据无法证明吉他是整体速度下降的因果来源。

---

# 22. 历史排练记忆

这是后续非常重要的功能，但不应该阻碍 MVP。

每次 Session 保存：

```text
Session #1
Session #2
Session #3
...
```

可以比较：

```text
Timing Score:
82 → 85 → 88

Guitar-Drum Sync:
72 → 75 → 81
```

然后 AI 可以回答：

> 最近三次排练中，吉他与鼓的同步性正在改善。

也可以发现：

> Breakdown 仍然是连续三次排练中问题最多的区域。

---

# 23. 后续 V2

V1 完成后再考虑：

### V2：Pitch

加入：

- Pitch Accuracy
- Note Accuracy
- Missed Note
- Extra Note

---

### V2：Source Separation

支持：

```text
mixed rehearsal.wav
↓
source separation
↓
stems
```

这样普通手机录音也可以分析。

---

### V2：Reference Audio Alignment

不再强制要求 MIDI。

可以：

```text
Original Song
+
Rehearsal
```

自动对齐。

---

# 24. 后续 V3

### Multimodal Rehearsal

加入视频：

```text
Video
+
Audio
+
Timeline
```

分析：

- 演奏动作
- 成员活动
- 音频事件
- 音视频同步

最终形成：

```text
Visual Events
+
Audio Events
↓
Multimodal Temporal Model
```

---

# 25. 后续 V4

### Rehearsal Agent

用户可以自然语言问：

> “我们今天最大的问题是什么？”

> “这首歌这周有没有进步？”

> “哪里最值得练？”

> “上次这个 Breakdown 解决了吗？”

系统从历史 Session Memory 查询并回答。

---

# 26. 推荐技术栈

## Backend

推荐：

```text
Python
FastAPI
```

负责：

- 音频分析
- 任务调度
- 算法 Pipeline
- API

Java/Spring Boot 可以作为第二阶段加入，负责：

- 用户
- Session
- 项目管理
- 数据持久化

不要为了体现 Java 而强行让 Java 做音频算法。

---

## Frontend

推荐：

```text
React
TypeScript
```

可视化：

```text
Waveform
Timeline
Charts
Audio Player
```

---

## Database

第一版：

```text
SQLite
```

即可。

项目稳定以后：

```text
PostgreSQL
```

---

## Async Task

音频分析不应该阻塞 HTTP 请求。

例如：

```text
POST /sessions
      ↓
create job
      ↓
background worker
      ↓
analysis
      ↓
store result
```

状态：

```text
PENDING
PROCESSING
COMPLETED
FAILED
```

---

# 27. Python 音频算法层

优先研究/评估：

```text
librosa
madmom
scipy
numpy
PyTorch
```

Onset / Beat Tracking 第一版优先采用成熟方法。

不要第一天训练神经网络。

---

# 28. 模型策略

项目必须遵循：

> **Model-agnostic**

不要依赖某一个必须 GPU 才能运行的大模型。

---

## 音频算法

优先：

```text
CPU-friendly
```

---

## Source Separation

可选：

```text
Demucs
BS-RoFormer
其他成熟模型
```

但是它不是 V1 的必要依赖。

---

## LLM

优先使用 API。

模型应该通过统一接口调用：

```python
class LLMProvider:
    def generate(self, prompt: str) -> str:
        ...
```

之后可以接：

```text
OpenAI
Anthropic
Gemini
DeepSeek
Qwen
Local Ollama
```

不允许把某个供应商写死在业务逻辑中。

---

# 29. 推荐项目目录

```text
ai-bandmate/
│
├── backend/
│   ├── app/
│   │   ├── api/
│   │   ├── services/
│   │   ├── models/
│   │   ├── schemas/
│   │   └── repositories/
│   │
│   └── main.py
│
├── analysis/
│   ├── preprocessing/
│   ├── beat_tracking/
│   ├── onset/
│   ├── alignment/
│   ├── timing/
│   ├── synchronization/
│   ├── aggregation/
│   └── scoring/
│
├── llm/
│   ├── providers/
│   ├── prompts/
│   └── report_generator.py
│
├── frontend/
│
├── data/
│   ├── references/
│   ├── sessions/
│   └── outputs/
│
├── tests/
│
├── docs/
│
├── scripts/
│
├── configs/
│
└── README.md
```

---

# 30. API 初版设计

## 创建 Session

```http
POST /api/sessions
```

---

## 上传轨道

```http
POST /api/sessions/{session_id}/tracks
```

---

## 开始分析

```http
POST /api/sessions/{session_id}/analyze
```

---

## 获取分析状态

```http
GET /api/sessions/{session_id}/status
```

---

## 获取结果

```http
GET /api/sessions/{session_id}/analysis
```

---

## 获取问题事件

```http
GET /api/sessions/{session_id}/events
```

---

## 生成 AI 报告

```http
POST /api/sessions/{session_id}/report
```

---

# 31. 数据库核心实体

## Session

```text
id
song_name
created_at
duration
status
reference_type
```

## Track

```text
id
session_id
instrument
file_path
sample_rate
duration
```

## Beat

```text
id
session_id
timestamp
beat_index
```

## PerformanceEvent

```text
id
session_id
instrument
timestamp
expected_timestamp
offset_ms
confidence
event_type
```

## ProblemWindow

```text
id
session_id
start_time
end_time
problem_type
severity
instrument
metrics
```

## Report

```text
id
session_id
provider
model
content
created_at
```

---

# 32. 第一阶段完成标准

必须达到：

### 功能

- [ ] 上传多轨录音
- [ ] 创建 Session
- [ ] 音频预处理
- [ ] Beat Tracking
- [ ] Onset Detection
- [ ] Timing Error
- [ ] 乐器间 Synchronization
- [ ] 时间窗口聚合
- [ ] 问题事件检测
- [ ] Dashboard
- [ ] 点击问题片段播放
- [ ] 保存历史 Session
- [ ] LLM 生成复盘报告

---

# 33. MVP 不需要做到的东西

不要为了“看起来高级”加入：

- 微服务
- Kubernetes
- Kafka
- Redis Cluster
- 大模型本地部署
- 自训练音频大模型
- 实时 WebSocket
- 视频理解
- Pitch Detection
- Source Separation

如果这些东西没有必要，全部推迟。

---

# 34. Codex 开发原则

Codex 在开发过程中必须遵循以下原则：

### 原则 1：先跑通 Pipeline，再优化算法

先做：

```text
Audio
→ Beat
→ Onset
→ Timing
→ Sync
→ JSON
```

再做：

```text
UI
LLM
History
```

---

### 原则 2：算法和 Web 解耦

不要在 API endpoint 中直接写：

```python
librosa...
numpy...
```

应该：

```text
API
 ↓
Service
 ↓
Analysis Pipeline
 ↓
Algorithm Modules
```

---

### 原则 3：所有算法结果结构化

不要直接返回自然语言。

算法层只返回：

```json
{
  "metrics": {},
  "events": [],
  "windows": []
}
```

---

### 原则 4：所有参数配置化

例如：

```yaml
timing:
  warning_ms: 40
  bad_ms: 80

sync:
  warning_ms: 40
  bad_ms: 60

window:
  bars: 4
```

不要在代码里大量出现 magic numbers。

---

### 原则 5：每个核心算法必须有单元测试

至少测试：

- Beat alignment
- Timing calculation
- Synchronization
- Window aggregation
- Score calculation

---

# 35. 最重要的研发策略

不要一上来就使用真实复杂乐队录音。

先构造**合成测试数据**。

例如人工生成：

```text
Beat:
0
0.5
1.0
1.5
```

模拟：

```text
Guitar:
0
0.52
1.03
1.57
```

然后系统应该得到：

```text
0ms
+20ms
+30ms
+70ms
```

确认数学算法正确以后，再放真实录音。

这样能够避免：

> 不知道是算法错了，还是音频模型错了。

---

# 36. 第一阶段最重要的实验数据

需要准备一个小型内部 Dataset。

例如：

```text
10 songs
×
3 takes
×
3 instruments
```

不需要公开发布。

同时建立人工标注：

```text
song
start
end
instrument
problem_type
severity
```

例如：

```text
song_01
102.3
110.4
guitar
timing
high
```

这样后面可以验证系统。

---

# 37. 项目的真正技术亮点

简历里不要写：

> “调用 AI API 分析乐队录音。”

应该强调：

### 亮点 1

**多轨音乐事件提取**

### 亮点 2

**Reference-to-Performance Temporal Alignment**

### 亮点 3

**基于 Timing Deviation 的演奏误差分析**

### 亮点 4

**乐器间 Synchronization Analysis**

### 亮点 5

**Problem Window Detection**

### 亮点 6

**基于历史排练数据的 Rehearsal Memory**

### 亮点 7

**LLM 作为解释层，而非核心测量器**

---

# 38. 项目未来可以形成的研究方向

如果以后想继续研究：

```text
Timing Event
↓
Temporal Graph
↓
Error Propagation
↓
Causal Reasoning
```

例如：

```text
Drum Tempo Drift
       ↓
Bass Drift
       ↓
Guitar Timing Error
       ↓
Band Synchronization Drop
```

可以研究：

> 如何从多轨演奏事件中判断错误传播和时序依赖。

这和单纯的音乐 App 有明显区别，也能与本人已有的时序建模研究经验形成技术延续。

---

# 39. Codex 当前任务优先级

严格按照下面顺序开发：

## Phase 1

```text
Project skeleton
↓
Audio loading
↓
Beat tracking
↓
Onset detection
↓
Timing analysis
↓
Synchronization
```

目标：

```text
一个 CLI 可以对测试音频输出 analysis.json
```

---

## Phase 2

```text
Window aggregation
↓
Problem events
↓
Scoring
```

目标：

```text
analysis.json
```

具有完整：

```text
metrics
events
windows
```

---

## Phase 3

```text
FastAPI
↓
Session management
↓
File upload
↓
Async analysis
```

---

## Phase 4

```text
React
↓
Dashboard
↓
Timeline
↓
Audio playback
```

---

## Phase 5

```text
LLM provider abstraction
↓
Report generation
```

---

## Phase 6

```text
History
↓
Comparison
↓
Rehearsal Memory
```

---

# 40. 第一版最终 Demo

用户打开网站：

```text
AI Bandmate
────────────────────────────

Song: Scream Aim Fire

[Upload Reference]
[Upload Guitar]
[Upload Bass]
[Upload Drums]

             [ Analyze ]
```

分析完成：

```text
BPM
158.3

Tempo Stability
91

Timing
84

Synchronization
77
```

然后：

```text
Timeline
────────────────────────────────────
            ⚠           ⚠⚠
────────────┼───────────┼───────────
           01:32       02:14
```

点击：

```text
02:14–02:23

Guitar ↔ Drums
47 ms

Bass ↔ Drums
14 ms

Problem:
Guitar timing instability

[▶ Play Segment]
```

下面：

> AI 复盘：整体速度稳定。本次最明显的问题发生在 02:14–02:23 的 Breakdown，吉他与鼓的同步误差明显高于贝斯与鼓。建议下一次优先循环练习该段，并先将速度降低至 80%。

这就是第一版完整产品。

---

# 41. 最终产品定位

项目正式名称：

**AI Bandmate**

副标题：

**AI-Powered Rehearsal Analysis and Performance Timing System**

中文：

**面向乐队排练的 AI 节奏分析与智能复盘系统**

核心关键词：

```text
Music Information Retrieval
Audio Analysis
Beat Tracking
Onset Detection
Temporal Alignment
Synchronization Analysis
Performance Analysis
LLM
Rehearsal Memory
```

---

# 42. 对 Codex 的最高优先级要求

在整个开发过程中牢记：

> **这是一个“音频分析工程项目”，不是一个“LLM 套壳项目”。**

核心价值排序：

```text
数据正确性
>
时间对齐正确性
>
节奏分析可靠性
>
同步分析可靠性
>
可视化
>
历史记忆
>
LLM
```

如果模型效果不可靠，不要用 LLM 生成看似合理的结论来掩盖算法问题。

如果没有足够数据支持某个结论，系统必须明确显示：

```text
Insufficient confidence
```

而不是编造。

第一版优先追求：

> **简单、稳定、可解释、可复现、能真正拿真实排练录音测试。**

而不是追求：

> **模型越大越好、功能越多越好。**

---

# 43. 当前开发起点

Codex 第一个目标：

> 创建最小可运行项目，并实现一个 CLI。

使用：

```bash
python analyze.py \
    --reference reference.json \
    --tracks ./tracks \
    --output ./output/analysis.json
```

最终生成：

```text
output/
├── analysis.json
├── metrics.json
└── events.json
```

第一阶段暂时**不要创建完整前端，不要接 LLM，不要接数据库，不要做登录系统**。

先确保：

```text
3 条 WAV
    ↓
Beat Tracking
    ↓
Onset Detection
    ↓
Timing Error
    ↓
Pairwise Synchronization
    ↓
JSON
```

这个核心 Pipeline 可以稳定工作，再向上叠加产品层。