# Yerel AI Kodlama Yığını

6 GB VRAM'li bir Windows dizüstünde, internete hiçbir kod göndermeden, editör içinden komut verip kod yazdıran bir kurulum.

Buradaki tüm sürümler, komutlar ve ölçümler **12 Eylül 2026**'da aşağıdaki makinede doğrulandı.

| | |
|---|---|
| GPU | NVIDIA RTX 4050 Laptop |
| VRAM | 6141 MiB |
| RAM | 15.7 GB |
| CPU | Intel i7-13620H |
| OS | Windows 11 Pro |

---

## Yığın

Dört parça var: modeli çalıştıran uygulama, modelin kendisi, editör ve editörü modele bağlayan eklenti. Hepsi ücretsiz, hiçbiri hesap istemiyor.

| Bileşen | Ne işe yarar | Sürüm | Nereden |
|---|---|---|---|
| **LM Studio** | Modeli çalıştırır, OpenAI uyumlu API sunar | 0.4.24 | winget · `ElementLabs.LMStudio` |
| **qwen/qwen3.5-4b** | Modelin kendisi — kod + görüntü | Q4_K_M · 3.38 GB | LM Studio Hub · `lms get` |
| **VS Code** | Editör | 1.134 | winget · `Microsoft.VisualStudioCode` |
| **Cline** | Ajan eklentisi — dosya yazar, terminal çalıştırır | 4.1.17 | Marketplace · `saoudrizwan.claude-dev` |
| **Continue** | Daha hafif alternatif — sohbet, satır içi düzenleme | 2.0.0 | Marketplace · `Continue.continue` |

