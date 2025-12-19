# 🌍 GitHub Models ile Otomatik Markdown Çevirmeni

Bu proje, GitHub üzerindeki Markdown (`.md`) dosyalarını **GitHub Models (GPT-4o)** kullanarak otomatik olarak İngilizceye (veya diğer dillere) çeviren bir otomasyon sistemidir.

Bu rehber; lokal kurulumdaki `PATH` ve `Permission` sorunlarıyla boğuşmadan, işlemlerin **tamamen GitHub sunucularında (Cloud)** nasıl güvenli ve ücretsiz bir şekilde yapılacağını anlatır.

---

## 🚀 Nasıl Çalışır?

1.  Siz Türkçe bir `.md` dosyası yazar ve GitHub'a yüklersiniz (`git push`).
2.  GitHub Actions devreye girer.
3.  Microsoft Azure üzerindeki **GPT-4o** modeline bağlanır.
4.  Dosyanızı çevirir ve `translations/` klasörü altına otomatik olarak kaydeder.
5.  Dosyanın en başına otomatik olarak dil seçeneklerini (Linkleri) ekler.

---

## 🛠️ Kurulum (Adım Adım)

### 1. Token (Anahtar) Alma
Sistemin çalışması için GitHub Models'e erişim izni gerekir.

1.  [GitHub Marketplace - Models](https://github.com/marketplace/models) sayfasına gidin (Veya Settings > Developer Settings).
2.  **Personal Access Token (Tokens - classic veya Fine-grained)** oluşturun.
3.  Token'ı kopyalayın (`github_pat_...` ile başlar).
    * ⚠️ **ÖNEMLİ UYARI:** Bu token'ı asla kodların içine, `.env` dosyasına veya `README` dosyasına yapıştırıp GitHub'a yüklemeyin. GitHub güvenlik sistemi (Push Protection) bunu engeller veya token'ı iptal eder.

### 2. Token'ı GitHub'a Tanımlama (Secret)
Anahtarı güvenli kasaya koymalıyız.

1.  Bu reponun **Settings** sekmesine gidin.
2.  Sol menüden **Secrets and variables** > **Actions** kısmına tıklayın.
3.  **New repository secret** butonuna basın.
    * **Name:** `OPENAI_API_KEY`
    * **Secret:** `(Kopyaladığınız github_pat_... kodunu buraya yapıştırın)`
4.  **Add secret** diyerek kaydedin.

### 3. Otomasyon Dosyasını Oluşturma
Reponuzda şu klasör yolunu oluşturun: `.github/workflows/`

Bu klasörün içine `cevirmen.yml` adında bir dosya açın ve şu kodları yapıştırın:

```yaml
name: AI Translator

on:
  push:
    branches: [ "main" ] # Ana dalınızın adı
    paths:
      - '**.md' # Sadece yazı dosyaları değişince çalışır

permissions:
  contents: write # Repoya dosya ekleyebilmesi için gerekli izin

jobs:
  translate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Python Kurulumu
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Aracı Yükle
        run: pip install co-op-translator

      - name: Çeviriyi Başlat
        env:
          # Secret olarak kaydettiğimiz anahtarı buradan çekiyoruz
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
          OPENAI_BASE_URL: '[https://models.inference.ai.azure.com](https://models.inference.ai.azure.com)'
          OPENAI_MODEL: 'gpt-4o'
        # Hedef dilleri buraya ekleyebilirsiniz (Örn: -l "en de fr")
        run: translate -l "en"

      - name: Kaydet ve Gönder
        run: |
          git config --global user.name "github-actions[bot]"
          git config --global user.email "github-actions[bot]@users.noreply.github.com"
          git add .
          git commit -m "🤖 Çeviri tamamlandı" || exit 0
          git push
```

---

## 💡 Tecrübe Notları (Neden Böyle Yaptık?)

Bu projeyi geliştirirken edindiğimiz kritik tecrübeler:

1.  **Lokal vs. Cloud:**
    * ❌ **Lokal (Bilgisayarda) Kurulum:** Windows ortamında Python `PATH` ayarları, yönetici izinleri ve dosya okuma (path) sorunları çok fazla zaman kaybettirebiliyor. Ayrıca `.env` dosyası oluşturmak ve bunu yanlışlıkla GitHub'a göndermek büyük güvenlik riski oluşturuyor.
    * ✅ **GitHub Actions (Cloud):** Sanal bir Linux makinesi her seferinde sıfırdan kurulur, işini yapar ve kapanır. `PATH` sorunu yoktur, izin sorunu yoktur. En temiz ve stabil yöntem budur.

2.  **Güvenlik (Secret Scanning):**
    * `.env` dosyasını `.gitignore` dosyasına eklemeden asla `git push` yapmayın.
    * Eğer yanlışlıkla şifreyi push ederseniz, GitHub bunu yakalar ve yüklemenize izin vermez (Push Protection). Bu durumda en temiz yol, repoyu silip sıfırdan başlamak veya git geçmişini (history) temizlemektir.

3.  **Model Seçimi:**
    * Bu proje, standart OpenAI API yerine **GitHub Models (Azure)** altyapısını kullanır. Bu sayede GitHub kullanıcıları belirli limitler dahilinde GPT-4o modelini ücretsiz deneyimleyebilir.

---

## 🏁 Kullanım

1.  Repoya yeni bir `deneme.md` dosyası ekleyin (Türkçe içerik yazın).
2.  Dosyanın en tepesine `` ve `` etiketlerini eklemeyi unutmayın.
3.  `Commit` ve `Push` yapın.
4.  İşlem bitince linkler otomatik eklenecektir.