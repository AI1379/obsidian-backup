---
tags: [journal]
---

# Dashboard

## 今日

```dataview
LIST
FROM "Journal"
WHERE file.name = dateformat(date(now), "yyyy-MM-dd")
```

> 没有今天的日记？用 `Templates/Journal` 模板新建一篇。

## 待办

```dataview
TASK
FROM "Courses" OR "Math" OR "Tech" OR "Misc"
WHERE !completed
SORT file.mtime DESC
LIMIT 20
```

## 最近修改

```dataview
TABLE file.mtime AS "时间"
FROM "Courses" OR "Math" OR "Tech" OR "Misc"
SORT file.mtime DESC
LIMIT 8
```

## 快捷入口

- [[Courses/期末复习计划|期末复习计划]]
- [[Tech/Agent/|Agent 调研]] · [[Tech/RAG/|RAG]] · [[Tech/Nahida-Bot/|Nahida-Bot]]
- [[Misc/Interviews/面试准备|面试准备]]
- [[Fictions/|同人创作]]
