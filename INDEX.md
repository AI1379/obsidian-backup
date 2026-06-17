# 📚 Vault Index

> 最近更新：2026-06-17 ｜ Phase 2 已完成（技术/研究边界重构）
> 标记：🟡 待 Phase 3 改名 ｜ ⚪ 待定边界

## 目录总览

| 目录 | 文件数 | 用途 | 状态 |
|------|--------|------|------|
| [[Math/README\|Math]] | 42 | 数学学科笔记（代数/ODE/数论/分析） | ✅ 组织良好 |
| [[研究/README\|研究]] | 18 | 探索性研究、论文调研、综述 | ✅ Phase 2 已重构 |
| [[技术/README\|技术]] | 17 | 命令、配置、工程实战 | ✅ Phase 2 已瘦身 |
| [[课程/README\|课程]] | 38 | 课堂笔记，5 门课 | 🟡 日期格式不一 |
| [[面试/README\|面试]] | 8 | 米哈游 2026 春招准备 | ✅ 清晰 |
| [[Fictions/README\|Fictions]] | 4 | 同人创作 | ✅ 清晰 |
| [[Zotero/README\|Zotero]] | 7 | 文献管理（RAG 向） | ✅ 清晰 |
| [[Templates/README\|Templates]] | 15 | 模板 | ✅ 清晰 |
| [[Journal/Dashboard\|Journal]] | 8 | 日记 | ✅ 清晰 |

## Phase 2 完成的迁移（2026-06-17）

✅ **从 `技术/` 迁回 `研究/`（9 篇研究内容）**
- `Agentic-Search-研究综述` → `研究/Agent/`
- `多模态RAG论文整理` → `研究/RAG/`
- Nahida-Bot 调研 8 篇 → `研究/Nahida-Bot/`

✅ **`研究/` 内部主题集中**
- 新建 `研究/Agent/`（3 篇：Agentic-Search 综述、Agentic Exploration of PDE、GAM Hierarchical Agentic Memory）
- 新建 `研究/RAG/`（3 篇：HiRAG、MoE-RAG、多模态RAG论文整理）

✅ **链接修复**
- `差异化战略分析.md` 内 3 条旧路径文本已更新
- 其余 wiki 链接均按文件名解析，不受影响
- `技术/调研/` 空目录已清理

## Phase 3 待办（统一命名规范）

### 🟡 目录英文统一
| 现名 | 目标 |
|------|------|
| `Math/` | （已英文）|
| `研究/` | `Research/` |
| `技术/` | `Tech/` |
| `课程/` | `Courses/` |
| `面试/` | `Interviews/` |
| `Fictions/` `Zotero/` `Templates/` `Journal/` | （已英文）|

### 🟡 日期格式统一为 `YYYY-MM-DD`
- `技术/学生会工作/25-10-28.md` → `2025-10-28.md`
- `研究/arXiv 订阅/` 的日报范围格式（保留或拆分）
- Nahida-Bot 文件名日期后缀（`-2026-04`）保留

### 🟡 分隔符统一
- 空格 / 连字符 / 下划线混用 → 统一为连字符或保留中文标题原样

## 散落文件说明

`研究/` 根目录有 3 篇独立主题论文暂未归类（因果推理、The Geometry of Knowing、LLM 方向高校调研）——每篇主题独特，不值得单独建目录，暂留根目录。
