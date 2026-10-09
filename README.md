# Verifikasi Ijazah: OCR Nomor Ijazah + Deteksi Tanda Tangan

Prototype pengolahan citra yang membaca satu citra ijazah dan menghasilkan dua keluaran.

```
Input  : ijazah_001.jpg

Output :
Nomor Ijazah : 571012022000056
Tanda Tangan : PRESENT
```

Seluruh kode ada di **satu notebook**: `ijazah_verification.ipynb`. Tidak ada file `.py`. Notebook yang ada di repo ini sudah berisi hasil eksekusi (gambar dan tabel), jadi bisa dibaca langsung di GitHub tanpa menjalankan apa pun.

## Struktur folder

```
.
├── ijazah_verification.ipynb   <- semua kode + penjelasan metode
├── README.md
├── requirements.txt
├── data/
│   ├── ijazah_001.jpg                       contoh ijazah asli
│   ├── ijazah_002_tanpa_ttd_sintetis.jpg    ijazah sintetis tanpa TTD (dibuat otomatis oleh notebook)
│   └── ground_truth.csv                     nomor ijazah sebenarnya, untuk hitung CER
└── results/                    <- tabel CER dan gambar hasil (dibuat saat notebook dijalankan)
```

## Cara menjalankan

Soal "apakah notebook perlu download OCR lagi": **perlu, tapi sudah diotomatisasi.** OCR memakai Tesseract, yaitu program terpisah dari Python. Bedanya dengan `.py`, di notebook pemasangannya ditaruh di sel pertama, jadi cukup tekan **Run All** dan sel itu mengurus semuanya.

### Opsi A: Google Colab (paling mudah, tanpa pasang apa pun)

1. Buka <https://colab.research.google.com>, pilih **File > Upload notebook**, lalu unggah `ijazah_verification.ipynb`.
2. Di panel file Colab (ikon folder), buat folder `data` lalu unggah isi folder `data/` dari repo ini (minimal `ijazah_001.jpg` dan `ground_truth.csv`).
3. Pilih **Runtime > Run all**. Sel pertama memasang Tesseract lewat `apt-get`, butuh sekitar setengah menit.

### Opsi B: Lokal dengan Jupyter Notebook

1. Pasang Tesseract satu kali saja.
   - **Windows:** unduh installer dari <https://github.com/UB-Mannheim/tesseract/wiki>, pasang, lalu di sel kedua notebook buka komentar baris `tesseract_cmd` dan sesuaikan path-nya.
   - **macOS:** `brew install tesseract`
   - **Ubuntu/Debian:** `sudo apt install tesseract-ocr` (opsional, sel pertama bisa melakukannya otomatis)
2. Pasang Jupyter dan library:
   ```bash
   pip install -r requirements.txt
   ```
3. Jalankan:
   ```bash
   jupyter notebook ijazah_verification.ipynb
   ```
4. Menu **Kernel > Restart & Run All**.

### Menguji ijazah lain

Taruh gambar (jpg, jpeg, png) di folder `data/`, lalu jalankan sel **Batch** di bagian 8. Kalau ingin CER ikut dihitung, tambahkan satu baris `namafile.jpg,nomorijazahsebenarnya` di `data/ground_truth.csv`. Orientasi gambar (termasuk yang terputar 90 derajat) dikoreksi otomatis.

## Pipeline

```
Citra -> koreksi orientasi -> Grayscale -> Enhancement umum (bilateral)
      -> Area nomor -> Enhancement terpilih -> OCR (Tesseract) -> Nomor Ijazah
      -> Area TTD   -> Thresholding -> Morphology -> Signature Detection
                                     => Hasil Verifikasi
```

## Metode

| Tahap | Metode |
|---|---|
| Koreksi orientasi | Coba 4 rotasi, OCR cepat pada gambar kecil, pilih rotasi dengan kata kunci terbanyak |
| Grayscale | Konversi BGR ke gray |
| Enhancement umum | Bilateral filter ringan (meredam noise, tepi huruf tetap tajam) |
| Area nomor / TTD | Crop ROI relatif (proporsi lebar dan tinggi gambar), jadi tidak bergantung resolusi |
| Enhancement area nomor | 11 metode dibandingkan memakai CER |
| OCR | Tesseract, `--psm 7` (satu baris), whitelist digit 0-9 |
| Thresholding TTD | Estimasi latar (dilasi + median blur), selisih mutlak dengan citra, lalu Otsu dengan batas kontras minimum 45 |
| Morphology | Opening 3x3 (hapus bintik), closing 9x9 (sambung goresan) |
| Signature detection | Connected components. Komponen terbesar harus punya luas minimal 0,4% ROI dan lebar minimal 25% ROI |
| Metrik OCR | CER = jarak Levenshtein / panjang ground truth |

Alasan memakai estimasi latar sebelum thresholding: kertas ijazah punya pola guilloche halus. Kalau Otsu dipakai langsung, pola itu ikut terdeteksi sebagai tinta. Batas kontras minimum memastikan area kosong benar-benar dibaca `ABSENT`.

## Hasil dan analisis enhancement berdasarkan CER

Setiap metode diuji pada citra asli dan 12 kondisi buruk simulasi (blur, noise, kontras rendah, cahaya tidak rata, resolusi rendah, JPEG berat, miring). Nilai 0 berarti OCR sempurna, nilai 1 berarti gagal total. Tabel lengkap ada di `results/cer_table.csv`, grafiknya di `results/cer_comparison.png`.

