# Bab 6. Mesin Rekomendasi (`crs/recommender.py`)

Bab ini adalah inti sistem. Di Bab 3, setiap pekerjaan sudah punya profil enam angka RIASEC (skala 1–7).
Di Bab 5, setiap siswa juga punya profil enam angka RIASEC (skala 0–40). Bab ini menjelaskan bagaimana kedua
profil itu dibandingkan untuk mendapat angka **kecocokan**, serta bagaimana kompetensi tiap pekerjaan
dipilih untuk ditampilkan.

Contoh yang dipakai tetap siswa dari Bab 5 dan pekerjaan **Data Scientists** (baris 119):

| | R | I | A | S | E | C |
|---|---|---|---|---|---|---|
| Siswa (0–40) | 9 | 36 | 17 | 12 | 8 | 28 |
| Data Scientists (1–7) | 2,17 | 6,98 | 2,61 | 1,66 | 1,71 | 5,39 |

## 6.1 Metode: Content-Based Filtering

Docstring `crs/recommender.py` baris 1–9:

```python
"""Career recommendation for high-school students, based on O*NET 31.0.

Content-based filtering on interests: an occupation's fit is the Pearson correlation between the
student's six RIASEC scores and the occupation's RIASEC profile, the method O*NET's own Interest
Profiler uses to link a profile to occupations.

Competencies do not affect the ranking. They are reported per occupation: what the student would
need to develop to get there.
"""
```

**Content-Based Filtering** merekomendasikan item berdasarkan kemiripan **ciri item** dengan **ciri
pengguna**. Di texin:

- **item** = pekerjaan, cirinya profil RIASEC dari O*NET,
- **pengguna** = siswa, cirinya profil RIASEC dari kuesioner,
- **kemiripan** = korelasi Pearson antara kedua profil.

Metode ini tidak butuh data pengguna lain. Hal ini cocok untuk texin, karena tidak ada riwayat siswa lain
yang bisa dipakai untuk metode seperti Collaborative Filtering.

Kompetensi sengaja **tidak** ikut menentukan urutan. Siswa SMA belum punya kompetensi kerja yang bisa
diukur, jadi kompetensi ditampilkan sebagai gambaran apa yang perlu dikembangkan, bukan sebagai syarat.

## 6.2 Rumus Pearson Correlation

Korelasi Pearson antara dua deret angka *x* dan *y* (di sini masing-masing enam angka):

```
            Σ (xᵢ − x̄)(yᵢ − ȳ)
r = ─────────────────────────────────
     √Σ (xᵢ − x̄)²  ·  √Σ (yᵢ − ȳ)²
```

- *x̄* dan *ȳ* adalah rata-rata masing-masing deret.
- Pembilang: seberapa sering kedua deret **sama-sama** di atas atau sama-sama di bawah rata-ratanya.
- Penyebut: membagi dengan "panjang" simpangan kedua deret, sehingga hasilnya selalu antara −1 dan 1.

| Nilai *r* | Arti |
|---|---|
| 1 | pola naik-turunnya identik |
| 0 | tidak ada hubungan pola |
| −1 | pola berlawanan: yang tinggi di satu deret rendah di deret lain |

### Kenapa Pearson cocok di sini

Korelasi Pearson hanya melihat **pola**, bukan besar angkanya. Ini penting karena:

1. **Skala siswa dan pekerjaan berbeda** (0–40 vs. 1–7). Pearson tidak terpengaruh perbedaan skala.
2. **Gaya menjawab siswa berbeda-beda.** Ada siswa yang cenderung menjawab "Suka" untuk semua hal, ada yang
   pelit memberi nilai tinggi. Pearson mengurangi rata-rata tiap siswa, sehingga yang dibandingkan hanya
   tipe mana yang **relatif** lebih disukai.

Ini juga metode yang dipakai O*NET Interest Profiler resmi untuk mencocokkan profil minat dengan pekerjaan.

## 6.3 Contoh perhitungan langkah demi langkah

### Langkah 1: Kurangi rata-rata

