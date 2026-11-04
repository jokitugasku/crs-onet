# Bab 11. Pengujian (`tests/`)

Setiap tes sudah dijelaskan di bab yang membahas kodenya. Bab ini merangkum semuanya: cara menjalankan tes,
apa yang diuji setiap file, hasil saat ini, dan apa yang **tidak** diuji.

## 11.1 Cara menjalankan

Paket yang dibutuhkan ada di `requirements-dev.txt`:

```
# Untuk pengembangan: tes
-r requirements.txt
httpx
```

Isinya semua paket aplikasi (`requirements.txt`) ditambah `httpx`, yang dibutuhkan `TestClient` FastAPI untuk
memanggil API tanpa menjalankan server.

Perintah untuk menjalankan semua tes, dari folder utama proyek:

```bash
.venv/bin/python -m unittest discover tests
```

`unittest` adalah modul tes bawaan Python. `discover tests` mencari semua file `test_*.py` di folder `tests/`
dan menjalankan setiap method yang namanya diawali `test_`.

Untuk menjalankan satu file saja:

```bash
.venv/bin/python -m unittest tests.test_recommender
```

### Tes Postgres

Tes penyimpanan untuk Postgres hanya berjalan jika ada basis data uji. Alamatnya diisi lewat variabel
`TEST_DATABASE_URL` (`tests/test_storage.py` baris 3–4):

```bash
TEST_DATABASE_URL=postgresql://localhost:54329/postgres .venv/bin/python -m unittest tests.test_storage
```

Tanpa variabel itu, tes Postgres dilewati (*skipped*), tidak dianggap gagal. Basis data yang dipakai harus
basis data kosong khusus untuk tes, **bukan** basis data asli di Neon, karena tes ini menulis akun dan hasil
percobaan.

## 11.2 Hasil saat ini

Hasil menjalankan semua tes:

```
Ran 42 tests in 1.752s

OK (skipped=5)

  siswa sintetis: Hit@10 67.4%, Related Recall@10 18.4%
```

**42 tes: 37 lulus, 5 dilewati, 0 gagal.** Kelima tes yang dilewati adalah tes Postgres, karena
`TEST_DATABASE_URL` tidak diisi. Versi SQLite dari tes yang sama lulus.

Baris terakhir dicetak oleh tes evaluasi siswa sintetis (Bab 6.8).

Saat tes dijalankan, mungkin muncul peringatan `StarletteDeprecationWarning` tentang `httpx`. Peringatan ini
berasal dari pustaka FastAPI/Starlette, bukan dari kode texin, dan tidak memengaruhi hasil tes.

## 11.3 Daftar file tes

| File | Jumlah tes | Yang diuji | Dibahas di |
|---|---|---|---|
| `test_recommender.py` | 7 | data lengkap, korelasi Pearson benar, skala tidak berpengaruh, kompetensi, evaluasi siswa sintetis | Bab 6.8 |
| `test_questionnaire.py` | 5 | 60 butir, penjumlahan skor, penolakan jawaban tidak valid dan profil datar, label area minat | Bab 5.9 |
| `test_i18n.py` | 3 | terjemahan lengkap, judul unik, file terjemahan cocok dengan data O*NET | Bab 4.5 |
| `test_details.py` | 3 | pendidikan, gaya kerja, software | Bab 7.10 |
| `test_storage.py` | 10 (5 dilewati) | tabel akun, sesi, progres, hasil, feedback; SQLite dan Postgres | Bab 8.8 |
| `test_web.py` | 14 | semua endpoint API, akun, riwayat, feedback, ekspor, file statis | Bab 9.11, 10.10 |
| **Total** | **42** | | |

`test_storage.py` berisi 7 method, tapi tiga di antaranya (`StorageContract`) dijalankan dua kali, untuk
SQLite dan untuk Postgres. Karena itu jumlah tesnya 10.

## 11.4 Jenis pengujian

Tes-tes ini bisa dikelompokkan menjadi tiga jenis.

### Tes unit: satu fungsi, satu perilaku

Menguji satu fungsi secara langsung, tanpa server dan tanpa basis data. Contohnya:

- `interest_fit` menghasilkan angka yang sama dengan `np.corrcoef` (`test_recommender.py`),
- `score` menjumlahkan jawaban dengan benar dan menolak profil datar (`test_questionnaire.py`),
- `education` hanya menampilkan pendidikan di atas 5% (`test_details.py`).

### Tes integrasi: beberapa bagian bekerja bersama

- `test_storage.py` menguji `Store` dengan basis data sungguhan: file SQLite sementara, atau Postgres jika
  ada.
