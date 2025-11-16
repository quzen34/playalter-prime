# PlayAlter – Blueprint V2 (Foundational Spec)

> **Purpose:** PlayAlter is a privacy-first “face layer” for the internet.  
> Creators can earn money (videos, photos, livestreams, calls) **without ever showing their real face**, using AI-generated personas and anonymization.

This document is the **canonical, high-level blueprint** for the project.  
Detaylar değişebilir, buradaki prensipler **değişmez anayasa** olarak kabul edilir.

---

## 0. Positioning & Reality Check

### 0.1 Problem

- Birçok içerik üreticisi:
  - Yüzünü göstermek istemiyor (güvenlik, aile, iş, gizlilik).
  - Buna rağmen **para kazanmak** istiyor.
- Mevcut çözümler:
  - Blur / mozaik: amatör ve zaman kaybı.
  - Klasik face swap: çoğu gerçek insanlar üzerinden, hukuki risk yaratıyor.
  - Mobil app’ler (ör. Pseudoface) genelde:
    - Telefon + tek platform odaklı,
    - Kapalı kutu, **SaaS/API tarafı zayıf**.

### 0.2 Solution (PlayAlter)

PlayAlter:

- **Sentetik maskeler** üretir (gerçek kişilere ait olmayan yüzler).
- Bu maskeleri:
  - Fotoğraf / video üzerinde uygular,
  - Canlı yayın sırasında takar,
  - Yasal ve etik sınırlar içinde tam anonimlik sağlar.
- Tamamen **web-tabanlı SaaS** + **API** + **Modal GPU compute**:
  - Tarayıcı, mobil, masaüstü, B2B entegrasyonlara açık,
  - Ölçeklenebilir, usage-based fiyatlandırma.

---

## 1. Core Concepts (3 Ana Core)

PlayAlter üç çekirdek moda ayrılır:

1. 🕶️ **Nohma – Full Anonymize Mode**
   - Amaç: Gerçek yüzleri tamamen anonim yapmak.
   - Çıktıdaki yüzler **hiç kimse değil**; geri döndürülemez.
   - Kullanım:
     - Hukuki/etik gereklilik,
     - Hassas içerik,
     - “Ben hiç görünmeyeyim” diyen kullanıcılar.

2. 🎭 **Reikuro – Persona Mode**
   - Amaç: Kullanıcı için **kalıcı, sentetik bir persona** (maske) oluşturmak.
   - Bu persona:
     - Selfie’ye kabaca benzer (yaş, cinsiyet, vibe),
     - Ama gerçek bir insan değil.
   - Kullanım:
     - Uzun vadeli içerik kimliği,
     - Marka/karakter oluşturmak,
     - Video/foto üzerinde persona maskesi ile görünmek.

3. ⚡ **HikariEdge – Live Session Mode**
   - Amaç: **Gerçek zamanlı maske** (stream / video call).
   - Kullanım:
     - Twitch, Kick, YouTube, cam siteleri, OF, Zoom tarzı canlı yayınlar.
   - Reikuro personasıyla veya anlık sentetik maskeyle çalışır.

**Kural:**  

- Reikuro maskeleri yeniden kullanılabilir (persona kimliği).  
- Nohma çıktılarını geri çözmeye çalışmak mümkün değildir.  
- HikariEdge, Reikuro maskelerini canlıya taşır.

---

## 2. High-Level Architecture

### 2.1 Overview

Mimari üç katmanlıdır:

1. **Edge Layer – Cloudflare**
2. **Control Plane – GCP (Cloud Run / Functions, Firestore, GCS)**
3. **Compute Plane – Modal (GPU Workers)**

---

### 2.2 Edge Layer – Cloudflare

- Domain: `playalter.com`
- Alt alan adları:
  - `api.playalter.com` – Public API (JSON/REST).
  - `app.playalter.com` – Web app / dashboard.
  - `gradio.playalter.com` – POC / internal testing UI.
  - Opsiyonel: `cdn.playalter.com` – sonuç dosyaları için CDN önbelleği.
