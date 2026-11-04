# Bab 4. Terjemahan ke Bahasa Indonesia (`crs/i18n/`)

Semua teks di data O*NET berbahasa Inggris, sedangkan pengguna texin adalah siswa SMA di Indonesia. Bab ini
menjelaskan bagaimana judul pekerjaan, deskripsi pekerjaan, dan nama kompetensi ditampilkan dalam bahasa
Indonesia.

## 4.1 Prinsip: data asli tidak diubah

Terjemahan **tidak** ditulis ke file O*NET di `data/`. `OnetData` (Bab 3) tetap menyimpan teks bahasa
Inggris aslinya. Terjemahan disimpan terpisah di folder `crs/i18n/` dan baru dipakai saat teks akan
ditampilkan ke siswa.

Pemisahan ini punya tiga keuntungan:

1. Data O*NET tetap sama persis dengan sumber resminya, jadi mudah diperbarui jika ada versi O*NET baru.
2. Terjemahan bisa dikoreksi langsung di file CSV, tanpa mengubah kode.
3. Jika sebuah terjemahan kosong atau tidak ada, aplikasi otomatis memakai teks bahasa Inggrisnya
   (*fallback*), sehingga tidak pernah ada teks yang hilang dari layar.

Prinsip ini tertulis di docstring `crs/i18n/__init__.py` baris 1–6:

```python
"""Indonesian text for what students see: occupation titles and descriptions and competency names.

The CSV files in this folder are the source of truth and can be corrected by hand. The
translations were written for this project and have not been validated by a language or
subject expert. Anything missing from a file falls back to the O*NET English text.
"""
```

### Cara terjemahan dibuat

Terjemahan judul pekerjaan, deskripsi pekerjaan, dan nama kompetensi dibuat untuk proyek ini **dengan
bantuan AI**. Terjemahan ini **belum divalidasi** oleh ahli bahasa maupun ahli di bidang pekerjaan
terkait. Karena itu, file CSV-nya sengaja dibuat mudah dikoreksi: setiap baris berisi teks asli bahasa
Inggris di samping terjemahannya.

## 4.2 File terjemahan

### `pekerjaan.csv`: judul dan deskripsi pekerjaan

Kolom: `kode, judul_en, judul_id, deskripsi_id`. Ada 878 baris, satu untuk setiap pekerjaan yang dipakai
aplikasi (Bab 3.5).

```
kode,judul_en,judul_id,deskripsi_id
15-2051.00,Data Scientists,Ilmuwan Data,"Mengembangkan dan menerapkan teknik atau aplikasi analitik
untuk mengubah data mentah menjadi informasi yang bermakna dengan bahasa pemrograman data dan perangkat
lunak visualisasi. ..."
```

| Kolom | Isi |
|---|---|
| `kode` | kode O*NET-SOC, kunci penghubung dengan data O*NET |
| `judul_en` | judul asli bahasa Inggris, sebagai pembanding saat mengoreksi |
| `judul_id` | judul bahasa Indonesia yang ditampilkan |
| `deskripsi_id` | deskripsi bahasa Indonesia yang ditampilkan |

Kolom `judul_en` tidak dipakai untuk menampilkan apa pun. Kolom ini ada supaya orang yang mengoreksi bisa
langsung melihat judul aslinya, dan supaya tes bisa memastikan isi file ini cocok dengan data O*NET
(subbab 4.5).

### `kompetensi.csv`: nama kompetensi

Kolom: `nama_en, nama_id`. Ada 119 baris.

```
nama_en,nama_id
Active Learning,Belajar Aktif
Active Listening,Mendengarkan Aktif
Critical Thinking,Berpikir Kritis
Mathematics,Matematika
Programming,Pemrograman
```

Jumlahnya 119, bukan 120 seperti jumlah kompetensi di Bab 3, karena file ini memakai **nama saja**
sebagai kunci, tanpa domain. "Mathematics" ada di dua domain (Knowledge dan EssentialSkills), tapi
terjemahannya sama, jadi cukup ditulis sekali.

## 4.3 Membaca file terjemahan

`crs/i18n/__init__.py` baris 11–26:

```python
HERE = Path(__file__).resolve().parent


def _rows(name):
    with open(HERE / name, encoding="utf-8") as f:
        return list(csv.DictReader(f))


@lru_cache(maxsize=1)
def _occupations():
    return {r["kode"]: r for r in _rows("pekerjaan.csv")}


@lru_cache(maxsize=1)
def _competencies():
    return {r["nama_en"]: r["nama_id"] for r in _rows("kompetensi.csv")}
```

