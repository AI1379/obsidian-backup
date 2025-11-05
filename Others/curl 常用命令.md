---
tags:
  - Linux
  - 运维
---
> 使用 `Gemini-2.5-flash` 生成

`curl` 和 `wget` 是在命令行中用于下载文件或与Web服务器进行交互的两个非常强大的工具。它们在功能上有一些重叠，但各有侧重。

### `curl` 常用命令

`curl` (Client URL) 主要用于传输数据，支持多种协议（HTTP, HTTPS, FTP, FTPS, SCP, SFTP, TFTP, DICT, TELNET, LDAP, FILE, IMAP, POP3, SMTP, RTSP, RTMP等）。它更像一个“瑞士军刀”，功能非常丰富，常用于API测试、发送各种HTTP请求等。

1.  **基本下载/查看网页内容:**

    ```bash
    curl <URL>
    # 例如：
    curl https://www.example.com
    ```

    这会将URL内容输出到标准输出（终端）。

2.  **将内容保存到文件:**
    *   使用 `-o` (小写o) 指定输出文件名：

        ```bash
        curl -o local_filename.html <URL>
        # 例如：
        curl -o index.html https://www.example.com
        ```

    *   使用 `-O` (大写O) 使用远程文件名保存：

        ```bash
        curl -O <URL_to_file>
        # 例如：
        curl -O https://www.example.com/some_file.zip
        ```

3.  **发送POST请求 (提交表单数据):**

    ```bash
    curl -X POST -d "param1=value1&param2=value2" <URL>
    # 或省略 -X POST，因为 -d 默认就是POST
    curl -d "username=user&password=pass" https://www.example.com/login
    ```

4.  **发送JSON格式的POST请求:**

    ```bash
    curl -X POST -H "Content-Type: application/json" -d '{"key":"value", "number":123}' <URL>
    # 例如：
    curl -X POST -H "Content-Type: application/json" -d '{"name":"John Doe", "age":30}' https://api.example.com/users
    ```

5.  **跟随重定向 (Location):**

    ```bash
    curl -L <URL>
    # 例如，访问一个短链接或被重定向的页面
    curl -L http://bit.ly/example
    ```

6.  **只获取HTTP头信息:**

    ```bash
    curl -I <URL>
    # 例如：
    curl -I https://www.example.com
    ```

    这会显示HTTP状态码、Content-Type、Server等信息。

7.  **显示详细的传输过程 (Verbose):**

    ```bash
    curl -v <URL>
    # 例如：
    curl -v https://www.example.com
    ```

    非常适合调试，可以看到请求和响应的完整HTTP头。

8.  **携带自定义HTTP头:**

    ```bash
    curl -H "User-Agent: MyCustomAgent/1.0" -H "Accept-Language: zh-CN" <URL>
    # 例如：
    curl -H "Authorization: Bearer my_token" https://api.example.com/data
    ```

9.  **进行HTTP认证 (Basic Authentication):**

    ```bash
    curl -u username:password <URL>
    # 例如：
    curl -u admin:mypassword http://example.com/admin
    ```

10. **设置Cookie:**

    ```bash
    curl -b "name=value; session=abc" <URL>
    # 从文件中读取Cookie
    curl -b cookiejar.txt <URL>
    # 将收到的Cookie保存到文件
    curl -c cookiejar.txt <URL>
    ```

### `wget` 常用命令

`wget` (Web Get) 主要用于从Web服务器下载文件，它以其强大的递归下载、断点续传和在后台下载的能力而闻名。它更适合批量下载、镜像网站等场景。

1.  **基本下载文件:**

    ```bash
    wget <URL>
    # 例如：
    wget https://www.example.com/file.zip
    ```

    文件会被保存到当前目录，文件名通常与远程文件名相同。

2.  **将文件保存为指定文件名:**

    ```bash
    wget -O new_filename.html <URL>
    # 例如：
    wget -O my_homepage.html https://www.example.com
    ```

    **注意:** `wget` 的 `-O` 是大写，功能类似 `curl` 的 `-o`。

3.  **断点续传 (继续下载):**

    ```bash
    wget -c <URL>
    # 例如，如果下载中断，再次运行此命令会从中断处继续
    wget -c https://www.example.com/large_file.iso
    ```

4.  **后台下载:**

    ```bash
    wget -b <URL>
    # 例如：
    wget -b https://www.example.com/very_large_file.zip
    ```

    下载会在后台进行，并将输出记录到 `wget-log` 文件中。

5.  **递归下载 (镜像网站):**

    ```bash
    wget -r <URL>
    # 或结合其他选项，例如：
    wget -r -np -P /tmp/mirror https://www.example.com/some_path/
    # -r: 递归下载
    # -np: 不遍历父目录 (No Parent)
    # -P: 指定下载目录前缀
    # --level=N: 指定递归深度
    ```

    这会下载指定URL及其链接到的所有页面/文件（在同一域名下，或根据配置）。

6.  **限制下载速度:**

    ```bash
    wget --limit-rate=100k <URL>
    # 例如：限制到100KB/s
    wget --limit-rate=5m https://www.example.com/big_file.rar
    ```

7.  **忽略SSL证书错误:**

    ```bash
    wget --no-check-certificate <URL>
    # 当遇到自签名证书或证书验证问题时使用
    wget --no-check-certificate https://badssl.com/expired
    ```

8.  **设置用户代理 (User-Agent):**

    ```bash
    wget -U "MyCustomBrowser/1.0" <URL>
    # 例如，伪装成浏览器
    wget -U "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36" https://www.example.com
    ```

9.  **下载整个网站 (更严格的镜像):**

    ```bash
    wget --mirror -p --convert-links -P /tmp/website_backup <URL>
    # --mirror: 开启递归下载，时间戳，不限深度，接受FTP/HTTP等
    # -p: 下载所有必需的文件（图片、CSS等）
    # --convert-links: 下载后转换链接，使其在本地可用
    # -P: 保存到指定目录
    ```

**总结:**

*   **`curl`**: 功能更全面，支持更多协议，常用于API测试、HTTP请求调试、发送复杂请求等场景。默认输出到标准输出，需要 `-o` 或 `-O` 保存文件。
*   **`wget`**: 更专注于文件下载，特别是大文件、断点续传、后台下载和递归下载（镜像网站）等场景。默认直接保存文件到当前目录。