- Cloudflare özellikleri:
  - **DNS + Anycast CDN** – Global hızlı erişim.
  - **WAF + DDoS koruması + Rate limiting** – API’lerin korunması.
  - **Cloudflare for SaaS** – İleride müşterilerin kendi domain’lerini bağlayabilmesi (white-label).

---

### 2.3 Control Plane – GCP

#### 2.3.1 Cloud Run / Cloud Functions

- **`upload-service` (Cloud Run/Functions)**
  - Görev:
    - Medya upload almak (dosya, URL, stream token).
    - Dosyayı GCS `uploads/` altına koymak.
    - Firestore `jobs` koleksiyonunda job dokümanı oluşturmak.
  - Giriş:
    - Auth’lu kullanıcı isteği.
  - Çıkış:
    - `job_id`.

- **`orchestrator-service` (Cloud Run/Functions)**
  - Görev:
    - Yeni job’ları dinlemek (Firestore trigger veya queue).
    - `mode` + `job_type` + `persona_id` alanlarına bakarak doğru Modal fonksiyonunu çağırmak.
    - Modal’dan gelen sonuçlara göre job `status` + `output_uri` güncellemek.

- **`status-service` (HTTP)**
  - Görev:
    - İstemcinin `job_id` ile job durumu sorgulayabilmesi.
  - Çıkış:
    - `status`, `output_uri`, hata mesajları.

- **`preprocess-service` (Cloud Run, CPU)**
  - Görev:
    - Girdi videoları karelere bölmek,
    - Yüz tespiti, landmark, bounding box, pose bilgisi önceden hesaplamak,
    - GPU maliyetini azaltmak (Modal worker sadece inference yapsın).

Cloud Run, container tabanlı bu servisleri tam yönetilen, otomatik ölçeklenen şekilde çalıştırır; istek yokken “scale-to-zero” olur.

#### 2.3.2 Firestore

Kullanım: Serverless NoSQL document DB, global erişim, otomatik ölçeklenme.

Temel koleksiyonlar:

- `users/{user_id}`
- `users/{user_id}/personas/{persona_id}`
- `jobs/{job_id}`
- `billing/{record_id}`
- `plans/{plan_id}` (statik konfigurasyon, opsiyonel)

Detay model 3. bölümde.

#### 2.3.3 Google Cloud Storage (GCS)

- `gs://playalter-uploads/` – ham user inputları.
- `gs://playalter-results/` – işlenmiş çıktı dosyaları.
- `gs://playalter-models/` – model ağırlıkları (Modal image build sürecinde senkronize edilir).

---

### 2.4 Compute Plane – Modal (GPU Workers)

Modal; Python tabanlı, serverless fonksiyonları GPU ile birlikte çalıştıran bir platformdur.

Her core için ayrı Modal uygulaması:

1. `reikuro_app`
   - `create_persona_fn` – sentetik maske üretimi.
   - `swap_media_fn` – video/foto üzerinde persona swap.
2. `nohma_app`
   - `anonymize_media_fn` – DeepPrivacy tarzı anonimleştirme.
3. `hikariedge_app`
   - `live_session_fn` – gerçek zamanlı swap + stream.

Modal tarafında genel prensipler:

- Her fonksiyon:
  - Uygun bir GPU tipi ile tanımlı (örn. `gpu="A10G"`).
  - Girdi olarak:
    - `job_id`, `input_uri`, `mode`, `job_type`, `persona_id?`, `config`.
- Çalışma adımları:
  - GCS’den input indir,
  - İlgili model stack’i yükle,
  - İşle,
  - Çıktıyı GCS `results/` altına yaz,
  - `output_uri` ve meta bilgileri geri dön.

---

## 3. Data Model (Firestore & Storage)

### 3.1 Firestore Collections

#### 3.1.1 `users/{user_id}`

Alanlar:

