# book-to-skill：安装 · 转换 · 发布 操作指南

> 本文档基于 2026-09 本机（Windows）实际执行的完整流程整理，记录每一步的**可用命令、镜像地址、故障处理**，可直接照做复用。
> 参考项目：[virgiliojr94/book-to-skill](https://github.com/virgiliojr94/book-to-skill)（MIT 协议，把书籍/文档 PDF、EPUB、DOCX、HTML、MD、TXT 等转换为结构化 Agent Skill）。

---

## 一、安装 book-to-skill 到技能库

**目标位置**：用户技能库 `C:\Users\78394\AppData\Local\Doubao\User Data\Default\.doubao\agent_mode\workspace\.user_skills\book-to-skill\`

**官方安装方式**（二选一）：
```bash
# 方式 A：git clone（需本机已装 Git）
git clone https://github.com/virgiliojr94/book-to-skill.git

# 方式 B：pip 安装 CLI
pip install book-to-skill
```

**本机实操（GitHub 直连不通时）**：本机 GitHub 直连反复超时（curl exit 28、Invoke-WebRequest 无法连接），改用 **gh-proxy 镜像**下载 master 分支 zip：

```powershell
# 1. 下载（约 1.39 MB，实测 ~244s）
curl.exe -L -o "$env:TEMP\book-to-skill.zip" `
  "https://gh-proxy.com/https://github.com/virgiliojr94/book-to-skill/archive/refs/heads/master.zip"

# 2. 解压
Expand-Archive "$env:TEMP\book-to-skill.zip" "$env:TEMP\book-to-skill-download" -Force

# 3. 移动到用户技能库（目录名保持 book-to-skill）
Move-Item "$env:TEMP\book-to-skill-download\book-to-skill-master" `
  "C:\Users\78394\AppData\Local\Doubao\User Data\Default\.doubao\agent_mode\workspace\.user_skills\book-to-skill"

# 4. 清理临时文件
Remove-Item "$env:TEMP\book-to-skill.zip", "$env:TEMP\book-to-skill-download" -Recurse -Force
```

**验证安装完整**（四要素缺一不可）：
- `book-to-skill\SKILL.md`（官方流程文档，含 Step 0–11 + Update 工作流）
- `book-to-skill\scripts\extract.py`（提取脚本）
- `book-to-skill\tools\scan_generated_skill.py`（安全扫描）
- `book-to-skill\tools\validate_skill.py`（校验）

---

## 二、用 PDF 生成技能

### 2.1 提取文本
```powershell
$PY = "C:\Users\78394\AppData\Local\Doubao\User Data\sandbox_runtime\bases\<base>\python\python.exe"  # 本机 Python 3.14.7

# 检查提取器依赖
& $PY "…\book-to-skill\scripts\extract.py" --check "F:\中华人民共和国民法典.pdf"

# 缺 pypdf 等依赖时安装（用华为云镜像加速）
& $PY -m pip install pypdf -i https://mirrors.huaweicloud.com/repository/pypi/simple

# 提取（--mode text 纯文本模式；默认会尝试多模式）
& $PY "…\book-to-skill\scripts\extract.py" --mode text "F:\中华人民共和国民法典.pdf"
```

**输出要点**：脚本会报告页数 / 字符数 / 估算 tokens / 检测到的章节标题，并给出 `Text ->` 路径（后续按行号分段精读的依据）与 `Workdir ->` 路径（存在 `metadata.json`，清理时用）。

### 2.2 生成章节与支持文件
按官方 Step 3–9 流程（本机以《民法典》为例，选"全用途 / 深读型"）：
- 按正文行号分段精读 → 生成 `chapters/ch01~chNN-*.md`（每章含 Core Idea / Frameworks / Key Concepts / Mental Models / Anti-patterns / Worked Example / Key Takeaways / Connects To）
- `glossary.md`（术语表）、`patterns.md`（制度适用模式）、`cheatsheet.md`（速查决策表）
- 主 `SKILL.md`：frontmatter（name + description）+ 核心框架（前 2000 tokens 最重要）+ 章节索引 + 主题索引 + 支持文件索引

### 2.3 安全扫描（发布/加载前必须）
```powershell
& $PY "…\book-to-skill\tools\scan_generated_skill.py" "…\.user_skills\<技能名>"
# 输出 "Generated-skill scan passed" 即通过；非零退出需人工审查后再继续
```

### 2.4 清理工作目录
只删**本次运行**的 workdir（从提取输出 / metadata.json 读取），勿删别家 run 的目录。

---

## 三、技能改名（可选）

技能生成后如需改中文名（示例：`minfadian` → `民法典`）：

```powershell
# 1. 重命名目录
Rename-Item "…\.user_skills\minfadian" "民法典"

# 2. 修改 SKILL.md frontmatter 的 name 字段（Edit 前必须先 Read 该文件）
name: 民法典
```

**要点**：内部章节链接均为相对路径（`chapters/...`），改名不受影响；目录名与 name 字段保持一致即可。

---

## 四、发布到 GitHub（Step 11）

### 4.1 发布前的两道门槛
1. **版权门槛**：第三方受版权保护的书籍 → 仓库必须 **private**；仅当来源为公有领域（如官方法律文本）、开放许可或用户明确确认有权公开时，才允许 public。
2. **可见性**：必须单独向用户确认，回答必须是裸词 `private` 或 `public`（不能从上下文推断；子串匹配不算数）。

### 4.2 安装 Git（本机无 Git 时的实测路径）
winget 安装会从 GitHub 下载安装包，本机失败（`0x80072efd` 连接失败）。改用 **npmmirror 镜像**（实测 62.35 MB 约 16 秒）：

```powershell
# 探测可用源
# 清华镜像 403 不可用；gh-proxy 慢（~50KB/s）；npmmirror 快（~4MB/s）
curl.exe -L --connect-timeout 30 -o "$env:TEMP\git-setup\Git-2.55.0.3-64-bit.exe" `
  "https://registry.npmmirror.com/-/binary/git-for-windows/v2.55.0.windows.3/Git-2.55.0.3-64-bit.exe"

# 静默安装
Start-Process "$env:TEMP\git-setup\Git-2.55.0.3-64-bit.exe" `
  -ArgumentList "/VERYSILENT","/NORESTART","/NOCANCEL","/SP-","/SUPPRESSMSGBOXES","/CLOSEAPPLICATIONS" -Wait -PassThru

# 验证
& "C:\Program Files\Git\cmd\git.exe" --version   # git version 2.55.0.windows.3
```

> 注：新安装的 git 不会自动进入已运行的进程 PATH，直接用完整路径调用即可。

### 4.3 no-gh 路径发布（gh CLI 未安装时）
1. 用户在 GitHub 网页创建**空仓库**（https://github.com/new，不要勾选 Add README），把地址发回。
2. 在技能目录内执行：

```powershell
$git = "C:\Program Files\Git\cmd\git.exe"
$dir = "…\.user_skills\<技能名>"

# 初始化 + 暂存 + 身份 + 提交
& $git -C $dir init -b main
& $git -C $dir add -A
& $git -C $dir config user.name "<GitHub用户名>"
& $git -C $dir config user.email "<用户名>@users.noreply.github.com"
& $git -C $dir commit -m "Add <技能名> skill generated from <书名> via book-to-skill"

# 关联远程 + 推送
& $git -C $dir remote add origin "https://github.com/<用户名>/<仓库名>.git"
& $git -C $dir push -u origin main

# 验证远程分支
& $git -C $dir ls-remote origin
```

**注意**：① 推送时的认证由 Git Credential Manager 处理，若弹出登录窗口，在浏览器中完成即可；② PowerShell 会把 git 的 stderr 输出显示为 `RemoteException` 红字——那是显示噪音，以 `push exit: 0` 和 `main -> main` 为准。

### 4.4 发布后交付
```
✅ 已发布：https://github.com/<用户名>/<仓库名>（private）

跨主机安装命令：
  npx skills add https://github.com/<用户名>/<仓库名> --skill <技能名>
```
- 技能目录即远程仓库的本地工作副本（非嵌套 git 仓库时），后续更新直接 `commit + push`。
- private 改 public：仓库 Settings → Danger Zone → Change visibility。

---

## 五、故障速查

| 现象 | 原因 | 处理 |
|---|---|---|
| GitHub 直连下载超时（curl exit 28 / 0x80072efd） | 本机网络到 github.com 不稳定 | 用 gh-proxy.com 或 npmmirror 镜像下载 |
| PowerShell 读 UTF-8 文件出现乱码 | PS 5.1 默认按 ANSI/GBK 解码 | 用 Python `read_text(encoding='utf-8')` 或 `Get-Content -Encoding UTF8` |
| Edit 工具拒绝修改 | 文件未在本会话 Read 过 | 先 Read 再 Edit |
| 推送出现红字 RemoteException | PowerShell 把 git stderr 当错误 | 看 `push exit` 与 `main -> main`，非故障 |
| 技能目录被外层 git 仓库包含（gitlink 问题） | 技能装在项目仓库内 | 复制到临时目录再发布，勿就地 git init |

---

## 六、本机现状速查（2026-09）

- **已安装**：Git 2.55.0（`C:\Program Files\Git\cmd\git.exe`）；Python 3.14.7 + pypdf 6.19.0；book-to-skill（用户技能库）
- **已生成技能**：`民法典`（`…\.user_skills\民法典\`，13 个文件，已发布 https://github.com/cjp1225/minfadian，private）
- **技能库根目录**：`C:\Users\78394\AppData\Local\Doubao\User Data\Default\.doubao\agent_mode\workspace\.user_skills\`
