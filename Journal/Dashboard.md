## Recent Notes

```dataview
TABLE file.ctime AS "Create time", file.tags AS "Tags" FROM "" WHERE file.cday >= date(now) - dur(7 days) SORT file.cday DESC
```

## TODOs

### LLM

```dataview
TASK
FROM "LLM"
WHERE !completed
SORT due ASC

```