Rata-rata skor siswa: (9 + 36 + 17 + 12 + 8 + 28) / 6 = **18,33**.
Rata-rata Data Scientists: (2,17 + 6,98 + 2,61 + 1,66 + 1,71 + 5,39) / 6 = **3,42**.

| | R | I | A | S | E | C |
|---|---|---|---|---|---|---|
| Siswa − 18,33 | −9,33 | 17,67 | −1,33 | −6,33 | −10,33 | 9,67 |
| Data Scientists − 3,42 | −1,25 | 3,56 | −0,81 | −1,76 | −1,71 | 1,97 |

Tanda **+** berarti tipe itu di atas rata-rata, **−** berarti di bawah rata-rata. Polanya sudah terlihat
mirip: keduanya positif di I dan C, negatif di R, A, S, dan E.

### Langkah 2: Bagi dengan panjangnya

Panjang (norma) simpangan siswa: √(9,33² + 17,67² + 1,33² + 6,33² + 10,33² + 9,67²) = **25,32**.
Panjang simpangan Data Scientists: √(1,25² + 3,56² + … + 1,97²) = **4,98**.

| | R | I | A | S | E | C |
|---|---|---|---|---|---|---|
| Siswa (terstandar) | −0,3685 | 0,6976 | −0,0526 | −0,2501 | −0,4080 | 0,3817 |
| Data Scientists (terstandar) | −0,2510 | 0,7149 | −0,1627 | −0,3535 | −0,3434 | 0,3956 |

Setelah dibagi, skala asalnya hilang. Kedua deret sekarang bisa dibandingkan langsung, walaupun aslinya
0–40 dan 1–7.

### Langkah 3: Kalikan lalu jumlahkan

| | R | I | A | S | E | C | **Jumlah** |
|---|---|---|---|---|---|---|---|
| Hasil kali | 0,0925 | 0,4987 | 0,0086 | 0,0884 | 0,1401 | 0,1510 | **0,979** |

Korelasi siswa ini dengan Data Scientists adalah **r = 0,979**, yang ditampilkan sebagai kecocokan **98%**.

Hasil kali bernilai positif jika kedua deret sama-sama di atas rata-rata (seperti I dan C) **atau**
sama-sama di bawah rata-rata (seperti R dan E). Keduanya menambah kecocokan. Minat yang sama-sama rendah
juga merupakan bentuk kecocokan.

## 6.4 `_standardise_rows`: korelasi sebagai perkalian

Langkah 1 dan 2 di atas dikerjakan oleh satu fungsi (`crs/recommender.py` baris 39–43):

```python
def _standardise_rows(X):
    """Centre each row and scale it to length 1, so a dot product of two rows is their Pearson correlation."""
    X = X - X.mean(axis=1, keepdims=True)
    norm = np.linalg.norm(X, axis=1, keepdims=True)
    return np.divide(X, norm, out=np.zeros_like(X), where=norm > 0)
```

| Baris kode | Langkah |
|---|---|
| `X - X.mean(axis=1, keepdims=True)` | Langkah 1: kurangi rata-rata **tiap baris** (`axis=1`) |
| `np.linalg.norm(X, axis=1, keepdims=True)` | hitung panjang tiap baris |
| `np.divide(X, norm, ...)` | Langkah 2: bagi tiap baris dengan panjangnya |

`keepdims=True` menjaga hasil rata-rata dan norma tetap berbentuk kolom (878 × 1), sehingga bisa langsung
dikurangkan atau dibagikan ke matriks 878 × 6.

`where=norm > 0` dan `out=np.zeros_like(X)` adalah pengaman: jika sebuah baris datar (semua angkanya sama),
panjangnya nol. Daripada membagi dengan nol, baris itu dibiarkan bernilai nol semua.

Setelah distandarkan, **Langkah 3 cukup berupa dot product** (kalikan per elemen lalu jumlahkan). Dengan kata
lain:

```
r(x, y) = standar(x) · standar(y)
```