**Editör alternatifleri:** Zed (`ZedIndustries.Zed`, LM Studio'yu yerleşik destekler), VSCodium (`VSCodium.VSCodium`, telemetrisiz VS Code), Cursor (kendi hesabını ister). Cline ve Continue hepsinde çalışır.

---

## Belirleyici kısıt: VRAM bütçesi

Model seçimi zevk meselesi değil, aritmetik. Model ağırlıkları artı 32k bağlam için KV önbelleği karta sığmazsa fazlası sistem RAM'ine taşar ve üretim hızı çöker.

| Model @ 32k bağlam | Gereken | 6141 MiB karta |
|---|---|---|
| `qwen3.5-4b` | 4.57 GiB | ✅ tamamen sığıyor |
| `qwen3.5-9b` | 8.39 GiB | ❌ %32'si RAM'e taşıyor |

**Pratik sonuç:** 9B model yüklenir ve izole testte 10.9 tok/s yapar — ama ajan konuşması büyüdükçe **1.95 tok/s**'e düşer, istekler 350 saniyeyi aşar ve eklenti bağlantıyı keser. 4B tamamen GPU'da kaldığı için hız sabit kalır. Bu kartta 4B doğru cevap.

---

## Kurulum

### 1. LM Studio'yu kur

```powershell
winget install --id ElementLabs.LMStudio -e --source winget --silent `
  --accept-package-agreements --accept-source-agreements
```

`--source winget` şart. Bu bayrak olmadan msstore kaynağı sertifika hatası verip komutu belirsizlik yüzünden düşürebiliyor.

### 2. Uygulamayı bir kez elle aç

`lms` komut satırı aracı, uygulama en az bir kez açılıp `~/.lmstudio` dizinini oluşturmadan çalışmaz. Açılışta sunucu da kendiliğinden kalkar.

### 3. CLI'ı PATH'e ekle

```powershell
& "$env:LOCALAPPDATA\Programs\LM Studio\resources\app\.webpack\lms.exe" bootstrap
```

Bundan sonra yeni bir terminalde `lms` doğrudan çalışır.

### 4. Modeli indir ve yükle

```powershell
lms get qwen/qwen3.5-4b -y --gguf
lms load qwen/qwen3.5-4b -y --context-length 32768 --parallel 1 --gpu max
lms server start --port 1234
```

`--parallel 1` atlanmamalı. Varsayılan 4 eşzamanlı tahmin, 6 GB kartta KV önbelleğini dörde katlayıp belleği taşırır.

`--identifier` **kullanma** — model kimliği hub anahtarından farklı olursa, JIT yükleme aynı modelin ikinci bir kopyasını yükleyebilir.

### 5. Editörü ve eklentiyi kur

```powershell
winget install --id Microsoft.VisualStudioCode -e --source winget --silent `
  --accept-package-agreements --accept-source-agreements

code --install-extension saoudrizwan.claude-dev --force
code --install-extension Continue.continue --force
```

### 6. Eklentiyi modele bağla

Aşağıdaki config dosyalarını yaz. Cline için ayrıca eklenti panelinden sağlayıcıyı bir kez seçmen gerekiyor — sebebi [Tuzaklar](#tuzaklar) bölümünde.

---

## Yapılandırma

Her eklentinin kendi dosyası var ve üçü de **ev dizininde** duruyor, editörün içinde değil. Bu yüzden aynı dosyalar VS Code, Cursor ve VSCodium tarafından paylaşılır — bir kez yazarsan hepsinde geçerli olur.

### Cline

`%USERPROFILE%\.cline\data\globalState.json`

```json
"planModeApiProvider": "lmstudio",
"actModeApiProvider": "lmstudio",
"planModeLmStudioModelId": "qwen/qwen3.5-4b",
"actModeLmStudioModelId": "qwen/qwen3.5-4b",
"lmStudioBaseUrl": "http://localhost:1234",
"lmStudioMaxTokens": "32768",
"planModeReasoningEffort": "low",
"actModeReasoningEffort": "low"
```

Base URL **çıplak** yazılır — Cline `/v1`'i kendi ekler. Mevcut dosyaya bu anahtarları ekle, üzerine tam dosya yazma; diğer ayarların orada duruyor.

### Continue

`%USERPROFILE%\.continue\config.yaml`

```yaml
models:
  - name: Local Qwen3.5 4B (LM Studio)
    provider: lmstudio
    model: qwen/qwen3.5-4b
    apiBase: http://localhost:1234/v1
    contextLength: 32768
    roles: [chat, edit, apply]
    capabilities: [tool_use, image_input]
```

Burada `/v1` **gerekli** — Cline'ın tersi. `image_input` olmadan görsel gönderemezsin.

### Zed

`%APPDATA%\Zed\settings.json`

```json
{
  "language_models": {
    "lmstudio": { "api_url": "http://localhost:1234/api/v0" }
  },
  "agent": {
    "default_model": { "provider": "lmstudio", "model": "qwen/qwen3.5-4b" }
  }
}
```

Üçüncü bir varyant: Zed ne `/v1` ne çıplak kök istiyor, `/api/v0` yani LM Studio'nun kendi REST API'si.

---

## Çalıştığını doğrulama

Editörü açıp denemeden önce bunları çalıştır. Üçü de geçiyorsa sorun editör tarafındadır, modelde değil.

```powershell
# 1. Sunucu ayakta ve model yuklu mu
lms ps

# 2. API model listesi donduruyor mu
curl http://localhost:1234/api/v0/models

# 3. En kritigi: tool calling calisiyor mu.
#    Govdeyi dosyadan ver - PowerShell satir ici JSON'u bozar.
curl -X POST http://localhost:1234/v1/chat/completions `
  -H "Content-Type: application/json" -d "@test.json"
```

Üçüncü testte yanıtta `"finish_reason": "tool_calls"` görmen gerekiyor. Model düz metinle cevap veriyorsa ajan eklentisi **hiç çalışmaz** — dosya yazma, terminal komutu, hepsi tool calling üzerinden gider.

<details>
<summary><code>test.json</code> içeriği</summary>

```json
{
  "model": "qwen/qwen3.5-4b",
  "messages": [
    { "role": "user", "content": "What is the weather in Istanbul? Use the tool." }
  ],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "Get the current weather for a city",
        "parameters": {
          "type": "object",
          "properties": { "city": { "type": "string" } },
          "required": ["city"]
        }
      }
    }
  ],
  "tool_choice": "auto",
  "stream": false,
  "max_tokens": 256
}
```
</details>

---

## Tuzaklar

Kurulum sırasında karşılaşılan ve sebebi hemen belli olmayan sorunlar. Çoğu sessizce başarısız oluyor.

### 🔴 BodyTimeoutError (UND_ERR_BODY_TIMEOUT)

Ajan birkaç tur sonra bağlantıyı kesiyor. Timeout ayarı sorunu değil: model karta sığmadığı için, konuşma büyüdükçe üretim 1.95 tok/s'e düşüyor ve istek 350 saniyeyi aşıyor.

**Çözüm:** tamamen VRAM'e sığan bir modele geç. Teşhis için LM Studio sunucu logundaki `eval time` satırlarına bak — gerçek tok/s orada yazıyor.

### 🟡 Cline ayarları geri alıyor

Çalışan Cline, bellek durumunu `globalState.json` üzerine yazıyor. Editör açıkken dosyayı düzenlersen değişiklik bir süre sonra kayboluyor — model kimliği eski değerine dönüyor.

**Çözüm:** dosyayı editör **kapalıyken** yaz, ya da sağlayıcıyı bir kez Cline panelinden seç. Panelden seçince eklentinin kendi durumu da güncellenir ve kalıcı olur.

### 🟡 JIT ikinci kopya yüklüyor

`justInTimeModelLoading` açıkken, bilinmeyen bir model kimliği için gelen istek modeli varsayılan ayarlarla otomatik yükler. `--identifier` ile özel isim verdiysen, hub anahtarına gelen istek **ikinci bir kopya** yükler ve kart taşar.

**Çözüm:** `--identifier` kullanma, kimlik hub anahtarıyla aynı kalsın. `lms ps` ile tek kopya olduğunu doğrula.

### 🟡 PowerShell config dosyalarını bozuyor

`Out-File -Encoding utf8` ve `Set-Content -Encoding utf8`, PowerShell 5.1'de dosyanın başına BOM ekler. Node'un `JSON.parse`'ı buna takılır; Cline dosyayı boş okuyup **üzerine yazabilir**.

**Çözüm:**

```powershell
[IO.File]::WriteAllText($p, $t, (New-Object System.Text.UTF8Encoding $false))
```

ya da dosyayı Bash/Node ile yaz.

### 🟡 Base URL her eklentide farklı

| Eklenti | Base URL |
|---|---|
| Cline | `http://localhost:1234` (çıplak kök) |
| Continue | `http://localhost:1234/v1` |
| Zed | `http://localhost:1234/api/v0` |