- **`HERE`**: folder `crs/i18n/` itu sendiri. File CSV disimpan di sebelah kodenya.
- **`_rows(name)`**: membaca satu file CSV menjadi daftar kamus, satu kamus per baris.
- **`_occupations()`**: kamus kode pekerjaan → satu baris lengkap, misalnya
  `{"15-2051.00": {"kode": ..., "judul_id": "Ilmuwan Data", "deskripsi_id": ...}}`.
- **`_competencies()`**: kamus nama Inggris → nama Indonesia, misalnya
  `{"Critical Thinking": "Berpikir Kritis"}`.

Kedua fungsi diberi `@lru_cache`, sama seperti `load()` di Bab 3, sehingga file hanya dibaca sekali. Nama
fungsinya diawali garis bawah (`_`), tanda dalam Python bahwa fungsi itu hanya untuk dipakai di dalam
modul ini.

## 4.4 Fungsi terjemahan dan fallback

`crs/i18n/__init__.py` baris 29–40:

```python
def occupation_title(code, fallback):
    row = _occupations().get(code)
    return row["judul_id"] if row and row["judul_id"] else fallback


def occupation_description(code, fallback):
    row = _occupations().get(code)
    return row["deskripsi_id"] if row and row["deskripsi_id"] else fallback


def competency(name):
    return _competencies().get(name) or name
```

Ketiga fungsi bekerja dengan pola yang sama: **cari terjemahannya; kalau tidak ada atau kosong, kembalikan
teks aslinya.**

| Fungsi | Contoh panggilan | Hasil |
|---|---|---|
| `occupation_title` | `("15-2051.00", "Data Scientists")` | `"Ilmuwan Data"` |
| `occupation_title` | kode yang tidak ada di file | teks `fallback` (judul Inggris) |
| `occupation_description` | `("15-2051.00", "Develop and implement ...")` | `"Mengembangkan dan menerapkan ..."` |
| `competency` | `("Critical Thinking")` | `"Berpikir Kritis"` |
| `competency` | `("Not A Competency")` | `"Not A Competency"` |

Untuk judul dan deskripsi, teks Inggris diberikan oleh pemanggil sebagai parameter `fallback`. Untuk
kompetensi, nama Inggrisnya sudah menjadi parameter, jadi nama itu sendiri yang dikembalikan
(`.get(name) or name`).

Kondisi `row and row["judul_id"]` memeriksa dua hal: barisnya ada di file, **dan** kolom terjemahannya
tidak kosong. Jadi baris yang terjemahannya dikosongkan juga aman.

### Di mana fungsi ini dipakai

Terjemahan hanya dipakai di `crs/service.py`, lapisan yang menyusun data untuk ditampilkan:

`crs/service.py` baris 44–46, judul pekerjaan:

```python
def title(i):
    d = recommender().data
    return i18n.occupation_title(d.codes[i], d.titles[i])
```

`crs/service.py` baris 54–58, kartu hasil rekomendasi:

```python
    return {
        "code": d.codes[i], "title": title(i), "title_en": d.titles[i],
        "description": i18n.occupation_description(d.codes[i], d.descriptions[i]),
        "job_zone": zone, "job_zone_name": JOB_ZONE_NAMES[zone],
    }
```

Kartu menyimpan judul Indonesia (`title`) **dan** judul aslinya (`title_en`). Judul asli tetap dikirim
supaya siswa bisa mencari pekerjaan itu sendiri di sumber berbahasa Inggris.

`crs/service.py` baris 69–70, nama kompetensi di halaman detail:

```python
    detail["skills"] = [{"name": i18n.competency(g.name), "importance": round(g.importance, 2),
                         "level": round(g.required * 7, 1)} for g in rec.competencies(i, limit=10)]
```

Mesin rekomendasi (`crs/recommender.py`) tidak pernah memakai terjemahan. Perhitungan selalu memakai data
O*NET asli; terjemahan baru diterapkan di tahap akhir, saat hasil disiapkan untuk layar.

## 4.5 Tes terjemahan

Karena terjemahan disimpan di file yang bisa diedit tangan, ada tes untuk memastikan file itu tetap lengkap
dan cocok dengan data. Tesnya ada di `tests/test_i18n.py`.

