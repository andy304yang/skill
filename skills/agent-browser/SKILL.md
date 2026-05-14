---
name: agent-browser
description: 使用浏览器对已部署的网站进行自动化测试 — 根据用户描述的测试项逐一验证，截图取证，生成测试报告并上传 COS 桶。
version: 1.0.0
metadata:
  hermes:
    tags: [browser, testing, qa, screenshot, cos]
    requires: []
---

# Agent Browser 测试 Skill

根据用户描述的文案和测试项，对已部署的网站进行自动化 QA 测试。测试完成后生成报告并上传 COS 桶。

## 触发条件

用户说：
- "测试 xxx 网站"
- "测试 /agent-browser"
- "帮我测试 xxx 功能"
- 类似需要访问网页并截图验证的请求

## 完整流程

### Step 1: 解析测试需求

用户提供：
- **URL**：要测试的网站地址（如 `https://mclarenai.cn/component/`）
- **测试项**：用户描述的测试文案（如"检查首页是否正常显示"、"验证登录流程"）

如果用户只给了路径（如 `/agent-browser`），拼接完整 URL：`https://mclarenai.cn/agent-browser`

### Step 2: 初始化输出目录

```python
import os
from datetime import datetime

timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
output_dir = f'/home/agentuser/dogfood-output/{timestamp}'
os.makedirs(f'{output_dir}/screenshots', exist_ok=True)
print(f"Output: {output_dir}")
```

### Step 3: 浏览器测试

使用 browser 工具集逐步测试：

```javascript
// 1. 导航到目标页面
browser_navigate(url="https://mclarenai.cn/component/")

// 2. 检查控制台错误
browser_console(clear=true)

// 3. 截图 + AI 分析
browser_vision(
  question="描述页面布局，识别任何视觉问题、broken 元素或可访问性问题",
  annotate=true
)

// 4. 滚动检查
browser_scroll(direction="down")
browser_vision(question="检查滚动后页面是否正常渲染", annotate=false)

// 5. 继续测试用户描述的测试项...
```

### Step 4: 生成测试报告

```markdown
# 测试报告

**测试时间**: {timestamp}
**测试地址**: {url}
**测试人**: Hermes Agent (自动化测试)

---

## 测试结果汇总

| 测试项 | 状态 | 说明 |
|--------|------|------|
| 页面加载 | ✅ 通过 | HTTP 200 |
| 控制台错误 | ✅ 无错误 | console.log 正常 |
| 资源加载 | ✅ 通过 | JS/CSS/图片均 200 |
| ... | ... | ... |

---

## 截图证据

### 1. 首页截图
![首页](screenshots/01_homepage.png)

### 2. 滚动后截图
![滚动后](screenshots/02_scrolled.png)

---

## 控制台日志
无 JavaScript 错误

---

## 结论
测试通过，网站运行正常。
```

### Step 5: 上传 COS

COS 配置（来自 `/tmp/cos_upload.py`，密钥不要硬编码在 skill 里）：
- Bucket: `yanghao-1303848059`
- Region: `ap-guangzhou`
- SecretId: `从 /tmp/cos_upload.py 读取`
- SecretKey: `从 /tmp/cos_upload.py 读取`

```python
import sys, os
sys.path.insert(0, '/home/agentuser/.hermes/hermes-agent/venv/lib/python3.11/site-packages')
from qcloud_cos import CosConfig, CosS3Client
import shutil
from datetime import datetime

# 从 /tmp/cos_upload.py 读取密钥
cos_cfg = {}
with open('/tmp/cos_upload.py') as f:
    for line in f:
        if '=' in line and not line.startswith('#'):
            key, _, val = line.partition('=')
            cos_cfg[key.strip()] = val.strip().strip("'\"?")

bucket = 'yanghao-1303848059'
region = 'ap-guangzhou'
config = CosConfig(Region=region, SecretId=cos_cfg['secret_id'], SecretKey=cos_cfg['secret_key'])
client = CosS3Client(config)

timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
output_dir = f'/home/agentuser/dogfood-output/{timestamp}'
cos_base = f'agent-browser/{timestamp}'

# 上传截图
screenshots_dir = f'{output_dir}/screenshots'
for fname in os.listdir(screenshots_dir):
    local_path = os.path.join(screenshots_dir, fname)
    cos_key = f'{cos_base}/screenshots/{fname}'
    with open(local_path, 'rb') as f:
        client.put_object(
            Bucket=bucket,
            Body=f,
            Key=cos_key,
            ContentType='image/png',
            ACL='public-read'
        )
    url = f'https://{bucket}.cos.{region}.myqcloud.com/{cos_key}'
    print(f"✅ {fname}: {url}")

# 上传报告
report_path = f'{output_dir}/report.md'
cos_key = f'{cos_base}/report.md'
with open(report_path, 'rb') as f:
    client.put_object(
        Bucket=bucket,
        Body=f,
        Key=cos_key,
        ContentType='text/markdown',
        ACL='public-read'
    )
report_url = f'https://{bucket}.cos.{region}.myqcloud.com/{cos_key}'
print(f"📄 报告: {report_url}")
```

## 测试项模板

根据用户描述的文案生成对应测试项：

| 用户描述 | 对应测试 |
|---------|---------|
| "检查页面是否正常显示" | 页面加载 + 截图 + 控制台检查 |
| "验证 xxx 功能" | 逐一点击/操作 + 截图 + 结果验证 |
| "检查控制台是否有报错" | browser_console + 错误分析 |
| "检查资源是否完整加载" | 验证 CSS/JS/图片返回 200 |
| "测试 xxx 交互" | 模拟交互操作 + 截图取证 |

## 报告格式

报告包含：
1. **测试概要** — 时间、地址、测试范围
2. **详细测试结果** — 每个测试项的状态、截图、控制台日志
3. **问题汇总** — 如有问题，按 severity 分类
4. **截图链接** — 所有截图的 COS 公开链接
5. **结论** — 通过/部分通过/失败

## 发送结果

测试完成后，返回：
- 测试结论（通过/有问题）
- 所有截图的 COS 链接
- 报告 COS 链接
- 让用户确认是否需要修复
