# Teknik Threshold, Reclassify, Masking, dan Clamping Data Raster Kontinu Dalam Bahasa R

Empat pendekatan yang sering dianggap sama padahal beda konsep, dan kapan masing-masing dipakai, menggunakan package `terra`.

## Daftar Isi

- [Pendahuluan](#pendahuluan)
- [Menampilkan Label Tick Sesuai Rentang Nilai Aktual Data](#menampilkan-label-tick-sesuai-rentang-nilai-aktual-data)
- [Reclassification](#reclassification)
- [Thresholding dengan Operator Perbandingan](#thresholding-dengan-operator-perbandingan)
- [Thresholding dengan ifel() untuk Nilai Kontinu yang Tetap Dipertahankan](#thresholding-dengan-ifel-untuk-nilai-kontinu-yang-tetap-dipertahankan)
- [Masking Berdasarkan Raster atau Vektor Lain](#masking-berdasarkan-raster-atau-vektor-lain)
- [Clamping: Membatasi Rentang Nilai Tanpa Membuang Piksel](#clamping-membatasi-rentang-nilai-tanpa-membuang-piksel)
- [Thresholding Adaptif](#thresholding-adaptif-ambang-dihitung-dari-statistik-raster-itu-sendiri)
- [Perbandingan Kategori Teknik](#perbandingan-kategori-teknik)
- [Ringkasan Istilah](#ringkasan-istilah)
- [Referensi](#referensi)

## Pendahuluan

Setelah sebuah indeks raster selesai dihitung dan berbentuk nilai kontinu, ada beberapa teknik yang bisa dipakai untuk mengolahnya lebih lanjut berdasarkan nilai piksel: thresholding, reclassification, masking, dan clamping.

Keempatnya sering dibahas berdampingan karena sama-sama "membatasi" atau "mengelompokkan" raster berdasarkan nilai, tapi secara konsep berbeda dan menghasilkan raster dengan karakteristik berbeda pula.

Repo ini membahas keempat kategori teknik itu menggunakan package `terra` di R, dengan AWEI (Automated Water Extraction Index) sebagai studi kasus.

Sebagai konteks, AWEI adalah indeks kontinu yang bisa bernilai negatif maupun positif, dengan konvensi umum nilai positif cenderung piksel air dan nilai negatif cenderung non-air (Feyisa et al., 2014). Titik potongnya bisa dicek dari histogram data yang digunakan, dan tidak selalu tepat di angka 0.

## Menampilkan Label Tick Sesuai Rentang Nilai Aktual Data

Legend default dari `plot()` tidak selalu menampilkan tick yang mencerminkan sebaran nilai asli raster. Tick-nya bisa dihitung eksplisit dari nilai minimum dan maksimum aktual data.

```r
library(terra)

r <- rast("D:/TUTOR AWEI/AWEI_sh.tif")

r_min <- minmax(r)[1, 1]
r_max <- minmax(r)[2, 1]
ticks <- seq(r_min, r_max, length.out = 10)

plot(r,
     col = hcl.colors(50, "GnBu", rev = TRUE),
     main = "AWEIsh — Raw Continuous Value",
     plg = list(at = ticks, labels = round(ticks, 3)))
```

`minmax(r)` mengembalikan matriks dua baris (min, max) satu kolom per layer raster. Karena `r` cuma satu layer, `minmax(r)[1, 1]` mengambil nilai minimum aktual data, `minmax(r)[2, 1]` mengambil nilai maksimum aktualnya — bukan angka yang ditebak manual. `seq(r_min, r_max, length.out = 10)` membuat 10 titik tick yang terbagi rata dari nilai min ke max aktual.

Argumen `plg` (plot legend) pada `plot()` versi terra dipakai untuk mengatur tampilan color bar/legend: `at` menentukan posisi tick sesuai nilai `ticks` yang sudah dihitung, `labels` menentukan teks yang tampil di tiap tick (dibulatkan 3 desimal pakai `round()`). Hasilnya, legend menampilkan 10 label tick yang mencerminkan sebaran nilai asli raster, dari minimum sampai maksimum data.

## Reclassification

`classify()` adalah fungsi reclassification: mengubah raster kontinu jadi raster kategorikal berdasarkan tabel rentang nilai. Threshold di sini berperan sebagai titik potong antar kelas, tapi hasil akhirnya bukan biner TRUE/FALSE — melainkan label kelas.

### Reclassify 2 Kelas

```r
r_bin <- classify(
  r,
  rcl = matrix(c(-Inf, 0, 0,
                 0,  Inf, 1),
               ncol = 2, byrow = TRUE),
  include.lowest = TRUE
)

plot(r_bin, col = c("lightgrey", "blue"))
```

Argumen `rcl` adalah matriks tiga kolom: batas bawah, batas atas, dan nilai kelas baru untuk rentang tersebut. Baris pertama berarti "semua nilai dari -Inf sampai 0 diubah jadi 0", baris kedua "semua nilai dari 0 sampai Inf diubah jadi 1". `include.lowest = TRUE` memastikan nilai persis di batas (0) ikut terhitung, bukan terlewat. Hasilnya, `r_bin` cuma berisi dua nilai: 0 dan 1.

### Reclassify Multi-Kelas

`classify()` tidak terbatas pada dua kelas, bisa menghasilkan beberapa kelas sekaligus dalam satu langkah, dengan threshold sebagai titik potong antar tiap kelas.

```r
r_multiclass <- classify(
  r,
  rcl = matrix(c(-Inf, -0.2,  1,
                 -0.2,  0.2,  2,
                 0.2,  Inf,  3),
               ncol = 3, byrow = TRUE),
  include.lowest = TRUE
)

plot(r_multiclass, col = c("lightgray", "blue", "navy"),
     main = "Reclassify 3 Kelas")
```

Di sini `rcl` punya tiga baris, jadi tiga rentang nilai dipetakan ke tiga kelas berbeda (1, 2, 3). Prinsipnya sama seperti reclassify biner, cuma jumlah barisnya menyesuaikan jumlah kelas yang diinginkan. Setelah reclassify, nilai asli setiap piksel tidak lagi tersimpan — yang tersisa hanya label kelas.

## Thresholding dengan Operator Perbandingan

Operator perbandingan (`>`, `<`, `>=`, dll.) adalah bentuk thresholding paling langsung: satu nilai ambang, hasilnya biner. Diterapkan ke SpatRaster, operator ini menghasilkan raster baru bertipe logical.

```r
r_logical <- r > 0

plot(r_logical, col = c("grey", "blue"),
     main = "Perbandingan Langsung (TRUE/FALSE)")

class(values(r_logical))
```

`r > 0` mengevaluasi setiap piksel: TRUE kalau nilainya lebih dari 0, FALSE kalau tidak. `values(r_logical)` mengambil nilai piksel sebagai vektor, dan `class()` menunjukkan tipe datanya berupa logical.

Secara nilai, ini setara dengan hasil reclassify biner (FALSE ≈ 0, TRUE ≈ 1), tapi didapat langsung dari satu ambang tanpa matriks reclassification.

Kalau raster logical ini mau disimpan sebagai output akhir, tipe datanya bisa dikonversi eksplisit ke numerik:

```r
r_bin_from_logical <- as.numeric(r_logical)
```

`as.numeric()` di sini mengubah tipe piksel dari logical menjadi numerik (0/1), supaya kompatibel kalau dibaca software GIS lain yang tidak mengenali tipe logical.

## Thresholding dengan ifel() untuk Nilai Kontinu yang Tetap Dipertahankan

`ifel()` juga bentuk thresholding, kondisinya tetap berupa perbandingan terhadap satu ambang, tapi nilai penggantinya tidak harus berupa label kelas. Nilai bisa dipertahankan tetap kontinu.

```r
r_masked <- ifel(r > 0, r, NA)

plot(r_masked, col = hcl.colors(50, "RdBu", rev = TRUE))
```

`ifel(kondisi, nilai_jika_benar, nilai_jika_salah)` — di sini, kalau `r > 0` benar, piksel mempertahankan nilai aslinya (`r`); kalau salah, piksel diisi NA. Berbeda dari `classify()`, tidak ada nilai yang diseragamkan menjadi label kelas.

Ambangnya bisa diganti sesuai kebutuhan, misalnya:

```r
r_masked_strict <- ifel(r > 0.2, r, NA)
plot(r_masked_strict, col = hcl.colors(50, "RdBu"))
```

## Masking Berdasarkan Raster atau Vektor Lain

### Masking dengan Referensi Vektor (AOI)

`vect()` membaca file vektor (shapefile) menjadi objek SpatVector. Tanpa argumen `maskvalues`, `mask(r, aoi)` secara default mempertahankan piksel yang berada di dalam geometri `aoi` dan menjadikan NA piksel di luarnya. Dengan demikian, raster `r` hanya akan mempertahankan nilai piksel yang berada di dalam batas Area of Interest (AOI) yang ditentukan oleh `Sketches.shp`. Pastikan Coordinate Reference System (CRS) antara raster dan AOI sesuai agar proses masking dilakukan pada lokasi yang tepat.

```r
library(terra)

aoi <- vect("D:/TUTOR AWEI/Sketches.shp")
aoi <- project(aoi, crs(r))
r_masked_aoi <- mask(r, aoi)
plot(r_masked_aoi)
```

### Masking dengan Referensi Raster Lain

Argumen pertama (`r`) adalah raster yang mau di-mask, argumen kedua (`r_logical`) adalah raster acuan, dan `maskvalues = FALSE` berarti piksel pada raster acuan yang bernilai FALSE akan diubah jadi NA pada raster hasil. Perlu dicatat, `r_logical` di sini sendiri adalah hasil thresholding dari bagian sebelumnya, jadi `mask()` pada contoh ini hanya menerapkan hasil threshold yang sudah dibuat sebelumnya, bukan melakukan thresholding baru.

```r
r_masked_alt <- mask(r, r_logical, maskvalues = FALSE)
plot(r_masked_alt)
```

## Clamping: Membatasi Rentang Nilai Tanpa Membuang Piksel

`clamp()` juga bukan thresholding, ini clamping (kadang disebut capping atau winsorizing): menekan nilai piksel yang berada di luar rentang tertentu ke batas rentang itu, tanpa menghapus piksel maupun mengubahnya jadi label kelas.

```r
r_clamped <- clamp(r, lower = 0, upper = 1, values = TRUE)

plot(r_clamped, col = hcl.colors(50, "RdBu"))
```

`lower` dan `upper` menentukan batas bawah dan atas rentang nilai. `values = TRUE` berarti piksel di luar rentang diganti dengan nilai batas terdekat; kalau `values = FALSE`, piksel di luar rentang akan jadi NA alih-alih dipotong ke batas.

## Thresholding Adaptif: Ambang Dihitung dari Statistik Raster Itu Sendiri

Masih dalam kategori thresholding, tapi ambangnya tidak ditulis manual sebagai angka tetap — dihitung dari statistik raster yang sedang diproses.

```r
q <- global(r, fun = quantile, probs = 0.75, na.rm = TRUE)[1, 1]

r_adaptive <- ifel(r > q, r, NA)

plot(r_adaptive, col = hcl.colors(50, "RdBu", rev = TRUE),
     main = paste0("Threshold Adaptif (>Q75 = ", round(q, 3), ")"))
```

`global()` menghitung statistik ringkasan dari seluruh piksel raster; di sini fungsinya `quantile` dengan `probs = 0.75`, jadi menghasilkan nilai kuantil ke-75 dari sebaran nilai raster, dengan `na.rm = TRUE` mengabaikan piksel NA dalam perhitungan. Hasilnya berupa data frame satu baris satu kolom, diambil nilainya dengan `[1, 1]`. Nilai `q` ini kemudian dipakai sebagai ambang di `ifel()`, sehingga ambangnya menyesuaikan sebaran nilai pada raster yang sedang diproses, bukan angka yang sama untuk semua citra.

## Perbandingan Kategori Teknik

| Teknik                  | Fungsi terra        | Kategori          | Nilai kontinu?  | Cakupan spasial                                       |
|---------------------------|-----------------------|--------------------|-------------------|----------------------------------------------------------|
| Reclassify 2 kelas       | `classify()`         | Reclassification  | Hilang           | Penuh                                                     |
| Reclassify multi-kelas   | `classify()`         | Reclassification  | Hilang           | Penuh                                                     |
| Operator perbandingan    | `>`, `<`, dst.        | Thresholding       | Hilang           | Penuh                                                     |
| Filter kondisional       | `ifel()`              | Thresholding       | Dipertahankan    | Menyempit (NA di luar ambang)                             |
| Mask referensi eksternal | `mask()`              | Masking            | Dipertahankan    | Sesuai raster/vektor acuan                                |
| Pembatasan rentang       | `clamp()`             | Clamping           | Dipertahankan    | Penuh (tidak ada NA baru, kecuali `values = FALSE`)       |
| Threshold adaptif        | `global()` + `ifel()` | Thresholding       | Dipertahankan    | Menyempit (NA di luar ambang)                             |

## Ringkasan Istilah

- **Thresholding** — operator perbandingan, `ifel()`, dan threshold adaptif: piksel dievaluasi terhadap satu nilai ambang, hasilnya bisa biner (logical) atau tetap kontinu tergantung fungsinya.
- **Reclassification** — `classify()`: nilai kontinu dikelompokkan jadi label kelas berdasarkan rentang, threshold jadi salah satu komponen di dalamnya (titik potong antar kelas).
- **Masking** — `mask()`: piksel dipertahankan/disembunyikan berdasarkan kriteria dari raster atau vektor referensi lain, bukan dari pembandingan nilai raster itu sendiri.
- **Clamping** — `clamp()`: rentang nilai dibatasi tanpa mengubah piksel jadi label kelas atau menyembunyikannya (kecuali diatur eksplisit).

## Referensi

Feyisa, G. L., Meilby, H., Fensholt, R., & Proud, S. R. (2014). Automated Water Extraction Index: A new technique for surface water mapping using Landsat imagery. *Remote Sensing of Environment*, 140, 23–35. https://doi.org/10.1016/j.rse.2013.08.029

Hijmans, R. J., Brown, A., Barbosa, A. M., Cordano, E., & Dyba, K. (2026). *terra: Spatial Data Analysis*. R package version 1.9-40.

- Repo GitHub terra: https://github.com/rspatial/terra
- Link CRAN terra: https://CRAN.R-project.org/package=terra

## Script dan Dataset

https://github.com/Defani/raster-logic-terra-r

---

<div align="left">
  <a href="https://linkedin.com/in/defaniarmanalfitriansyah"><img src="https://img.shields.io/badge/LinkedIn-0077B5?style=flat-square&logo=linkedin&logoColor=white" alt="LinkedIn" /></a>
  <a href="https://medium.com/@defaniarman"><img src="https://img.shields.io/badge/Medium-12100E?style=flat-square&logo=medium&logoColor=white" alt="Medium" /></a>
  <a href="https://tiktok.com/@defaniarman"><img src="https://img.shields.io/badge/TikTok-000000?style=flat-square&logo=tiktok&logoColor=white" alt="TikTok" /></a>
  <a href="https://www.instagram.com/de.fanii"><img src="https://img.shields.io/badge/Instagram-E4405F?style=flat-square&logo=instagram&logoColor=white" alt="Instagram" /></a>
  <a href="https://www.kaggle.com/defani123"><img src="https://img.shields.io/badge/Kaggle-20BEFF?style=flat-square&logo=kaggle&logoColor=white" alt="Kaggle" /></a>
  <a href="https://rpubs.com/defanii"><img src="https://img.shields.io/badge/RPubs-75AADB?style=flat-square&logo=r&logoColor=white" alt="RPubs" /></a>
  <a href="https://www.behance.net/defaniarman"><img src="https://img.shields.io/badge/Behance-1769FF?style=flat-square&logo=behance&logoColor=white" alt="Behance" /></a>
  <a href="mailto:defaniarman@gmail.com"><img src="https://img.shields.io/badge/Email-D14836?style=flat-square&logo=gmail&logoColor=white" alt="Email" /></a>
</div>
