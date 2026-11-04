# Bab 3. Memuat Data ke Memori (`crs/data.py`)

Bab 2 menjelaskan isi file CSV. Bab ini menjelaskan bagaimana `crs/data.py` membaca file-file itu dan
mengubahnya menjadi **matriks NumPy** yang siap dipakai untuk perhitungan. Semua modul lain (`recommender.py`,
`service.py`) mengambil data O*NET dari sini, bukan langsung dari CSV.

File ini hanya membaca tiga file: `onet_ratings.csv`, `onet_occupations.csv`, dan
`onet_related_occupations.csv`. Tiga file lainnya (pendidikan, gaya kerja, software) dibaca oleh
`crs/details.py` dan dibahas di Bab 7.

Contoh yang dipakai tetap **Data Scientists** (`15-2051.00`).

## 3.1 Konstanta

`crs/data.py` baris 10–19:

```python
DATA_DIR = Path(__file__).resolve().parent.parent / "data"

RIASEC = ["Realistic", "Investigative", "Artistic", "Social", "Enterprising", "Conventional"]
COMPETENCY_DOMAINS = ["Abilities", "Knowledge", "EssentialSkills", "TransferableSkills"]
JOB_ZONE_NAMES = {
    2: "Job Zone 1-2: sedikit persiapan (SMA/SMK atau pelatihan singkat)",
    3: "Job Zone 3: persiapan menengah (pelatihan vokasi / D3)",
    4: "Job Zone 4: persiapan tinggi (umumnya S1)",
    5: "Job Zone 5: persiapan sangat tinggi (umumnya S2/S3)",
}
```

- **`DATA_DIR`**: lokasi folder `data/`. `Path(__file__)` adalah lokasi `crs/data.py`; `.parent.parent` naik
  dua tingkat ke folder utama proyek. Dengan cara ini, data tetap ditemukan dari mana pun program dijalankan.
- **`RIASEC`**: urutan enam tipe minat. Urutan ini **tetap** dan dipakai di seluruh program: kolom ke-0
  matriks RIASEC selalu Realistic, kolom ke-1 selalu Investigative, dan seterusnya. Modul lain mengimpor
  daftar ini dari sini supaya urutannya tidak pernah berbeda.
- **`COMPETENCY_DOMAINS`**: empat domain yang dihitung sebagai kompetensi. `WorkActivities` sengaja tidak
  dimasukkan, jadi walaupun ada di CSV, domain itu tidak pernah dibaca.
- **`JOB_ZONE_NAMES`**: label Job Zone untuk ditampilkan (Bab 2.4).

## 3.2 Fungsi `load()` dan cache

`crs/data.py` baris 42–44:

```python
@lru_cache(maxsize=1)
def load(data_dir=DATA_DIR):
    """Occupations that lack interest or competency ratings are left out (878 remain in O*NET 31.0)."""
```

`onet_ratings.csv` berisi lebih dari 370 ribu baris, jadi membacanya butuh beberapa detik. Dekorator
`@lru_cache(maxsize=1)` membuat hasil `load()` disimpan setelah pemanggilan pertama. Pemanggilan
berikutnya langsung mengembalikan hasil yang sama tanpa membaca file lagi.

Jadi data dibaca **sekali saja** selama server berjalan. Inilah sebabnya README menyebut permintaan
pertama setelah server lama tidak aktif terasa lebih lambat beberapa detik.

`load()` dipanggil dari `Recommender.__init__` (`crs/recommender.py` baris 48):

```python
        self.data = data or load()
```

Parameter `data_dir` bisa diganti, misalnya untuk tes yang memakai data lain. Selain itu, nilai bawaannya
`DATA_DIR`.

## 3.3 Membaca rating menjadi kamus

`crs/data.py` baris 45–48:

```python
    ratings = defaultdict(dict)  # soc -> {(domain, element, scale): value}
    with open(Path(data_dir) / "onet_ratings.csv", encoding="utf-8") as f:
        for r in csv.DictReader(f):
            ratings[r["soc_code"]][(r["domain"], r["element_name"], r["scale_id"])] = float(r["value"])
```

Setiap baris CSV dimasukkan ke kamus bertingkat dua:

- kunci luar: kode pekerjaan,
- kunci dalam: tuple **(domain, nama elemen, skala)**,
- nilai: angka rating (diubah dari teks ke `float`).

Untuk Data Scientists, isinya kira-kira seperti ini:

