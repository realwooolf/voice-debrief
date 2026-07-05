---
name: voice-debrief
description: 面试/工作会议录音转译复盘助手，处理面试或会议录音、转录文本，支持保存到飞书文档并自动归档到对应的多维表格（面试记录表 / 会议记录表，两者独立不混存）。触发场景：用户提到"面试复盘"、"会议纪要"、"整理这场会议"、上传音频文件（.wav/.m4a/.mp3/.mp4）需要转录、上传 PDF 转录文件需要整理。
---

# 录音复盘（面试 / 会议）

根据传入的文件类型自动判断执行阶段：
- 传入音频文件（.wav / .m4a / .mp3）或视频文件（.mp4）→ 执行阶段一：本地转录
- 传入 PDF 文件（已有转录文本）→ 执行阶段二：整理并输出结果

> **注意：** Claude Code 不支持直接上传音频/视频文件。请粘贴文件路径，或截图文件所在位置（如 Finder 窗口底部会显示完整路径），例如：
> `帮我整理这场面试 /Users/xxx/Desktop/某公司二面.mp3`

**多文件处理：** 如果用户传入多个音频/视频文件，在阶段一完成后自动执行阶段零（多文件合并排序），再进入阶段二。单文件则跳过合并直接处理。

---

## 前置依赖

使用前确保以下工具已安装：

```bash
# 转录（二选一，按平台选）
pip install mlx-whisper          # Apple Silicon Mac 用，GPU 加速，推荐
pip install faster-whisper        # 其他平台 fallback，比 openai-whisper 快 2-4 倍

# 音视频处理（whisper 内部依赖 ffmpeg 解码音频）
# macOS
brew install ffmpeg
# Ubuntu / Debian
sudo apt install ffmpeg
# Windows
# 前往 https://ffmpeg.org/download.html 下载，或用 winget install ffmpeg
```

**平台说明：**
- Apple Silicon Mac（M1/M2/M3/M4）→ 使用 mlx-whisper，速度快 5-10 倍
- Linux / Windows + NVIDIA GPU → faster-whisper + CUDA，自动检测
- Intel Mac / 无显卡 Linux / Windows → faster-whisper + CPU int8，比 openai-whisper 快 2-4 倍

---

## 飞书配置

若选择保存到飞书，需要先完成以下一次性配置。选本地保存可跳过此章节。

### 第一步：创建飞书自建应用

