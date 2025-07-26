---
title: "Automatically perform AI review on modified code during git commit"
date: 2025-07-25
---

# git commit时自动对修改代码进行AI review

研发人员经常需要修改，提交代码。但有时会因为粗心，或者疏忽，导致提交的代码存在一些错误或者隐患。如果能有人帮忙review一下，那就太好了。
现在AI发展迅猛，我们可以借用AI来帮我们review代码，消除大部分常见问题，甚至能提出优化建议，使你的代码质量大大提高。

下面我来教大家怎么用git + ChatGPT 来实现git commit时的AI自动review。

## 1. 获取OPENAI的API key

### 1.1 创建 API key

打开链接 https://platform.openai.com/account/api-keys ，登录你的OpenAPI账户。
点击`Create new secret key`，输入`Name`,点击创建即可。

<img width="229" height="245" alt="image" src="https://github.com/user-attachments/assets/f0be43e8-05eb-4b7e-8ea2-eb99499e2c60" />

<img width="223" height="189" alt="image" src="https://github.com/user-attachments/assets/fd114c98-ab70-4b28-a0e5-c5370250bc36" />

创建后记得把key保存备用，记得千万不要暴露在网上，也不要轻易借给他人使用。

<img width="838" height="621" alt="image" src="https://github.com/user-attachments/assets/2cd2b4a4-f155-472a-ba92-35075c9e4521" />

### 1.2 充值账户

因为openai API key是预付费，按token收费，所以我们要先给账户充值。

打开链接 https://platform.openai.com/settings/organization/billing/overview ，可以看到你的余额。

<img width="384" height="257" alt="image" src="https://github.com/user-attachments/assets/872d5fa8-0e33-44b2-beff-94d0677e1eee" />

然后选择充值 `Add to credit balance`

<img width="224" height="173" alt="image" src="https://github.com/user-attachments/assets/56b26410-c1a7-46e4-8b4b-99c69d3cd603" />

最低充值$5， 我们添加信用卡账户，确认后充值成功。

## 2. 安装依赖

在开发机器上，需要安装openai依赖包。

```bash
pip install openai
```

PS：如果需要root权限，记得在命令前加上`sudo`

## 3. 编辑pre-commit

```bash
cp .git/hooks/pre-commit.sample .git/hooks/pre-commit
vim .git/hooks/pre-commit
```

pre-commit 已经有了一些基础脚本，可以删除，也可以保留。
我们在后面添加以下脚本内容：

```bash
diff=$(git diff --cached --function-context)

if [ -z "$diff" ]; then
  echo "No staged changes to review."
  exit 0
fi

python3 .git/hooks/ai_review.py "$diff"
exit_code=$?

if [ $exit_code -eq 1 ]; then
    echo "Commit aborted due to AI review feedback."
    exit 1
fi
```

这部分主要是生成diff，并调用 ai_review.py 去review 它。

## 4. 编写ai_review.py 脚本

创建 `.git/hooks/ai_review.py` 文件，并赋予执行权限

```bash
touch .git/hooks/ai_review.py
chmod 777 .git/hooks/ai_review.py
vim .git/hooks/ai_review.py
```

编辑脚本，写入以下内容

```bash
import openai
import sys
import os

diff_text = sys.argv[1] if len(sys.argv) > 1 else ""

if len(diff_text.strip()) == 0:
    print("No diff content provided.")
    sys.exit(0)

client = openai.OpenAI(
    api_key = os.getenv("OPENAI_API_KEY", "paste-your-openai-key-here")
)

prompt = (
    "You are a senior code reviewer. Please review the following Git diff and point out:\n"
    "- Possible bugs or logic flaws\n"
    "- Code quality or best practice issues\n"
    "- Suggestions for improvement\n\n"
    "Only comment on meaningful issues. Here is the Git diff:\n\n"
    "```\n" + diff_text + "\n```"
)

response = client.chat.completions.create(
    #model="gpt-3.5-turbo",
    model="gpt-4",
    messages=[{"role": "user", "content": prompt}],
    temperature=0.2,
)

review = response.choices[0].message.content.strip()

print("\n===== AI Review Summary =====")
print(review)
print("=============================\n")

if "bug" in review.lower() or "issue" in review.lower():
    print("⚠️  Potential issues detected. Please review before pushing.")
    sys.exit(1)
```

在 `paste-your-openai-key-here` 位置粘贴你之前在openai网址获取的openai key字串。

在model这里填入你想使用的模型，"gpt-3.5-turbo性价比更好，gpt-4性能更佳。根据自己的情况做出合适的选择。

## 5. 测试验证

随便找一段代码，故意在其中添加几行错误代码，然后提交。
观察是否能够给出AI review结果。