```python
ratings["15-2051.00"] = {
    ("CareerInterestTypes", "Realistic", "OI"): 2.17,
    ("CareerInterestTypes", "Investigative", "OI"): 6.98,
    ...
    ("SpecificInterestAreas", "Mathematics/Statistics", "OI"): 6.85,
    ...
    ("EssentialSkills", "Mathematics", "IM"): 4.38,
    ("EssentialSkills", "Mathematics", "LV"): 5.00,
    ("Knowledge", "Mathematics", "IM"): 4.52,
    ("Knowledge", "Mathematics", "LV"): 5.68,
    ...
}
```

Domain ikut menjadi bagian kunci karena ada nama elemen yang sama di domain berbeda, seperti
"Mathematics" di atas. Kalau kuncinya hanya nama elemen, nilai yang satu akan menimpa yang lain.

`defaultdict(dict)` membuat kamus dalam secara otomatis saat sebuah kode pekerjaan pertama kali muncul,
jadi tidak perlu dicek dulu apakah kodenya sudah ada.

Setelah langkah ini, `ratings` berisi **923 pekerjaan**, yaitu semua pekerjaan yang punya setidaknya satu
rating.

## 3.4 Menentukan elemen yang wajib ada

`crs/data.py` baris 50–54:

```python
    areas = sorted({k[1] for v in ratings.values() for k in v if k[0] == "SpecificInterestAreas"})
    competencies = sorted({(k[0], k[1]) for v in ratings.values() for k in v if k[0] in COMPETENCY_DOMAINS})
    required = ([("CareerInterestTypes", n, "OI") for n in RIASEC]
                + [("SpecificInterestAreas", a, "OI") for a in areas]
                + [(d, e, s) for d, e in competencies for s in ("IM", "LV")])
```

1. **`areas`**: semua nama area minat yang muncul di data, tanpa duplikat (pakai `set`), lalu diurutkan
   menurut abjad. Hasilnya 41 area: `['Accounting', 'Agriculture', 'Animal Service', ...]`.
2. **`competencies`**: semua pasangan (domain, nama) dari empat domain kompetensi, juga diurutkan. Hasilnya
   120 kompetensi: `[('Abilities', 'Arm-Hand Steadiness'), ('Abilities', 'Auditory Attention'), ...]`.
3. **`required`**: daftar semua kunci yang **harus** dimiliki sebuah pekerjaan supaya bisa dipakai:

   | Bagian | Jumlah kunci |
   |---|---|
   | 6 tipe RIASEC, skala OI | 6 |
   | 41 area minat, skala OI | 41 |
   | 120 kompetensi × 2 skala (IM dan LV) | 240 |
   | **Total** | **287** |

Daftar area dan kompetensi diurutkan supaya urutan kolom matriks selalu sama setiap kali program
dijalankan. Tanpa `sorted`, urutan isi `set` bisa berbeda-beda.

## 3.5 Menyaring pekerjaan: dari 1.016 menjadi 878

Pertama, informasi dasar pekerjaan dibaca dari `onet_occupations.csv` (`crs/data.py` baris 56–59):

```python
    meta = {}
    with open(Path(data_dir) / "onet_occupations.csv", encoding="utf-8") as f:
        for r in csv.DictReader(f):
            meta[r["soc_code"]] = r
```

`meta` memetakan kode pekerjaan ke barisnya (judul, Job Zone, deskripsi).

Lalu pekerjaan disaring (`crs/data.py` baris 61–63):

```python
    codes = sorted(c for c, v in ratings.items()
                   if all(k in v for k in required) and meta[c]["job_zone"])
    row = {c: i for i, c in enumerate(codes)}
```

Sebuah pekerjaan hanya dipakai jika:

1. **semua 287 kunci wajib ada** di ratingnya (`all(k in v for k in required)`), dan
2. **punya Job Zone** (`meta[c]["job_zone"]` tidak kosong).

Pekerjaan yang datanya tidak lengkap dibuang karena tidak bisa dibandingkan secara adil. Matriks juga butuh
setiap baris terisi penuh.

Rincian penyaringannya:

| Tahap | Jumlah pekerjaan |
|---|---|
| Semua pekerjaan di `onet_occupations.csv` | 1.016 |
| Punya rating sama sekali (ada di `onet_ratings.csv`) | 923 |
| Dibuang: tidak punya rating area minat | −32 |
| Dibuang: tidak punya rating kompetensi | −13 |
| **Dipakai** | **878** |