Penyebut pada rumus Pearson sudah "dimasukkan" ke Langkah 2, jadi rumus yang tadinya rumit menjadi satu
perkalian sederhana.

## 6.5 Kelas `Recommender`

### Menyiapkan data sekali di awal

`crs/recommender.py` baris 46–49:

```python
class Recommender:
    def __init__(self, data: OnetData | None = None):
        self.data = data or load()
        self._riasec_z = _standardise_rows(self.data.riasec)
```

Saat `Recommender` dibuat:

1. data O*NET dimuat (Bab 3),
2. **profil RIASEC 878 pekerjaan langsung distandarkan** dan disimpan di `_riasec_z` (878 × 6).

Profil pekerjaan tidak pernah berubah, jadi cukup distandarkan sekali. Setiap kali ada siswa baru, yang
perlu distandarkan hanya profil siswa itu.

Objek `Recommender` sendiri juga hanya dibuat sekali, lewat `service.recommender()` yang diberi
`@lru_cache` (`crs/service.py` baris 39–41):

```python
@lru_cache(maxsize=1)
def recommender():
    return Recommender()
```

### `interest_fit`: kecocokan dengan semua pekerjaan sekaligus

`crs/recommender.py` baris 51–58:

```python
    def interest_fit(self, riasec):
        """Pearson correlation (-1..1) between the student's RIASEC scores and every occupation's.
        riasec: {type: score} on any scale."""
        missing = [n for n in RIASEC if n not in riasec]
        if missing:
            raise ValueError(f"Skor RIASEC belum lengkap: {missing}")
        student = _standardise_rows(np.array([[float(riasec[n]) for n in RIASEC]]))[0]
        return self._riasec_z @ student
```

1. **Periksa kelengkapan.** Keenam tipe harus ada. Jika tidak, fungsi berhenti dengan pesan kesalahan.
2. **Susun skor siswa sesuai urutan `RIASEC`.** Kamus `{"Realistic": 9, ...}` diubah menjadi
   `[9, 36, 17, 12, 8, 28]`. Urutannya harus sama dengan urutan kolom `_riasec_z`, karena itulah daftar
   `RIASEC` dipakai bersama (Bab 3.1).
3. **Standarkan profil siswa.** `_standardise_rows` menerima matriks, jadi daftar enam angka dibungkus
   menjadi matriks 1 × 6 (`[[...]]`), lalu barisnya diambil kembali dengan `[0]`.
4. **Kalikan.** `@` adalah operator perkalian matriks di Python:

```
_riasec_z  @  student   =   fit
(878 × 6)     (6,)          (878,)
```

Satu perkalian ini menghitung dot product profil siswa dengan **setiap** baris pekerjaan. Hasilnya 878
angka korelasi, satu per pekerjaan, sesuai urutan baris di `OnetData`. Untuk contoh siswa kita,
`fit[119]` (Data Scientists) bernilai 0,979.

Parameter `riasec` bisa berskala apa pun ("on any scale"), karena standardisasi menghilangkan skalanya.
Skor 0–40 dari kuesioner bisa langsung dipakai tanpa diubah dulu.

### Hasil untuk contoh siswa

Dari 878 korelasi, sepuluh tertinggi untuk siswa contoh:

| # | Pekerjaan | r | Kecocokan | Label |
|---|---|---|---|---|
| 1 | Sejarawan (Historians) | 0,981 | 98% | Sangat cocok |
| 2 | Ilmuwan Data (Data Scientists) | 0,979 | 98% | Sangat cocok |
| 3 | Ahli Biostatistika (Biostatisticians) | 0,952 | 95% | Sangat cocok |
| 4 | Matematikawan (Mathematicians) | 0,943 | 94% | Sangat cocok |
| 5 | Statistikawan (Statisticians) | 0,938 | 94% | Sangat cocok |
| 6 | Peneliti Survei (Survey Researchers) | 0,922 | 92% | Sangat cocok |
| 7 | Asisten Riset Ilmu Sosial | 0,912 | 91% | Sangat cocok |
| 8 | Pengembang Web (Web Developers) | 0,907 | 91% | Sangat cocok |
| 9 | Analis Riset Operasi | 0,902 | 90% | Sangat cocok |
| 10 | Ilmuwan Riset Komputer dan Informasi | 0,891 | 89% | Sangat cocok |

