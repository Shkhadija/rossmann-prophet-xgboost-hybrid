# Hybrid Demand Forecasting: Prophet + XGBoost (Rossmann)

## 1. Seriya qurulması qərarı

Prophet tək zaman seriyası tələb edir, datada isə 1115 mağaza var. Üç variant nəzərdən keçirildi:

| Variant | Üstünlük | Problem |
|---|---|---|
| A: bütün mağazaların gündəlik cəmi | Sadə | Açıq mağaza sayı dəyişəndə (bazar günü, bayram) cəm süni şəkildə dəyişir |
| **B: açıq mağazaların gündəlik orta satışı** | Bağlı mağazaların təsiri aradan qalxır, trend və mövsümilik daha təmiz görünür | Tərkib effekti (aşağıda) |
| C: mağaza tipi üzrə ayrı seriyalar | Daha dəqiq ola bilər | 4 Prophet + 4 XGBoost lazımdır; tip a/c/d orta satışa görə oxşardır, tip b-də mağaza az olduğu üçün seriya qeyri-sabit olardı |

**Seçim: Variant B.**

**Təmizləmə:** 172 817 bağlı sətir və 54 "açıq, amma satış 0" sətri (qeydiyyat səhvi ehtimalı) çıxarıldı, 844 338 sətir qaldı. `Customers` gələcək üçün məlum olmadığından (leakage riski), tələb olunmayan və çox boşluğu olan `CompetitionOpenSince*`, `Promo2Since*`, `PromoInterval` isə istifadə edilmədiyi üçün atıldı. `CompetitionDistance`-in 3 mağazadakı boşluğu median ilə dolduruldu.

**Xarici amillərin gündəlik səviyyəyə çevrilməsi:** `Promo`, `Promo2`, `SchoolHoliday` açıq mağazalar arasında pay kimi, `StateHoliday` 0/1 kimi, `StoreType` tip b payı (`TypeB_share`), `Assortment` extended çeşid payı (`AssortC_share`), `CompetitionDistance` median kimi, əlavə olaraq `OpenStores` (açıq mağaza sayı) verildi.

**Məhdudiyyətlər:**
- **Tərkib effekti.** Orta satış hansı mağazaların açıq olmasından asılıdır. Bazar günü və bayramlarda yalnız bir hissə açıq qalır, ona görə orta satış real tələb artımını deyil, tərkibin dəyişməsini də əks etdirir.
- **Mağaza atributlarının zəifləməsi.** Mağaza səviyyəli atributlar gündəlik paya çevrildiyi üçün fərdi mağaza təsiri itir.
- **Az mağazalı kateqoriyalar.** `TypeB_share` az sayda mağazaya əsaslanır, ona görə daha çox "bu gün həmin mağazalardan neçəsi açıqdır" siqnalıdır. `Assortment` üçün b (9 mağaza) səs-küylü olacağı üçün c payı götürüldü. Bu iki seçim eyni məntiqlə edilməyib, `TypeB_share` bayram günlərində açıq qalan mağazaların tərkibini ayırd etdiyi üçün saxlanıldı.

## 2. Bölgü

Data zamana görə bölündü: **train 894 gün (2013-01-01 → 2015-06-13), test 48 gün (2015-06-14 → 2015-07-31)**. Təsadüfi bölgü istifadə edilmədi, çünki test günlərinin qonşularını görmək gələcəyə baxmaq (leakage) olardı. Hər üç model eyni train və eyni test dövründə qiymətləndirildi.

## 3. Modellər

- **Prophet (baseline):** yalnız tarix və satış, həftəlik və illik mövsümilik, Almaniya bayramları.
- **Hybrid:** XGBoost Prophet-in train qalıqları (`real − Prophet proqnozu`) üzərində `Promo`, `Promo2`, `SchoolHoliday`, `StateHoliday`, `TypeB_share`, `AssortC_share`, `CompDist`, `OpenStores`, həftə günü və ay ilə öyrədildi. Yekun proqnoz = Prophet proqnozu + XGBoost düzəlişi.
- **End-to-end XGBoost:** Prophet olmadan, eyni xüsusiyyətlər və əlavə olaraq `dayofyear`, `weekofyear`, trend indeksi `t` ilə birbaşa satış proqnozu.

