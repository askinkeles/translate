# 🌍 Automatic Markdown Translator with GitHub Models

This project is an automation system that translates Markdown (`.md`) files on GitHub into **English (or other languages)** using **GitHub Models (GPT-4o)**.

This guide explains how you can perform the process **entirely on GitHub servers (Cloud)** securely and for free, without wrestling with `PATH` and `Permission` issues in local setups.

---

## 🚀 How Does It Work?

1. You write a `.md` file in Turkish and upload it to GitHub (`git push`).
2. GitHub Actions triggers.
3. It connects to the **GPT-4o** model on Microsoft Azure.
4. It translates your file and automatically saves it under the `translations/` folder.
5. It adds language options (links) automatically to the top of the file.

---

## 🛠️ Setup (Step-by-Step)

### 1. Get a Token (Key)
You need permission to access GitHub Models for the system to work.

1. Go to the [GitHub Marketplace - Models](https://github.com/marketplace/models) page (or navigate to Settings > Developer Settings).
2. Create a **Personal Access Token (Tokens - classic or Fine-grained)**.
3. Copy the token (starts with `github_pat_...`).
    * ⚠️ **IMPORTANT WARNING:** Never paste this token into your code, `.env` file, or `README` document and upload it to GitHub. GitHub's security system (Push Protection) will block it or revoke the token.

### 2. Add the Token to GitHub (Secret)
We need to store the key securely.

1. Go to the **Settings** tab of this repository.
2. Click on **Secrets and variables** > **Actions** in the left menu.
3. Press the **New repository secret** button.
    * **Name:** `OPENAI_API_KEY`
    * **Secret:** (Paste the copied `github_pat_...` key here.)
4. Save by clicking **Add secret**.

### 3. Create the Automation File
Create the following folder path in your repository: `.github/workflows/`

Inside this folder, create a file named `translator.yml` and paste the following code:

```yaml
name: AI Translator

on:
  push:
    branches: [ "main" ] # The name of your main branch
    paths:
      - '**.md' # Works only when markdown files are changed

permissions:
  contents: write # Required permissions to add files to the repo

jobs:
  translate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install Tool
        run: pip install co-op-translator

      - name: Start Translation
        env:
          # Pulling the key we saved as a secret
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
          OPENAI_BASE_URL: 'https://models.inference.ai.azure.com'
          OPENAI_MODEL: 'gpt-4o'
        # You can add target languages here (e.g., -l "en de fr")
        run: translate -l "en"

      - name: Save and Push
        run: |
          git config --global user.name "github-actions[bot]"
          git config --global user.email "github-actions[bot]@users.noreply.github.com"
          git add .
          git commit -m "🤖 Translation completed" || exit 0
          git push
```

---

## 💡 Experience Notes (Why This Setup?)

Critical insights we gained while developing this project:

1. **Local vs Cloud:**
    * ❌ **Local (On Your Computer):** Python `PATH` settings, admin permissions, and file-reading (path) issues on Windows can waste a lot of time. Additionally, creating a `.env` file and accidentally pushing it to GitHub poses a huge security risk.
    * ✅ **GitHub Actions (Cloud):** A virtual Linux machine is freshly set up each time, completes its task, and shuts down. There are no `PATH` issues, no permission issues. This is the cleanest and most stable method.

2. **Security (Secret Scanning):**
    * Never push `.env` files without adding them to `.gitignore`.
    * If you accidentally push a key, GitHub detects it and prevents the upload (Push Protection). In this case, the cleanest option is to delete the repository and start fresh or clean the git history.

3. **Model Selection:**
    * This project uses the **GitHub Models (Azure)** infrastructure instead of the standard OpenAI API. This allows GitHub users to experience the GPT-4o model free of charge within certain limits.

---

## 🏁 Usage

1. Add a new `example.md` file to the repo (write content in Turkish).
2. Don’t forget to add `<en>` and `<tr>` tags at the very top of the file.
3. Commit and push it.
4. Links will be automatically added once the process is complete.