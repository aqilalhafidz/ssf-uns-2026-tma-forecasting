# Prediksi Tinggi Muka Air (TMA) DAS Bengawan Solo - SSF UNS 2026

[English](README.md)

**Ringkasan.** Memprediksi tinggi muka air (`tma_mdpl`, m dpl) di 30 pos pemantauan DAS Bengawan Solo untuk kompetisi SSF UNS 2026 (kompetisi Kaggle privat; metrik RMSE; horizon 8 bulan; 3 observasi per hari). Pendekatan akhir: LightGBM dengan fitur cuaca, kelembapan tanah, dan "memori" hujan; target dinormalisasi dengan rata-rata tiap pos; spike label dibuang; prediksi dirata-ratakan dari model yang dilatih dengan cutoff dan seed berbeda. Repo ini mendokumentasikan apa yang berhasil, apa yang tidak, dan bukti di balik tiap keputusan, dengan skema validasi yang dirancang meniru jarak train-uji sebenarnya.

---

## 1. Masalah

- **Tujuan:** memprediksi `tma_mdpl` di 30 pos pemantauan BBWS Bengawan Solo, pukul 06.00, 12.00, dan 18.00.
- **Data latih:** 2023-01-01 s.d. 2025-09-18. **Data uji:** 2025-09-19 s.d. 2026-05-18 (8 bulan, 21.780 baris, tanpa riwayat TMA).
- **Data pendukung:** variabel lingkungan per jam (hujan, kelembapan, kelembapan tanah, tekanan, MJO, ENSO, dll.).
- **Metrik:** RMSE, yang sangat memberi bobot pada kesalahan besar.
- **Kompetisi:** SSF UNS 2026, kompetisi Kaggle privat (hanya peserta undangan). Data diberikan oleh panitia dan tidak didistribusikan ulang di sini.

## 2. Hasil

| Tahap | RMSE private |
|---|---|
| Baseline awal | 1.815 |
| Kebocoran data dibuang, spike label ditangani | 1.685 |
| Target dikurangi rata-rata pos + fitur akumulasi hujan | 1.676 |
| Rata-rata dua model (cutoff berbeda) | 1.672 |
| Rata-rata empat cutoff x 5 seed | 1.662 |
| Rata-rata dua versi akhir | **1.661** (public 1.601) |

- Entri asli saat kompetisi: private 1.676, peringkat resmi 47 dari 95 tim. Submission setelah batas waktu tidak masuk peringkat resmi.
- Sebagai konteks, skor private teratas di leaderboard sekitar 1.42. Penyebab jarak ini belum teridentifikasi (lihat Bagian 6).

## 3. Pendekatan

1. **Pemeriksaan dan pembersihan data:** pemeriksaan otomatis (duplikat, gap, kode data hilang seperti `-999` pada kolom radiasi surya, nilai di luar rentang train), lalu pembuangan spike label dengan aturan median 5 observasi (simpangan > 30 m atau nilai nol/negatif mendadak).
2. **Normalisasi target:** TMA dikurangi rata-rata pos (dihitung hanya dari data latih), sehingga satu model bersama bisa belajar dari 30 pos berlevel sangat berbeda.
3. **Fitur:** akumulasi hujan beberapa jendela (2 sampai 90 hari), kelembapan tanah jangka panjang, hujan rata-rata se-DAS, indeks hujan antesedan (rata-rata berbobot eksponensial, waktu paruh 12 sampai 1.440 jam), kalender, jam observasi, dan harmonik pasang-surut astronomis.
4. **Model:** LightGBM dengan parameter default, 300 pohon.
5. **Ensemble:** rata-rata model yang dilatih dengan cutoff berbeda (2024-01-18, 2024-09-19, 2025-01-18, seluruh train), masing-masing rata-rata 5 seed dengan bagging.
6. **Validasi:** tiga fold maju, masing-masing menguji 8 bulan setelah cutoff (meniru jarak train-uji sebenarnya), jumlah pohon tetap (tanpa early stopping pada validasi), RMSE dilaporkan juga khusus musim hujan.

## 4. Keputusan dan buktinya

Semua berdasarkan validasi maju, bukan leaderboard. Hasil negatif sengaja dicatat.