**Setiap pekerjaan punya judul dan deskripsi Indonesia** (baris 8–14):

```python
    def test_every_occupation_has_indonesian_title_and_description(self):
        d = load()
        for code, title, description in zip(d.codes, d.titles, d.descriptions):
            self.assertNotEqual(i18n.occupation_description(code, description), description, code)
            self.assertTrue(i18n.occupation_title(code, title), code)
        titles = [i18n.occupation_title(c, t) for c, t in zip(d.codes, d.titles)]
        self.assertEqual(len(set(titles)), len(titles))   # students must be able to tell them apart
```

- Deskripsi hasil terjemahan harus **berbeda** dari deskripsi Inggris. Jika sama, berarti fallback
  terpakai dan terjemahannya belum ada.
- Semua judul Indonesia harus **unik**. Dua pekerjaan berbeda tidak boleh punya judul Indonesia yang sama,
  supaya siswa bisa membedakannya.

**Isi `pekerjaan.csv` cocok dengan data O*NET** (baris 16–20):

```python
    def test_translation_file_matches_the_onet_titles(self):
        d = load()
        with open(i18n.HERE / "pekerjaan.csv", encoding="utf-8") as f:
            rows = {r["kode"]: r["judul_en"] for r in csv.DictReader(f)}
        self.assertEqual(rows, dict(zip(d.codes, d.titles)))
```

Pasangan kode dan judul Inggris di file harus sama persis dengan 878 pekerjaan di `OnetData`: tidak ada
yang kurang, tidak ada yang lebih, dan tidak ada yang tertukar. Tes ini akan gagal jika data O*NET diganti
tapi file terjemahan belum diperbarui.

**Setiap kompetensi punya terjemahan** (baris 22–26):

```python
    def test_every_competency_is_translated(self):
        for _, name in load().competencies:
            self.assertNotEqual(i18n.competency(name), "", name)
        self.assertEqual(i18n.competency("Critical Thinking"), "Berpikir Kritis")
        self.assertEqual(i18n.competency("Not A Competency"), "Not A Competency")
```

Baris terakhir memastikan fallback bekerja: nama yang tidak ada di file dikembalikan apa adanya.

## 4.6 Teks Indonesia lain di luar `crs/i18n/`

Selain dua file CSV di atas, ada beberapa daftar label Indonesia yang ditulis langsung di kode karena
jumlahnya sedikit. Semuanya juga dibuat dengan bantuan AI dan belum divalidasi.

| Label | Lokasi | Jumlah | Dibahas di |
|---|---|---|---|
| 60 butir kuesioner | `crs/questionnaire.py`, `_ITEMS` | 60 | Bab 5 |
| Nama 41 area minat | `crs/questionnaire.py`, `AREA_LABELS` | 41 | Bab 7 |
| Nama tipe RIASEC | `crs/service.py`, `RIASEC_LABELS` dan `RIASEC_NICKNAMES` | 6 | Bab 1 |
| Kategori pendidikan | `crs/details.py`, `EDUCATION_LABELS` | 12 | Bab 7 |
| Gaya kerja | `crs/details.py`, `WORK_STYLE_LABELS` | 21 | Bab 7 |

Untuk 60 butir kuesioner, hal ini juga dicatat di docstring `crs/questionnaire.py` baris 7–9. Lisensi
O*NET Tools Developer License meminta versi adaptasi (termasuk terjemahan) divalidasi untuk penggunanya
sebelum dirilis:

```python
The Indonesian wording is an unvalidated translation made for this project. The O*NET Tools
Developer License asks that adapted versions be validated for their audience before release:
https://www.onetcenter.org/license_toolsdev.html
```

## 4.7 Ringkasan

```
OnetData (teks Inggris asli)
      │
      │  crs/service.py saat menyusun kartu dan detail
      ▼
crs/i18n/  occupation_title(), occupation_description(), competency()
      │      ├─ ada terjemahan  → teks Indonesia
      │      └─ tidak ada       → teks Inggris (fallback)
      ▼
JSON ke browser (judul Indonesia + judul Inggris)
```

- Terjemahan disimpan **terpisah** dari data O*NET dan hanya dipakai saat menampilkan.
- Terjemahan dibuat **dengan bantuan AI** dan belum divalidasi ahli.
- Jika terjemahan tidak ada, **teks Inggris** yang ditampilkan.
- Tes memastikan file terjemahan lengkap, judulnya unik, dan cocok dengan data O*NET.
