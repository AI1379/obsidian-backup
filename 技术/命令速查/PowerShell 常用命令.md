---
tags:
  - 运维
---
> 使用 Gemini-2.5-pro 生成

好的，这是一个非常棒的问题！从 Bash 切换到 PowerShell (pwsh) 的开发者经常需要寻找熟悉的命令。PowerShell 的设计哲学与 Bash 有很大不同（**对象管道 vs. 文本流**），但为了方便过渡，PowerShell 团队创建了很多与 Bash/Linux 命令同名的**别名 (Alias)**。

下面我将列出 Bash 中最常用的命令及其在 PowerShell 中的等价命令、常用别名以及关键区别。

### 核心理念差异

在看具体命令前，请记住这个核心差异：

*   **Bash/Linux**: 一切皆文件，命令之间通过**纯文本流**进行通信。工具如 `grep`, `awk`, `sed` 都是为了解析和处理这些文本。
*   **PowerShell**: 一切皆对象（`.NET Object`），Cmdlet之间通过**对象管道**通信。你操作的是结构化数据，而不是简单的字符串。

---

### 命令对照表

| 分类          | Bash 命令           | PowerShell 等价命令                      | 别名 (Aliases)          | 注释 / 示例                                                                                                |
| :---------- | :---------------- | :----------------------------------- | :-------------------- | :----------------------------------------------------------------------------------------------------- |
| **文件 & 目录** | `ls`, `ls -l`     | `Get-ChildItem`                      | `ls`, `dir`, `gci`    | `ls` 返回的是文件/目录对象，而不仅仅是文本。`ls -l` 类似于 `Get-ChildItem \| Format-Table`                                   |
|             | `cd`              | `Set-Location`                       | `cd`, `sl`            | 功能几乎完全相同。`cd ~` 对应 `Set-Location ~`                                                                    |
|             | `pwd`             | `Get-Location`                       | `pwd`, `gl`           | 返回一个包含路径信息的对象。                                                                                         |
|             | `cp`              | `Copy-Item`                          | `cp`, `copy`, `cpi`   | `cp -r sourcedir destdir` 对应 `Copy-Item -Path sourcedir -Destination destdir -Recurse`                 |
|             | `mv`              | `Move-Item`                          | `mv`, `move`, `mi`    | 用法与 `Copy-Item` 类似。                                                                                    |
|             | `rm`              | `Remove-Item`                        | `rm`, `del`, `ri`     | `rm -rf dirname` 对应 `Remove-Item -Path dirname -Recurse -Force`。**务必小心！**                              |
|             | `mkdir`           | `New-Item -ItemType Directory`       | `mkdir`, `md`         | `mkdir -p a/b/c` 对应 `New-Item -Path "a/b/c" -ItemType Directory -Force`                                |
|             | `touch`           | `New-Item -ItemType File`            | `ni`                  | `New-Item my_file.txt -ItemType File`。或用重定向 `"" > my_file.txt`                                         |
|             | `cat`             | `Get-Content`                        | `cat`, `gc`, `type`   | `Get-Content` 可以读取文件内容为字符串数组。                                                                          |
| **文本处理**    | `grep`            | `Select-String`                      | `sls`                 | 功能非常强大，返回匹配对象（包含行号、文件名等）。`grep -v` 对应 `Select-String -NotMatch`。                                       |
|             | `head`            | `Select-Object -First <N>`           | `select`              | `Get-Content file.txt \| Select-Object -First 10`                                                      |
|             | `tail`            | `Select-Object -Last <N>`            | `select`              | `Get-Content file.txt \| Select-Object -Last 10`。`tail -f` 对应 `Get-Content file.txt -Wait`             |
|             | `sort`            | `Sort-Object`                        | `sort`                | 可以按对象属性排序。`ls \| Sort-Object -Property Length -Descending`                                             |
|             | `uniq`            | `Get-Unique` / `Sort-Object -Unique` | `gu`                  | `Get-Unique` 处理已排序的文本行。`Sort-Object -Unique` 更常用，集排序和去重于一体。                                            |
|             | `wc -l`           | `Measure-Object -Line`               | `measure`             | `Get-Content file.txt \| Measure-Object -Line`。更简单的方式是 `(Get-Content file.txt).Count`                  |
|             | `sed` / `awk`     | `ForEach-Object`, `-replace` 操作符     | `%`, `foreach`        | **无直接对应**。PowerShell 通过对象操作实现。例如 `sed 's/foo/bar/g'` 对应 `(Get-Content file.txt) -replace 'foo', 'bar'` |
| **进程管理**    | `ps`              | `Get-Process`                        | `ps`, `gps`           | 返回进程对象，可以轻松地排序、筛选和操作。`ps aux` 类似于 `Get-Process`                                                        |
|             | `kill`            | `Stop-Process`                       | `kill`, `spps`        | `Stop-Process -Name "notepad"` 或 `Stop-Process -Id 1234`                                               |
|             | `top`             | (无内置)                                |                       | `Get-Process \| Sort-Object -Descending CPU \| Select-Object -First 20` 可以实现一个静态快照。                    |
| **网络**      | `curl`            | `Invoke-WebRequest`                  | `curl`, `iwr`, `wget` | `iwr` 返回一个包含状态码、头、内容的丰富对象。                                                                             |
|             | `wget`            | `Invoke-WebRequest -OutFile <file>`  | `curl`, `iwr`, `wget` | `wget http://.../file.zip` 对应 `Invoke-WebRequest http://.../file.zip -OutFile file.zip`                |
|             |                   | `Invoke-RestMethod`                  | `irm`                 | 专门用于API调用，会自动将JSON/XML响应转换为PowerShell对象。                                                               |
|             | `ping`            | `Test-Connection`                    | `ping`, `tcm`         | 返回包含延迟、TTL等信息的对象。                                                                                      |
|             | `ifconfig` / `ip` | `Get-NetIPAddress`, `Get-NetAdapter` |                       | `Get-NetAdapter` 查看网络适配器，`Get-NetIPAddress` 查看IP地址。                                                    |
| **系统 & 权限** | `echo`            | `Write-Output`                       | `echo`, `write`       | `Write-Output` 是将对象写入管道的标准方式。`Write-Host` 则是直接打印到控制台，一般仅用于脚本的用户交互提示。                                   |
|             | `export`          | `$env:VAR="value"`                   |                       | `export FOO=bar` 对应 `$env:FOO = "bar"`。这会设置当前会话的环境变量。                                                  |
|             | `man`             | `Get-Help`                           | `help`, `man`         | `man ls` 对应 `Get-Help Get-ChildItem`。使用 `Get-Help Get-ChildItem -Examples` 或 `-Full` 获取更详细信息。          |
|             | `chmod` / `chown` | `Get-Acl`, `Set-Acl`                 |                       | **复杂得多**。因为Windows使用访问控制列表(ACL)而不是简单的`rwx`权限。这是一个较大的话题。                                                |
|             | `which`           | `Get-Command`                        | `gcm`                 | `gcm ls` 会告诉你 `ls` 是 `Get-ChildItem` 的一个别名。                                                            |