- `test_web.py` memanggil API lewat `TestClient`, sehingga yang diuji adalah seluruh rantai: Pydantic →
  endpoint → `service` → `recommender` → `storage`. Setiap tes memakai basis data SQLite baru di folder
  sementara (`setUp`, baris 29–39), jadi tes tidak saling memengaruhi dan tidak menyentuh data asli.

Beberapa tes di `test_web.py` membuat **dua `TestClient`** untuk meniru dua orang atau dua perangkat yang
berbeda, misalnya untuk memastikan siswa B tidak bisa membuka Riwayat siswa A, atau bahwa tes yang dikerjakan
di satu perangkat terlihat di perangkat lain.

### Evaluasi kualitas: siswa sintetis

`test_quality_on_synthetic_students` (`test_recommender.py` baris 54–69) tidak menguji kebenaran kode,
melainkan **kualitas rekomendasi**. Tes ini membuat 878 siswa tiruan dari profil pekerjaan yang diberi gangguan
acak, lalu mengukur:

| Ukuran | Hasil | Batas minimal |
|---|---|---|
| Hit@10 | 67,4% | 60% |
| Related Recall@10 | 18,4% | 15% |

Batas minimal berfungsi sebagai **pengaman regresi**: jika suatu saat kode diubah dan kualitas rekomendasi
turun di bawah batas itu, tes ini gagal. Penjelasan lengkap dan cara membaca angkanya ada di Bab 6.8.

## 11.5 Hal penting yang dijaga oleh tes

Beberapa jaminan penting sistem ini dipastikan oleh tes:

| Jaminan | Tes |
|---|---|
| Kecocokan benar-benar korelasi Pearson | `test_fit_is_the_pearson_correlation` |
| Perbedaan skala siswa dan pekerjaan tidak berpengaruh | `test_riasec_scale_does_not_matter` |
| Profil datar tidak menghasilkan rekomendasi | `test_rejects_incomplete_invalid_and_flat_answers`, `test_identical_answers_give_no_cards` |
| Rencana pendidikan menyaring Job Zone | `test_education_plan_limits_the_job_zone` |
| Password tidak pernah disimpan dalam bentuk asli | `test_register_signs_in_and_never_stores_the_password` |
| Pesan salah masuk tidak membocorkan email yang terdaftar | `test_login_logout_and_wrong_password` |
| Percobaan masuk berulang diblokir | `test_repeated_wrong_passwords_are_blocked_for_that_email` |
| Data siswa hanya bisa diakses pemiliknya | `test_history_belongs_to_the_account`, `test_feedback_is_kept_with_the_students_own_result`, `test_results_and_feedback_belong_to_their_user` |
| Ekspor feedback tidak memuat nama dan email | `test_feedback_is_kept_with_the_students_own_result` |
| SQLite dan Postgres berperilaku sama | `StorageContract` di `test_storage.py` |
| Semua pekerjaan punya terjemahan | `test_every_occupation_has_indonesian_title_and_description` |

## 11.6 Yang tidak diuji

Supaya laporan jujur, berikut hal yang **tidak** dicakup tes otomatis:

1. **Tampilan dan interaksi di browser.** Klik, animasi, perpindahan halaman, diagram radar, dan jaringan
   karier tidak diuji otomatis. Tes hanya memastikan file tampilan terkirim dan semua `import` JavaScript
   valid (Bab 10.10). Bagian ini diperiksa secara manual dengan membuka situsnya.
2. **Sinkronisasi browser–server di `sync.js`.** Aturan di Bab 10.7 berjalan di browser dan tidak punya tes.
   Sisi servernya (`/api/state`, `/api/progress`) diuji.
3. **Postgres**, kecuali `TEST_DATABASE_URL` diisi. Pada hasil di 11.2, tes Postgres dilewati.
4. **Situs yang sudah dipasang di Vercel.** Tes berjalan di komputer pengembang, bukan terhadap situs daring.
5. **Kualitas rekomendasi untuk siswa sungguhan.** Evaluasi hanya memakai siswa sintetis. Belum ada data dari
   siswa SMA nyata, dan terjemahan Indonesia butir kuesioner belum divalidasi (Bab 4.6 dan 5.1).

## 11.7 Ringkasan

- Jalankan semua tes dengan `.venv/bin/python -m unittest discover tests`.
- Hasil saat ini: **42 tes, 37 lulus, 5 dilewati (Postgres), 0 gagal.**
- Tes mencakup perhitungan rekomendasi, kuesioner, terjemahan, detail pekerjaan, penyimpanan, dan seluruh
  API.
- Kualitas rekomendasi dijaga oleh evaluasi siswa sintetis: Hit@10 67,4%.
- Tampilan browser dan kualitas untuk siswa sungguhan belum diuji otomatis.
