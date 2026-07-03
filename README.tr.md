🌍 [English](README.md) | [Türkçe](README.tr.md) | [Español](README.es.md)

# 🛠️ ServiceNow Developer Suite — Fix Scripts & Toplu Kayıt Güncellemeleri

> **Üretime hazır ServiceNow Fix Script örnekleri, performans en iyi pratikleri ve toplu veri taşıma (migration) rehberleri.**

<div align="center">

![Platform](https://img.shields.io/badge/platform-ServiceNow-brightgreen)
![Lisans](https://img.shields.io/badge/license-MIT-green)
![Gereksinim](https://img.shields.io/badge/compatibility-Utah%20%7C%20Vancouver%20%7C%20Washington-blue)

</div>

---

## 📚 İçindekiler

- [Giriş](#giriş)
- [Fix Script'ler İçin En İyi Pratikler](#fix-scriptler-için-en-iyi-pratikler)
- [Örnek 1: Güvenli Dry-Run (Önizleme) Toplu Güncelleme](#örnek-1-güvenli-dry-run-önizleme-toplu-güncelleme)
- [Örnek 2: Parçalı (Chunked) Toplu Taşıma](#örnek-2-parçalı-chunked-toplu-taşıma)
- [Örnek 3: Yüksek Performanslı GlideMultipleUpdate](#örnek-3-yüksek-performanslı-glidemultipleupdate)
- [Hızlı Başvuru Kartı](#hızlı-başvuru-kartı)

---

## Giriş

ServiceNow kurumsal uygulama geliştirmede, **Fix Script'ler** veri başlatma, toplu veri taşıma, temizlik ve yama dağıtımı için kritik araçlardır. Ancak, milyonlarca kayıt içeren tablolar üzerinde kod çalıştırmak; işlem zaman aşımlarına (transaction timeouts), bellek tükenmesine veya yanlışlıkla iş kurallarının (business rules) tetiklenmesine yol açabilir.

Bu depo, ServiceNow üzerinde güvenli, yüksek performanslı ve tekrarlanabilir veri güncellemeleri yapabilmeniz için standart şablonları ve en iyi pratikleri sunar.

---

## Fix Script'ler İçin En İyi Pratikler

1. **Her Zaman setWorkflow(false) Kullanın** (uygun durumlarda): İş Kurallarının (Business Rules), İş Akışlarının (Workflows), Flow Designer motorlarının ve denetimlerin (auditing) tetiklenmesini engelleyerek yürütmeyi büyük ölçüde hızlandırır.
2. **Her Zaman autoSysFields(false) Kullanın:** Teknik veri taşımalarında `sys_updated_on`, `sys_updated_by` ve `sys_mod_count` gibi sistem denetim alanlarının orijinal değerlerini korur.
3. **Dry-Run (Önizleme) Modu Uygulayın:** Kodunuzun gerçek verileri güncellemeden önce etkilenen kayıt sayısını ve sys_id'leri loglamasını sağlayan bir `dryRun` bayrağı ekleyin.
4. **Büyük Veri Setleri İçin Parçalama (Chunking) Kullanın:** Milyonlarca kaydı tek seferde belleğe yüklemekten kaçının. `setLimit()` ve offset takibi kullanarak verileri 5.000 veya 10.000'lik paketler halinde işleyin.

---

## Örnek 1: Güvenli Dry-Run (Önizleme) Toplu Güncelleme

Yerleşik bir kuru çalıştırma (dry-run) önizleme modu ile kayıtları güvenli bir şekilde güncellemek için bu şablonu kullanın.

```javascript
(function() {
    var dryRun = true; // Değişiklikleri uygulamak için false yapın
    var targetTable = 'incident';
    var encodedQuery = 'active=true^state=3'; // Beklemede (On Hold) olan olaylar
    
    var gr = new GlideRecord(targetTable);
    gr.addEncodedQuery(encodedQuery);
    gr.query();
    
    var count = 0;
    gs.info('Fix Script Başlatıldı. Dry Run: ' + dryRun);
    
    while (gr.next()) {
        count++;
        if (!dryRun) {
            gr.setWorkflow(false);
            gr.autoSysFields(false);
            gr.work_notes = "Sistem bakımı: Beklemede durum açıklamaları güncellendi.";
            gr.update();
        } else {
            gs.info('[DRY RUN] Güncellenecek kayıt: ' + gr.number);
        }
    }
    gs.info('Yürütme tamamlandı. Etkilenen kayıt sayısı: ' + count);
})();
```

---

## Örnek 2: Parçalı (Chunked) Toplu Taşıma

Büyük veri kümeleri için, bellek yığını hatalarından ve işlem zaman aşımlarından kaçınmak amacıyla kayıtları paketler halinde sorgulamak ve güncellemek için bu döngüyü kullanın.

```javascript
(function() {
    var BATCH_SIZE = 5000;
    var targetTable = 'u_custom_data';
    var encodedQuery = 'u_processed=false';
    
    var recordsLeft = true;
    var totalUpdated = 0;
    
    while (recordsLeft) {
        var gr = new GlideRecord(targetTable);
        gr.addEncodedQuery(encodedQuery);
        gr.setLimit(BATCH_SIZE);
        gr.query();
        
        var batchCount = 0;
        if (!gr.hasNext()) {
            recordsLeft = false;
            break;
        }
        
        while (gr.next()) {
            gr.setWorkflow(false);
            gr.autoSysFields(false);
            gr.u_processed = true;
            gr.update();
            batchCount++;
        }
        
        totalUpdated += batchCount;
        gs.info(batchCount + ' adet kayıt güncellendi. Toplam: ' + totalUpdated);
        
        // Sorgu kriterinin değişmemesi durumunda sonsuz döngüyü önleme
        if (batchCount < BATCH_SIZE) {
            recordsLeft = false;
        }
    }
    gs.info('Veri taşıma tamamlandı. Toplam güncellenen: ' + totalUpdated);
})();
```

---

## Örnek 3: Yüksek Performanslı GlideMultipleUpdate

Milyonlarca kayıtta tek bir sütunu aynı değere güncellemeniz gerektiğinde, `GlideMultipleUpdate` en hızlı yöntemdir çünkü veritabanında doğrudan bir SQL `UPDATE` ifadesi çalıştırır, betik belleğini atlar ve standart GlideRecord döngüsünden kaçınır.

> ⚠️ **Dikkat:** Bu API belgelenmemiştir ve yalnızca maksimum performans gerektiğinde kapsamlı (scoped) veya global komut dosyalarında kullanılmalıdır. Tüm iş akışlarını ve sistem alanlarını otomatik olarak atlar.

```javascript
(function() {
    var targetTable = 'incident';
    var mu = new GlideMultipleUpdate(targetTable);
    mu.addQuery('active', 'true');
    mu.addQuery('priority', '1');
    mu.setValue('work_notes', 'GlideMultipleUpdate aracılığıyla toplu güncelleme yürütüldü.');
    
    gs.info('Veritabanında toplu güncelleme yürütülüyor...');
    mu.execute();
    gs.info('Toplu güncelleme başarıyla tamamlandı.');
})();
```

---

## Hızlı Başvuru Kartı

```
┌──────────────────────────────────────────────────────────┐
│              🛠️ ServiceNow Hızlı Başvuru                 │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  GLIDERECORD API           BEST PRACTICES                │
│  gr.query()      .......  Sorgula         setWorkflow()  │
│  gr.next()       .......  Sonraki kayıt   autoSysFields()│
│  gr.update()     .......  Kaydet          setLimit()     │
│                                                          │
│  TOPLU GÜNCELLEME METOTLARI PERFORMANS                   │
│  GlideMultipleUpd.......  Hızlı toplu güncChooseColumns()│
│  GlideQuery      .......  Modern sorgu APIisInteractive()│
│                                                          │
│  GÜVENLİ YÜRÜTME                                         │
│  Dry Run         .......  Önizleme testi                 │
│  Chunking        .......  Zaman aşımını önleme           │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

> **ServiceNow Developer Suite** — MIT Lisanslı — ❤️ ile oluşturuldu
