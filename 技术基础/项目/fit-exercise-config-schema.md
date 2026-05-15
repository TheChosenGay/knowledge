---
tags: [project, fit, ios, exercise-model, configuration]
created: 2026-05-14
status: 设计中
---

# 动作配置驱动方案

将运动解剖学知识从代码中剥离为 JSON 配置文件，代码只做通用引擎。

## 三层分离

```
配置层:  exercise_configs/squat.json        ← AI 生成，人说 OK
         ├── 主关节、辅助关节
         ├── 关键帧定义
         └── 常见错误模式 + 反馈文案

模型层:  exercise_models/squat_v1.json       ← 标准动作数据（来自视频提取）
         └── 关键帧角度范围、轨迹曲线

引擎层:  ExerciseFormEvaluator.swift         ← 不变，读配置运行
         └── 不关心具体是什么动作
```

**核心原则：** 加新动作 = 加一个 JSON 配置，不改代码。

## JSON Schema 设计

### 动作配置 `exercise_configs/<name>.json`

```json
{
  "$schema": "exercise_config_schema.json",
  "exercise_id": "squat",
  "display_name": { "zh": "深蹲", "en": "Squat" },
  "category": "bodyweight",
  "equipment": null,

  "primary_joints": [
    {
      "joint": "knee",
      "side": "both",
      "definition": {
        "type": "angle",
        "points": ["hip", "knee", "ankle"],
        "description": "髋-膝-踝夹角"
      }
    }
  ],

  "secondary_joints": [
    {
      "joint": "hip",
      "side": "both",
      "definition": {
        "type": "angle",
        "points": ["shoulder", "hip", "knee"],
        "description": "躯干-髋-膝夹角，监控前倾"
      }
    }
  ],

  "keyframe_detection": {
    "method": "extremum",
    "signal": "primary_joints.knee",
    "keyframes": [
      {
        "name": "top",
        "label": { "zh": "顶点（站立）" },
        "detection": "local_maximum",
        "tolerance_degrees": 5
      },
      {
        "name": "bottom",
        "label": { "zh": "底点（蹲下）" },
        "detection": "local_minimum",
        "tolerance_degrees": 5
      }
    ],
    "cycle_definition": "top → bottom → top"
  },

  "rep_counting": {
    "triggers_on": "bottom_to_top_transition",
    "min_amplitude_degrees": 40,
    "debounce_frames": 10
  },

  "standard_model_ref": "squat_v1",

  "common_errors": [
    {
      "id": "knees_caving_in",
      "condition": "primary_joints.knee.left - primary_joints.knee.right > 15",
      "label": { "zh": "膝盖内扣" },
      "feedback": { "zh": "膝盖向外打开，对准脚尖方向" },
      "severity": "warning"
    },
    {
      "id": "excessive_forward_lean",
      "condition": "secondary_joints.hip < 45",
      "label": { "zh": "躯干过度前倾" },
      "feedback": { "zh": "保持背部挺直，挺胸" },
      "severity": "warning"
    },
    {
      "id": "not_deep_enough",
      "condition": "keyframes.bottom.primary_joints.knee > 100",
      "label": { "zh": "下蹲深度不足" },
      "feedback": { "zh": "再蹲深一点，大腿至少平行地面" },
      "severity": "correction"
    }
  ],

  "scoring": {
    "weights": {
      "depth_accuracy": 0.4,
      "symmetry": 0.3,
      "tempo_consistency": 0.15,
      "stability": 0.15
    }
  }
}
```

### 器械动作示例 `exercise_configs/leg_press.json`

```json
{
  "exercise_id": "leg_press",
  "category": "machine",
  "equipment": {
    "name": "倒蹬机",
    "type": "plate_loaded"
  },

  "primary_joints": [
    {
      "joint": "knee",
      "side": "both",
      "definition": {
        "type": "angle",
        "points": ["hip", "knee", "ankle"]
      }
    }
  ],

  "keyframe_detection": {
    "method": "extremum",
    "signal": "primary_joints.knee",
    "keyframes": [
      { "name": "extended",  "label": { "zh": "伸展" }, "detection": "local_maximum" },
      { "name": "flexed",    "label": { "zh": "屈膝" }, "detection": "local_minimum" }
    ],
    "cycle_definition": "extended → flexed → extended"
  },

  "common_errors": [
    {
      "id": "knee_hyperextension",
      "condition": "keyframes.extended.primary_joints.knee > 180",
      "label": { "zh": "膝盖超伸" },
      "feedback": { "zh": "腿不要完全蹬直，保持微屈" },
      "severity": "critical"
    }
  ]
}
```

## 配置文件清单

当配置方案稳定后，可以用 AI 批量生成以下常见动作：

**自由重量（7 个）：**
- 深蹲、硬拉、卧推、杠铃划船、肩推、引体向上、弓步蹲

**自体重（6 个）：**
- 俯卧撑、平板支撑、仰卧起坐、臀桥、波比跳、深蹲跳

**器械（8 个）：**
- 腿举（倒蹬机）、坐姿腿屈伸、俯卧腿弯举、坐姿划船、高位下拉、蝴蝶机夹胸、史密斯机深蹲、哈克深蹲

## 引擎读取流程

```
ExerciseFormEvaluator 启动
  → 加载 exercise_configs/<id>.json
  → 根据 primary_joints 构建角度计算器
  → 根据 keyframe_detection 构建极值检测器
  → 根据 rep_counting 构建计次状态机
  → 每帧 evaluate():
      ├→ 计算所有 primary/secondary 关节角度
      ├→ 极值检测 → 匹配 keyframes
      ├→ 周期判定 → 更新 repCount
      ├→ common_errors 规则匹配 → 生成反馈
      └→ 读取 standard_model_ref 对应模型 → 评分
```

## 与标准模型的关系

两套 JSON 分工明确：
- **配置** (`exercise_configs/`)：告诉引擎"检测什么"——哪些关节、关键帧在哪、错误模式
- **模型** (`exercise_models/`)：告诉引擎"标准是什么"——关键帧的角度范围参考值

同一个动作可以有多版模型（如 `squat_v1`、`squat_v2`），配置指向当前版本即可。

## 相关

- [[fit-standard-exercise-model|标准动作模型库]]
- [[fit-refactoring-targets|重构目标]] — 第 7 项：评分阈值科学化
