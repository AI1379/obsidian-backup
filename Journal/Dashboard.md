---
tags: [journal]
---

# Dashboard

## 最近更新的笔记

```dataview
TABLE file.mtime AS "更新时间", file.tags AS "标签"
FROM -"Templates"
SORT file.mtime DESC
LIMIT 10
```

## 最近 7 天新建

```dataview
TABLE file.ctime AS "创建时间", file.tags AS "标签"
FROM ""
WHERE file.cday >= date(now) - dur(7 days)
SORT file.cday DESC
```

## 待办事项

```dataview
TASK
FROM -"Templates"
WHERE !completed
SORT file.mtime DESC
LIMIT 30
```

## 课程笔记

```dataview
LIST
FROM "Courses"
SORT file.name ASC
```

## 数学笔记

```dataview
LIST
FROM "Math"
SORT file.name ASC
```

## 近期日记

```dataview
LIST
FROM "Journal"
WHERE file.name != "Dashboard"
SORT file.name DESC
LIMIT 5
```