| Keputusan | Bukti |
|---|---|
| Target dikurangi rata-rata pos | RMSE fold 1: 2.85 -> 0.77, fold 2: 1.62 -> 1.00 (model target asli rapuh) |
| Fitur jumlah hujan jangka panjang dengan jendela penuh | Jendela parsial membuat jumlah hujan awal 2023 terlalu kecil; setelah diperbaiki 0.814 -> 0.771 |
| Fitur hujan rata-rata se-DAS | 0.850 -> 0.770 |
| Indeks hujan antesedan (API) | 0.770 -> 0.745; RMSE Wonogiri Dam 1.54 -> 1.22 (kuat di validasi, netral di leaderboard) |
| Tanpa `solar_radiation_mj_m2` | Berisi kode `-999` sejak 2025-12-31 (57% baris uji) |
| Tanpa `nino_34` dan fitur MJO | Lebih baik tanpa di ketiga fold setelah fitur dibersihkan |
| Rata-rata 5 seed (bagging) | RMSE 0.7698 -> 0.7624, konsisten di ketiga fold dan muncul di leaderboard |
| Rata-rata antar cutoff | Jendela 2025: 0.7556 -> 0.7104; leaderboard membaik 0.0085 RMSE (gain terbesar yang tertransfer) |
| Jam observasi dan harmonik pasang-surut | Satu pos (Arjowinangun) jelas berpasang-surut; 0.7454 -> 0.7405 (kecil tapi konsisten) |
| Parameter default | Tuning Optuna tidak mengalahkan default pada 3 seed (0.8419 vs 0.8372) |

**Dicoba dan ditolak:** fitur dari jaringan sungai HydroRIVERS (lebih buruk dari rata-rata hujan se-DAS), lag target 365 hari (merugikan), CatBoost dan model linear per pos (jauh lebih lemah), blend model per pos (+0.009 di validasi, tidak berubah di leaderboard), block bagging, jendela latih yang bergeser (anggota lemah mengencerkan rata-rata), median antar model, clipping, kalibrasi linear, smoothing temporal.

## 5. Pelajaran

1. **Periksa data periode uji lebih dulu.** Kode data hilang di satu kolom baru ketahuan di tengah proyek lewat diagnostik ekstrapolasi.
2. **Validasi harus meniru jarak dan musim antara train dan uji,** dan jendelanya tidak boleh tumpang tindih kalau dipakai untuk kalibrasi.
3. **Selisih besar antara validasi dan leaderboard adalah informasi.** Perbaikan fitur yang besar di validasi (hujan se-DAS, API) nyaris tidak terlihat di leaderboard, sedangkan pengurangan variansi (rata-rata seed dan cutoff) terlihat.
4. **Rata-rata dua model yang sama akuratnya menghemat MSE sebesar seperempat kuadrat jarak antar prediksi.** Ini cocok dengan leaderboard (0.0065 / 0.0057 teramati vs 0.0059 diprediksi) dan memberi batas atas yang bisa dihitung tanpa label.
5. **Catat setiap keputusan dengan buktinya,** termasuk yang negatif.

## 6. Keterbatasan dan pertanyaan terbuka

- Hampir seluruh MSE leaderboard (sekitar 2.56) adalah kesalahan yang dibagi bersama semua variasi model di repo ini. Sumbernya belum teridentifikasi. Hipotesis (belum teruji): perbedaan rezim antar tahun di pos seperti Wonogiri Dam, dan pergeseran level di Kali Anyar.
- Jarak ke skor teratas (sekitar 0.24 RMSE) tidak terjelaskan oleh apa pun yang dicoba di sini.
- Selisih di bawah sekitar 0.004 RMSE tidak bisa dibedakan dari noise leaderboard.

## 7. Cara menjalankan

1. Dapatkan data kompetisi dari panitia (tidak disertakan di repo ini): `train.csv`, `test.csv`, `sample_submission.csv`, `data_pendukung/data_lingkungan.csv`.
2. Letakkan di satu folder (misalnya di Google Drive).
3. Buka `ssf_uns_2026_tma_final.ipynb` di Google Colab (Google Drive dipasang oleh sel pertama), atur `DATA_DIR` di sel konfigurasi, lalu jalankan semua sel. Atur `RUN_CUTOFF_CV = False` untuk melewati validasi rata-rata cutoff yang memakan waktu.
4. Keluaran: `submission_final.csv` di `OUTPUT_DIR`.

Dependensi utama: `pandas`, `numpy`, `scikit-learn`, `lightgbm` (lihat `requirements.txt`).

## 8. Struktur repo

```
.
├── README.md
├── README_id.md
├── requirements.txt
├── .gitignore
├── LICENSE
└── ssf_uns_2026_tma_final.ipynb
```

## 9. Catatan reprodusibilitas

Seed diatur tetap. Hasil ulang bisa berbeda sangat kecil (urutan thread LightGBM) tanpa mengubah kesimpulan apa pun.

## 10. Lisensi dan data

Kode dilisensikan MIT (lihat `LICENSE`). Data kompetisi mengikuti ketentuan panitia dan tidak didistribusikan ulang.