### 示例对比：查找并终止所有名为 "chrome" 的进程

**Bash:**

```bash
# 获取PID，然后用xargs传给kill
ps aux | grep chrome | awk '{print $2}' | xargs kill
```

这个过程涉及多个文本处理工具 (`grep` 筛选, `awk` 提取第二列)。

**PowerShell:**

```powershell
# 获取进程对象，然后直接通过管道传给 Stop-Process
Get-Process -Name "chrome" | Stop-Process
```

或者更简洁地：

```powershell
# 使用别名
ps chrome | kill
```

这里的 `ps chrome` 获取的是所有名为 "chrome" 的**进程对象**，这些对象被直接传递给 `kill` (Stop-Process) cmdlet，`Stop-Process` 知道如何从这些对象中提取所需信息（如PID）来终止进程。这就是对象管道的威力。

### 给初学者的建议

1.  **善用别名**：开始时，你可以继续使用 `ls`, `cat`, `ps`, `kill` 等熟悉的别名来降低学习曲线。
2.  **拥抱 `Get-Help`**：PowerShell 的帮助系统非常完善。遇到不熟悉的命令，立即 `Get-Help <command-name> -Full`。
3.  **使用 `Get-Member`**：想知道一个命令返回的对象有什么属性和方法？把它通过管道传给 `Get-Member` (别名 `gm`)。例如：`Get-Process | Get-Member`。
4.  **逐步转变思维**：慢慢地从“我如何用文本处理实现这个功能？”转变为“我有什么对象？我能对这个对象做什么？”。

希望这个列表能帮助你顺利地从 Bash 过渡到 PowerShell！