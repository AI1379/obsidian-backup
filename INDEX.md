# 📚 Vault Index

> [[Journal/Dashboard|Dashboard]] ｜ 标签体系：`course` `math` `tech` `misc` `fiction` `paper` `journal`

## 最近更新

```dataview
TABLE file.mtime AS "修改时间", file.tags AS "标签"
FROM -"Templates"
SORT file.mtime DESC
LIMIT 8
```

## 目录总览

| 目录 | 用途 |
|------|------|
| [[Math/README\|Math]] | 数学学科笔记（代数/ODE/数论/分析） |
| [[Tech/README\|Tech]] | 技术探索与工程实战（Agent/RAG/LLM/命令速查/开发配置） |
| [[Courses/README\|Courses]] | 课堂笔记，5 门课 |
| [[Fictions/README\|Fictions]] | 同人创作 |
| [[Zotero/README\|Zotero]] | 文献管理（RAG 向） |
| [[Templates/README\|Templates]] | 模板 |
| [[Misc/README\|Misc]] | 杂项（面试准备、项目点子、学生会、数模） |
| [[Journal/Dashboard\|Journal]] | 日记 |

## 按标签浏览

### course（课堂笔记）
```dataview
LIST
FROM #course
SORT file.name ASC
```

### math（数学）
```dataview
LIST
FROM #math
SORT file.name ASC
```

### tech（技术）
```dataview
LIST
FROM #tech
SORT file.name ASC
```
