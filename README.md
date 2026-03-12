这是一个用于 **防止 Supabase 免费层级项目因长期闲置而进入“暂停”（Paused）状态** 的 GitHub Actions 自动化脚本。

Supabase 的免费层级（Free Tier）规定，如果项目在一段时间内（通常是7天）没有收到任何 API 请求，数据库和 API 服务会自动暂停以节省资源。当用户再次访问时，需要等待几十秒甚至更长时间来“唤醒”项目（Cold Start）。

这个脚本通过定期（每天）向 Supabase 项目发送一个轻量级的 HTTP 请求，欺骗系统认为该项目仍在活跃使用中，从而保持其处于“运行中”（Active）状态。

以下是脚本的详细逐段解析：

### 1. 触发机制 (`on`)
```yaml
on:
  schedule:
    # 每天一次：这里示例 UTC 03:17
    - cron: "17 3 * * *"
  workflow_dispatch:
```
-   **`schedule`**: 使用 Cron 表达式 `17 3 * * *` 设定定时任务。
    -   这意味着脚本会在 **UTC 时间每天凌晨 03:17** 自动运行。
    -   *注意：你需要根据你的时区换算这个时间，确保它在你的业务低峰期运行。*
-   **`workflow_dispatch`**: 允许你在 GitHub 网页界面上手动点击按钮立即运行此脚本（用于测试或紧急唤醒）。

### 2. 运行环境与多项目策略 (`jobs` & `strategy`)
```yaml
jobs:
  keepalive:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        include:
          - name: go-project
            supabase_url: SUPABASE_URL_GO
            supabase_key: SUPABASE_SERVICE_ROLE_KEY_GO
            expected_ref: ""
          - name: test-project
            supabase_url: SUPABASE_URL_TEST
            supabase_key: SUPABASE_SERVICE_ROLE_KEY_TEST
            expected_ref: ""
```
-   **`runs-on: ubuntu-latest`**: 在最新的 Ubuntu 虚拟机上运行。
-   **`matrix` (矩阵策略)**: 这是脚本最强大的部分。它允许你**在一个工作流中同时维护多个 Supabase 项目**。
    -   你可以看到定义了 `go-project` 和 `test-project` 两个配置。
    -   `supabase_url` 和 `supabase_key` 的值（如 `SUPABASE_URL_GO`）**不是真实的 URL 或密钥**，而是指向 GitHub Secrets 中存储的变量名。
    -   `expected_ref`: 可选的安全校验字段，用于防止配置错误（详见下文）。
-   **`fail-fast: false`**: 即使其中一个项目（例如 `go-project`）的检查失败了，脚本也会继续尝试检查其他项目（例如 `test-project`），而不是直接终止整个流程。

### 3. 执行步骤 (`steps`)

#### A. 环境变量设置
```yaml
- name: Ping Supabase (${{ matrix.name }})
  env:
    SUPABASE_URL: ${{ secrets[matrix.supabase_url] }}
    SUPABASE_KEY: ${{ secrets[matrix.supabase_key] }}
    EXPECTED_REF: ${{ matrix.expected_ref }}
    PROJECT_NAME: ${{ matrix.name }}
```
-   这里动态地从 GitHub Secrets 读取真实的 URL 和 Service Role Key。
-   **重要安全提示**: 必须使用 `SUPABASE_SERVICE_ROLE_KEY`（服务角色密钥），因为它拥有绕过行级安全策略（RLS）的权限，确保健康检查一定能成功，不会被数据库权限规则拦截。

#### B. 核心逻辑脚本 (`run`)

**1. 基础信息输出**
```bash
echo "Project: $PROJECT_NAME"
echo "URL: ${SUPABASE_URL}/rest/v1/health_check"
```
打印当前正在检查的项目名称和目标 URL。