93 pekerjaan yang tidak punya rating sama sekali kebetulan juga pekerjaan yang tidak punya Job Zone. Jadi
syarat Job Zone di data 31.0 ini tidak membuang pekerjaan tambahan. Syarat itu tetap perlu sebagai
pengaman, karena tanpa Job Zone, `int(meta[c]["job_zone"])` di baris 79 akan gagal.

`codes` diurutkan menurut kode pekerjaan. Lalu `row` dibuat sebagai kamus kebalikannya, yaitu kode →
nomor baris:

```python
row = {"11-1011.00": 0, "11-1011.03": 1, ..., "15-2051.00": 119, ...}
```

**Nomor baris inilah identitas pekerjaan di dalam program.** Data Scientists ada di baris 119, artinya
baris ke-119 di setiap matriks (RIASEC, area minat, kompetensi) adalah milik Data Scientists.

## 3.6 Pekerjaan terkait

`crs/data.py` baris 65–70:

```python
    related = defaultdict(list)
    with open(Path(data_dir) / "onet_related_occupations.csv", encoding="utf-8") as f:
        links = sorted(csv.DictReader(f), key=lambda r: (r["soc_code"], int(r["rank"])))
    for r in links:
        if r["soc_code"] in row and r["related_soc_code"] in row:
            related[row[r["soc_code"]]].append(row[r["related_soc_code"]])
```

1. Semua tautan dibaca lalu diurutkan per pekerjaan asal, kemudian per `rank` (1 = paling dekat).
   `int(...)` perlu karena `rank` dibaca sebagai teks; tanpa itu urutannya menjadi "1, 10, 11, ..., 2".
2. Tautan hanya disimpan jika **pekerjaan asal dan tujuannya sama-sama termasuk 878 pekerjaan**. Tautan
   ke pekerjaan yang sudah dibuang tidak berguna, karena pekerjaan itu tidak punya data untuk ditampilkan.
3. Tautan disimpan dalam bentuk nomor baris, bukan kode.

Untuk Data Scientists (baris 119):

```python
related[119] = [117, 606, 85, 116, 181, ...]   # 20 pekerjaan
#              Statisticians, Statistical Assistants, Financial Quantitative Analysts, ...
```

Dari 18.460 tautan di CSV, 16.803 tersisa setelah penyaringan. Setiap pekerjaan punya 6 sampai 20 pekerjaan
terkait.

## 3.7 Menyusun matriks

`crs/data.py` baris 72–73:

```python
    def matrix(keys):
        return np.array([[ratings[c][k] for k in keys] for c in codes])
```

Fungsi kecil ini membuat matriks dengan:

- **baris** = pekerjaan, sesuai urutan `codes`,
- **kolom** = kunci rating, sesuai urutan `keys`.

Karena semua pekerjaan di `codes` sudah dipastikan punya semua kunci wajib (subbab 3.5), `ratings[c][k]`
tidak akan pernah gagal.

Fungsi ini dipanggil empat kali (`crs/data.py` baris 80–85):

```python
        riasec=matrix([("CareerInterestTypes", n, "OI") for n in RIASEC]),
        ...
        area_scores=matrix([("SpecificInterestAreas", a, "OI") for a in areas]),
        ...
        importance=matrix([(d, e, "IM") for d, e in competencies]),
        level=matrix([(d, e, "LV") for d, e in competencies]),
```

| Matriks | Ukuran | Skala | Dipakai untuk |
|---|---|---|---|
| `riasec` | 878 × 6 | OI, 1–7 | **perhitungan kecocokan** (Bab 6) |
| `area_scores` | 878 × 41 | OI, 1–7 | alasan "Mengapa cocok" (Bab 7) |
| `importance` | 878 × 120 | IM, 1–5 | memilih kompetensi penting (Bab 6) |
| `level` | 878 × 120 | LV, 0–7 | level kompetensi yang dibutuhkan (Bab 6) |

Contoh baris Data Scientists di matriks `riasec`:

```python
riasec[119] = [2.17, 6.98, 2.61, 1.66, 1.71, 5.39]
#               R     I     A     S     E     C
```

Menyimpan data dalam bentuk matriks membuat perhitungan jauh lebih cepat. Di Bab 6, kecocokan siswa
dengan 878 pekerjaan dihitung dengan **satu** perkalian matriks, bukan 878 kali perulangan.

