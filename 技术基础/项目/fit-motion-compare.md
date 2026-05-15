---
tags: [姿态检测, dtw, 动作对比, 实时, 算法, swift, 项目]
created: 2026-05-14
---

# 动作对比方案

姿态检测已能输出每帧的关节角度序列。动作对比的核心难题是**时间对齐**：
用户做动作的速度和标准不同，不能简单逐帧对比。

---

## 两种场景，两种方案

| 场景 | 推荐方案 | 原因 |
|---|---|---|
| **实时指导**（边做边反馈） | 阶段识别 | 轻量、对速度免疫、延迟低 |
| **录制视频分析**（做完后复盘） | DTW | 精确时间对齐、可定位问题时段 |

### 实时方案：阶段识别

不逐帧对比，把动作切成有限阶段，识别当前阶段后只和该阶段标准姿态比：

```
深蹲：站立准备 → 下降中 → 最低点 → 上升中
```

- 标准姿态只需定义几个关键帧
- 基于关节角度 + 运动方向判断阶段
- 实时触发语音反馈（见 [[fit-ai-voice-coach|AI 实时语音教练方案]]）

### 录制分析方案：DTW

动作做完后，整段序列用 DTW 对齐，精确找出哪一段、哪个关节有问题。

---

## DTW 详解

### 问题：为什么不能直接逐帧对比

```
标准序列（30帧）：  [s0][s1][s2]...[s29]
用户序列（45帧）：  [u0][u1][u2]...[u44]  ← 用户做慢了

逐帧对比：s0↔u0, s1↔u1, ...  ← 序列长度不同，无法对齐
```

即使长度相同，用户在某段做慢、某段做快，对应关系也会错乱。

### 核心思想：允许时间轴"弯曲"

DTW（Dynamic Time Warping，动态时间规整）的本质：

> 找一条从 (s0,u0) 到 (sm,un) 的路径，使路径上所有对齐帧的距离之和最小。

```
标准帧索引 →  0  1  2  3  4  5
              ↕  ↕  ↕  ↓  ↕  ↕
用户帧索引 →  0  1  2  2  3  4
                        ↑
                  用户在第2帧停留了两拍（做慢了）
                  DTW 把 s3 也对应到 u2，自动"拉伸"
```

### 帧如何被"拉伸"——DP 转移方程

DTW 用动态规划构建一个 m×n 的代价矩阵，每个格子 dp[i][j] 表示：
**标准序列前 i 帧和用户序列前 j 帧对齐的最小总代价。**

```
dp[i][j] = cost(si, uj) + min(
    dp[i-1][j-1],   // 路径①：两者同步前进（正常对齐）
    dp[i-1][j],     // 路径②：标准前进，用户不动（用户做慢了，si 对应同一个 uj）
    dp[i][j-1]      // 路径③：用户前进，标准不动（用户做快了，多个 ui 对应同一个 sj）
)
```

**路径②就是"帧拉伸"的机制**：允许标准序列的多帧对应用户的同一帧，相当于把用户这一帧在时间轴上拉长。

```
可视化三条路径：
         用户帧 j →
         0  1  2  3  4
标  0    .  .  .  .  .
准  1    .  .  .  .  .
帧  2    .  .  *  ↑  .    ← 路径②（i前进j不动）= 标准帧2对应用户帧2
i  3    .  .  ↑  ↖  .    ← 路径③（j前进i不动）= 用户帧3对应标准帧2
↓  4    .  .  .  ↖  ↖
```

最终从右下角 dp[m][n] 回溯，得到最优对齐路径。

### 完整实现

#### 姿态帧定义与距离

```swift
struct PoseFrame {
    let joints: [JointAngle]
    let timestamp: TimeInterval
}

struct JointAngle {
    let joint: Joint
    let angle: Float
}

enum Joint: CaseIterable {
    case leftKnee, rightKnee
    case leftHip, rightHip
    case leftShoulder, rightShoulder
    case spine
}

// 两帧距离：加权关节角度差
func distance(_ a: PoseFrame, _ b: PoseFrame) -> Float {
    let weights: [Joint: Float] = [
        .leftKnee: 1.5, .rightKnee: 1.5,
        .leftHip: 1.2,  .rightHip: 1.2,
        .spine: 1.0,
        .leftShoulder: 0.8, .rightShoulder: 0.8
    ]
    var total: Float = 0
    var totalWeight: Float = 0
    for joint in Joint.allCases {
        let w = weights[joint] ?? 1.0
        total += w * abs(a.angle(for: joint) - b.angle(for: joint))
        totalWeight += w
    }
    return total / totalWeight  // 加权平均角度差（度）
}
```

#### DTW 核心