**2. 防误配校验 (Safety Check)**
```bash
if [ -n "${EXPECTED_REF}" ]; then
  ref=$(echo "$SUPABASE_URL" | sed -E 's#https://([^.]+)\..*#\1#')
  echo "Project ref: $ref (expected: $EXPECTED_REF)"
  test "$ref" = "$EXPECTED_REF"
fi
```
-   **目的**: 防止你在 Secrets 里填错了 URL。例如，你想检查项目 A，但不小心把项目 B 的 URL 填进去了。
-   **逻辑**:
    -   从 URL (如 `https://abcdefg.supabase.co`) 中提取出 Ref ID (`abcdefg`)。
    -   将其与你在矩阵配置中填写的 `expected_ref` 进行比对。
    -   如果不一致，`test` 命令会失败，脚本立即终止，避免向错误的项目发送请求或掩盖配置错误。

**3. 构造请求 Payload**
```bash
payload=$(printf '{"source":"github-actions","run_id":"%s"}' "${GITHUB_RUN_ID}")
```
创建一个简单的 JSON 数据，包含 GitHub 的运行 ID，作为请求体发送。虽然健康检查接口通常不需要 body，但带上数据可以模拟更真实的业务请求。

**4. 发送 HTTP 请求 (curl)**
```bash
status=$(curl -sS -o resp.txt -w "%{http_code}" \
  -X POST "${SUPABASE_URL}/rest/v1/health_check" \
  -H "apikey: ${SUPABASE_KEY}" \
  -H "Authorization: Bearer ${SUPABASE_KEY}" \
  -H "Content-Type: application/json" \
  -H "Prefer: return=representation" \
  -d "$payload"
)
```
-   **目标端点**: `/rest/v1/health_check`。这是 Supabase PostgREST 提供的一个标准健康检查接口。
-   **Headers**:
    -   `apikey` 和 `Authorization`: 填入 Service Role Key 以通过验证。
    -   `Prefer: return=representation`: 告诉服务器返回响应体（尽管健康检查通常返回简单状态）。
-   **动作**: 发送 `POST` 请求。使用 `POST` 而不是 `GET` 有时能更好地触发某些缓存刷新机制，且符合写入操作的语义（虽然这里只是 ping）。

**5. 结果验证**
```bash
echo "HTTP Status: $status"
cat resp.txt
test "$status" -ge 200 -a "$status" -lt 300
```
-   打印 HTTP 状态码和响应内容。
-   **关键判断**: `test "$status" -ge 200 -a "$status" -lt 300`。
    -   如果状态码不在 200-299 之间（即请求失败），脚本会以非零退出码结束。
    -   这将导致 GitHub Actions 显示为 **Failed**，并通过邮件或通知提醒你：你的 Supabase 项目可能出问题了（不仅仅是休眠，可能是挂了或密钥过期）。

---

### 如何使用此脚本？

1.  **创建文件**: 在你的 GitHub 仓库中创建文件 `.github/workflows/supabase-keepalive.yml`，并将上述代码粘贴进去。
2.  **配置 Secrets**:
    进入仓库的 `Settings` -> `Secrets and variables` -> `Actions`，添加以下 New repository secrets：
    -   `SUPABASE_URL_GO`: 你的第一个项目 URL (例如 `https://xxxxx.supabase.co`)
    -   `SUPABASE_SERVICE_ROLE_KEY_GO`: 对应项目的 Service Role Key (在 Supabase  Dashboard -> Settings -> API 中找到)
    -   `SUPABASE_URL_TEST`: 第二个项目 URL
    -   `SUPABASE_SERVICE_ROLE_KEY_TEST`: 第二个项目 Key
    *(名字必须与脚本 matrix 部分引用的名字完全一致)*
3.  **(可选) 填写 Ref**:
    如果你希望启用安全检查，将 URL 中的子域名部分（即 `xxxxx`）填入脚本中的 `expected_ref` 字段。
4.  **提交并运行**:
    提交代码后，GitHub Actions 会自动识别。你可以去 `Actions` 标签页手动触发一次测试，或者等待第二天自动运行。

### 总结
这是一个**健壮、支持多项目、具备自我校验能力**的保活脚本。它不仅解决了免费层休眠问题，还通过失败报警机制充当了简单的监控工具。