- `email`
- `auth_provider` (password, Google, etc.)
- `plan_id`
- `created_at`, `last_login_at`
- `usage_counters`:
  - `personas_created`
  - `media_jobs_run`
  - `live_minutes_used`
- `flags`:
  - `is_banned`
  - `requires_manual_review`

#### 3.1.2 `users/{user_id}/personas/{persona_id}`

- `display_name`
- `style_tags` (örn: `"soft"`, `"sharp"`, `"anime"`, `"realistic"`)
- `ref_image_uri` (GCS path)
- `embedding_ref` veya inline `embedding_vector`
- `generator_version` (örn: `"stylegan3_v1"`)
- `created_at`
- `is_default`
- `is_deleted`
- Plan bazlı limit kontrolü için: `storage_bytes`, `usage_count`.

#### 3.1.3 `jobs/{job_id}`

- `user_id`
- `mode` – `"reikuro" | "nohma" | "hikariedge"`
- `job_type` – örn:
  - `"create_persona"`
  - `"swap_image"`
  - `"swap_video"`
  - `"anonymize_image"`
  - `"anonymize_video"`
  - `"live_session_record"`
- `persona_id` (opsiyonel)
- `input_uri`
- `output_uri` (işlem bitince dolar)
- `status` – `"queued" | "running" | "done" | "error"`
- `progress` (0–100, opsiyonel)
- `error_code`, `error_message`
- `runtime_stats`:
  - `started_at`, `finished_at`
  - `modal_function`
  - `gpu_type`
  - `billed_seconds` / tahmini maliyet

#### 3.1.4 `billing/{record_id}`

- `user_id`
- `job_id`
- `plan_id`
- `price_usd`
- `units` (saniye, adet frame, adet persona, vs.)
- `created_at`
- Senkron Stripe invoice id vs.

#### 3.1.5 `plans/{plan_id}`

- `name`, `price_monthly`
- `max_personas`
- `max_storage_gb`
- `included_media_minutes`
- `included_live_minutes`
- `overage_price_per_min`

### 3.2 Google Cloud Storage Yapısı

- `gs://playalter-uploads/{user_id}/{job_id}/…`
  - Raw giriş dosyaları.
- `gs://playalter-results/{user_id}/{job_id}/…`
  - İşlenmiş sonuçlar.
- `gs://playalter-personas/{user_id}/{persona_id}/ref.png`
  - Persona referans görüntüleri.
- `gs://playalter-models/...`
  - Model ağırlıkları (detectors, generators, anonymizers).

---

## 4. Model Stack by Core

### 4.1 Ortak Bileşenler

- **Face Detection & Tracking**
  - RetinaFace / SCRFD / YOLOv8 vb. modeller.
- **Face Embeddings**
  - InsightFace benzeri encoder’lar.
- **Quality / Upscaling**
  - GFPGAN (face restoration),
  - RealESRGAN (upscale).

### 4.2 Reikuro – Persona & Swap

#### 4.2.1 Persona Creation (MaskFactory)

- Input: Selfie.
- İş akışı:
  1. Detector ile yüz + landmark + pose tespiti.
  2. InsightFace ile embedding çıkarma.
  3. Persona generator (StyleGAN3 veya diffusion tabanlı):
     - Latent uzayda örnekleme,
     - Orijinal embedding ile benzerlik kontrolü (çok yakınsa discard),
     - İstenilen attribute’lara (yaş bandı, cinsiyet, stil) göre filtre.
  4. 4–8 maske adayı üret,
  5. Her biri için preview imajını GCS’ye kaydet,
  6. Kullanıcı seçtiğinde `personas/{persona_id}` dokümanı yarat.

#### 4.2.2 Persona Swap (Media)

- Input: Video/foto + `persona_id`.
- İş akışı:
  1. Preprocess:
     - Videoyu karelere böl,
     - Frame başına yüz/landmark tespiti.
  2. Reikuro worker:
     - Persona ref embedding’i yükler,
     - InSwapper benzeri modelle frame frame swap,
     - GFPGAN / RealESRGAN ile kalite iyileştirme.
  3. Frame’leri tekrar video haline getir,
  4. `results/` altına kaydet, `job` güncelle.