| Metode | asli | blur-3.5 | noise-35 | noise-50 | uneven-0.3 | uneven-0.2 | lowres | jpeg | miring-2.5 | CER rata-rata |
|---|---|---|---|---|---|---|---|---|---|---|
| M9 Blur + BG norm + Upscale + Otsu | 0.00 | 0.07 | 0.00 | 0.00 | 0.00 | 0.00 | 0.00 | 0.00 | 0.00 | 0.01 |
| M10 Median + BG norm + Upscale + Otsu | 0.07 | 0.13 | 0.02 | 0.02 | 0.00 | 0.00 | 0.00 | 0.00 | 0.07 | 0.03 |
| M7 Background norm + Upscale + Otsu | 0.00 | 0.00 | 1.00 | 1.00 | 0.00 | 0.00 | 0.00 | 0.00 | 0.00 | 0.15 |
| M4 Adaptive threshold | 0.00 | 0.20 | 1.00 | 1.00 | 0.00 | 0.00 | 0.00 | 0.00 | 0.00 | 0.25 |
| M3 Gaussian blur + Otsu | 0.00 | 0.47 | 0.00 | 0.00 | 1.00 | 1.00 | 0.00 | 0.00 | 0.00 | 0.27 |
| M6 Denoise (NLM) + CLAHE | 0.00 | 0.13 | 1.00 | 1.00 | 1.00 | 0.87 | 0.00 | 0.00 | 0.00 | 0.39 |
| M5 Sharpening | 0.00 | 0.00 | 1.00 | 1.00 | 1.00 | 1.00 | 0.00 | 0.00 | 0.00 | 0.39 |
| M1 Upscale 2x | 0.00 | 0.00 | 1.00 | 1.00 | 1.00 | 1.00 | 0.00 | 0.00 | 0.00 | 0.39 |
| M0 Grayscale (baseline) | 0.00 | 0.00 | 1.00 | 1.00 | 1.00 | 1.00 | 0.00 | 0.00 | 0.00 | 0.39 |
| M2 CLAHE | 0.00 | 0.07 | 1.00 | 1.00 | 0.33 | 0.80 | 0.00 | 0.00 | 0.00 | 0.40 |
| M8 CLAHE + Upscale + Otsu | 0.07 | 0.07 | 1.00 | 1.00 | 1.00 | 1.00 | 0.00 | 0.00 | 0.00 | 0.47 |

**Metode paling efektif: M9, yaitu Gaussian blur 3x3, normalisasi latar, upscale 2x, lalu Otsu.** CER rata-ratanya 0,005, dibanding 0,386 untuk baseline grayscale.

Pembahasannya:

- Normalisasi latar adalah faktor terbesar. Ijazah hasil foto atau scan hampir selalu punya sisi yang lebih gelap, dan pada kondisi itu metode tanpa penyesuaian latar (M0, M1, M3, M5, M6, M8) gagal dengan CER mendekati 1.
- Normalisasi latar saja (M7) belum cukup, sebab ia ikut memperkuat noise. CER M7 melonjak ke 1 pada noise sedang dan berat. Blur ringan sebelum normalisasi (M9) menutup celah itu.
- M3 (blur + Otsu global) tahan noise, tapi Otsu global jatuh saat cahaya tidak rata. M4 (adaptive threshold) kebalikannya.
- CLAHE (M2, M8) tidak membantu. Ia menaikkan kontras noise dan pola kertas sama banyaknya dengan kontras huruf.
- Sharpening dan upscale saja tidak memberi perbaikan dibanding baseline.

## Hasil deteksi tanda tangan

| File | Tanda Tangan | Keterangan |
|---|---|---|
| `ijazah_001.jpg` | PRESENT | ijazah asli |
| `ijazah_002_tanpa_ttd_sintetis.jpg` | ABSENT | area TTD Rektor dihapus secara digital |

## Keterbatasan

- Pengujian memakai **satu ijazah** nyata. Kondisi buruk dibuat secara simulasi, jadi CER sebaiknya dibaca sebagai perbandingan relatif antarmetode, bukan angka akurasi di lapangan.
- Area nomor dan tanda tangan ditentukan dengan ROI relatif untuk **template ijazah Universitas Indonesia**. Template kampus lain butuh penyesuaian `NUM_BOX` dan `SIG_BOX` di notebook.
- Sistem hanya mendeteksi **ada atau tidaknya** goresan tinta pada area tanda tangan. Keaslian tanda tangan tidak diperiksa.
- Yang dicek adalah tanda tangan Rektor. Tanda tangan Dekan bisa ditambahkan dengan satu ROI baru.
- Tesseract membaca digit dengan baik pada font cetak seperti ini, tetapi belum dilatih khusus untuk nomor ijazah.

## Catatan privasi sebelum upload ke GitHub

`data/ijazah_001.jpg` berisi nama, tanggal lahir, NPM, foto, dan nomor ijazah asli. Kalau repo akan dibuat **publik**, pertimbangkan menyamarkan gambar itu atau membuat repo **private** dan memberi akses ke dosen.

## Upload ke GitHub

```bash
git init
git add .
git commit -m "Prototype verifikasi ijazah: OCR + deteksi tanda tangan"
git branch -M main
git remote add origin https://github.com/<username>/<nama-repo>.git
git push -u origin main
```