Semuanya pekerjaan dengan pola Investigative–Conventional, sesuai profil siswa. Pengurutan dan penyusunan
kartu ini dikerjakan oleh `service.recommend_by_interest()` (Bab 7). `Recommender` hanya menghitung
korelasinya.

## 6.6 `fit_label`: label kecocokan

`crs/recommender.py` baris 16–19 dan 31–36:

```python
IMPORTANT = 3.0   # O*NET Importance scale: 3 = "Important"
# Cut-offs O*NET's own Interest Profiler uses for the RIASEC profile correlation
# (Gregory & Lewis, 2016, "Linking Client Assessment Profiles to O*NET Occupational Profiles").
BEST_FIT, GREAT_FIT = 0.729, 0.608
...
def fit_label(r):
    if r >= BEST_FIT:
        return "Sangat cocok"
    if r >= GREAT_FIT:
        return "Cocok"
    return "Cukup cocok" if r >= 0 else "Kurang cocok"
```

Angka korelasi diterjemahkan menjadi label yang mudah dipahami siswa:

| Korelasi *r* | Label | Pekerjaan untuk siswa contoh |
|---|---|---|
| ≥ 0,729 | Sangat cocok | 38 |
| 0,608 – 0,728 | Cocok | 46 |
| 0 – 0,607 | Cukup cocok | 384 |
| < 0 | Kurang cocok | 410 |

Batas 0,729 dan 0,608 tidak dipilih sembarangan. Keduanya adalah batas *Best Fit* dan *Great Fit* yang
dipakai O*NET Interest Profiler resmi, dari laporan Gregory & Lewis (2016), *Linking Client Assessment
Profiles to O\*NET Occupational Profiles*.

### Dari korelasi ke persen

Persen kecocokan di layar dihitung di `crs/service.py` baris 90–93:

```python
def match_percent(correlation):
    """"Kecocokan" shown to the student: the Pearson correlation between the student's six RIASEC
    scores and the occupation's, as a percentage; negative correlations count as 0."""
    return max(0, round(100 * float(correlation)))
```

Korelasi dikalikan 100, dan korelasi negatif ditampilkan 0%, karena "kecocokan −40%" membingungkan bagi
siswa. Pekerjaan dengan korelasi negatif tidak akan masuk sepuluh besar kecuali hampir semua pekerjaan
lain juga negatif, yang praktis tidak terjadi.

## 6.7 `competencies`: kompetensi yang dibutuhkan

Bagian kedua `Recommender` tidak berhubungan dengan urutan rekomendasi. Fungsinya memilih kompetensi
yang ditampilkan di halaman detail pekerjaan.

`crs/recommender.py` baris 20–28:

```python
LEVEL_MAX = 7.0   # O*NET Level scale runs 0..7


@dataclass
class Competency:
    domain: str
    name: str
    importance: float   # 1..5
    required: float     # the occupation's required level, 0..1
```

`crs/recommender.py` baris 60–69:

```python
    def competencies(self, occupation, limit=10):
        """Competencies that are important for the occupation (Importance >= 3), most important first.
        Required level is the O*NET Level scale / 7."""
        d = self.data
        i = occupation if isinstance(occupation, (int, np.integer)) else d.index_of(occupation)
        found = [Competency(domain=dom, name=name, importance=float(d.importance[i, j]),
                            required=float(d.level[i, j]) / LEVEL_MAX)
                 for j, (dom, name) in enumerate(d.competencies) if d.importance[i, j] >= IMPORTANT]
        found.sort(key=lambda c: (-c.importance, -c.required))
        return found[:limit]
```

1. **Tentukan baris pekerjaannya.** `occupation` boleh berupa nomor baris, atau kode/judul pekerjaan yang
   dicari dengan `index_of` (Bab 3.8).