### 4.3 Nohma – Full Anonymization

- Input: Video/foto.
- Model stack:
  - Detectors (yüz + opsiyonel vücut),
  - DeepPrivacy-tarzı face/body anonymizer,
  - GFPGAN / RealESRGAN (gerekirse).
- Çıktı:
  - Her yüz **tamamen yeni bir yüz** ile değiştirilir,
  - Hiçbir persona id yoktur, mapping tutulmaz.

### 4.4 HikariEdge – Live Masking

- Input: RTMP/WebRTC stream + `persona_id` veya anlık mask config.
- Model stack:
  - Lightweight face detector (low latency),
  - Matting / segmentation modelleri (MODNet, BiSeNet-CelebAMaskHQ),
  - FP16 optimize swap modeli,
  - Minimal restorasyon (low latency için).
- Çıktı:
  - Maskeli video stream → hedef platforma (veya kayıt için GCS’ye).

---

## 5. Main User Flows

### 5.1 Kayıt & Plan

1. Kullanıcı kayıt olur (email + password veya sosyal login).
2. Stripe üzerinden plan seçer (Free / Pro / Creator).
3. `users` ve `billing` kayıtları oluşturulur.

### 5.2 Persona Oluşturma (Reikuro – MaskFactory)

1. Kullanıcı selfielerini / kısa video yükler.
2. `upload-service`:
   - Dosyayı `uploads/` altına koyar,
   - `jobs` koleksiyonunda `mode="reikuro"`, `job_type="create_persona"` job’ı yaratır.
3. `orchestrator-service` Modal `create_persona_fn` fonksiyonunu çağırır.
4. Reikuro worker:
   - Sentetik maskeleri üretir,
   - Preview’ları GCS’ye yazar,
   - Firestore’da persona adaylarını döner.
5. Kullanıcı UI’dan maskesini seçer:
   - Seçilenler `personas/{persona_id}` altında kalıcı hale gelir,
   - Plan limitleri kontrol edilir.

### 5.3 Video/Foto İçerik Maskeleme (Reikuro / Nohma)

#### Reikuro (persona swap)

1. Kullanıcı bir `persona_id` seçer.
2. Video/foto yükler → `mode="reikuro"`, `job_type="swap_video"` veya `"swap_image"`.
3. İşlem tamamlandığında:
   - `results/` altından indirilebilir URL döner,
   - Kullanım `billing`’e yazılır.

#### Nohma (tam anonim)

1. Kullanıcı “full anonymous” modu seçer.
2. Video/foto yükler → `mode="nohma"`, `job_type="anonymize_*"`.
3. DeepPrivacy-tarzı worker anonimleştirir.

### 5.4 Live Streaming (HikariEdge)

1. Kullanıcı:
   - Mevcut `persona_id` seçer veya
   - “Random synthetic mask” işaretler (önce Reikuro çağrılır).
2. `hikariedge-session-service`:
   - Kullanıcıya giriş RTMP URL/token’ı geri döner.
3. Kullanıcı OBS / client ile videoyu bu adrese yollar.
4. `hikariedge_app` üzerindeki Modal worker:
   - Frame’leri alır,
   - Maskeyi uygular,
   - Çıkışı hedef RTMP veya WebRTC endpoint’ine iletir.
5. Opsiyonel: Yayın eş zamanlı olarak GCS’de kaydedilir.

---

## 6. SaaS & Monetization Layer

### 6.1 Pricing Model (Örnek Taslak)

- **Free**
  - 1 persona
  - Aylık kısıtlı swap süresi (ör. 10 dakika video)
  - Live yok / sınırlı deneme

- **Creator**
  - 5–10 persona
  - Daha fazla swap dakikası
  - Sınırlı live dakikası

