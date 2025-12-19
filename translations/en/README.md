<!-- LANGUAGE_TABLE_START -->
[ 🇹🇷 Türkçe ](../../README.md) | [ 🇺🇸 English ](README.md)
<!-- LANGUAGE_TABLE_END -->

# 🌍 Automatic Document Translator with GitHub Models (All-in-One Translator)
<div align="center">

  [![AI Translator](https://github.com/askinkeles/translate/actions/workflows/cevirmen.yml/badge.svg)](https://github.com/askinkeles/translate/actions/workflows/cevirmen.yml)
  
  [![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE.md)

  ![Visitors](https://visitor-badge.laobi.icu/badge?page_id=askinkeles.translate)

  [![GitHub forks](https://img.shields.io/github/forks/askinkeles/translate?style=social)](https://github.com/askinkeles/translate/network)
  [![GitHub stars](https://img.shields.io/github/stars/askinkeles/translate?style=social)](https://github.com/askinkeles/translate/stargazers)

</div>
---
This project automatically detects **all Markdown (`.md`) files** (e.g., `README.md`, `CONTRIBUTING.md`, `LICENSE.md`, etc.) in your repository, translates them into English using **GitHub Models (GPT-4o)**, and adds navigation links for language switching at the top of each file.

> **🎯 Purpose:** Write your technical documentation only in Turkish; the system will automatically create all other files and their English versions.

---

## 🏗️ Why Use This Custom Method? (Technical Background)

Here are the critical reasons why we use a **Custom Script** instead of standard translation tools (e.g., `co-op-translator`):

1.  **Token Format:** GitHub Models generates tokens in the `github_pat_` format. Ready-made tools expect the OpenAI `sk-` format, so they don't work.
2.  **Beta Permission Issue:** GitHub Models is in "Public Beta." If "Only select repositories" is chosen in token settings, AI permissions are hidden from the menu. The **"All repositories"** setting in this guide solves this issue.
3.  **Smart Linking:** When translation files are moved to a subfolder (`translations/en/`), links returning to the main page (`../../FileName.md`) need to be dynamically calculated. This project handles this for each file individually.

---

## 🚀 Installation Guide (Step-by-Step)

Follow these steps to set up this system.

### Step 0: Preparation (Marketplace and Local Setup)

1.  **Marketplace Access:**
    * Go to [GitHub Marketplace Models](https://github.com/marketplace/models).
    * If you don't have access, click "Join Waitlist" to register (approval is quick).
    * If you see the "Playground" button, you have access.

2.  **Start the Project Locally:**
    If you don't have a repository yet, start on your computer with the following commands:
    ```bash
    mkdir my-translator-project
    cd my-translator-project
    echo "# Project Title" > README.md
    echo "# Contribution Guidelines" > CONTRIBUTING.md
    git init
    git branch -M main
    ```

### Step 1: Create a Token (Access Key) ⚠️
This step is critical. Follow the settings **exactly**.

1.  Go to **Settings** > **Developer settings** > **Personal access tokens** > **Fine-grained tokens** in GitHub.
2.  Click the **Generate new token** button.
3.  **Token Name:** `Translator-Token`.
4.  **Expiration:** `90 days`.
5.  **Repository access:** 🔴 **VERY IMPORTANT!**
    * Be sure to select **"All repositories"**.
    * *(If you select "Only select repositories," the Models permission may not appear).*
6.  **Permissions:**
    * Expand the **Repository permissions** section:
        * `Contents` -> **Read and write** (To write files).
    * Expand the **Account permissions** section:
        * `Models` -> **Read-only** (To use AI).
7.  Click the **Generate token** button and copy the code.

### Step 2: Create the Repository on GitHub and Add a Secret

1.  Create a new repository on GitHub.
2.  Go to your repository's **Settings** > **Secrets and variables** > **Actions** page.
3.  Click the **New repository secret** button.
4.  **Name:** `OPENAI_API_KEY`
5.  **Value:** Paste the token you copied and save it.

### Step 3: Create the Workflow File

On your computer, create a `.github/workflows/` folder. Inside it, create a file named `cevirmen.yml` and paste the following code.

*(This code finds all `.md` files in the folder and processes them in a loop)*

```yaml
name: AI Translator (Robust)

on:
  push:
    branches: ["main"]
    paths:
      - '**.md' # Triggered when any MD file changes

permissions:
  contents: write

jobs:
  translate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install Required Libraries
        run: pip install openai

      - name: Batch Translation Script
        env:
          GITHUB_TOKEN: ${{ secrets.OPENAI_API_KEY }}
        shell: python
        run: |
          import os
          import sys
          import re
          from openai import OpenAI

          # --- 1. SETTINGS ---
          endpoint = "https://models.github.ai/inference"
          token = os.environ.get("GITHUB_TOKEN")
          model_name = "gpt-4o"
          
          # --- CRITICAL FIX: CREATE TAGS WITH ASCII ---
          # Prevent YAML parser from removing HTML comments
          # by creating characters with code.
          # chr(60) = '<', chr(62) = '>'
          
          TAG_START = chr(60) + "!-- LANGUAGE_TABLE_START --" + chr(62)
          TAG_END   = chr(60) + "!-- LANGUAGE_TABLE_END --" + chr(62)

          # Print for debugging (visible in logs)
          print(f"Tags created: {TAG_START} ... {TAG_END}")

          if not token:
              print("::error::Token not found! Check secret settings.")
              sys.exit(1)

          client = OpenAI(base_url=endpoint, api_key=token)

          # --- 2. FIND FILES ---
          md_files = [f for f in os.listdir('.') if f.endswith('.md') and os.path.isfile(f)]

          if not md_files:
              print("No .md files found to process.")
              sys.exit(0)

          print(f"Files found: {md_files}")

          # --- 3. START LOOP ---
          for file_name in md_files:
              print(f"\n--- Processing: {file_name} ---")

              # Link Templates
              header_root = f"{TAG_START}\n[ 🇹🇷 Turkish ]({file_name}) | [ 🇺🇸 English ](translations/en/{file_name})\n{TAG_END}\n"
              
              header_en = f"{TAG_START}\n[ 🇹🇷 Turkish ](../../{file_name}) | [ 🇺🇸 English ]({file_name})\n{TAG_END}\n"

              # Read File
              try:
                  with open(file_name, "r", encoding="utf-8") as f:
                      content = f.read()
              except Exception as e:
                  print(f"::error::{file_name} could not be read: {e}")
                  continue

              # Add Link to Main File
              if TAG_START in content:
                  # Instead of regex, use simple replacement to avoid errors with special characters
                  # Simple logic: Remove between Start and End, then add new content.
                  # However, regex is cleaner; just escape variables.
                  pattern = re.escape(TAG_START) + r".*?" + re.escape(TAG_END)
                  content = re.sub(pattern, header_root.strip(), content, flags=re.DOTALL)
              else:
                  content = header_root.strip() + "\n\n" + content

              with open(file_name, "w", encoding="utf-8") as f:
                  f.write(content)

              # --- TRANSLATION PART (This was the error-prone section) ---
              
              # Check before splitting
              parts = content.split(TAG_END)
              
              if len(parts) > 1:
                  text_to_translate = parts[-1].strip()
              else:
                  # If split fails (no tag), take the entire content
                  text_to_translate = content.replace(header_root.strip(), "").strip()

              if not text_to_translate:
                  print(f"WARNING: {file_name} content is empty or only contains links.")
                  continue

              # AI Call
              try:
                  response = client.chat.completions.create(
                      messages=[
                          {"role": "system", "content": "You are a professional technical translator. Translate the markdown content to English. Do not explain. Preserve code blocks exactly."},
                          {"role": "user", "content": text_to_translate}
                      ],
                      model=model_name,
                      temperature=0.1
                  )
                  translated_body = response.choices[0].message.content
                  
                  # Save English Translation
                  final_english_content = header_en.strip() + "\n\n" + translated_body
                  
                  os.makedirs("translations/en", exist_ok=True)
                  output_path = f"translations/en/{file_name}"
                  
                  with open(output_path, "w", encoding="utf-8") as f:
                      f.write(final_english_content)
                      
                  print(f"✅ {file_name} successfully translated.")

              except Exception as e:
                  print(f"::error::{file_name} encountered an error during translation: {e}")
                  continue

      - name: Push to GitHub
        run: |
          git config --global user.name "github-actions[bot]"
          git config --global user.email "github-actions[bot]@users.noreply.github.com"
          git add .
          git commit -m "🤖 All Documents Translated (Fix)" || echo "No changes"
          git push
```

### Step 4: Publish

Push your files to GitHub from the VS Code terminal:

```bash
git add .
git commit -m "System setup completed"
git push -u origin main
```

---

## 🚦 How to Use and Test?

The system is fully automated.

1.  Edit or create any `.md` file (e.g., `README.md`, `LICENSE.md`, etc.) in your repository.
2.  Save and push the changes (`git push`).
3.  Go to the **Actions** tab in your GitHub repository.

### What You'll See in the Actions Tab
1.  **Yellow Circle:** The process has started.
2.  **Logs:** When you click on the process, you'll see a list like `Files found: ['README.md', 'CONTRIBUTING.md']`. The script processes them one by one.
3.  **Green Checkmark (✅):** Once completed, links will appear at the top of your files in the main directory, and English versions will be created in the `translations/en/` folder.

---

## ❓ Frequently Asked Questions (FAQ)

**Q: Can I manually edit the English translation?**  
A: No. The `translations` folder is **automatically overwritten** during each run. You should make edits in the main Turkish file.

**Q: What happens if I add a new file?**  
A: For example, if you add `NEW_DOCUMENT.md`, the system will automatically detect it in the next run, add links, and create its translation as `translations/en/NEW_DOCUMENT.md`.

**Q: Why is there no `.env` file?**  
A: Storing API keys in code is insecure. GitHub Secrets creates a virtual environment variable during runtime to ensure security.