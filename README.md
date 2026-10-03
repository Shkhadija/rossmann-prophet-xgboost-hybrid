# Rossmann: Prophet + XGBoost Hybrid Demand Forecasting

Rossmann mağazalarının gündəlik satışını proqnozlaşdırmaq üçün hybrid yanaşma: **Prophet** trend və mövsümiliyi, **XGBoost** isə Prophet-in qalıqlarını (promo, bayram və s.) öyrənir.

## Yanaşma
1. `train.csv` və `store.csv` birləşdirilir, bağlı günlər çıxarılır.
2. Seriya: hər gün üçün açıq mağazaların orta satışı.
3. Zamana görə bölgü: son 48 gün test.
4. Üç model müqayisə olunur: Prophet, XGBoost (tək), Hybrid.

## Nəticələr (test: 48 gün)

| Model | MAE | MAPE % | RMSPE % |
|---|---|---|---|
| Prophet | 1001.8 | 14.05 | 16.89 |
| XGBoost | 514.5 | 7.08 | 8.21 |
| Hybrid | 526.3 | 7.07 | 8.24 |

Əsas qazanc xarici amillərdən gəlir. Hybrid XGBoost-dan əhəmiyyətli dərəcədə yaxşı deyil.

## Data
Fayllar repozitoriyaya daxil deyil. Buradan yükləyin: https://www.kaggle.com/c/rossmann-store-sales/data
`train.csv` və `store.csv` notebook ilə eyni qovluqda olmalıdır.

## Fayllar
- `Prophet and XGBoost for Demand Forecasting.ipynb`: əsas notebook
- `note.md`: qərarlar və nəticə
