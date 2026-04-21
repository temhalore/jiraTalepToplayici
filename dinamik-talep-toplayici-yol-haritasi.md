# Dinamik Talep Toplayıcı Sistemi — Yol Haritası

> **Versiyon:** 1.0  
> **Mimari:** Angular (Frontend) + .NET Core (Backend)  
> **Hazırlayan:** Claude  
> **Amaç:** Bu döküman, ileride projelerin analiz edilerek geliştirmenin projeye uyumlu şekilde uygulanabilmesi için referans kaynağıdır.

---

## İÇİNDEKİLER

1. [Genel Mimari](#1-genel-mimari)
2. [Veritabanı Tasarımı](#2-veritabanı-tasarımı)
3. [t_kod Tablosu — Enum Tanımları](#3-t_kod-tablosu--enum-tanımları)
4. [Backend Enum'ları](#4-backend-enumları)
5. [Alan JSON Şeması](#5-alan-json-şeması)
6. [Backend — DTO'lar](#6-backend--dtolar)
7. [Backend — Servis ve Controller Akışı](#7-backend--servis-ve-controller-akışı)
8. [Backend — Dosya Yükleme (MinIO)](#8-backend--dosya-yükleme-minio)
9. [Angular — Model ve Enum'lar](#9-angular--model-ve-enumlar)
10. [Angular — Dinamik Form Builder](#10-angular--dinamik-form-builder)
11. [Angular — Template Yapısı](#11-angular--template-yapısı)
12. [Angular — Dosya Yükleme Bileşeni](#12-angular--dosya-yükleme-bileşeni)
13. [Güvenlik Kontrol Listesi](#13-güvenlik-kontrol-listesi)
14. [Sprint Planı](#14-sprint-planı)
15. [Proje Entegrasyon Notları](#15-proje-entegrasyon-notları)

---

## 1. Genel Mimari

```
┌─────────────────────────────┐     ┌─────────────────────────────┐
│   Angular — Admin Modülü    │     │  Angular — Public Talep     │
│                             │     │                             │
│  UygulamaTanim Liste/Form   │     │  /talep/:appKey             │
│  Alan Editörü (drag-drop)   │     │  Dinamik Form               │
│  JSON Preview               │     │  reCAPTCHA v3               │
└────────────┬────────────────┘     └────────────┬────────────────┘
             │                                   │
             ▼                                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                        .NET Core API                            │
│                                                                 │
│  /api/admin/uygulamaTanim  (CRUD — yetkili)                     │
│  /api/talep/getUygulamaBilgi/:appKey  (public)                  │
│  /api/talep/dosyaYukle  (public — MinIO)                        │
│  /api/talep/talepGonder  (public — reCAPTCHA korumalı)          │
└──────────────┬──────────────────────────┬───────────────────────┘
               │                          │
               ▼                          ▼
    ┌─────────────────┐        ┌─────────────────────┐
    │   Veritabanı    │        │   Jira Entegrasyonu │
    │ UygulamaTanim   │        │  (Mevcut JiraService│
    │ t_kod           │        │   değişmez)         │
    └─────────────────┘        └─────────────────────┘
```

### Temel Prensipler

- **JiraService değişmez.** `CreateIssue(CreateJiraIssueRequestDto)` metodu olduğu gibi kullanılır. Bu sisteme bağlanmak için sadece dinamik form verisi `CreateJiraIssueRequestDto`'ya dönüştürülür.
- **JiraApiKey asla frontend'e gönderilmez.** Sadece backend'de, `UygulamaTanim` tablosundan alınır.
- **Alan tipleri string değil enum (t_kod) ile yönetilir.** Tüm karşılaştırmalar `KodDTO.id` üzerinden yapılır.
- **Dosyalar MinIO'ya yüklenir,** Jira'ya sadece URL gönderilir.

---

## 2. Veritabanı Tasarımı

### `UygulamaTanim` Tablosu

```sql
CREATE TABLE UygulamaTanim (
    Id                INT IDENTITY(1,1) PRIMARY KEY,
    AppKey            NVARCHAR(100)    NOT NULL UNIQUE,  -- URL'de kullanılır: /talep/mezun
    AppAdi            NVARCHAR(200)    NOT NULL,
    Baslik            NVARCHAR(500)    NULL,              -- Form üstü açıklama metni
    LogoUrl           NVARCHAR(500)    NULL,
    AnaAlanlarJson    NVARCHAR(MAX)    NOT NULL,          -- Sabit alanlar (ad, email, konu, mesaj vb.)
    EkAlanlarJson     NVARCHAR(MAX)    NULL,              -- Uygulama bazlı ek alanlar
    JiraApiKey        NVARCHAR(500)    NOT NULL,          -- Şifreli saklanmalı, frontend'e GÖNDERİLMEZ
    StatuKodId        BIGINT           NOT NULL DEFAULT 2120001,  -- UYGULAMA_TANIM_STATU
    OlusturmaTarihi   DATETIME         NOT NULL DEFAULT GETDATE(),
    GuncellemeTarihi  DATETIME         NULL
);
```

### Alan İlişkisi

```
UygulamaTanim.StatuKodId  →  t_kod.Id  (TipId = 2120)
FormAlanDto.ModelTipiKodDto.Id  →  t_kod.Id  (TipId = 2100)
FormAlanDto.MapTipiKodDto.Id    →  t_kod.Id  (TipId = 2110)
```

---

## 3. t_kod Tablosu — Enum Tanımları

### SQL Insert'ler

```sql
-- ============================================================
-- TIP: FORM_ALAN_MODEL_TIPI (TipId = 2100)
-- Formda gösterilecek input türleri
-- ============================================================
INSERT INTO dbo.t_kod (Id, TipId, Sira, Kod, KisaAd, DigerUygulamaEnumAd, DigerUygulamaEnumDeger)
VALUES (2100, 0, 0, 'Form Alan Model Tipi', '', null, null);

INSERT INTO dbo.t_kod (Id, TipId, Sira, Kod, KisaAd, DigerUygulamaEnumAd, DigerUygulamaEnumDeger)
VALUES (2100001, 2100, 1, 'STRING',   'Metin',          null, null);
INSERT INTO dbo.t_kod (Id, TipId, Sira, Kod, KisaAd, DigerUygulamaEnumAd, DigerUygulamaEnumDeger)
VALUES (2100002, 2100, 2, 'EMAIL',    'E-Posta',        null, null);
INSERT INTO dbo.t_kod (Id, TipId, Sira, Kod, KisaAd, DigerUygulamaEnumAd, DigerUygulamaEnumDeger)
VALUES (2100003, 2100, 3, 'CEPTEL',   'Cep Telefonu',   null, null);
INSERT INTO dbo.t_kod (Id, TipId, Sira, Kod, KisaAd, DigerUygulamaEnumAd, DigerUygulamaEnumDeger)
VALUES (2100004, 2100, 4, 'COMBO',    'Açılır Liste',   null, null);
INSERT INTO dbo.t_kod (Id, TipId, Sira, Kod, KisaAd, DigerUygulamaEnumAd, DigerUygulamaEnumDeger)
VALUES (2100005, 2100, 5, 'TEXTAREA', 'Uzun Metin',     null, null);
INSERT INTO dbo.t_kod (Id, TipId, Sira, Kod, KisaAd, DigerUygulamaEnumAd, DigerUygulamaEnumDeger)
VALUES (2100006, 2100, 6, 'DOSYA',    'Dosya Eki',      null, null);

-- ============================================================
-- TIP: FORM_ALAN_MAP_TIPI (TipId = 2110)
-- Alanın CreateJiraIssueRequestDto'nun hangi property'sine map'leneceği
-- ============================================================
INSERT INTO dbo.t_kod (Id, TipId, Sira, Kod, KisaAd, DigerUygulamaEnumAd, DigerUygulamaEnumDeger)
VALUES (2110, 0, 0, 'Form Alan Map Tipi', '', null, null);

INSERT INTO dbo.t_kod (Id, TipId, Sira, Kod, KisaAd, DigerUygulamaEnumAd, DigerUygulamaEnumDeger)
VALUES (2110001, 2110, 1, 'SUBJECT',              'Konu Alanı',          null, null);
INSERT INTO dbo.t_kod (Id, TipId, Sira, Kod, KisaAd, DigerUygulamaEnumAd, DigerUygulamaEnumDeger)
VALUES (2110002, 2110, 2, 'DESCRIPTION',          'Açıklama Alanı',      null, null);
INSERT INTO dbo.t_kod (Id, TipId, Sira, Kod, KisaAd, DigerUygulamaEnumAd, DigerUygulamaEnumDeger)
VALUES (2110003, 2110, 3, 'REQUESTER_NAME',       'Talep Eden Ad',       null, null);
INSERT INTO dbo.t_kod (Id, TipId, Sira, Kod, KisaAd, DigerUygulamaEnumAd, DigerUygulamaEnumDeger)
VALUES (2110004, 2110, 4, 'REQUESTER_EMAIL',      'Talep Eden E-Posta',  null, null);
INSERT INTO dbo.t_kod (Id, TipId, Sira, Kod, KisaAd, DigerUygulamaEnumAd, DigerUygulamaEnumDeger)
VALUES (2110005, 2110, 5, 'REQUESTER_IDENTIFIER', 'Talep Eden TC/ID',    null, null);
INSERT INTO dbo.t_kod (Id, TipId, Sira, Kod, KisaAd, DigerUygulamaEnumAd, DigerUygulamaEnumDeger)
VALUES (2110006, 2110, 6, 'ADDITIONAL_DATA',      'Ek Veri (JSON)',      null, null);
INSERT INTO dbo.t_kod (Id, TipId, Sira, Kod, KisaAd, DigerUygulamaEnumAd, DigerUygulamaEnumDeger)
VALUES (2110007, 2110, 7, 'ATTACHMENT_URL',       'Dosya URL',           null, null);

-- ============================================================
-- TIP: UYGULAMA_TANIM_STATU (TipId = 2120)
-- ============================================================
INSERT INTO dbo.t_kod (Id, TipId, Sira, Kod, KisaAd, DigerUygulamaEnumAd, DigerUygulamaEnumDeger)
VALUES (2120, 0, 0, 'Uygulama Tanım Statü', '', null, null);

INSERT INTO dbo.t_kod (Id, TipId, Sira, Kod, KisaAd, DigerUygulamaEnumAd, DigerUygulamaEnumDeger)
VALUES (2120001, 2120, 1, 'AKTIF', 'Aktif', null, null);
INSERT INTO dbo.t_kod (Id, TipId, Sira, Kod, KisaAd, DigerUygulamaEnumAd, DigerUygulamaEnumDeger)
VALUES (2120002, 2120, 2, 'PASIF', 'Pasif', null, null);

-- ============================================================
-- TIP: DOSYA_YUKLEME_STATU (TipId = 2130)
-- ============================================================
INSERT INTO dbo.t_kod (Id, TipId, Sira, Kod, KisaAd, DigerUygulamaEnumAd, DigerUygulamaEnumDeger)
VALUES (2130, 0, 0, 'Dosya Yükleme Statü', '', null, null);

INSERT INTO dbo.t_kod (Id, TipId, Sira, Kod, KisaAd, DigerUygulamaEnumAd, DigerUygulamaEnumDeger)
VALUES (2130001, 2130, 1, 'YUKLENDI',    'Yüklendi',    null, null);
INSERT INTO dbo.t_kod (Id, TipId, Sira, Kod, KisaAd, DigerUygulamaEnumAd, DigerUygulamaEnumDeger)
VALUES (2130002, 2130, 2, 'YUKLENEMEDI', 'Yüklenemedi', null, null);
INSERT INTO dbo.t_kod (Id, TipId, Sira, Kod, KisaAd, DigerUygulamaEnumAd, DigerUygulamaEnumDeger)
VALUES (2130003, 2130, 3, 'BOYUT_ASIMI', 'Boyut Aşımı', null, null);
INSERT INTO dbo.t_kod (Id, TipId, Sira, Kod, KisaAd, DigerUygulamaEnumAd, DigerUygulamaEnumDeger)
VALUES (2130004, 2130, 4, 'TIP_HATASI',  'İzinsiz Tip', null, null);
```

---

## 4. Backend Enum'ları

> **Kural:** Enum değerleri her zaman `t_kod.Id` ile birebir eşleşir. Karşılaştırma kodda `(ENUM_ADI)kodDto.id` cast'i ile yapılır.

```csharp
namespace MEZUN.COMMON.DTO.Enums
{
    // t_kod TipId = 2100
    public enum FORM_ALAN_MODEL_TIPI
    {
        STRING   = 2100001,
        EMAIL    = 2100002,
        CEPTEL   = 2100003,
        COMBO    = 2100004,
        TEXTAREA = 2100005,
        DOSYA    = 2100006,
    }

    // t_kod TipId = 2110
    public enum FORM_ALAN_MAP_TIPI
    {
        SUBJECT              = 2110001,
        DESCRIPTION          = 2110002,
        REQUESTER_NAME       = 2110003,
        REQUESTER_EMAIL      = 2110004,
        REQUESTER_IDENTIFIER = 2110005,
        ADDITIONAL_DATA      = 2110006,
        ATTACHMENT_URL       = 2110007,
    }

    // t_kod TipId = 2120
    public enum UYGULAMA_TANIM_STATU
    {
        AKTIF = 2120001,
        PASIF = 2120002,
    }

    // t_kod TipId = 2130
    public enum DOSYA_YUKLEME_STATU
    {
        YUKLENDI    = 2130001,
        YUKLENEMEDI = 2130002,
        BOYUT_ASIMI = 2130003,
        TIP_HATASI  = 2130004,
    }
}
```

---

## 5. Alan JSON Şeması

`AnaAlanlarJson` ve `EkAlanlarJson` aynı şemayı paylaşır. `modelTipiKodDto` ve `mapTipiKodDto` her zaman tam `KodDTO` nesnesidir — int/string değil.

```json
[
  {
    "fieldKey": "requesterName",
    "label": "Ad Soyad",
    "modelTipiKodDto": {
      "id": 2100001,
      "kod": "STRING",
      "tipId": 2100,
      "kisaAd": "Metin"
    },
    "mapTipiKodDto": {
      "id": 2110003,
      "kod": "REQUESTER_NAME",
      "tipId": 2110,
      "kisaAd": "Talep Eden Ad"
    },
    "zorunlu": true,
    "maxLength": 200,
    "placeholder": "Adınızı giriniz",
    "sira": 1
  },
  {
    "fieldKey": "requesterEmail",
    "label": "E-Posta",
    "modelTipiKodDto": {
      "id": 2100002,
      "kod": "EMAIL",
      "tipId": 2100,
      "kisaAd": "E-Posta"
    },
    "mapTipiKodDto": {
      "id": 2110004,
      "kod": "REQUESTER_EMAIL",
      "tipId": 2110,
      "kisaAd": "Talep Eden E-Posta"
    },
    "zorunlu": true,
    "placeholder": "ornek@mail.com",
    "sira": 2
  },
  {
    "fieldKey": "talepTipi",
    "label": "Talep Tipi",
    "modelTipiKodDto": {
      "id": 2100004,
      "kod": "COMBO",
      "tipId": 2100,
      "kisaAd": "Açılır Liste"
    },
    "mapTipiKodDto": {
      "id": 2110006,
      "kod": "ADDITIONAL_DATA",
      "tipId": 2110,
      "kisaAd": "Ek Veri (JSON)"
    },
    "zorunlu": true,
    "comboItems": [
      { "value": "TEKNIK", "label": "Teknik Destek" },
      { "value": "BILGI",  "label": "Bilgi Talebi"  }
    ],
    "sira": 3
  },
  {
    "fieldKey": "konu",
    "label": "Konu",
    "modelTipiKodDto": {
      "id": 2100001,
      "kod": "STRING",
      "tipId": 2100,
      "kisaAd": "Metin"
    },
    "mapTipiKodDto": {
      "id": 2110001,
      "kod": "SUBJECT",
      "tipId": 2110,
      "kisaAd": "Konu Alanı"
    },
    "zorunlu": true,
    "maxLength": 500,
    "sira": 4
  },
  {
    "fieldKey": "mesaj",
    "label": "Mesaj",
    "modelTipiKodDto": {
      "id": 2100005,
      "kod": "TEXTAREA",
      "tipId": 2100,
      "kisaAd": "Uzun Metin"
    },
    "mapTipiKodDto": {
      "id": 2110002,
      "kod": "DESCRIPTION",
      "tipId": 2110,
      "kisaAd": "Açıklama Alanı"
    },
    "zorunlu": true,
    "minLength": 10,
    "maxLength": 1000,
    "sira": 5
  },
  {
    "fieldKey": "ekDosya",
    "label": "Dosya Ekle",
    "modelTipiKodDto": {
      "id": 2100006,
      "kod": "DOSYA",
      "tipId": 2100,
      "kisaAd": "Dosya Eki"
    },
    "mapTipiKodDto": {
      "id": 2110007,
      "kod": "ATTACHMENT_URL",
      "tipId": 2110,
      "kisaAd": "Dosya URL"
    },
    "zorunlu": false,
    "dosyaMaxBoyutMb": 5,
    "dosyaIzinliTipler": ["pdf", "jpg", "png", "docx"],
    "sira": 6
  }
]
```

### Alan Şeması Referansı

| Property | Tip | Açıklama |
|---|---|---|
| `fieldKey` | string | Form control adı, `FormData` key'i |
| `label` | string | Ekranda gösterim adı, AdditionalData'da alan başlığı |
| `modelTipiKodDto` | KodDTO | Input türü (STRING, EMAIL, COMBO, DOSYA...) |
| `mapTipiKodDto` | KodDTO | Jira DTO'sunun hangi alanına gideceği |
| `zorunlu` | bool | Frontend ve backend validasyonu |
| `maxLength` / `minLength` | int? | STRING ve TEXTAREA için |
| `placeholder` | string? | Input placeholder |
| `sira` | int | Form içindeki görüntülenme sırası |
| `comboItems` | list? | Sadece COMBO tipinde — `{value, label}` |
| `dosyaMaxBoyutMb` | int? | Sadece DOSYA tipinde |
| `dosyaIzinliTipler` | list? | Sadece DOSYA tipinde — örn: `["pdf","jpg"]` |

---

## 6. Backend — DTO'lar

### KodDTO (Mevcut model — değişmez)

```csharp
public class KodDTO
{
    public long id { get; set; }
    public string kod { get; set; }
    public long tipId { get; set; }
    public string kisaAd { get; set; }
    public string digerUygEnumAd { get; set; }
    public string digerUygEnumDeger { get; set; }
    public long sira { get; set; }
    public bool isAktif { get; set; }
    public KodDTO ustKodDTO { get; set; }
}
```

### ComboItemDto

```csharp
public class ComboItemDto
{
    public string Value { get; set; }
    public string Label { get; set; }
}
```

### FormAlanDto

```csharp
public class FormAlanDto
{
    public string FieldKey { get; set; }
    public string Label { get; set; }

    // KodDTO ile tip güvenliği — int/string KULLANILMAZ
    public KodDTO ModelTipiKodDto { get; set; }
    public KodDTO MapTipiKodDto { get; set; }

    public bool Zorunlu { get; set; }
    public int? MaxLength { get; set; }
    public int? MinLength { get; set; }
    public string Placeholder { get; set; }
    public int Sira { get; set; }

    // Sadece COMBO (2100004) tipinde dolu gelir
    public List<ComboItemDto> ComboItems { get; set; }

    // Sadece DOSYA (2100006) tipinde dolu gelir
    public int? DosyaMaxBoyutMb { get; set; }
    public List<string> DosyaIzinliTipler { get; set; }
}
```

### UygulamaBilgiResponseDto (GET endpoint — public)

```csharp
public class UygulamaBilgiResponseDto
{
    public string AppKey { get; set; }
    public string AppAdi { get; set; }
    public string Baslik { get; set; }
    public string LogoUrl { get; set; }
    public List<FormAlanDto> AnaAlanlar { get; set; }
    public List<FormAlanDto> EkAlanlar { get; set; }
    // JiraApiKey ASLA bu DTO'ya girmez
}
```

### DynamicTalepRequestDto (POST endpoint — public)

```csharp
public class DynamicTalepRequestDto
{
    [Required]
    public string AppKey { get; set; }

    [Required]
    public string RecaptchaToken { get; set; }

    // Formdan gelen tüm alan değerleri — key: fieldKey, value: giriş değeri
    public Dictionary<string, string> FormData { get; set; }
}
```

### DosyaYuklemeResponseDto

```csharp
public class DosyaYuklemeResponseDto
{
    public string Url { get; set; }
    public string DosyaAdi { get; set; }
    public int Statu { get; set; }  // DOSYA_YUKLEME_STATU enum değeri
}
```

---

## 7. Backend — Servis ve Controller Akışı

### UygulamaTanimService — Temel Metodlar

```csharp
public interface IUygulamaTanimService
{
    UygulamaBilgiResponseDto GetByAppKey(string appKey);
    List<UygulamaTanimListDto> GetAll();
    void Create(UygulamaTanimCreateDto dto);
    void Update(int id, UygulamaTanimUpdateDto dto);
    void Delete(int id);
}
```

### DynamicTalepService — Map Akışı

`talepGonder` endpoint'ine gelen `DynamicTalepRequestDto`, burada `CreateJiraIssueRequestDto`'ya dönüştürülür:

```csharp
private CreateJiraIssueRequestDto MapFormDataToJiraRequest(
    List<FormAlanDto> alanlar,
    Dictionary<string, string> formData)
{
    var jiraRequest = new CreateJiraIssueRequestDto();
    var additionalDict = new Dictionary<string, string>();

    foreach (var alan in alanlar)
    {
        if (!formData.TryGetValue(alan.FieldKey, out var deger)) continue;

        // KodDTO.id üzerinden enum cast
        var mapTipi = (FORM_ALAN_MAP_TIPI)alan.MapTipiKodDto.id;

        switch (mapTipi)
        {
            case FORM_ALAN_MAP_TIPI.SUBJECT:
                jiraRequest.Subject = deger; break;
            case FORM_ALAN_MAP_TIPI.DESCRIPTION:
                jiraRequest.Description = deger; break;
            case FORM_ALAN_MAP_TIPI.REQUESTER_NAME:
                jiraRequest.RequesterName = deger; break;
            case FORM_ALAN_MAP_TIPI.REQUESTER_EMAIL:
                jiraRequest.RequesterEmail = deger; break;
            case FORM_ALAN_MAP_TIPI.REQUESTER_IDENTIFIER:
                jiraRequest.RequesterIdentifier = deger; break;
            case FORM_ALAN_MAP_TIPI.ATTACHMENT_URL:
            case FORM_ALAN_MAP_TIPI.ADDITIONAL_DATA:
                // label ile birlikte AdditionalData'ya düşer
                additionalDict[alan.Label] = deger; break;
        }
    }

    if (additionalDict.Any())
        jiraRequest.AdditionalData = JsonConvert.SerializeObject(additionalDict);

    return jiraRequest;
}
```

### DynamicTalepService — Sunucu Tarafı Validasyon

```csharp
private void ValidateFormData(List<FormAlanDto> alanlar, Dictionary<string, string> formData)
{
    foreach (var alan in alanlar)
    {
        formData.TryGetValue(alan.FieldKey, out var deger);
        var modelTipi = (FORM_ALAN_MODEL_TIPI)alan.ModelTipiKodDto.id;

        if (alan.Zorunlu && string.IsNullOrWhiteSpace(deger))
            throw new OYSException(MessageCode.ERROR_500,
                $"{alan.Label} alanı zorunludur.");

        if (string.IsNullOrWhiteSpace(deger)) continue;

        switch (modelTipi)
        {
            case FORM_ALAN_MODEL_TIPI.EMAIL:
                var emailValid = EmailValidator.ValidateEmail(deger);
                if (!emailValid.isValid)
                    throw new OYSException(MessageCode.ERROR_500,
                        $"{alan.Label} geçerli bir e-posta adresi olmalıdır.");
                break;

            case FORM_ALAN_MODEL_TIPI.CEPTEL:
                if (!Regex.IsMatch(deger, @"^05[0-9]{9}$"))
                    throw new OYSException(MessageCode.ERROR_500,
                        $"{alan.Label} geçerli bir cep telefonu olmalıdır. (05XXXXXXXXX)");
                break;

            case FORM_ALAN_MODEL_TIPI.STRING:
            case FORM_ALAN_MODEL_TIPI.TEXTAREA:
                if (alan.MaxLength.HasValue && deger.Length > alan.MaxLength.Value)
                    throw new OYSException(MessageCode.ERROR_500,
                        $"{alan.Label} en fazla {alan.MaxLength} karakter olabilir.");
                if (alan.MinLength.HasValue && deger.Length < alan.MinLength.Value)
                    throw new OYSException(MessageCode.ERROR_500,
                        $"{alan.Label} en az {alan.MinLength} karakter olmalıdır.");
                break;
        }
    }
}
```

### TalepController

```csharp
// GET — Public
[HttpGet]
[Route("getUygulamaBilgi/{appKey}")]
[DirectAccess]
public IActionResult GetUygulamaBilgi(string appKey)
{
    var response = new ServiceResponse<UygulamaBilgiResponseDto>();
    try
    {
        var bilgi = _uygulamaTanimService.GetByAppKey(appKey);
        if (bilgi == null)
        {
            response.exceptionMessage = "Uygulama bulunamadı.";
            return Ok(response);
        }
        response.data = bilgi;
        return Ok(response);
    }
    catch (Exception ex)
    {
        OYSLog.Instance.SetLogType(CommonEnums.LOG_TYPE.USER)
            .SetLogIndexTip(CommonEnums.LOG_INDEX_TIP.GENEL_EXCEPTION)
            .Error($"GetUygulamaBilgi - Exception. Message:{ex.Message}");
        response.exceptionMessage = "Servis hatası.";
        return Ok(response);
    }
}

// POST — Public, reCAPTCHA + rate limit korumalı
[CustomIpRateLimit(maxRequests: 3, periodInSeconds: 300)]
[HttpPost]
[Route("talepGonder")]
[DirectAccess]
public IActionResult TalepGonder(DynamicTalepRequestDto dto)
{
    var response = new ServiceResponse<object>();

    // 1. reCAPTCHA
    bool recaptchaOk = false;
    try
    {
        recaptchaOk = _commonManager.VerifyRecaptcha(dto.RecaptchaToken);
    }
    catch (Exception recaptchaEx)
    {
        OYSLog.Instance.SetLogType(CommonEnums.LOG_TYPE.USER)
            .SetLogIndexTip(CommonEnums.LOG_INDEX_TIP.GENEL_EXCEPTION)
            .Error($"TalepGonder - Recaptcha exception. Message:{recaptchaEx.Message}");
        response.exceptionMessage = "Doğrulama sırasında hata oluştu.";
        return Ok(response);
    }

    if (!recaptchaOk)
    {
        response.exceptionMessage = "Doğrulama Hatası";
        response.messageType = ServiceResponseMessageType.Error;
        return Ok(response);
    }

    try
    {
        // 2. Uygulama tanımını al — JiraApiKey dahil (DB'den)
        var uygulama = _uygulamaTanimService.GetByAppKeyWithApiKey(dto.AppKey);
        if (uygulama == null)
            throw new OYSException(MessageCode.ERROR_500, "Geçersiz uygulama.");

        // 3. Tüm alanları birleştir (ana + ek)
        var tumAlanlar = uygulama.AnaAlanlar
            .Concat(uygulama.EkAlanlar ?? new List<FormAlanDto>())
            .ToList();

        // 4. Sunucu tarafı validasyon
        _dynamicTalepService.ValidateFormData(tumAlanlar, dto.FormData);

        // 5. Jira DTO'ya map'le
        var jiraRequest = _dynamicTalepService.MapFormDataToJiraRequest(
            tumAlanlar, dto.FormData);

        // 6. Mevcut JiraService — değişmez
        var headers = new Dictionary<string, string>
        {
            { "X-Api-Key", uygulama.JiraApiKey }  // Her uygulama kendi key'ini kullanır
        };
        var jiraResponse = _jiraService.CreateIssue(jiraRequest);

        response.data = jiraResponse;
        response.message = "Talebiniz başarıyla oluşturulmuştur. En yakın zamanda sizinle iletişime geçeceğiz.";
        return Ok(response);
    }
    catch (OYSException oysEx)
    {
        OYSLog.Instance.SetLogType(CommonEnums.LOG_TYPE.USER)
            .SetLogIndexTip(CommonEnums.LOG_INDEX_TIP.GENEL_EXCEPTION)
            .Error($"TalepGonder - OYSException. Message:{oysEx.oysMessage}");
        response = new ServiceResponse<object>(oysEx);
        return Ok(response);
    }
    catch (Exception ex)
    {
        OYSLog.Instance.SetLogType(CommonEnums.LOG_TYPE.USER)
            .SetLogIndexTip(CommonEnums.LOG_INDEX_TIP.GENEL_EXCEPTION)
            .Error($"TalepGonder - Exception. Message:{ex.Message}");
        response.exceptionMessage = ex.Message;
        return Ok(response);
    }
}
```

---

## 8. Backend — Dosya Yükleme (MinIO)

### Akış

```
Angular  →  dosyayı /api/talep/dosyaYukle endpoint'ine gönderir
Backend  →  boyut + uzantı kontrolü yapar
         →  MinIO'ya yükler (bucket: talep-{appKey})
         ←  MinIO public URL döner
Angular  →  URL'yi ilgili form control'e yazar
         →  talepGonder'de bu URL FormData içinde gider
Backend  →  URL, AdditionalData veya ATTACHMENT_URL map'i ile Jira'ya iletilir
```

### DosyaYukle Controller

```csharp
[HttpPost]
[Route("dosyaYukle")]
[DirectAccess]
[RequestSizeLimit(10_000_000)]  // 10MB hard limit
public async Task<IActionResult> DosyaYukle(
    IFormFile dosya,
    [FromQuery] string appKey)
{
    var response = new ServiceResponse<DosyaYuklemeResponseDto>();
    try
    {
        var uygulama = _uygulamaTanimService.GetByAppKey(appKey);
        if (uygulama == null)
        {
            response.exceptionMessage = "Geçersiz uygulama.";
            return Ok(response);
        }

        // Dosya alanı tanımını bul
        var dosyaAlan = uygulama.AnaAlanlar
            .Concat(uygulama.EkAlanlar ?? new List<FormAlanDto>())
            .FirstOrDefault(a => a.ModelTipiKodDto.id == (long)FORM_ALAN_MODEL_TIPI.DOSYA);

        // Boyut kontrolü
        var maxByte = (dosyaAlan?.DosyaMaxBoyutMb ?? 5) * 1024 * 1024;
        if (dosya.Length > maxByte)
        {
            response.exceptionMessage = $"Dosya boyutu {dosyaAlan?.DosyaMaxBoyutMb ?? 5}MB'ı geçemez.";
            return Ok(response);
        }

        // Uzantı kontrolü
        var uzanti = Path.GetExtension(dosya.FileName).TrimStart('.').ToLower();
        var izinliTipler = dosyaAlan?.DosyaIzinliTipler ?? new List<string> { "pdf", "jpg", "png" };
        if (!izinliTipler.Contains(uzanti))
        {
            response.exceptionMessage = $"İzin verilmeyen dosya tipi: {uzanti}";
            return Ok(response);
        }

        // MinIO'ya yükle
        var minioUrl = await _minioService.UploadAsync(
            bucketName: $"talep-{appKey.ToLower()}",
            fileName: $"{Guid.NewGuid()}.{uzanti}",
            stream: dosya.OpenReadStream(),
            contentType: dosya.ContentType
        );

        response.data = new DosyaYuklemeResponseDto
        {
            Url = minioUrl,
            DosyaAdi = dosya.FileName,
            Statu = (int)DOSYA_YUKLEME_STATU.YUKLENDI
        };
        return Ok(response);
    }
    catch (Exception ex)
    {
        OYSLog.Instance.SetLogType(CommonEnums.LOG_TYPE.USER)
            .SetLogIndexTip(CommonEnums.LOG_INDEX_TIP.GENEL_EXCEPTION)
            .Error($"DosyaYukle - Exception. Message:{ex.Message}");
        response.exceptionMessage = "Dosya yükleme sırasında hata oluştu.";
        return Ok(response);
    }
}
```

---

## 9. Angular — Model ve Enum'lar

### KodDTO Model

```typescript
// src/app/core/models/kod-dto.model.ts
export class KodDTO {
  id: number;
  kod: string;
  tipId: number;
  kisaAd: string;
  digerUygEnumAd: string;
  digerUygEnumDeger: string;
  sira: number;
  isAktif: boolean;
  ustKodDTO: KodDTO;
}
```

### FormAlanDto Model

```typescript
// src/app/core/models/form-alan-dto.model.ts
import { KodDTO } from './kod-dto.model';

export class ComboItemDto {
  value: string;
  label: string;
}

export class FormAlanDto {
  fieldKey: string;
  label: string;

  modelTipiKodDto: KodDTO;   // Alan tipi (STRING, EMAIL, COMBO, DOSYA...)
  mapTipiKodDto: KodDTO;     // Jira map tipi (SUBJECT, DESCRIPTION...)

  zorunlu: boolean;
  maxLength?: number;
  minLength?: number;
  placeholder?: string;
  sira: number;

  comboItems?: ComboItemDto[];    // Sadece COMBO tipinde

  dosyaMaxBoyutMb?: number;       // Sadece DOSYA tipinde
  dosyaIzinliTipler?: string[];   // Sadece DOSYA tipinde
}
```

### Enum Sabitleri

```typescript
// src/app/core/enums/form-alan.enum.ts

// t_kod TipId = 2100
export enum FormAlanModelTipi {
  STRING   = 2100001,
  EMAIL    = 2100002,
  CEPTEL   = 2100003,
  COMBO    = 2100004,
  TEXTAREA = 2100005,
  DOSYA    = 2100006,
}

// t_kod TipId = 2110
export enum FormAlanMapTipi {
  SUBJECT              = 2110001,
  DESCRIPTION          = 2110002,
  REQUESTER_NAME       = 2110003,
  REQUESTER_EMAIL      = 2110004,
  REQUESTER_IDENTIFIER = 2110005,
  ADDITIONAL_DATA      = 2110006,
  ATTACHMENT_URL       = 2110007,
}

// t_kod TipId = 2120
export enum UygulamaTanimStatu {
  AKTIF = 2120001,
  PASIF = 2120002,
}

// t_kod TipId = 2130
export enum DosyaYuklemeStatu {
  YUKLENDI    = 2130001,
  YUKLENEMEDI = 2130002,
  BOYUT_ASIMI = 2130003,
  TIP_HATASI  = 2130004,
}
```

---

## 10. Angular — Dinamik Form Builder

```typescript
// talep-form.component.ts
import { FormAlanModelTipi, DosyaYuklemeStatu } from 'src/app/core/enums/form-alan.enum';
import { FormAlanDto } from 'src/app/core/models/form-alan-dto.model';

export class TalepFormComponent implements OnInit {

  // Template'e enum'u açıyoruz — magic string KULLANILMAZ
  FormAlanModelTipi = FormAlanModelTipi;
  DosyaYuklemeStatu = DosyaYuklemeStatu;

  appKey: string;
  alanlar: FormAlanDto[] = [];
  dynamicForm: UntypedFormGroup;
  isSubmitting = false;

  dosyaYuklemeMap: Record<string, {
    yukleniyor: boolean;
    url: string;
    dosyaAdi: string;
    statu: DosyaYuklemeStatu;
  }> = {};

  ngOnInit(): void {
    this.appKey = this.route.snapshot.paramMap.get('appKey');
    this.talepService.getUygulamaBilgi(this.appKey).subscribe({
      next: (res) => {
        if (!res.hasExceptionMessage) {
          // Ana + ek alanları sırayla birleştir
          this.alanlar = [
            ...(res.data.anaAlanlar ?? []),
            ...(res.data.ekAlanlar ?? [])
          ].sort((a, b) => a.sira - b.sira);

          this.buildDynamicForm(this.alanlar);
        }
      }
    });
  }

  buildDynamicForm(alanlar: FormAlanDto[]): void {
    const group: Record<string, any> = {};
    alanlar.forEach(alan => {
      group[alan.fieldKey] = ['', Validators.compose(this.buildValidators(alan))];
    });
    this.dynamicForm = this.fb.group(group);
  }

  buildValidators(alan: FormAlanDto): ValidatorFn[] {
    const validators: ValidatorFn[] = [];
    if (alan.zorunlu) validators.push(Validators.required);
    if (alan.maxLength) validators.push(Validators.maxLength(alan.maxLength));
    if (alan.minLength) validators.push(Validators.minLength(alan.minLength));

    // modelTipiKodDto.id üzerinden — string karşılaştırma YOK
    if (alan.modelTipiKodDto?.id === FormAlanModelTipi.EMAIL)
      validators.push(Validators.email);
    if (alan.modelTipiKodDto?.id === FormAlanModelTipi.CEPTEL)
      validators.push(Validators.pattern(/^05[0-9]{9}$/));

    return validators;
  }

  talepGonder(): void {
    if (this.isSubmitting || this.dynamicForm.invalid) return;
    this.isSubmitting = true;

    const recaptcha$ = this.reChapchaAktif
      ? this.recaptchaV3Service.execute('talep')
      : of('');

    recaptcha$.pipe(
      switchMap(token => this.talepService.talepGonder({
        appKey: this.appKey,
        recaptchaToken: token,
        formData: this.dynamicForm.value
      }).pipe(first()))
    ).subscribe({
      next: (res) => {
        if (!res.hasExceptionMessage) {
          this.sweetAlertService.showMessage('success', 'Talebiniz alınmıştır.');
          this.dynamicForm.reset();
          this.dosyaYuklemeMap = {};
        } else {
          this.sweetAlertService.showError(res.exceptionMessage);
        }
      },
      complete: () => { this.isSubmitting = false; },
      error: () => { this.isSubmitting = false; }
    });
  }
}
```

---

## 11. Angular — Template Yapısı

```html
<!-- ngSwitch doğrudan KodDTO.id üzerinde — magic string YOK -->
<ng-container *ngFor="let alan of alanlar">
  <div class="mb-3">
    <label class="il-label">
      {{ alan.label }}
      <span *ngIf="alan.zorunlu" class="text-danger">*</span>
    </label>

    <ng-container [ngSwitch]="alan.modelTipiKodDto?.id">

      <!-- STRING -->
      <input *ngSwitchCase="FormAlanModelTipi.STRING"
             type="text" class="il-input"
             [formControlName]="alan.fieldKey"
             [placeholder]="alan.placeholder || ''"
             [maxlength]="alan.maxLength || 500" />

      <!-- EMAIL -->
      <input *ngSwitchCase="FormAlanModelTipi.EMAIL"
             type="email" class="il-input"
             [formControlName]="alan.fieldKey"
             [placeholder]="alan.placeholder || 'ornek@mail.com'" />

      <!-- CEPTEL -->
      <input *ngSwitchCase="FormAlanModelTipi.CEPTEL"
             type="tel" class="il-input"
             [formControlName]="alan.fieldKey"
             placeholder="05XXXXXXXXX"
             maxlength="11" />

      <!-- COMBO -->
      <select *ngSwitchCase="FormAlanModelTipi.COMBO"
              class="il-input"
              [formControlName]="alan.fieldKey">
        <option value="">Seçiniz...</option>
        <option *ngFor="let item of alan.comboItems"
                [value]="item.value">
          {{ item.label }}
        </option>
      </select>

      <!-- TEXTAREA -->
      <textarea *ngSwitchCase="FormAlanModelTipi.TEXTAREA"
                class="il-input il-textarea" rows="5"
                [formControlName]="alan.fieldKey"
                [placeholder]="alan.placeholder || ''"
                [maxlength]="alan.maxLength || 1000">
      </textarea>

      <!-- DOSYA -->
      <div *ngSwitchCase="FormAlanModelTipi.DOSYA" class="il-file-upload">
        <input type="file"
               [accept]="getAcceptString(alan)"
               (change)="onDosyaSec($event, alan)" />
        <ng-container *ngIf="dosyaYuklemeMap[alan.fieldKey] as d">
          <span *ngIf="d.yukleniyor">
            <span class="spinner-border spinner-border-sm"></span> Yükleniyor...
          </span>
          <span *ngIf="d.url" class="text-success">
            <i class="fas fa-check-circle"></i> {{ d.dosyaAdi }}
          </span>
          <span *ngIf="d.statu === DosyaYuklemeStatu.BOYUT_ASIMI" class="text-danger">
            <i class="fas fa-circle-exclamation"></i> Dosya boyutu aşıldı.
          </span>
          <span *ngIf="d.statu === DosyaYuklemeStatu.TIP_HATASI" class="text-danger">
            <i class="fas fa-circle-exclamation"></i> İzin verilmeyen dosya tipi.
          </span>
        </ng-container>
      </div>

    </ng-container>

    <!-- Validasyon hata mesajları -->
    <div *ngIf="dynamicForm.get(alan.fieldKey)?.invalid &&
                dynamicForm.get(alan.fieldKey)?.touched"
         class="il-form-error">
      <i class="fas fa-circle-exclamation"></i>
      <span *ngIf="dynamicForm.get(alan.fieldKey)?.hasError('required')">
        {{ alan.label }} zorunludur.
      </span>
      <span *ngIf="dynamicForm.get(alan.fieldKey)?.hasError('email')">
        Geçerli bir e-posta adresi giriniz.
      </span>
      <span *ngIf="dynamicForm.get(alan.fieldKey)?.hasError('pattern')">
        Geçerli bir cep telefonu giriniz. (05XXXXXXXXX)
      </span>
      <span *ngIf="dynamicForm.get(alan.fieldKey)?.hasError('maxlength')">
        {{ alan.label }} en fazla {{ alan.maxLength }} karakter olabilir.
      </span>
      <span *ngIf="dynamicForm.get(alan.fieldKey)?.hasError('minlength')">
        {{ alan.label }} en az {{ alan.minLength }} karakter olmalıdır.
      </span>
    </div>

  </div>
</ng-container>
```

---

## 12. Angular — Dosya Yükleme Bileşeni

```typescript
getAcceptString(alan: FormAlanDto): string {
  return (alan.dosyaIzinliTipler ?? []).map(t => `.${t}`).join(',');
}

onDosyaSec(event: Event, alan: FormAlanDto): void {
  const input = event.target as HTMLInputElement;
  const dosya = input.files?.[0];
  if (!dosya) return;

  // Client-side ön kontrol
  const maxMb = alan.dosyaMaxBoyutMb ?? 5;
  if (dosya.size > maxMb * 1024 * 1024) {
    this.sweetAlertService.showError(`Dosya boyutu ${maxMb}MB'ı geçemez.`);
    this.dosyaYuklemeMap[alan.fieldKey] = {
      yukleniyor: false, url: '', dosyaAdi: '',
      statu: DosyaYuklemeStatu.BOYUT_ASIMI
    };
    return;
  }

  this.dosyaYuklemeMap[alan.fieldKey] = {
    yukleniyor: true, url: '', dosyaAdi: dosya.name,
    statu: DosyaYuklemeStatu.YUKLENDI
  };

  this.talepService.dosyaYukle(dosya, this.appKey).subscribe({
    next: (res) => {
      if (!res.hasExceptionMessage) {
        // MinIO URL'yi form control'e yaz — talepGonder'de FormData ile gider
        this.dynamicForm.get(alan.fieldKey)?.setValue(res.data.url);
        this.dosyaYuklemeMap[alan.fieldKey] = {
          yukleniyor: false,
          url: res.data.url,
          dosyaAdi: dosya.name,
          statu: DosyaYuklemeStatu.YUKLENDI
        };
      } else {
        this.dosyaYuklemeMap[alan.fieldKey].yukleniyor = false;
        this.dosyaYuklemeMap[alan.fieldKey].statu = DosyaYuklemeStatu.YUKLENEMEDI;
        this.sweetAlertService.showError(res.exceptionMessage);
      }
    },
    error: () => {
      this.dosyaYuklemeMap[alan.fieldKey].yukleniyor = false;
      this.sweetAlertService.showError('Dosya yüklenemedi.');
    }
  });
}
```

---

## 13. Güvenlik Kontrol Listesi

| Madde | Durum | Not |
|---|---|---|
| reCAPTCHA v3 (frontend + backend verify) | Uygulanacak | Mevcut `VerifyRecaptcha` metodu kullanılır |
| Duplicate submit (hash + semaphore) | Uygulanacak | Mevcut `JiraService` hash mekanizması taşınır |
| IP rate limiting | Uygulanacak | Mevcut `CustomIpRateLimit` attribute kullanılır |
| JiraApiKey frontend'e gönderilmez | Tasarımda sağlandı | `UygulamaBilgiResponseDto`'ya dahil edilmez |
| Sunucu tarafı validasyon | Yeni eklenecek | `ValidateFormData` metodu |
| Dosya boyut + uzantı kontrolü (backend) | Yeni eklenecek | `DosyaYukle` controller'ında |
| Admin ekranı auth koruması | Uygulanacak | Mevcut `[Authorize]` + role yapısı |
| JiraApiKey DB'de encrypted saklanması | Önerilir | Proje şifreleme altyapısı varsa eklenir |

---

## 14. Sprint Planı

| Sprint | Kapsam | Çıktı |
|---|---|---|
| **S1** | DB migration + `UygulamaTanim` tablosu + `t_kod` SQL insert'leri | DB hazır |
| **S2** | Backend enum'lar + DTO'lar + `UygulamaTanimService` CRUD + admin controller | API hazır |
| **S3** | Angular enum sabitleri + `KodDTO` modeli + admin liste/form + alan editörü | Admin ekranı hazır |
| **S4** | `getUygulamaBilgi` endpoint + Angular dinamik form builder + tüm alan tipleri template | Form çalışır |
| **S5** | MinIO `dosyaYukle` endpoint + Angular dosya bileşeni | Dosya yükleme hazır |
| **S6** | `talepGonder` endpoint → map akışı → mevcut `JiraService` bağlantısı | E2E akış hazır |
| **S7** | reCAPTCHA + duplicate submit + IP rate limit + sunucu validasyonu | Güvenlik hazır |
| **S8** | Test, edge case'ler, güvenlik hardening, admin yetkilendirme | Prod'a hazır |

---

## 15. Proje Entegrasyon Notları

> Bu bölüm, proje dosyaları paylaşıldığında doldurulacaktır.

### Projeyi Aldığımda Yapacaklarım

Proje dosyaları paylaşıldığında aşağıdaki adımları izleyerek bu yol haritasını projeye uyumlu hale getireceğim:

**Backend Projesi İçin:**
1. Mevcut proje klasör ve namespace yapısını inceleyeceğim
2. `OYSException`, `MessageCode`, `ServiceResponse`, `OYSLog` gibi ortak sınıfların tam namespace'lerini tespit edeceğim
3. Mevcut `JiraService` ve `IMinioService` arayüzlerini inceleyerek entegrasyon noktalarını belirleyeceğim
4. `t_kod` tablosunun mevcut yapısını ve varsa diğer `TipId` değerlerini inceleyerek çakışma olmadığını doğrulayacağım
5. Mevcut migration altyapısını (EF Core / Dapper / Stored Proc) tespit edip buna uygun migration yazacağım
6. `[Authorize]` ve role yapısını inceleyerek admin controller'ı projeye uyumlu şekilde koruyacağım

**Frontend Projesi İçin:**
1. Mevcut modül ve routing yapısını inceleyeceğim — admin modülü varsa oraya, yoksa yeni modül oluşturacağım
2. `HttpHelper` / `httpHelper.Post` yapısını inceleyerek `TalepService` metod imzalarını uyumlu yazacağım
3. Mevcut `SweetAlertService`, `FormValidationService`, `isLoading$` gibi shared servislerin import yollarını tespit edeceğim
4. `environment.recaptchaAktif` gibi environment değişkenlerinin varlığını kontrol edeceğim
5. Mevcut CSS sınıf yapısını (il- prefix vs. proje prefix'i) inceleyerek template'i uyumlu yazacağım
6. `i18n` / `TranslateService` kullanımı varsa label'ları translation key'e dönüştüreceğim

### Paylaşım Sırasında Belirtilmesi Gerekenler

Projeyi paylaşırken şu bilgileri de iletirseniz geliştirme daha hızlı ilerler:

- Hangi sprint'ten başlamamı istiyorsunuz?
- `t_kod` tablosunda mevcut en yüksek `TipId` değeri nedir? (2100 çakışmasın diye)
- MinIO servis arayüzü (`IMinioService`) mevcut mu, yoksa yeni mi yazılacak?
- Admin ekranı için mevcut bir yetkili kullanıcı modülü var mı?
