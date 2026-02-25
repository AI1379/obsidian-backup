---
title: Linux find 命令使用
created: 2026-02-24
source: Cherry Studio
tags:
  - Linux
  - 运维
---

## 🤖 Assistant

# Linux find 命令使用指南

`find` 是 Linux 系统中用于在指定目录下查找文件和目录的强大工具，支持根据文件名、类型、大小、修改时间等多种条件进行搜索，并可对搜索结果执行操作 [1][2]。

## 一、基本语法

```bash
find [搜索路径] [表达式]
```

- **搜索路径**：指定从哪些目录开始搜索，默认是当前目录（`.`），可指定多个路径 [2]
- **表达式**：由测试条件、动作和逻辑运算符组成，用于定义搜索规则 [2]

## 二、常用搜索条件

### 1. 按文件名搜索

| 选项 | 说明 | 示例 |
|------|------|------|
| `-name` | 按文件名查找（区分大小写） | `find . -name "*.txt"` [1] |
| `-iname` | 按文件名查找（不区分大小写） | `find /etc -iname "*host*"` [2] |

支持通配符：`*` 匹配任意字符，`?` 匹配单个字符 [1]。

### 2. 按文件类型搜索

```bash
find . -type f    # 查找普通文件 [1]
find . -type d    # 查找目录 [1]
find . -type l    # 查找符号链接 [2]
```

常见类型：`f`（普通文件）、`d`（目录）、`l`（符号链接）、`s`（套接字）等 [2]。

### 3. 按文件大小搜索

```bash
find /home -size +1M      # 查找大于1MB的文件 [1]
find . -size -1k          # 查找小于1KB的文件 [3]
find . -size 512c         # 查找恰好512字节的文件 [2]
```

单位：`c`（字节）、`k`（KB）、`M`（MB）、`G`（GB）[1]。符号：`+` 表示大于，`-` 表示小于 [2]。

### 4. 按时间搜索

| 选项 | 含义 | 单位 |
|------|------|------|
| `-mtime` | 修改时间 | 天 [1] |
| `-atime` | 访问时间 | 天 [1] |
| `-ctime` | 状态变更时间 | 天 [1] |
| `-mmin` | 修改时间 | 分钟 [2] |

```bash
find /var/log -mtime +7    # 查找7天前修改的文件 [1]
find . -mtime -7           # 查找最近7天内修改的文件 [3]
find . -amin -60           # 查找1小时内访问过的文件 [2]
```

**时间参数说明**：`+n` 表示n天前更早，`-n` 表示n天内，`n` 表示正好n天前 [1]。

### 5. 按所有者和权限搜索

```bash
find /path -user username     # 按文件所有者查找 [1]
find /path -group groupname   # 按文件所属组查找 [1]
find /path -perm 644          # 按文件权限查找 [4]
```

## 三、高级用法

### 1. 逻辑运算符组合条件

```bash
# 逻辑与（-and 可省略）
find /var/log -type f -name "*.log" -mtime -1 -size +1M [2]

# 逻辑或（需要分组转义）
find . -type f $ -name "*.txt" -or -name "*.md" $ [2]

# 逻辑非
find . -type f -not -name "*.tmp" [2]
```

> ⚠️ 注意：`-or` 优先级低于 `-and`，需用 `$` 和 `$` 分组，括号需转义 [2]。

### 2. 对搜索结果执行命令（-exec）

```bash
# 对每个文件执行一次命令
find . -name "*.tmp" -exec rm {} \; [3]

# 更高效：所有文件一次执行
find . -type f -name "*.c" -exec wc -l {} + [2]

# 删除前确认
find /var/log -type f -mtime +7 -ok rm {} \; [1]
```

- `{}`：占位符，表示匹配到的文件名 [1]
- `\;`：命令结束符（需转义）[1]
- `+`：将所有文件作为参数传给命令（更高效）[2]

### 3. 处理特殊文件名

```bash
# 文件名含空格时的正确方式
find . -name "*.txt" -print0 | xargs -0 grep "keyword" [2]
```

`-print0` 用 null 字符分隔结果，`xargs -0` 正确读取 [2]。

### 4. 限制搜索深度

```bash
find . -maxdepth 1 -name "*.log"    # 仅当前目录，不搜索子目录 [4]
```

### 5. 直接删除（-delete）

```bash
find /path -type f -size -1k -delete    # 删除小于1KB的文件 [4]
```

## 四、最佳实践与注意事项

1. **缩小搜索范围**：避免从根目录 `/` 开始搜索，直接指定可能路径 [2]
2. **先测试再执行**：使用 `-exec` 删除文件前，先用 `ls -l` 或 `echo` 预览 [2]
3. **优先使用 `-exec +`**：比 `-exec \;` 更高效，减少进程创建开销 [2]
4. **注意大小写**：`-name` 区分大小写，`-iname` 不区分 [2]
5. **权限问题**：查找系统文件时可能需要 `sudo` 提升权限 [3]

## 五、常用命令速查表

| 目的 | 命令 |
|------|------|
| 查找特定文件 | `find /path -name "filename.txt"` [3] |
| 查找所有目录 | `find /path -type d` [3] |
| 查找大文件 | `find /path -size +100M` [3] |
| 查找近期修改文件 | `find /path -mtime -7` [3] |
| 查找并删除 | `find /path -name "*.tmp" -exec rm {} \;` [3] |
| 排除特定目录 | `find /home -path "/home/user/.cache" -prune -o -name "*.log" -print` [2] |

---

掌握 `find` 命令能极大提高文件查找和管理效率。记住：**明确搜索目标、先测试再执行、妥善处理特殊文件名** [2]。如有更复杂的需求，可查阅 `man find` 获取完整文档。

[1] [Linux find 命令 | 菜鸟教程](https://www.runoob.com/linux/linux-comm-find.html)

[2] [Linux 下的 `find` 命令完全指南：从基础到高级用法](https://geek-blogs.com/blog/manual-find-linux/)

[3] [全面掌握find命令：使用与用法详解-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/2469108)

[4] [Linux find命令指南|极客教程](https://geek-docs.com/linux/linux-basic/t_guide-to-the-linux-find-command.html)