1. 打开 [open.feishu.cn](https://open.feishu.cn)，登录你的飞书账号
2. 点击右上角「开发者后台」→「创建应用」→ 选「自建应用」
3. 填写应用名称（随意，如 `interview-debrief`）和描述，创建完成

### 第二步：开启权限

1. 进入应用后，左侧菜单点「权限管理」
2. 点击「**批量导入/导出权限**」→「导入」，将以下 JSON 粘贴进去，点「下一步，确认新增权限」：
   ```json
   {
     "scopes": {
       "user": [
         "docx:document",
         "bitable:app",
         "docs:doc",
         "docs:doc:readonly",
         "docs:document:import",
         "drive:drive",
         "drive:file",
         "wiki:wiki"
       ]
     }
   }
   ```
3. 确认后权限全部生效（测试企业环境下免审核）

### 第三步：添加回调地址

1. 左侧菜单点「安全设置」
2. 找到「重定向 URL」，点击添加，填入：
   ```
   http://localhost:9998/callback
   ```
3. 保存

### 第四步：发布应用

1. 左侧菜单点「版本管理与发布」→「创建版本」
2. 版本号填 `1.0.0`，更新说明随意填
3. 可用范围保持默认（部分成员，仅自己可见）
4. 点「保存」，飞书个人版免审核，保存后立即生效

### 第五步：获取凭证并创建配置文件

1. 左侧菜单点「凭证与基础信息」，复制 **App ID** 和 **App Secret**

2. 在终端运行以下命令创建配置文件（替换为你的实际值）：

   ```bash
   cat > ~/.claude/feishu_config.json << 'EOF'
   {
     "app_id": "你的 App ID",
     "app_secret": "你的 App Secret",
     "interview_bitable_app_token": "",
     "interview_bitable_table_id": "",
     "meeting_bitable_app_token": "",
     "meeting_bitable_table_id": ""
   }
   EOF
   ```

3. 多维表格配置（面试记录表、会议记录表各自独立，二选一）：
   - **没有表格**：对应两个字段留空，第一次运行时 skill 会按类型自动创建对应的表并写入
   - **已有表格**：打开飞书多维表格，URL 格式为 `https://xxx.feishu.cn/base/ClG9xxxxxxx?table=tblxxxxxxx`，`ClG9xxxxxxx` 填入对应的 `_bitable_app_token`，`tblxxxxxxx` 填入对应的 `_bitable_table_id`

   面试应用（app_id/app_secret）和授权是共用的，只有多维表格是分开的两张表，互不影响。

配置完成后，第一次使用时会自动打开浏览器完成飞书 OAuth 授权，之后 token 自动缓存，无需重复操作。

---

## 阶段一：音频/视频处理

**触发条件：** 文件扩展名为 .wav、.m4a、.mp3、.mp4

### 检测已有转录文件

收到音频/视频文件路径后，**先检查同目录下是否存在同名 `.txt` 文件**（如 `interview.m4a` → 检查 `interview.txt`）：
- **存在**：直接读取 `.txt` 内容，跳过转录，进入阶段二信息收集
- **不存在**：启动转录（见下方）

### 转录

先检测平台，选择对应的转录方式。**使用 `run_in_background=True` 运行转录脚本**，转录在后台进行，Claude 立即在对话框中询问阶段二所需的信息（保存方式、日期、公司名等）。收到通知转录完成后，直接用已收集的信息进入阶段零（多文件）或阶段二（单文件）。

```python
import platform, subprocess, os

def is_apple_silicon():
    return platform.system() == "Darwin" and platform.machine() == "arm64"

def transcribe(files):
    for f in files:
        out = f.rsplit('.', 1)[0] + '.txt'
        if is_apple_silicon():
            # mlx-whisper：Apple Silicon GPU 加速，41 分钟音频约 3-5 分钟
            import mlx_whisper
            result = mlx_whisper.transcribe(
                f, path_or_hf_repo="mlx-community/whisper-large-v3-turbo", language="zh",
                initial_prompt="以下是普通话的转录内容："
            )
        else:
            # faster-whisper：CPU int8 量化（2-4x vs openai-whisper），有 NVIDIA GPU 自动用 CUDA
            from faster_whisper import WhisperModel
            import torch
            device = "cuda" if torch.cuda.is_available() else "cpu"
            compute_type = "float16" if device == "cuda" else "int8"
            model = WhisperModel("small", device=device, compute_type=compute_type)
            segments, _ = model.transcribe(f, language="zh", initial_prompt="以下是普通话的转录内容：")
            result = {"segments": [{"text": s.text} for s in segments]}
        open(out, 'w', encoding='utf-8').write('\n'.join(s['text'].strip() for s in result['segments']))
        print('完成:', out)
```

注意：
- mlx-whisper 首次运行会自动下载模型（约 800MB），之后缓存本地
- faster-whisper 首次运行会自动下载模型（small 约 244MB），之后缓存本地
- 转录脚本必须用 `run_in_background=True` 启动，否则 Claude 会阻塞无法收集信息

---

## 阶段零：多文件合并排序（自动触发）

**触发条件：** 阶段一产出 2 个及以上 .txt 转录文件时自动执行，无需用户操作。

执行步骤：

1. 读取所有 .txt 文件内容
2. 根据文件名时间戳 + 内容语义判断顺序：
   - 文件名含时间戳（如 1733、215639）→ 按时间升序排列
   - 内容含开场白（自我介绍、你好）→ 排最前
   - 内容含道别（拜拜、谢谢）→ 排最后
   - 内容有明显上下文衔接 → 按语义连贯性判断
3. 将排序结果告知用户确认（列出文件顺序和判断依据），确认后进入阶段二
4. 若顺序不确定，询问用户

---

## 阶段二：整理 → 输出结果

**触发条件：** 文件扩展名为 .pdf，或阶段一/阶段零完成后的转录文本

启动前分两步收集信息：

**第一问（单独问，合并成一条消息）：**
- 保存方式：本地文件 还是 飞书文档？
- 类型：面试 / 工作会议 / 其他（不确定就默认面试；内容明显是会议讨论则默认会议）

收到回答后，若选飞书，检查 `~/.claude/feishu_config.json` 是否存在且 `app_id` / `app_secret` 均为非空非占位符（不等于「你的 App ID」）。若未配置或配置不完整，先引导完成飞书配置（见「飞书配置」章节），配置完成后再继续。

**第二问（按类型分别收集，合并成一条消息）：**

**面试：**
- 日期（格式：YYYY-MM-DD）
- 公司名、岗位名、面试轮次（如一面、二面、终面）
- 面试官姓名（选填）、面试时长（选填，如 45min）

**工作会议：**
- 日期（格式：YYYY-MM-DD）
- 会议主题、项目/类别（用于多维表格筛选，如所属项目名）
- 参会人（选填）、会议时长（选填，如 45min）

**其他：**
- 日期
- 对方身份、主题
（跳过多维表格更新）

根据类型生成对应的文档标题和信息头：

**面试：**
```
{公司名} 完整对话
{公司名} 问答整理
{公司名} 面试总评

信息头：公司 / 岗位 / 轮次 / 日期
```

**工作会议：**
```
{会议主题} 完整对话
{会议主题} 会议纪要（内含：要点整理 / 结论 / 待办事项）

信息头：会议主题 / 项目类别 / 参会人 / 日期
```

**其他：**
```
{对方身份} 完整对话
{对方身份} 问答整理
{对方身份} 总评

信息头：对方身份 / 主题 / 日期
```

### 1. 本地保存（选本地时执行）

将整理结果写入录音文件同目录下的 .md 文件。面试/其他类型是三个文件（完整对话/问答整理/总评），工作会议是两个文件（完整对话/会议纪要）：

```python
import os

def save_local(source_path, title, parts):
    # source_path 可以是音频文件或 PDF 文件路径
    # parts: [(后缀, 内容), ...]，面试/其他传三项，会议传两项（完整对话、会议纪要）
    base_dir = os.path.dirname(os.path.abspath(source_path))
    for suffix, content in parts:
        out = os.path.join(base_dir, f"{title}-{suffix}.md")
        with open(out, "w", encoding="utf-8") as f:
            f.write(content)
        print(f"已保存：{out}")
```

保存完成后，将总评/会议纪要内容直接输出在对话框中。后续飞书相关步骤全部跳过。

---

### 2. 获取飞书 user_access_token（选飞书时执行）

从配置中读取 APP_ID / APP_SECRET，TOKEN_CACHE 路径做 `~` 展开：

```python
import json, time, os, subprocess, tempfile
import http.server, threading, webbrowser, urllib.parse

# 从本地配置文件读取（~/.claude/feishu_config.json），不写入 skill 避免泄露
_cfg = json.load(open(os.path.expanduser("~/.claude/feishu_config.json")))
APP_ID = _cfg["app_id"]
APP_SECRET = _cfg["app_secret"]
TOKEN_CACHE = os.path.expanduser("~/.claude/feishu_token.json")
REDIRECT_URI = "http://localhost:9998/callback"

def load_token():
    try:
        with open(TOKEN_CACHE) as f:
            return json.load(f)
    except:
        return None

def save_token(data):
    os.makedirs(os.path.dirname(TOKEN_CACHE), exist_ok=True)
    with open(TOKEN_CACHE, "w") as f:
        json.dump(data, f)

def curl_post_auth(url, data, headers=None):
    payload = json.dumps(data, ensure_ascii=False)
    with tempfile.NamedTemporaryFile(mode='w', suffix='.json', delete=False, encoding='utf-8') as f:
        f.write(payload)
        fname = f.name
    cmd = ["curl", "-sk", "-X", "POST", url, "--noproxy", "*",
           "-H", "Content-Type: application/json"]
    if headers:
        for h in headers:
            cmd += ["-H", h]
    cmd += ["--data", f"@{fname}"]
    try:
        result = subprocess.run(cmd, capture_output=True, text=True, timeout=30)
    finally:
        os.unlink(fname)
    if not result.stdout.strip():
        return {}
    return json.loads(result.stdout)

def get_app_token():
    d = curl_post_auth(
        "https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal",
        {"app_id": APP_ID, "app_secret": APP_SECRET})
    return d.get("tenant_access_token")

def oauth_flow():
    # 先释放端口，防止上次未完成进程占用
    subprocess.run("lsof -ti:9998 | xargs kill -9 2>/dev/null", shell=True)
    app_token = get_app_token()
    code_holder = {}
    class Handler(http.server.BaseHTTPRequestHandler):
        def do_GET(self):
            parsed = urllib.parse.urlparse(self.path)
            params = urllib.parse.parse_qs(parsed.query)
            code_holder["code"] = params.get("code", [None])[0]
            self.send_response(200)
            self.end_headers()
            self.wfile.write(b"<html><body>Done! You can close this tab.</body></html>")
        def log_message(self, *args): pass
    server = http.server.HTTPServer(("localhost", 9998), Handler)
    t = threading.Thread(target=server.handle_request)
    t.start()
    auth_url = (f"https://open.feishu.cn/open-apis/authen/v1/authorize"
                f"?app_id={APP_ID}&redirect_uri={urllib.parse.quote(REDIRECT_URI)}"
                f"&scope=docx%3Adocument%20bitable%3Aapp%20docs%3Adoc%20docs%3Adoc%3Areadonly"
                f"%20docs%3Adocument%3Aimport%20drive%3Adrive%20drive%3Afile%20wiki%3Awiki"
                f"&response_type=code")
    print("正在打开飞书授权页面...")
    webbrowser.open(auth_url)
    t.join(timeout=120)
    code = code_holder.get("code")
    if not code:
        raise Exception("未获取到授权码，请重试")
    d = curl_post_auth(
        "https://open.feishu.cn/open-apis/authen/v1/access_token",
        {"grant_type": "authorization_code", "code": code},
        headers=[f"Authorization: Bearer {app_token}"])
    d = d.get("data", {})
    if not d.get("access_token"):
        raise Exception(f"获取 access_token 失败，请检查飞书应用配置：{d}")
    token_data = {
        "access_token": d["access_token"],
        "refresh_token": d.get("refresh_token", ""),
        "expires_at": int(time.time()) + d.get("expires_in", 7200) - 300
    }
    save_token(token_data)
    return token_data["access_token"]

def get_token():
    cached = load_token()
    if cached and time.time() < cached.get("expires_at", 0):
        return cached["access_token"]
    return oauth_flow()
```

首次运行会打开浏览器授权，之后 token 自动缓存复用，无需重复操作。

> **关键：** 获取 token 后，立即在同一脚本中继续执行后续所有操作（创建文档、写 bitable），不要中断等待用户确认，否则 token 会在等待中过期。

> **端口冲突处理：** 启动 OAuth 服务器前先释放端口，避免上次未完成的进程占用：
> ```python
> import subprocess
> subprocess.run("lsof -ti:9998 | xargs kill -9 2>/dev/null", shell=True)
> ```

### 3. 读取 PDF

用 Read 工具读取 PDF 全文。

### 4. 识别发言人

**面试/其他：** 根据对话内容判断：
- 做自我介绍、介绍项目、回答问题的是**我**
- 提问、追问、给反馈的是**面试官**

**工作会议：** 按参会人姓名/角色区分发言人（如有多人），无法确定具体姓名时用「发言人A」「发言人B」等标注，保持前后一致。

### 5. 生成飞书文档（面试/其他为三个文档，会议为两个文档）

**重要注意事项（调试确认）：**
- 用 subprocess curl 调用飞书 API，绕过 Python SSL 栈（Python 3.14 + 本地代理会导致 SSL handshake 失败）
- 不要使用 `text_color` 参数（会报 99992402 field validation failed）
- 批量写入每批不超过 20 块，超过容易超时
- token 获取方式：先用 app_id/app_secret 换 tenant_access_token，再用它换 user_access_token

```python
import json, time, subprocess, tempfile, os

def curl_post(url, data, token):
    payload = json.dumps(data, ensure_ascii=False)
    with tempfile.NamedTemporaryFile(mode='w', suffix='.json', delete=False, encoding='utf-8') as f:
        f.write(payload)
        fname = f.name
    try:
        result = subprocess.run(
            ["curl", "-sk", "-X", "POST", url, "--noproxy", "*",
             "-H", "Content-Type: application/json",
             "-H", f"Authorization: Bearer {token}",
             "--data", f"@{fname}"],
            capture_output=True, text=True, timeout=60)
    finally:
        os.unlink(fname)
    if not result.stdout.strip():
        return {}
    return json.loads(result.stdout)

def create_doc(token, title):
    d = curl_post("https://open.feishu.cn/open-apis/docx/v1/documents", {"title": title}, token)
    if d.get("code") != 0:
        print(f"create_doc failed: {d}")
        return None
    return d["data"]["document"]["document_id"]

def write_blocks(token, doc_id, blocks):
    for i in range(0, len(blocks), 20):
        batch = blocks[i:i+20]
        for attempt in range(3):
            r = curl_post(
                f"https://open.feishu.cn/open-apis/docx/v1/documents/{doc_id}/blocks/{doc_id}/children",
                {"children": batch, "index": -1}, token)
            if r.get("code") == 0:
                break
            print(f"  batch {i} attempt {attempt+1} failed: {r.get('code')}, retrying...")
            time.sleep(1.5)
        time.sleep(0.5)

def p(text):
    return {"block_type": 2, "text": {"elements": [{"text_run": {"content": text}}], "style": {}}}

def bold(text):
    return {"block_type": 2, "text": {"elements": [{"text_run": {"content": text, "text_element_style": {"bold": True}}}], "style": {}}}

def mixed_p(parts):
    # parts = [("文字", is_bold), ...]，用于发言人加粗+正文不加粗的混排段落
    elements = []
    for text, is_bold in parts:
        el = {"text_run": {"content": text}}
        if is_bold:
            el["text_run"]["text_element_style"] = {"bold": True}
        elements.append(el)
    return {"block_type": 2, "text": {"elements": elements, "style": {}}}

def h1(text):
    return {"block_type": 3, "heading1": {"elements": [{"text_run": {"content": text}}], "style": {}}}

def h2(text):
    return {"block_type": 4, "heading2": {"elements": [{"text_run": {"content": text}}], "style": {}}}

def divider():
    return {"block_type": 22, "divider": {}}
```

#### 每个文档开头写入基础信息块

**面试/其他：**
```
公司：{公司名}
岗位：{岗位名}
轮次：{面试轮次}
日期：{面试日期}
────────────────────────
```

**工作会议：**
```
会议主题：{会议主题}
项目/类别：{项目类别}
参会人：{参会人}
日期：{会议日期}
────────────────────────
```

---

#### 文档一：完整对话

- 全程对话按时间顺序，发言人加粗（面试官：/我：，或会议参会人姓名/角色）
- 保留完整内容，不删减

---

#### 面试/其他 — 文档二：问答整理

- 每组问答独立分块，加二级标题（Q1、Q2……）
- 标题概括该组问答的核心话题
- 包含候选人的反问部分
- 每组问答之间空一行

#### 面试/其他 — 文档三：面试总评

根据对话内容灵活调整结构，基础框架：

**整体印象**（一句话定调）

**做得好的地方**（具体举例，引用原文）

**明显问题**（具体举例，说明为什么是问题）

**优化建议**（可操作，针对这场面试的具体情况）

如有特殊情况（面试官明确反馈、压力性问题、明显失误等），单独增加板块说明。

---

#### 工作会议 — 文档二：会议纪要

单个文档内依次包含三个板块（二级标题分隔）：

**要点整理**：按讨论议题分块，每个议题加二级标题概括主题，正文提炼讨论要点（不是逐字稿）

**结论 / 决策**：本次会议达成的结论、拍板事项，逐条列出

**待办事项**：逐条列出，格式为「事项 —— 负责人（如有提及）—— 截止时间（如有提及）」，未提及负责人或截止时间的项就只写事项本身，不编造

### 6. 总评/纪要同时在对话框内输出

文档创建完毕后，**必须**将面试总评（面试/其他）或会议纪要（工作会议）内容完整输出在对话框中，不能省略。这一步是强制的，不能跳过。

### 7. 更新多维表格（面试类型 → 面试记录表；工作会议类型 → monee会议记录表；其他类型跳过）

文档创建完成后，自动更新 `~/.claude/feishu_config.json` 中配置的对应多维表格。面试和会议是两张完全独立的表，互不影响，各自有各自的 `_bitable_app_token` / `_bitable_table_id`。

#### 面试类型

**首先检查并自动创建多维表格（若未配置）：**

```python
def create_bitable_if_needed(token):
    cfg_path = os.path.expanduser("~/.claude/feishu_config.json")
    cfg = json.load(open(cfg_path))
    if cfg.get("interview_bitable_app_token") and cfg.get("interview_bitable_table_id"):
        return  # 已配置，跳过

    print("未检测到多维表格，正在自动创建面试记录表...")

    # 创建多维表格应用
    r = curl_post("https://open.feishu.cn/open-apis/bitable/v1/apps",
                  {"name": "面试记录"}, token)
    if r.get("code") != 0:
        print(f"创建多维表格失败: {r}")
        return
    app_token = r["data"]["app"]["app_token"]

    # 获取默认表格 ID
    r2 = subprocess.run(
        ["curl", "-sk", f"https://open.feishu.cn/open-apis/bitable/v1/apps/{app_token}/tables",
         "--noproxy", "*", "-H", f"Authorization: Bearer {token}"],
        capture_output=True, text=True, timeout=30)
    table_id = json.loads(r2.stdout)["data"]["items"][0]["table_id"]

    fields_url = f"https://open.feishu.cn/open-apis/bitable/v1/apps/{app_token}/tables/{table_id}/fields"
    records_url = f"https://open.feishu.cn/open-apis/bitable/v1/apps/{app_token}/tables/{table_id}/records"

    # 删除默认空行（飞书新建表格自带 10 条空记录，逐条删除，batch_delete 会报 RecordIdNotFound）
    recs_r = subprocess.run(
        ["curl", "-sk", f"{records_url}?page_size=100", "--noproxy", "*",
         "-H", f"Authorization: Bearer {token}"],
        capture_output=True, text=True, timeout=30)
    for rec in json.loads(recs_r.stdout).get("data", {}).get("items", []):
        if not rec.get("fields"):
            subprocess.run(
                ["curl", "-sk", "-X", "DELETE", f"{records_url}/{rec['record_id']}",
                 "--noproxy", "*", "-H", f"Authorization: Bearer {token}"],
                capture_output=True, text=True, timeout=15)

    # 删除默认多余列（单选、日期、附件），保留主字段
    fields_r = subprocess.run(
        ["curl", "-sk", f"{fields_url}?page_size=100", "--noproxy", "*",
         "-H", f"Authorization: Bearer {token}"],
        capture_output=True, text=True, timeout=30)
    keep_names = {"岗位","面试轮次","面试时间","面试时长","面试官","面试整理版","面试问答整理","面试总评","备注"}
    for f in json.loads(fields_r.stdout).get("data", {}).get("items", []):
        if not f.get("is_primary") and f["field_name"] not in keep_names:
            subprocess.run(
                ["curl", "-sk", "-X", "DELETE", f"{fields_url}/{f['field_id']}",
                 "--noproxy", "*", "-H", f"Authorization: Bearer {token}"],
                capture_output=True, text=True, timeout=15)

    # 主字段改名为「公司」（主字段不可删，直接 PUT 改名）
    fields_r2 = subprocess.run(
        ["curl", "-sk", f"{fields_url}?page_size=100", "--noproxy", "*",
         "-H", f"Authorization: Bearer {token}"],
        capture_output=True, text=True, timeout=30)
    for f in json.loads(fields_r2.stdout).get("data", {}).get("items", []):
        if f.get("is_primary"):
            import tempfile as _tf
            _payload = json.dumps({"field_name": "公司", "type": 1}, ensure_ascii=False)
            with _tf.NamedTemporaryFile(mode='w', suffix='.json', delete=False, encoding='utf-8') as _ff:
                _ff.write(_payload); _ffname = _ff.name
            subprocess.run(
                ["curl", "-sk", "-X", "PUT", f"{fields_url}/{f['field_id']}", "--noproxy", "*",
                 "-H", "Content-Type: application/json", "-H", f"Authorization: Bearer {token}",
                 "--data", f"@{_ffname}"],
                capture_output=True, text=True, timeout=15)
            import os as _os; _os.unlink(_ffname)
            break

    # 新建业务字段（不含「公司」，它已是主字段）
    for field in [
        {"field_name": "岗位",         "type": 1},
        {"field_name": "面试轮次",     "type": 1},
        {"field_name": "面试时间",     "type": 5},
        {"field_name": "面试时长",     "type": 1},
        {"field_name": "面试官",       "type": 1},
        {"field_name": "面试整理版",   "type": 15},
        {"field_name": "面试问答整理", "type": 15},
        {"field_name": "面试总评",     "type": 15},
        {"field_name": "备注",         "type": 1},
    ]:
        curl_post(fields_url, field, token)

    # 写回配置文件
    cfg["interview_bitable_app_token"] = app_token
    cfg["interview_bitable_table_id"] = table_id
    with open(cfg_path, "w", encoding="utf-8") as f:
        json.dump(cfg, f, ensure_ascii=False, indent=2)

    print(f"✅ 多维表格已创建：https://feishu.cn/base/{app_token}")
```

执行顺序：先调用 `create_bitable_if_needed(token)` 确保表格存在，再调用 `update_bitable` 写入记录。

**逻辑：**
1. 每次**永远新建**一条记录，不查找、不覆盖已有记录
2. 写入以下字段：公司（见命名规则）、岗位、面试轮次、面试时间、面试时长、面试官、三个文档链接
3. 面试官和面试时长为选填，用户未提供则跳过

**公司字段命名规则：**
- 新建前先查询表格，看是否已有同名公司的记录
- 如果**没有**已有记录：`公司` 字段直接写公司名（如「某公司」）
- 如果**已有 1 条**记录且该记录的公司字段等于纯公司名（说明之前只有一面）：
  - 先把那条老记录的 `公司` 字段更新为「公司名+一面」（如「某公司一面」）
  - 新记录的 `公司` 字段写「公司名+当前轮次」（如「某公司二面」、「某公司HR面」）
- 如果**已有 2 条及以上**记录：新记录直接写「公司名+当前轮次」，不动已有记录

轮次后缀直接用收集到的面试轮次字段（如「一面」「二面」「HR面」「终面」）。

```python
import json, subprocess, os, tempfile

def curl_put_bitable(url, data, token):
    payload = json.dumps(data, ensure_ascii=False)
    with tempfile.NamedTemporaryFile(mode='w', suffix='.json', delete=False, encoding='utf-8') as f:
        f.write(payload)
        fname = f.name
    try:
        result = subprocess.run(
            ["curl", "-sk", "-X", "PUT", url, "--noproxy", "*",
             "-H", "Content-Type: application/json",
             "-H", f"Authorization: Bearer {token}",
             "--data", f"@{fname}"],
            capture_output=True, text=True, timeout=30)
    finally:
        os.unlink(fname)
    if not result.stdout.strip():
        return {}
    decoder = json.JSONDecoder()
    obj, _ = decoder.raw_decode(result.stdout.strip())
    return obj

def curl_post_bitable(url, data, token):
    payload = json.dumps(data, ensure_ascii=False)
    with tempfile.NamedTemporaryFile(mode='w', suffix='.json', delete=False, encoding='utf-8') as f:
        f.write(payload)
        fname = f.name
    try:
        result = subprocess.run(
            ["curl", "-sk", "-X", "POST", url, "--noproxy", "*",
             "-H", "Content-Type: application/json",
             "-H", f"Authorization: Bearer {token}",
             "--data", f"@{fname}"],
            capture_output=True, text=True, timeout=30)
    finally:
        os.unlink(fname)
    if not result.stdout.strip():
        return {}
    decoder = json.JSONDecoder()
    obj, _ = decoder.raw_decode(result.stdout.strip())
    return obj

def update_bitable(token, company, round_name, date_str, position, interviewer, duration,
                   doc1_id, doc2_id, doc3_id):
    cfg = json.load(open(os.path.expanduser("~/.claude/feishu_config.json")))
    app_token = cfg.get("interview_bitable_app_token")
    table_id = cfg.get("interview_bitable_table_id")
    if not app_token or not table_id:
        print("bitable 未配置，跳过")
        return

    base_url = f"https://open.feishu.cn/open-apis/bitable/v1/apps/{app_token}/tables/{table_id}"

    # 查询该公司已有的所有记录
    records_r = subprocess.run(
        ["curl", "-sk", f"{base_url}/records?page_size=100", "--noproxy", "*",
         "-H", f"Authorization: Bearer {token}"],
        capture_output=True, text=True, timeout=30)
    records_data = json.loads(records_r.stdout)
    existing = [
        rec for rec in records_data.get("data", {}).get("items", [])
        if rec["fields"].get("公司", "").startswith(company)
    ]

    # 决定新记录的公司字段名
    if len(existing) == 0:
        # 第一条记录，直接写公司名
        company_label = company
    else:
        # 已有记录，新记录加轮次后缀
        company_label = f"{company}{round_name}"
        # 如果只有 1 条老记录且它的公司字段等于纯公司名，补上「一面」后缀
        if len(existing) == 1 and existing[0]["fields"].get("公司", "") == company:
            old_id = existing[0]["record_id"]
            curl_put_bitable(
                f"{base_url}/records/{old_id}",
                {"fields": {"公司": f"{company}一面"}},
                token)
            print(f"已将旧记录公司名更新为「{company}一面」")

    # 日期转 Unix 毫秒时间戳（飞书日期字段要求此格式，不能传字符串）
    import datetime
    y, m, d = map(int, date_str.split("-"))
    date_ts_ms = int(datetime.datetime(y, m, d).timestamp() * 1000)

    fields = {
        "公司": company_label,   # 主字段，直接按名称写入
        "岗位": position,
        "面试轮次": round_name,
        "面试时间": date_ts_ms,
        "面试整理版": {
            "text": f"{company_label}-面试整理版",
            "link": f"https://feishu.cn/docx/{doc1_id}"
        },
        "面试问答整理": {
            "text": f"{company_label}-面试问答整理",
            "link": f"https://feishu.cn/docx/{doc2_id}"
        },
        "面试总评": {
            "text": f"{company_label}-面试总评",
            "link": f"https://feishu.cn/docx/{doc3_id}"
        }
    }
    if interviewer:
        fields["面试官"] = interviewer
    if duration:
        fields["面试时长"] = duration

    r = curl_post_bitable(f"{base_url}/records", {"fields": fields}, token)
    print("bitable 新建：", "成功" if r.get("code") == 0 else r)
```

调用时传入收集好的信息：
```python
update_bitable(token, company=公司名, round_name=面试轮次,
               date_str=面试日期, position=岗位名,
               interviewer=面试官(可为None), duration=时长(可为None),
               doc1_id=doc1_id, doc2_id=doc2_id, doc3_id=doc3_id)
```

#### 工作会议类型

会议记录表和面试记录表是两张独立的多维表格，不共用、不合并。表名固定为「monee会议记录」。

**逻辑：** 每次会议永远新建一条记录，不做面试表那种同名合并/轮次后缀逻辑（会议主题本身通常每次不同，不需要合并）。

```python
def create_meeting_bitable_if_needed(token):
    cfg_path = os.path.expanduser("~/.claude/feishu_config.json")
    cfg = json.load(open(cfg_path))
    if cfg.get("meeting_bitable_app_token") and cfg.get("meeting_bitable_table_id"):
        return  # 已配置，跳过

    print("未检测到会议记录表，正在自动创建...")

    r = curl_post("https://open.feishu.cn/open-apis/bitable/v1/apps",
                  {"name": "monee会议记录"}, token)
    if r.get("code") != 0:
        print(f"创建多维表格失败: {r}")
        return
    app_token = r["data"]["app"]["app_token"]

    r2 = subprocess.run(
        ["curl", "-sk", f"https://open.feishu.cn/open-apis/bitable/v1/apps/{app_token}/tables",
         "--noproxy", "*", "-H", f"Authorization: Bearer {token}"],
        capture_output=True, text=True, timeout=30)
    table_id = json.loads(r2.stdout)["data"]["items"][0]["table_id"]

    fields_url = f"https://open.feishu.cn/open-apis/bitable/v1/apps/{app_token}/tables/{table_id}/fields"
    records_url = f"https://open.feishu.cn/open-apis/bitable/v1/apps/{app_token}/tables/{table_id}/records"

    # 删除默认空行
    recs_r = subprocess.run(
        ["curl", "-sk", f"{records_url}?page_size=100", "--noproxy", "*",
         "-H", f"Authorization: Bearer {token}"],
        capture_output=True, text=True, timeout=30)
    for rec in json.loads(recs_r.stdout).get("data", {}).get("items", []):
        if not rec.get("fields"):
            subprocess.run(
                ["curl", "-sk", "-X", "DELETE", f"{records_url}/{rec['record_id']}",
                 "--noproxy", "*", "-H", f"Authorization: Bearer {token}"],
                capture_output=True, text=True, timeout=15)

    # 删除默认多余列，保留主字段
    fields_r = subprocess.run(
        ["curl", "-sk", f"{fields_url}?page_size=100", "--noproxy", "*",
         "-H", f"Authorization: Bearer {token}"],
        capture_output=True, text=True, timeout=30)
    keep_names = {"项目类别","日期","时长","参会人","完整对话","会议纪要","备注"}
    for f in json.loads(fields_r.stdout).get("data", {}).get("items", []):
        if not f.get("is_primary") and f["field_name"] not in keep_names:
            subprocess.run(
                ["curl", "-sk", "-X", "DELETE", f"{fields_url}/{f['field_id']}",
                 "--noproxy", "*", "-H", f"Authorization: Bearer {token}"],
                capture_output=True, text=True, timeout=15)

    # 主字段改名为「会议主题」
    fields_r2 = subprocess.run(
        ["curl", "-sk", f"{fields_url}?page_size=100", "--noproxy", "*",
         "-H", f"Authorization: Bearer {token}"],
        capture_output=True, text=True, timeout=30)
    for f in json.loads(fields_r2.stdout).get("data", {}).get("items", []):
        if f.get("is_primary"):
            import tempfile as _tf
            _payload = json.dumps({"field_name": "会议主题", "type": 1}, ensure_ascii=False)
            with _tf.NamedTemporaryFile(mode='w', suffix='.json', delete=False, encoding='utf-8') as _ff:
                _ff.write(_payload); _ffname = _ff.name
            subprocess.run(
                ["curl", "-sk", "-X", "PUT", f"{fields_url}/{f['field_id']}", "--noproxy", "*",
                 "-H", "Content-Type: application/json", "-H", f"Authorization: Bearer {token}",
                 "--data", f"@{_ffname}"],
                capture_output=True, text=True, timeout=15)
            import os as _os; _os.unlink(_ffname)
            break

    # 新建业务字段（不含「会议主题」，它已是主字段）
    for field in [
        {"field_name": "项目类别", "type": 1},
        {"field_name": "日期",     "type": 5},
        {"field_name": "时长",     "type": 1},
        {"field_name": "参会人",   "type": 1},
        {"field_name": "完整对话", "type": 15},
        {"field_name": "会议纪要", "type": 15},
        {"field_name": "备注",     "type": 1},
    ]:
        curl_post(fields_url, field, token)

    cfg["meeting_bitable_app_token"] = app_token
    cfg["meeting_bitable_table_id"] = table_id
    with open(cfg_path, "w", encoding="utf-8") as f:
        json.dump(cfg, f, ensure_ascii=False, indent=2)

    print(f"✅ monee会议记录表已创建：https://feishu.cn/base/{app_token}")


def update_meeting_bitable(token, topic, category, date_str, attendees, duration,
                            doc1_id, doc2_id):
    cfg = json.load(open(os.path.expanduser("~/.claude/feishu_config.json")))
    app_token = cfg.get("meeting_bitable_app_token")
    table_id = cfg.get("meeting_bitable_table_id")
    if not app_token or not table_id:
        print("会议记录表未配置，跳过")
        return

    base_url = f"https://open.feishu.cn/open-apis/bitable/v1/apps/{app_token}/tables/{table_id}"

    import datetime
    y, m, d = map(int, date_str.split("-"))
    date_ts_ms = int(datetime.datetime(y, m, d).timestamp() * 1000)

    fields = {
        "会议主题": topic,   # 主字段
        "项目类别": category,
        "日期": date_ts_ms,
        "完整对话": {
            "text": f"{topic}-完整对话",
            "link": f"https://feishu.cn/docx/{doc1_id}"
        },
        "会议纪要": {
            "text": f"{topic}-会议纪要",
            "link": f"https://feishu.cn/docx/{doc2_id}"
        }
    }
    if attendees:
        fields["参会人"] = attendees
    if duration:
        fields["时长"] = duration

    r = curl_post_bitable(f"{base_url}/records", {"fields": fields}, token)
    print("会议记录表新建：", "成功" if r.get("code") == 0 else r)
```

调用时传入收集好的信息：
```python
create_meeting_bitable_if_needed(token)
update_meeting_bitable(token, topic=会议主题, category=项目类别,
                       date_str=会议日期, attendees=参会人(可为None),
                       duration=时长(可为None),
                       doc1_id=doc1_id, doc2_id=doc2_id)
```

### 8. 完成提示 + 询问复盘追问模式

面试/其他类型收尾文档是「总评」，工作会议类型收尾文档是「会议纪要」——以下统称"收尾文档"。

在对话框中按以下顺序输出，**不能拆分成多条消息**：

1. 告知飞书文档已创建（面试/其他三个，会议两个），对应多维表格已更新
2. 输出每个文档的链接（格式：`https://feishu.cn/docx/{document_id}`）
3. 完整输出收尾文档内容（总评 或 会议纪要）
4. 用 AskUserQuestion 询问是否开启复盘追问模式

**询问复盘追问模式：**

用 AskUserQuestion 给出选项：

**问题：** 是否需要把后续追问内容保存到收尾文档里？
**选项 A（开启）：** 后续追问说「存」时自动整理并追加到收尾文档
**选项 B（不需要）：** 只看文档，不保存追问

用户选 B 则流程结束。

用户选 A 则回复：

> 📌 **复盘追问模式已开启。** 说「**存**」时我把这段追问整理后追加到收尾文档的「复盘追问」章节。

**收集范围：** 从用户选择开启起，到用户说触发词之前，所有和本场面试/会议相关的问答内容。

**触发词：** 用户说「存」「存进去」「保存」「存到飞书」任意一个即触发保存。

**保存逻辑：**
1. 回顾本次对话中从开启追问模式之后的所有 Q&A
2. 整理为问答格式：每组以 `Q：` 开头写用户问题，正文写回答
3. 用 `h2` 格式分组，每个问题一个 `h2` 标题
4. 调用飞书 API，在对应收尾文档末尾追加 `divider` + `h1("复盘追问")` + 各问答块
5. 如果收尾文档里已有「复盘追问」章节（非首次保存），则只追加新内容，不重复写标题
6. 保存完成后提示「已存入文档」，继续保持追问模式——用户可继续追问，下次说触发词时追加新增部分

**多场处理：** 如果本次 skill 处理了多场面试/会议，回复时说明：「说『存虾皮』『存追觅』可以分别保存到对应文档，或说『都存』同时保存。」

**本地保存模式下：** 跳过此步骤，不询问。