Yanlış olanı yazarsan bağlantı sessizce kurulmaz ya da `/v1/v1` gibi geçersiz bir adrese gider.

### 🟡 GUI uygulamaları betikten açılmıyor

Bir otomasyon betiğinden `Start-Process` ile açılan GUI uygulamaları (LM Studio dahil) saniyeler içinde ölebiliyor. `explorer.exe "yol\uygulama.exe"` ile açmak işe yarıyor — uygulama kabuğun bağlamında başlatılıyor.

---

## Ölçümler

| Ölçüm | qwen3.5-4b | qwen3.5-9b |
|---|---|---|
| 2000 token üretimi | **63.9 s** | 183.7 s |
| Tek dosya kod yazma (tool call) | **18.5 s** | 39.5 s |
| Model yükleme | **6.6 s** | 34.8 s |
| Kalan boş VRAM | **914 MiB** | 95 MiB |
| Prompt işleme (prefill) | ~720 tok/s | ~720 tok/s |
| Ajan konuşmasında üretim | **sabit** | 1.95 tok/s'e düşer |

Prompt işleme her iki modelde de hızlı — darboğaz orada değil, token üretiminde. Bu yüzden "prompt çok uzun" diye bağlamı kısmak sorunu çözmez; modelin karta sığması çözer.

### Beklentiler

- **İyi çalıştığı işler:** tek dosyalık fonksiyon yazma, test üretme, refactor, hata mesajı yorumlama, ekran görüntüsü okuma.
- **Zorlandığı işler:** çok dosyalı mimari değişiklikler, uzun ajan döngüleri, belirsiz talimatlar. 4B model birkaç adımdan sonra dağılabilir.
- **Kod kalitesi:** çalışır kod üretiyor ama gözden geçirmen gerekiyor. Test sırasında ürettiği `slugify` fonksiyonunda regex bayrağı eksikti — kod çalışıyordu, sadece yanlış çalışıyordu.
- **Görüntü:** model `vlm` tipinde. Test görselindeki şekilleri, renkleri ve metni doğru okudu, ama olmayan bir detay uydurdu.

---

## Bakım

- **LM Studio açık olmalı.** Kapalıyken hiçbir eklenti bağlanamaz. Sunucu ayarı `autoStartOnLaunch` olduğu için uygulamayı açman yeterli.
- **Model silme:** `lms` setinde silme komutu yok. Model klasörünü elle kaldır: `%USERPROFILE%\.lmstudio\models\lmstudio-community\<model>`
- **Disk:** her model 3–7 GB. İndirmeden önce yer olduğundan emin ol; yarım kalan indirmeler `.part` uzantısıyla yer kaplamaya devam eder.
- **Model değiştirme:** `lms unload --all` sonra yeni modeli yükle, ardından her iki config dosyasındaki model adını güncelle. Eski ad kalırsa eklenti "model bulunamadı" hatası verir.

---

## Bu repoda

| Dosya | Ne |
|---|---|
| `README.md` | Bu rehber |
| `yerel-ai-kurulum.html` | Aynı rehberin tek dosyalık HTML sürümü |

Farklı bir GPU'da VRAM bütçesi bölümündeki aritmetiği kendi kartına göre yeniden yap — model seçimi oradan çıkar.
