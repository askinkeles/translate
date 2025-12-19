# 🌍 GitHub Models ile Otomatik Doküman Çevirmeni (All-in-One Translator)

Bu proje, reponuzdaki **tüm Markdown (`.md`) dosyalarını** (örneğin `README.md`, `CONTRIBUTING.md`, `LICENSE.md` vb.) otomatik olarak algılar, **GitHub Models (GPT-4o)** kullanarak İngilizceye çevirir ve her dosyanın başına diller arası geçiş sağlayan navigasyon linklerini ekler.

> **🎯 Amaç:** Teknik dokümantasyonunuzu sadece Türkçe yazın; sistem diğer tüm dosyaları ve İngilizce versiyonlarını otomatik oluştursun.

---

## 🏗️ Neden Bu Özel Yöntemi Kullanıyoruz? (Teknik Arkaplan)

Standart çeviri araçları (`co-op-translator` vb.) yerine neden **Custom Script** kullandığımızın kritik sebepleri şunlardır:

1.  **Token Formatı:** GitHub Models, `github_pat_` formatında token üretir. Hazır araçlar ise OpenAI `sk-` formatı beklediği için çalışmaz.
2.  **Beta İzin Sorunu:** GitHub Models "Public Beta" sürecindedir. Token ayarlarında "Only select repositories" seçilirse, yapay zeka izinleri menüden gizlenmektedir. Bu rehberdeki **"All repositories"** ayarı bu sorunu çözer.
3.  **Akıllı Linkleme:** Çeviri dosyaları alt klasöre (`translations/en/`) taşındığında, ana sayfaya dönen linklerin (`../../DosyaAdi.md`) dinamik olarak hesaplanması gerekir. Bu proje bunu her dosya için ayrı ayrı yapar.

---

## 🚀 Kurulum Rehberi (Sıfırdan Adım Adım)

Bu sistemi kurmak için aşağıdaki adımları sırasıyla uygulayın.

### Adım 0: Ön Hazırlık (Marketplace ve Lokal Kurulum)