2. **Saring kompetensi penting.** Dari 120 kompetensi, hanya yang Importance-nya **≥ 3** yang diambil. Di
   skala Importance O*NET (1–5), nilai 3 berarti "Important". Data Scientists punya 34 kompetensi seperti
   itu.
3. **Ubah level ke 0–1.** Level O*NET berskala 0–7, lalu dibagi `LEVEL_MAX` agar mudah ditampilkan sebagai
   bar persen.
4. **Urutkan.** Paling penting lebih dulu (`-c.importance`). Jika sama penting, level yang lebih tinggi
   lebih dulu (`-c.required`).
5. **Ambil sejumlah `limit`**, bawaannya 10.

Hasil untuk Data Scientists:

| # | Domain | Kompetensi | Importance | Level / 7 |
|---|---|---|---|---|
| 1 | Knowledge | Mathematics | 4,52 | 0,81 |
| 2 | EssentialSkills | Mathematics | 4,38 | 0,71 |
| 3 | Abilities | Deductive Reasoning | 4,25 | 0,68 |
| 4 | Abilities | Inductive Reasoning | 4,25 | 0,68 |
| 5 | EssentialSkills | Critical Thinking | 4,25 | 0,66 |
| 6 | Knowledge | Computers and Electronics | 4,24 | 0,83 |
| 7 | Abilities | Mathematical Reasoning | 4,12 | 0,63 |
| 8 | Abilities | Information Ordering | 4,12 | 0,59 |
| 9 | Abilities | Written Comprehension | 4,00 | 0,64 |
| 10 | EssentialSkills | Reading Comprehension | 3,88 | 0,66 |

Di halaman detail, `crs/service.py` mengubah level kembali ke skala 0–7 (`g.required * 7`) dan
menerjemahkan nama kompetensinya (Bab 4.4).

## 6.8 Tes mesin rekomendasi

Tesnya ada di `tests/test_recommender.py`.

**Data lengkap** (baris 15–21): 878 pekerjaan, 41 area minat, matriks RIASEC 878 × 6, tidak ada nilai
kosong (`NaN`) di matriks level, dan Job Zone hanya bernilai 2–5.

**Hasilnya benar-benar korelasi Pearson** (baris 23–29):

```python
    def test_fit_is_the_pearson_correlation(self):
        student = {"Realistic": 2, "Investigative": 9, "Artistic": 4, "Social": 5, "Enterprising": 1, "Conventional": 6}
        fit = self.rec.interest_fit(student)
        self.assertEqual(fit.shape, (878,))
        for i in (0, 100, 877):
            expected = np.corrcoef([student[t] for t in RIASEC], self.data.riasec[i])[0, 1]
            self.assertAlmostEqual(fit[i], expected)
```

Hasil `interest_fit` dibandingkan dengan `np.corrcoef`, fungsi korelasi bawaan NumPy, untuk pekerjaan
pertama, tengah, dan terakhir. Tes ini membuktikan bahwa cara cepat di 6.4 (standarkan lalu kalikan)
menghasilkan angka yang sama dengan rumus Pearson biasa.

**Profil pekerjaan sendiri cocok sempurna** (baris 31–35): jika profil Physicists dimasukkan sebagai
"siswa", korelasinya dengan Physicists harus 1,0 dan menjadi nilai tertinggi.

**Skala tidak berpengaruh** (baris 37–40):

```python
    def test_riasec_scale_does_not_matter(self):
        a = {"Realistic": 2, "Investigative": 9, "Artistic": 4, "Social": 5, "Enterprising": 1, "Conventional": 6}
        b = {k: v * 4 + 3 for k, v in a.items()}
        np.testing.assert_allclose(self.rec.interest_fit(a), self.rec.interest_fit(b))
```

Semua skor dikali 4 lalu ditambah 3. Hasil korelasinya harus tetap sama persis. Tes ini membuktikan sifat
Pearson yang dijelaskan di 6.2.

