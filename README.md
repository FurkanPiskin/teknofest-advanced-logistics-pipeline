# Desi Talep Tahmini

**TEKNOFEST 2026 — Hepsiburada Mid-Mile Linehaul Optimizasyonu**  
Gelişmiş Çözüm Aşaması · talep tahmini katmanı

Türkiye’deki kargo transfer merkezleri arasındaki **rota × gün × vardiya** yük hacmini (desi) tahmin eden uçtan uca makine öğrenmesi hattı. Çıktı, taşıma planı / ALNS katmanına resmi talep formatında gider.

Sabah (**09:00**) ve akşam (**17:00**) vardiyaları ayrı modellerdir; hacim dağılımları ve gürültüleri farklıdır.

---

## Ne işe yarar?

Planlayıcı her talep için çıkış, varış, tarih, tamamlanma saati ve **desi** ister. Bu depo o desiyi üretir:

1. Tarihsel `Master_Dataset_Tum_Turkiye.xlsx` ile modeller eğitilir.
2. Gelecek pencere için `Master_Dataset_Tum_Ozellikler_Hazir_V2.xlsx` üzerinde gün gün (recursive) tahmin yapılır.
3. Excel raporlar (`rota_id` + `Tahmin_Edilen_Desi`) optimizasyon tarafındaki `convert_ml_forecast` ile resmi 6 sütunlu talep dosyasına çevrilir.

Hedef kolon: **`toplam_desi`**. Tahminler 0’ın altına düşmesin diye `clip` + tam sayıya yuvarlanır.

---

## Mimari

```
Master_Dataset_Tum_Turkiye.xlsx
        │
        ├── sabah_mi == 1  →  TargetEncoder + XGBRegressor (SABAH)
        └── sabah_mi == 0  →  TargetEncoder + XGBRegressor (AKŞAM)
                                    │
                                    ▼
                    01_target_encoder*.pkl
                    xgboost_lojistik_modeli_*.pkl
                                    │
Master_Dataset_Tum_Ozellikler_Hazir_V2.xlsx
        │  (tarih aralığında gün gün döngü)
        │  lag özellikleri gerçek / önceki günün tahminiyle güncellenir
        ▼
Excel rapor  →  (opsiyonel) convert_ml_forecast  →  Talep-tahmini.xlsx
```