- **Pro / Studio**
  - Çok daha yüksek limitler
  - B2B kullanım
  - Custom domain (Cloudflare for SaaS ile)

Faturalama:

- Stripe ile abonelik + usage-based ek ücret.
- Her `job` → `billing` kaydı, plan limit kontrolü.

### 6.2 Feature Set vs. Pseudoface

Pseudoface’in ana özellikleri:

- AI-mask ile anonimlik,
- Video/foto maskesi,
- Creator geliri artırma.

PlayAlter’in eklediği farklar:

- **Tam web-tabanlı SaaS + API**
  - Mobil app’e sıkışık değil.
- **Modal GPU ile ölçeklenebilir backend**
  - Birden çok modeli tek yerde koşturan serverless GPU platformu.
- **Persona yönetimi + B2B kullanımlar**
  - Birden fazla persona, takım account’ları, white-label imkânı.

---

## 7. Non-Functional Requirements

1. **Güvenlik**
   - Cloudflare WAF, DDoS ve rate limits aktif.
   - Tüm API’ler HTTPS, JWT-based auth.
   - RBAC: user / admin / support roller.

2. **Gizlilik**
   - Kullanıcı isterse:
     - Input medya otomatik silinebilir (örn. 7 gün sonra).
     - Personas’ını silebilir.
   - Nohma çıktıları için hiçbir yüz embedding’i tutulmaz.

3. **Gözlemlenebilirlik**
   - Job bazlı logging (Cloud Logging).
   - Modal fonksiyon metrikleri, invocation süreleri.
   - Billing ile ilişkilendirilen cost raporları.

4. **Maliyet Kontrolü**
   - Modal fonksiyonlarında:
     - Maksimum runtime,
     - Batching ve cache kullanımına dikkat.
   - Cloud Run için:
     - Min instance = 0,
     - Uygun concurrency.

---

## 8. Roadmap (Yüksek Seviye)

1. **Phase 0 – POC**
   - Tek persona core (Reikuro swap) için:
     - Basit upload → job → result akışı,
     - Modal üzerinde tek worker,
     - Firestore `jobs` + basit `users`.
   - Gradio arayüzü ile test.

2. **Phase 1 – MVP SaaS**
   - Cloudflare + `api/app` domainleri.
   - Auth, plan’lar, basit Stripe entegrasyonu.
   - Reikuro + Nohma destekli:
     - Image + kısa video.
   - Dashboard:
     - Job listesi, persona listesi, basit usage grafikleri.

3. **Phase 2 – Live (HikariEdge)**
   - RTMP giriş ve çıkış pipeline’ı.
   - Reikuro personası ile live swap.
   - İlk ücretli planlar ve beta kullanıcılar.

4. **Phase 3 – Büyüme & B2B**
   - White-label domain (Cloudflare for SaaS).
   - Team account’lar, API anahtarları, rate limits.
   - Gelişmiş logging, SOC-friendly export.

---

## 9. Mental Checklist (Para Kazanma İçin Gerekli Ana Taşlar)

Bu blueprint, projenin gerçekten para kazanan bir SaaS olması için şu kritik taşları içeriyor:

- ✅ Net müşteri problemi (anonim ama kazanç odaklı creator’lar).
- ✅ Ürünü taşıyan 3 core (Nohma, Reikuro, HikariEdge).
- ✅ SaaS mimarisi:
  - Cloudflare edge,
  - GCP control plane (Cloud Run + Firestore + GCS),
  - Modal GPU compute.
- ✅ Veri modeli:
  - users / personas / jobs / billing / plans.
- ✅ Kullanıcı akışları:
  - Persona yaratma,
  - Medya maskeleme,
  - Live yayın.
- ✅ Ücretlendirme planları ve usage tracking.
- ✅ Güvenlik, gizlilik ve maliyet kontrol prensipleri.

Bu doküman, ileride detaylar değişse bile **PlayAlter için referans “BluePrintV2.md”** olarak kullanılmalıdır.