1.  **Marketplace Erişimi:**
    * [GitHub Marketplace Models](https://github.com/marketplace/models) sayfasına gidin.
    * Erişiminiz yoksa "Join Waitlist" diyerek kaydolun (Hızlı onaylanır).
    * "Playground" butonunu görüyorsanız erişiminiz var demektir.

2.  **Lokalde Projeyi Başlatma:**
    Henüz bir reponuz yoksa bilgisayarınızda şu komutlarla başlayın:
    ```bash
    mkdir my-translator-project
    cd my-translator-project
    echo "# Proje Başlığı" > README.md
    echo "# Katkıda Bulunma" > CONTRIBUTING.md
    git init
    git branch -M main
    ```

### Adım 1: Token (Erişim Anahtarı) Oluşturma ⚠️
Bu adım en kritik kısımdır. Ayarları **birebir** uygulayın.

1.  GitHub'da **Settings** > **Developer settings** > **Personal access tokens** > **Fine-grained tokens** sayfasına gidin.
2.  **Generate new token** butonuna basın.
3.  **Token Name:** `Translator-Token`.
4.  **Expiration:** `90 days`.
5.  **Repository access:** 🔴 **ÇOK ÖNEMLİ!**
    * Mutlaka **"All repositories"** seçeneğini seçin.
    * *("Only select repositories" seçerseniz Models izni görünmeyebilir).*
6.  **Permissions (İzinler):**
    * **Repository permissions** başlığını genişletin:
        * `Contents` -> **Read and write** (Dosya yazmak için).
    * **Account permissions** başlığını genişletin:
        * `Models` -> **Read-only** (Yapay zekayı kullanmak için).
7.  **Generate token** butonuna basın ve kodu kopyalayın.

### Adım 2: Repoyu GitHub'da Oluşturma ve Secret Ekleme

1.  GitHub'da yeni bir repository oluşturun.
2.  Reponuzun **Settings** > **Secrets and variables** > **Actions** sayfasına gidin.
3.  **New repository secret** butonuna basın.
4.  **Name:** `OPENAI_API_KEY`
5.  **Value:** Kopyaladığınız token'ı yapıştırın ve kaydedin.

### Adım 3: Workflow Dosyasını Oluşturma

Bilgisayarınızda `.github/workflows/` klasörünü oluşturun. İçine `cevirmen.yml` adında bir dosya açın ve aşağıdaki kodu yapıştırın.

*(Bu kod klasördeki tüm .md dosyalarını bulur ve döngüye sokar)*

```yaml
name: AI Translator (Robust)

on:
  push:
    branches: ["main"]
    paths:
      - '**.md' # Herhangi bir MD dosyası değişince tetiklenir

permissions:
  contents: write

jobs:
  translate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Python Kurulumu
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Gerekli Kütüphaneler
        run: pip install openai

      - name: Toplu Çeviri Scripti
        env:
          GITHUB_TOKEN: ${{ secrets.OPENAI_API_KEY }}
        shell: python
        run: |
          import os
          import sys
          import re
          from openai import OpenAI

          # --- 1. AYARLAR ---
          endpoint = "https://models.github.ai/inference"
          token = os.environ.get("GITHUB_TOKEN")
          model_name = "gpt-4o"
          
          # --- KRİTİK DÜZELTME: ASCII İLE ETİKET OLUŞTURMA ---
          # YAML parser'ın HTML yorumlarını silmesini engellemek için
          # karakterleri kodla oluşturuyoruz. 
          # chr(60) = '<', chr(62) = '>'
          
          TAG_START = chr(60) + "!-- LANGUAGE_TABLE_START --" + chr(62)
          TAG_END   = chr(60) + "!-- LANGUAGE_TABLE_END --" + chr(62)

          # Debug için yazdıralım (Loglarda görebilirsiniz)
          print(f"Etiketler oluşturuldu: {TAG_START} ... {TAG_END}")

          if not token:
              print("::error::Token bulunamadi! Secret ayarlarini kontrol edin.")
              sys.exit(1)

          client = OpenAI(base_url=endpoint, api_key=token)

          # --- 2. DOSYALARI BULMA ---
          md_files = [f for f in os.listdir('.') if f.endswith('.md') and os.path.isfile(f)]

          if not md_files:
              print("İşlenecek .md dosyası bulunamadı.")
              sys.exit(0)

          print(f"Bulunan dosyalar: {md_files}")

          # --- 3. DÖNGÜ BAŞLIYOR ---
          for file_name in md_files:
              print(f"\n--- İşleniyor: {file_name} ---")

              # Link Şablonları
              header_root = f"{TAG_START}\n[ 🇹🇷 Türkçe ]({file_name}) | [ 🇺🇸 English ](translations/en/{file_name})\n{TAG_END}\n"
              
              header_en = f"{TAG_START}\n[ 🇹🇷 Türkçe ](../../{file_name}) | [ 🇺🇸 English ]({file_name})\n{TAG_END}\n"

              # Dosyayı Oku
              try:
                  with open(file_name, "r", encoding="utf-8") as f:
                      content = f.read()
              except Exception as e:
                  print(f"::error::{file_name} okunamadı: {e}")
                  continue

              # Ana Dosyaya Link Ekleme
              if TAG_START in content:
                  # Regex yerine düz değiştirme yapıyoruz, çünkü regex özel karakterlerde hata verebilir
                  # Basit mantık: Start ve End arasını sil, yenisini koy.
                  # Ancak regex daha temizdir, sadece değişkenleri escape edelim.
                  pattern = re.escape(TAG_START) + r".*?" + re.escape(TAG_END)
                  content = re.sub(pattern, header_root.strip(), content, flags=re.DOTALL)
              else:
                  content = header_root.strip() + "\n\n" + content

              with open(file_name, "w", encoding="utf-8") as f:
                  f.write(content)

              # --- ÇEVİRİ KISMI (Hata veren yer burasıydı) ---
              
              # Split etmeden önce kontrol ediyoruz
              parts = content.split(TAG_END)
              
              if len(parts) > 1:
                  text_to_translate = parts[-1].strip()
              else:
                  # Eğer split çalışmazsa (etiket yoksa) tüm içeriği al
                  text_to_translate = content.replace(header_root.strip(), "").strip()

              if not text_to_translate:
                  print(f"UYARI: {file_name} içeriği boş veya sadece linklerden oluşuyor.")
                  continue

              # Yapay Zeka Çağrısı
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
                  
                  # İngilizce Kaydetme
                  final_english_content = header_en.strip() + "\n\n" + translated_body
                  
                  os.makedirs("translations/en", exist_ok=True)
                  output_path = f"translations/en/{file_name}"
                  
                  with open(output_path, "w", encoding="utf-8") as f:
                      f.write(final_english_content)
                      
                  print(f"✅ {file_name} başarıyla çevrildi.")

              except Exception as e:
                  print(f"::error::{file_name} çevrilirken hata: {e}")
                  continue

      - name: GitHub'a Gönder (Push)
        run: |
          git config --global user.name "github-actions[bot]"
          git config --global user.email "github-actions[bot]@users.noreply.github.com"
          git add .
          git commit -m "🤖 Tüm Belgeler Çevrildi (Fix)" || echo "Değişiklik yok"
          git push
```

### Adım 4: Yayınlama (Publish)

VS Code terminalinden dosyalarınızı GitHub'a gönderin:

```bash
git add .
git commit -m "Sistem kurulumu tamamlandi"
git push -u origin main
```

---

## 🚦 Nasıl Kullanılır ve Test Edilir?

Sistem tamamen otomatiktir.

1.  Reponuzda herhangi bir `.md` dosyasını (`README.md`, `LICENSE.md` vb.) düzenleyin veya yeni bir markdown dosyası oluşturun.
2.  Değişikliği kaydedip gönderin (`git push`).
3.  GitHub reponuzda **Actions** sekmesine tıklayın.

### Actions Ekranında Görecekleriniz
1.  **Sarı Daire:** İşlem başladı.
2.  **Logs:** İşleme tıkladığınızda `Bulunan dosyalar: ['README.md', 'CONTRIBUTING.md']` gibi bir liste göreceksiniz. Script hepsini sırayla işler.
3.  **Yeşil Tık (✅):** Tamamlandığında, ana dizindeki dosyalarınızın tepesinde linkler belirir ve `translations/en/` klasöründe İngilizce versiyonları oluşur.

---

## ❓ Sıkça Sorulan Sorular (FAQ)

**S: İngilizce çeviriyi elle düzeltebilir miyim?**
C: Hayır. `translations` klasörü her çalışmada **otomatik olarak üzerine yazılır**. Düzeltmeleri Türkçe ana dosyada yapmalısınız.


**S: Yeni bir dosya eklersem ne olur?**
C: Örneğin `YENI_BELGE.md` eklerseniz, sistem bir sonraki çalışmada onu otomatik algılar, link ekler ve `translations/en/YENI_BELGE.md` olarak çevirisini oluşturur.

**S: Neden `.env` dosyası yok?**
C: API anahtarlarını kod içinde tutmak güvensizdir. GitHub Secrets özelliği, çalışma anında sanal bir ortam değişkeni oluşturarak güvenliği sağlar.
