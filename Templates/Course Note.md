---
date: <% tp.date.now("YYYY-MM-DD") %>
course: "<% await tp.system.prompt('课程名') %>"
period: "<% await tp.system.prompt('节次（如 第3-5节）') %>"
instructor: "<% await tp.system.prompt('教师') %>"
tags: [course, course/<% await tp.system.prompt('课程简称（如 复变函数）') %>]
---

# <% tp.date.now("YYYY-MM-DD") %> <% await tp.system.prompt('课程名') %> <% await tp.system.prompt('节次（如 第3-5节）') %>

## 本节要点