```swift
struct DTWResult {
    let totalDistance: Float
    let normalizedScore: Float           // 0-100 相似度得分
    let alignmentPath: [(Int, Int)]      // (标准帧索引, 用户帧索引)
    let frameDistances: [Float]          // 每对对齐帧的距离
}

func dtw(standard: [PoseFrame], user: [PoseFrame]) -> DTWResult {
    let m = standard.count, n = user.count
    var dp = Array(repeating: Array(repeating: Float.infinity, count: n + 1), count: m + 1)
    dp[0][0] = 0

    for i in 1...m {
        for j in 1...n {
            let cost = distance(standard[i-1], user[j-1])
            dp[i][j] = cost + min(dp[i-1][j-1], dp[i-1][j], dp[i][j-1])
        }
    }

    // 回溯最优路径
    var path: [(Int, Int)] = []
    var i = m, j = n
    while i > 0 && j > 0 {
        path.append((i-1, j-1))
        let diag = dp[i-1][j-1], up = dp[i-1][j], left = dp[i][j-1]
        if diag <= up && diag <= left { i -= 1; j -= 1 }
        else if up <= left            { i -= 1 }
        else                          { j -= 1 }
    }
    path.reverse()

    let frameDistances = path.map { distance(standard[$0.0], user[$0.1]) }
    let normalizedDist = dp[m][n] / Float(path.count)
    let score = max(0, 100 - normalizedDist)

    return DTWResult(
        totalDistance: dp[m][n],
        normalizedScore: score,
        alignmentPath: path,
        frameDistances: frameDistances
    )
}
```

#### 找出问题时段

```swift
struct ProblemSegment {
    let userFrameRange: Range<Int>
    let avgError: Float
    let worstJoint: Joint
    let description: String
}

func analyzeProblemSegments(
    result: DTWResult,
    standard: [PoseFrame],
    user: [PoseFrame],
    threshold: Float = 15.0   // 超过15度算有问题
) -> [ProblemSegment] {
    var segments: [ProblemSegment] = []
    var problemStart: Int? = nil

    for (idx, dist) in result.frameDistances.enumerated() {
        let (si, ui) = result.alignmentPath[idx]
        if dist > threshold {
            if problemStart == nil { problemStart = ui }
        } else if let start = problemStart {
            let worstJoint = findWorstJoint(standard[si], user[ui])
            segments.append(ProblemSegment(
                userFrameRange: start..<ui,
                avgError: dist,
                worstJoint: worstJoint,
                description: describeError(worstJoint, error: dist)
            ))
            problemStart = nil
        }
    }
    return segments.sorted { $0.avgError > $1.avgError }
}
```

#### 生成报告 + 喂给 AI

```swift
func buildAIPrompt(score: Float, problems: [ProblemSegment], jointScores: [Joint: Float]) -> String {
    """
    用户完成了一次深蹲，整体相似度得分 \(Int(score)) 分。
    主要问题：
    \(problems.map { "- 第\($0.userFrameRange.lowerBound/30)秒附近：\($0.description)" }.joined(separator: "\n"))
    各关节评分：\(jointScores.map { "\($0.key): \(Int($0.value))分" }.joined(separator: "，"))
    请用教练口吻给出针对性建议，100字以内。
    """
}
```

### 完整分析流程

```
用户录制视频
    ↓
逐帧姿态检测 → [PoseFrame 序列]
    ↓
DTW 对比标准序列（构建 m×n DP 矩阵）
    ↓
回溯最优对齐路径
    ↓
找出误差超阈值的时段 + 最差关节
    ↓
生成结构化报告 → 喂给 DeepSeek → 文字/语音反馈
    ↓
在视频时间轴上高亮问题时段（红色标注）
```

### DTW 的两个核心约束

DTW 路径必须满足：

**1. 单调性（Monotonicity）** — 路径只能前进，不能回头

```
✅ 合法方向：→（j+1）、↓（i+1）、↘（i+1,j+1）
❌ 不合法：← 或 ↑（回头）
```

保证动作顺序不乱：深蹲的"最低点"不会被对应到"站立准备"前面。

**2. 连续性（Continuity）** — 每步最多前进一格，不能跳帧

```
✅ (3,3) → (4,4)   合法
❌ (3,3) → (5,5)   不合法（跳了两格）
```

保证每一帧都被对应上，不会漏掉某段动作。

**为什么用 DP 而不是贪心**（逐帧找最相似的往后走）：

贪心是局部最优，当前帧找到最相似的不代表整条路径代价最小。DP 把所有可能对齐方式全算出来，保证**整条路径总代价全局最优**。

### 性能

- 时间复杂度：O(m×n)
- 30秒@30fps：900×900 ≈ 81万次运算，iOS 上毫秒级，离线分析完全够用
- 实时场景不适合（每帧重算代价太高），实时用阶段识别

---

## 相关

- [[fit-overview|PostureAI 项目]] — 整体架构
- [[fit-pose-detection-implementation|姿态检测实现]] — PoseFrame 数据来源
- [[fit-ai-voice-coach|AI 实时语音教练]] — 实时阶段识别方案
