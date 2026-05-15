---
tags: [project, fit, ios, pose-detection, exercise-model, backend]
created: 2026-05-14
status: 设计中
---

# 标准动作模型库设计

为实时训练的动作评估提供标准化参考模型，替代当前的魔法数字阈值评分。

## 动机

当前 `ExerciseFormEvaluator` 的评分依赖硬编码角度范围（如深蹲膝角 80°-170° = 100 分），没有运动科学依据，无法区分"勉强完成"和"标准完成"。

需要一个**标准动作模型库**，将专业教练的标准动作数据化，作为评分基准。

## 模型方案对比

### 方案 A：关键帧角度模板（推荐起步）

将标准动作拆解为几个关键帧，每帧存储目标关节角度：

```json
{
  "exercise": "squat",
  "version": 1,
  "body_scale_normalized": true,
  "phases": [
    {
      "name": "bottom",
      "progress": 0.5,
      "angles": {
        "left_knee":   {"min": 75, "max": 90},
        "right_knee":  {"min": 75, "max": 90},
        "left_hip":    {"min": 60, "max": 80},
        "torso_lean":  {"min": 30, "max": 50}
      }
    },
    {
      "name": "top",
      "progress": 1.0,
      "angles": {
        "left_knee":  {"min": 170, "max": 180},
        "right_knee": {"min": 170, "max": 180},
        "torso_lean": {"min": 0, "max": 10}
      }
    }
  ]
}
```

**优点：** 体积小（几 KB），客户端计算量低，容易理解  
**缺点：** 只检查离散点，不检查轨迹平滑度

### 方案 B：关节轨迹曲线

录制完整动作周期，提取各关节角度随时间的变化曲线（归一化到 [0,1] 进度）：

```json
{
  "exercise": "squat",
  "trajectories": {
    "left_knee":  [160, 155, 140, ..., 80, 85, 100, ..., 170],
    "right_hip":  [170, 165, 150, ..., 70, 75, 90, ..., 175]
  },
  "samples": 100,
  "tolerance": 8.0
}
```

比较时使用 DTW（Dynamic Time Warping）匹配用户曲线与标准曲线。

**优点：** 全程评估，检测动作流畅度  
**缺点：** 模型体积中等，DTW 计算稍重（仍可在移动端实时运行）

### 方案 C：完整时空约束

结合角度 + 速度 + 加速度 + 对称性 + 稳定性的多维度模型，类似专业运动生物力学分析。

**优点：** 最全面  
**缺点：** 模型复杂，需要大量标定数据，不适合初期

### 建议路线

先用**方案 A**跑通深蹲（模型打包在 Bundle 里），验证端到端链路，再逐步演进到方案 B。

## 后台模型生成管线

```
网络获取标准动作视频 → Python + MediaPipe → 逐帧姿态提取
  → 动作周期检测（自相关/峰值检测）→ 归一化（身体比例）
  → 标准模型 JSON → 存入数据库 → FastAPI 端点
```

### 技术选型

| 环节 | 技术 | 原因 |
|---|---|---|
| 姿态提取 | MediaPipe Pose | 成熟稳定，33 点足以覆盖主要关节 |
| 周期检测 | 膝角自相关 | 周期运动的自动分割 |
| 归一化 | 腿长/躯干比例缩放 | 适配不同身材用户 |
| 后端框架 | FastAPI (Python) / Gin (Go) | 轻量、快速搭建 |
| 存储 | PostgreSQL JSONB 或 MongoDB | 模型结构灵活，JSON 友好 |

### API 设计

```
GET /api/v1/exercises              → 可用动作列表
GET /api/v1/exercises/squat/model  → 深蹲标准模型 JSON
GET /api/v1/exercises/squat/model?version=2 → 指定版本
```

## 客户端设计

### 模型缓存策略

- 首次登录 → 从后台拉取所有动作模型 → 存入本地（文件 / SwiftData）
- 后续训练 → 检查版本号，有更新则增量下载
- 默认内置一个深蹲模型在 App Bundle，离线也可用

### 评分比较流程（改造 ExerciseFormEvaluator）

```
用户完成一个动作周期
  → 提取关键帧角度
  → 与标准模型对应帧比较
    ├→ 每帧角度偏差 → 逐帧评分
    ├→ 对称性（左右偏差）→ 对称分
    └→ 加权综合 → 0-100 总分 + 分项反馈
```

### 反馈生成

根据偏差类型生成针对性中文提示：
- 膝角偏大 → "蹲得再深一点"
- 膝角偏小 → "不用蹲太深，到大腿平行即可"
- 左右不对称 → "重心偏左，注意平衡"
- 躯干前倾过多 → "保持背部挺直"

## 实施步骤

1. 找一个标准深蹲视频（或请教练录制），用 MediaPipe 跑一遍提取数据，手工标定关键帧角度，生成第一版 `squat_model_v1.json`
2. 把模型放 App Bundle 中，改造 `ExerciseFormEvaluator` 读取模型并比较
3. 用真机测试对比（有模型 vs 无模型），看评分是否更合理
4. 验证通过后搭建后台管线，支持在线模型下发和更新
5. 扩展更多动作：俯卧撑、硬拉、平板支撑

## 相关

- [[fit-pose-detection-implementation|姿态检测实现]]
- [[fit-rtmpose-plan|RTMPose 升级方案]]
- [[fit-motion-compare|动作对比]]
- [[fit-refactoring-targets|重构目标]] — 第 7 项：评分阈值科学化