Kaynak: [`(Atılcak Kod)Canlıya_Alma_v2.ipynb`](./(Teknofest_Code.ipynb)  
Colab not defteri dört işi sırayla yapar: akşam eğitimi → akşam recursive tahmin → sabah eğitimi → sabah recursive tahmin.

---

## Veri dosyaları

| Dosya | Rol |
|---|---|
| `Master_Dataset_Tum_Turkiye.xlsx` | Eğitim / hold-out test. Tarihsel desi + özellikler. |
| `Master_Dataset_Tum_Ozellikler_Hazir_V2.xlsx` | Canlı tahmin. Takvim ve rota özellikleri hazır; gelecek günlerde `toplam_desi` henüz yok / boş. |
| `01_target_encoder.pkl` | Akşam `rota_id` → hedef ortalaması dönüştürücüsü. |
| `01_target_encoder_SABAH.pkl` | Sabah encoder. |
| `xgboost_lojistik_modeli_AKSAM.pkl` | Akşam XGBoost. |
| `xgboost_lojistik_modeli_SABAH.pkl` | Sabah XGBoost. |

Notebook Colab yolları kullanır (`/content/...`). Yerelde çalıştırırken bu yolları kendi klasörünüze çekin.

---

## Sütunlar

Kod iki grup kullanır: **hedef**, **modele giren özellikler**, **bilerek dışarıda bırakılanlar** (`cop_sutunlar`). İsimler master Excel’den gelir.

### Hedef

| Sütun | Amaç |
|---|---|
| `toplam_desi` | O rota, o gün, o vardiyada taşınacak hacim. Eğitimde `y`, tahminde üretilen değer. |

### Modele giren özellikler (bilinenler)

Eğitimde `toplam_desi` ve `cop_sutunlar` düşülür; kalan her sayısal / kodlanmış kolon XGBoost’a gider. Recursive adımda **açıkça yeniden hesaplanan** üçü:

| Sütun | Nasıl üretilir | Neden var |
|---|---|---|
| `rota_id` | Çıkış–varış çifti kimliği. Ham hali kategorik. | Yüzlerce rotanın “bu hat genelde ne kadar yük taşır” karakterini taşır. `TargetEncoder(smoothing=10)` ile eğitim setindeki ortalama desiye çevrilir; test/tahminde yalnızca `transform`. |
| `son_7_gun_hareketli_ortalama` | Aynı `rota_id` + vardiya için `toplam_desi.shift(1).rolling(7).mean()` | Kısa dönem seviye (geçen haftanın ortalaması). `shift(1)` bugünün hedefini sızdırmaz. |
| `operasyonel_std_7` | Aynı grupta 7 günlük kayan std (`min_periods=2`), NaN → 0 | Haftalık hacim oynaklığı. |
| `operasyonel_std_30` | 30 günlük kayan std | Daha uzun vadeli oynaklık. |

Master dosyada takvim ve operasyon bayrakları da bulunur (hafta içi/sonu, resmi tatil, dini bayram vb.). Bunlar `cop_sutunlar` listesinde **yoksa** modele girer. Listede olan gecikmeli tatil bayrakları (`lag_7_*`, `lag_14_*`) ve `tarihsel_gun_ortalamasi` **bilinçli olarak çıkarılır**.

Tahmin sırasında sütun sırası `model.feature_names_in_` ile hizalanır; eksik kolon 0 doldurulur.

### Dışarıda bırakılan sütunlar (`cop_sutunlar`)

Bunlar ya kimlik / rapor alanı, ya geleceği ele veren sızıntı, ya da vardiya ayırıcısıdır (vardiya zaten ayrı modelde).

| Sütun | Neden modelde yok |
|---|---|
| `tarih` | Zaman damgası; split ve recursive döngüde kullanılır, özellik değil. Ham tarih ağaç modelinde ezber üretir. |
| `gun_ismi` | Rapor / okunabilirlik. |
| `talep_tamamlanma_saati` | 09:00 / 17:00; vardiya zaten `sabah_mi` ile ayrıldı. Çıktı raporunda tutulur. |
| `cikis`, `varis` | Merkez adları. Model `rota_id` kullanır; isimler resmi formata çevirirken eşlenir. |
| `gunluk_desi_hesabi` | Hedefle ilişkili türetilmiş hesap; sızıntı riski. |
| `talep_id` | Yarışma kimliği; hacmi açıklamaz, ezber riski. |
| `sentetik_mi` | Sentetik satır işareti; modelin “sentetik = düşük/yüksek desi” öğrenmesini istemeyiz. |
| `cikis_makro_hacim`, `varis_makro_hacim` | Merkez düzeyinde makro hacim; rota hedefini dolaylı sızdırabilir. |
| `anomali_suphesi` | Etiket / kalite bayrağı, gelecekte bilinmez. |
| `varis_doygunluk_orani`, `cikis_bagimlilik_orani` | Operasyonel oranlar; hedefle iç içe geçebilir. |
| `son_ayni_vardiya_acik_desi` | Aynı vardiyanın “açık desi”si; sızıntı adayı. |
| `_yeni_eklendi` | Pipeline işaretçisi. |
| `lag_7_resmi_tatil_mi`, `lag_7_dini_bayram_mi`, `lag_14_resmi_tatil_mi`, `lag_14_dini_bayram_mi` | Gecikmeli tatil; denemede tutulmamış / gürültü. |
| `gecmis_kor_mu` / `gecmisi_kor_mu` | Geçmiş kör pencere bayrağı. |
| `tarihsel_gun_ortalamasi` | Gün tipi ortalaması; hedefe çok yakın, sızıntı. |
| `sabah_mi` | 1 = sabah, 0 = akşam. Veri ikiye bölündüğü için modele verilmez. |

---

## Model nasıl eğitilir?

Her vardiya için aynı tarif:

1. Excel yükle, `tarih` datetime, `tarih` + `sabah_mi` ile sırala.
2. Vardiya filtrele (`sabah_mi == 0` veya `== 1`).
3. **Zaman temelli hold-out:** son 7 gün test, öncesi train. Rastgele split yok. Notebook koşusunda kesim **2026-06-21**; test **2026-06-22 … 2026-06-28**.
4. `X`: `toplam_desi` + `cop_sutunlar` düşülmüş tablo. `y`: `toplam_desi`.
5. **TargetEncoder** yalnız train üzerinde `fit_transform`; test `transform`. `rota_id` için `smoothing=10`.
6. **XGBRegressor** (`tree_method='hist'`, `random_state=42`).
7. **RandomizedSearchCV**, 30 aday, skor `neg_mean_absolute_error`.
8. Çapraz doğrulama: **TimeSeriesSplit(n_splits=3)** — gelecek fold geçmişe bakmaz.
9. En iyi model testte tahmin eder; çıktı `round` + `clip(min=0)`.
10. Encoder ve model `joblib` ile `.pkl` kaydedilir. Gain önem grafiği çizilir.

Hiperparametre ızgarası:

| Parametre | Adaylar |
|---|---|
| `max_depth` | 3, 5, 7, 9 |
| `learning_rate` | 0.01, 0.03, 0.05, 0.1 |
| `n_estimators` | 300, 500, 700, 1000 |
| `subsample` | 0.6, 0.7, 0.8, 0.9 |
| `colsample_bytree` | 0.6, 0.7, 0.8, 0.9 |

Bu koşudaki şampiyon (her iki vardiya):  
`max_depth=9`, `learning_rate=0.01`, `n_estimators=700`, `subsample=0.6`, `colsample_bytree=0.7`.

### Hold-out metrikler (notebook çıktısı)

| Vardiya | MAE (desi) | RMSE | R² | WAPE |
|---|---|---|---|---|
| Akşam | 598.88 | 1150.01 | 0.9605 | %18.49 |
| Sabah | 235.68 | 903.12 | 0.6051 | %53.41 |

WAPE = Σ\|y − ŷ\| / (Σy + ε). Sabah hacimleri daha küçük ve daha gürültülü olduğu için WAPE daha yüksek çıkar; MAE mutlak olarak daha düşüktür.

---

## Recursive (canlı) tahmin

Gelecek günlerin lag’i henüz gerçek desi değildir. Döngü her takvim günü:

1. `tarih <= bugün` satırlarında `son_7_gun_hareketli_ortalama`, `operasyonel_std_7`, `operasyonel_std_30` güncelle (`shift(1)` ile).
2. O günün vardiya satırlarını al, encoder + model ile tahmin et.
3. Tahmini `toplam_desi` olarak yaz; ertesi günün rolling hesabı bunu kullanır.

Notebook örnek penceresi: **2026-06-29 … 2026-07-05**.

Çıktı kolonları: `tarih`, `gun_ismi`, `rota_id`, `talep_tamamlanma_saati`, `Tahmin_Edilen_Desi`.

Optimizasyon reposunda bu raporlar `rota_id` → `cikis`/`varis` eşlemesiyle resmi talep şablonuna çevrilir (`python -m src.forecasting.convert_ml_forecast`).

---

## Kurulum

Python 3.10+ önerilir.

```bash
pip install pandas numpy xgboost scikit-learn category_encoders openpyxl matplotlib joblib
```

Not defterini Jupyter veya Google Colab’de açın. Excel ve (tahmin adımı için) `.pkl` dosyalarını notebook’un beklediği yola koyun.

Eğitim uzun sürebilir: 30 × 3 = 90 XGBoost fit’i vardiya başına.

---

## Klasör önerisi (GitHub)

```
Desi-Talep-Tahmini/
├── README.md
├── (Atılcak Kod)Canlıya_Alma_v2.ipynb
├── Master_Dataset_Tum_Turkiye.xlsx          # büyük; Git LFS veya release
├── Master_Dataset_Tum_Ozellikler_Hazir_V2.xlsx
├── 01_target_encoder.pkl
├── 01_target_encoder_SABAH.pkl
├── xgboost_lojistik_modeli_AKSAM.pkl
└── xgboost_lojistik_modeli_SABAH.pkl
```

Excel ve pickle dosyaları büyükse GitHub’a LFS veya ayrı bir drive/release linki koyun; README’de yolu belirtin.

---

## Tasarım seçimleri (kısa)

- **İki model:** sabah ve akşam aynı ağaçta karışmasın.
- **Target encoding:** yüksek kardinaliteli `rota_id` one-hot şişirmesin; sızıntı olmaması için encoder yalnız train’de fit.
- **TimeSeriesSplit + son 7 gün test:** lojistik “yarın / gelecek hafta” sorusuna uygun.
- **Recursive lag:** 7 günlük pencerede her gün bir önceki tahmine dayanır; tek seferde 7 günü bağımsız tahmin etmek rolling ortalamayı dondururdu.
- **MAE odaklı arama:** desi sapması operasyonel olarak mutlak hacim hatasına yakın; WAPE rapor için yüzde hata verir.
