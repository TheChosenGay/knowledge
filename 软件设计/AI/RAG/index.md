---
tags: [moc, 软件设计, AI]
created: 2026-05-15
publish: true
---

# RAG 检索增强生成专题

> 构建基于私有知识库的 AI 问答系统

## 核心知识点

### RAG 基础架构
```
用户问题
    ↓
向量化（Embedding）
    ↓
向量检索（相似度搜索）
    ↓
召回相关文档
    ↓
LLM 生成答案（携带上下文）
```

### 关键组件

#### 文档处理
- 分块策略（Chunk Size / Overlap）
- 元数据提取与过滤
- 多模态文档解析（PDF / Word / 图片）

#### Embedding 模型
- OpenAI text-embedding-3 / BGE / M3E
- 向量维度与精度权衡

#### 向量数据库
- Milvus / Weaviate / Qdrant / Chroma
- HNSW 索引原理
- 混合检索（稠密 + 稀疏）

### 优化方向
- **检索优化**：HyDE、Rerank 重排序
- **生成优化**：Few-shot 示例、格式约束
- **评估**：RAGAS 框架（Faithfulness / Relevancy）

---

## 练习题（待补充）

内容正在整理中，敬请期待。