Hybrid və end-to-end XGBoost eyni hyperparametrlərlə qurulub (`n_estimators=300`, `max_depth=3`, `learning_rate=0.05`).

## 4. Metrik seçimi

- **MAE:** satış vahidi ilə orta mütləq xəta, şərh etmək asandır.
- **MAPE:** nisbi xəta (%), müxtəlif səviyyəli günləri müqayisə etməyə imkan verir.
- **RMSPE:** Kaggle-ın rəsmi metrikidir. Nisbi xətaları kvadrata yüksəltdiyi üçün böyük səhvləri daha çox cəzalandırır.

RMSPE real dəyərə bölündüyü üçün `Sales = 0` olan sətirlər sıfıra bölmə yaratmasın deyə çıxarılır (Kaggle-ın qaydası). Bu qayda hər üç modelə eyni tətbiq olunub. Bizim təmizlənmiş datada sıfır satış qalmadığı üçün praktikada heç bir sətir çıxarılmayıb.

## 5. Nəticələr (test: 48 gün)

| Model | MAE | MAPE % | RMSPE % |
|---|---|---|---|
| Prophet | 1001.8 | 14.05 | 16.89 |
| XGBoost (tək) | **514.5** | 7.08 | **8.21** |
| Hybrid | 526.3 | **7.07** | 8.24 |

**Prophet-ə qarşı:** hybrid MAE-ni təxminən **47%**, MAPE-ni **50%**, RMSPE-ni **51%** azaldır. XGBoost tək də təxminən eyni qazancı verir (MAE −49%). Prophet-in səhvi sistematik idi: promo günlərində orta hesabla ~1 017 az, promosuz günlərdə ~623 çox proqnoz verirdi. Bu səhv promo ilə izah olunur.

**XGBoost tək ilə hybrid:** fərq çox kiçikdir. Hybrid MAE-də təxminən **2.3% pis** (526.3 vs 514.5), MAPE-də praktiki olaraq bərabərdir (7.07 vs 7.08), RMSPE-də təxminən 0.4% pisdir (8.24 vs 8.21). Heç bir metrikdə hybrid XGBoost-u aydın şəkildə qabaqlamır.

**Həssaslıq:** `AssortC_share` əlavə edilməzdən əvvəl hybrid MAE-də XGBoost-dan cüzi yaxşı idi (530.7 vs 536.0). Bir xüsusiyyət əlavə edəndə sıralama tərsinə döndü. Bu, hybrid ilə XGBoost arasındakı fərqin test səs-küyü daxilində olduğunu göstərir.

## 6. Hybrid mürəkkəbliyi doğrultdumu?

**Xeyr, bu datada.** Əsas qazanc xarici amillərdən (ilk növbədə `Promo`) gəlir, Prophet-in özündən yox. End-to-end XGBoost zaman xüsusiyyətləri ilə (həftə günü, ay, ilin günü, `t`) zaman strukturunu özü yaxşı öyrəndi. Bunun səbəbi seriyada trendin zəif, mövsümiliyin isə sadə (həftəlik və illik) olmasıdır. Hybrid iki model, iki mərhələ və qalıq hesablama tələb edir, amma XGBoost tək modeldən daha yaxşı nəticə vermir. Sadə XGBoost bu tapşırıq üçün kifayətdir.

## 7. Xüsusiyyət əhəmiyyəti (end-to-end XGBoost)

`Promo`, `CompDist` və `TypeB_share` ən yüksək əhəmiyyət göstərdi. `Promo`-nun yüksək olması EDA ilə uyğundur (promo günlərində orta satış təxminən 39% yüksəkdir). `CompDist` və `TypeB_share` isə real rəqib və ya mağaza tipi təsirini yox, açıq mağaza tərkibinin dəyişməsini əks etdirir, çünki bunlar gündəlik median və pay kimi hesablanıb. Yəni bu iki sütun Variant B-nin tərkib effektinin dolayı sübutudur. `feature_importances_` korrelyasiyalı xüsusiyyətlər arasında əhəmiyyəti qeyri-sabit bölür, ona görə dəqiq sıralamaya həddindən artıq güvənmək olmaz.