**Kompetensi memakai skala yang benar** (baris 42–48): Mathematical Reasoning untuk Physicists punya
Level 6,00, jadi `required` harus 6/7. Semua kompetensi yang dikembalikan harus Importance ≥ 3, terurut
menurun, dan berjumlah 10.

**Skor tidak lengkap ditolak** (baris 50–52).

### Evaluasi dengan siswa sintetis

Tes terakhir (baris 54–69) mengukur kualitas rekomendasi. Karena belum ada data siswa sungguhan, dipakai
**siswa sintetis**: untuk setiap pekerjaan, dibuat siswa tiruan yang profil RIASEC-nya sama dengan
pekerjaan itu ditambah sedikit gangguan acak.

```python
        d, rng = self.data, np.random.default_rng(0)
        R = np.clip((d.riasec - 1) / 6 + rng.normal(0, 0.10, d.riasec.shape), 0, 1)
```

- `(d.riasec - 1) / 6` mengubah skala 1–7 menjadi 0–1.
- `rng.normal(0, 0.10, ...)` menambah gangguan acak dengan simpangan baku 0,10, untuk meniru siswa yang
  minatnya mirip pekerjaan itu tapi tidak persis sama.
- `np.clip(..., 0, 1)` menjaga hasilnya tetap 0–1.
- `default_rng(0)` memakai *seed* tetap, jadi angka acaknya selalu sama setiap kali tes dijalankan.

Dua ukuran dihitung:

| Ukuran | Pertanyaan | Hasil | Batas minimal tes |
|---|---|---|---|
| **Hit@10** | Apakah pekerjaan asal siswa tiruan masuk 10 besar rekomendasinya? | **67,4%** | 60% |
| **Related Recall@10** | Dari 10 pekerjaan terkait pekerjaan asal (menurut O*NET), berapa yang ikut masuk 10 besar? | **18,4%** | 15% |

Sebagai pembanding, jika rekomendasi dipilih secara acak, peluang sebuah pekerjaan masuk 10 besar dari 878
hanya sekitar 10/878 ≈ 1,1%.

Dua catatan penting untuk membaca angka ini:

1. **Ini siswa sintetis, bukan siswa sungguhan.** Angka ini menunjukkan bahwa sistem bekerja sesuai
   rancangannya, bukan bahwa rekomendasinya terbukti tepat untuk siswa SMA nyata.
2. **Recall pekerjaan terkait memang rendah.** Daftar Related Occupations O*NET disusun dari banyak hal
   (tugas, kompetensi, pendidikan), bukan hanya minat. Jadi wajar jika rekomendasi yang hanya berdasarkan
   minat tidak selalu sama dengan daftar itu.

Angka ini dicetak setiap kali tes dijalankan:

```
siswa sintetis: Hit@10 67.4%, Related Recall@10 18.4%
```

## 6.9 Ringkasan

```
Profil RIASEC 878 pekerjaan (1..7)          Skor RIASEC siswa (0..40)
        │                                           │
        │ _standardise_rows                         │ _standardise_rows
        │ (sekali, saat Recommender dibuat)         │ (setiap siswa)
        ▼                                           ▼
   _riasec_z  (878 × 6)          @            student  (6,)
                              │
                              ▼
               fit: 878 korelasi Pearson (−1..1)
                              │
               ┌──────────────┼───────────────┐
               ▼              ▼               ▼
         fit_label()    match_percent()   Bab 7: urutkan, ambil 10 teratas
       Sangat cocok ...    0..100%
```

- Kecocokan = **korelasi Pearson** antara enam skor RIASEC siswa dan pekerjaan.
- Korelasi dihitung untuk **878 pekerjaan sekaligus** dengan satu perkalian matriks.
- Label kecocokan memakai **batas resmi O*NET** (0,729 dan 0,608).
- Kompetensi **hanya ditampilkan**: yang Importance-nya ≥ 3, maksimal 10, paling penting lebih dulu.
- Kualitasnya diuji dengan siswa sintetis: Hit@10 67,4%.