## 3.8 Kelas `OnetData`

Semua hasil di atas dikumpulkan ke satu objek (`crs/data.py` baris 75–87):

```python
    return OnetData(
        codes=codes,
        titles=[meta[c]["title"] for c in codes],
        descriptions=[meta[c]["description"] for c in codes],
        job_zones=np.array([int(meta[c]["job_zone"]) for c in codes]),
        riasec=matrix([("CareerInterestTypes", n, "OI") for n in RIASEC]),
        areas=areas,
        area_scores=matrix([("SpecificInterestAreas", a, "OI") for a in areas]),
        competencies=competencies,
        importance=matrix([(d, e, "IM") for d, e in competencies]),
        level=matrix([(d, e, "LV") for d, e in competencies]),
        related=dict(related),
    )
```

Bentuk objeknya didefinisikan sebagai `dataclass` (`crs/data.py` baris 22–34):

```python
@dataclass
class OnetData:
    codes: list            # O*NET-SOC codes, row order of every matrix below
    titles: list
    descriptions: list
    job_zones: np.ndarray  # 2..5 (O*NET 31.0 merges zones 1 and 2)
    riasec: np.ndarray     # occupations x 6, Occupational Interest scale 1..7
    areas: list            # the 41 Specific Interest Area names
    area_scores: np.ndarray        # occupations x 41, scale 1..7
    competencies: list             # (domain, element name)
    importance: np.ndarray         # occupations x competencies, scale 1..5
    level: np.ndarray              # occupations x competencies, scale 0..7
    related: dict                  # row index -> [row index, ...] ordered by O*NET's related index
```

`@dataclass` membuat Python otomatis menyediakan konstruktor dari daftar atribut ini, jadi kelasnya cukup
berisi deklarasi atribut.

Hal terpenting dari kelas ini: **semua daftar dan matriks memakai urutan baris yang sama**. Untuk
pekerjaan di baris `i`:

| Atribut | Isi untuk Data Scientists (`i = 119`) |
|---|---|
| `codes[i]` | `"15-2051.00"` |
| `titles[i]` | `"Data Scientists"` |
| `descriptions[i]` | `"Develop and implement a set of techniques ..."` |
| `job_zones[i]` | `4` |
| `riasec[i]` | `[2.17, 6.98, 2.61, 1.66, 1.71, 5.39]` |
| `area_scores[i]` | 41 angka, urut sesuai `areas` |
| `importance[i]`, `level[i]` | 120 angka, urut sesuai `competencies` |
| `related[i]` | `[117, 606, 85, ...]` |

Begitu juga untuk kolom: kolom `j` di `area_scores` adalah area `areas[j]`, dan kolom `j` di `importance`
serta `level` adalah kompetensi `competencies[j]`.

### Method `index_of`

`crs/data.py` baris 36–39:

```python
    def index_of(self, code_or_title):
        if code_or_title in self.codes:
            return self.codes.index(code_or_title)
        return self.titles.index(code_or_title)
```

Method ini mencari nomor baris sebuah pekerjaan dari kodenya (`"15-2051.00"`) atau judul bahasa Inggrisnya
(`"Data Scientists"`). Keduanya menghasilkan `119`. Method ini dipakai oleh `Recommender.competencies()`
saat yang diberikan berupa kode, bukan nomor baris (Bab 6).

## 3.9 Ringkasan

```
onet_ratings.csv ──► ratings: {kode: {(domain, elemen, skala): nilai}}      923 pekerjaan
                          │
                          │  wajib punya 287 kunci + Job Zone
                          ▼
onet_occupations.csv ──► codes (878, terurut) dan row {kode: nomor baris}
                          │
                          ├──► riasec        878 × 6
                          ├──► area_scores   878 × 41
                          ├──► importance    878 × 120
                          ├──► level         878 × 120
                          │
onet_related_occupations.csv ──► related {nomor baris: [nomor baris, ...]}
                          │
                          ▼
                     OnetData   (disimpan oleh lru_cache, dibaca sekali)
```

- Data dibaca **sekali** dan disimpan di memori selama server berjalan.
- Hanya **878 pekerjaan dengan data lengkap** yang dipakai.
- Setiap pekerjaan dikenali dari **nomor barisnya**, yang sama di semua matriks.
- Bab 4 menjelaskan bagaimana judul dan deskripsi bahasa Inggris di `OnetData` diganti dengan bahasa
  Indonesia saat ditampilkan.
