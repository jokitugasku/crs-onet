# Bab 7. Layanan Aplikasi (`crs/service.py` dan `crs/details.py`)

Bab 6 menghasilkan 878 angka korelasi. Angka-angka itu belum bisa langsung ditampilkan ke siswa: harus
diurutkan, disaring sesuai rencana pendidikan, diberi judul Indonesia, diberi alasan kecocokan, dan
dilengkapi detail pekerjaan. Semua itu dikerjakan oleh `crs/service.py`, dibantu `crs/details.py`.

Contoh yang dipakai tetap siswa dari Bab 5 (skor R 9, I 36, A 17, S 12, E 8, C 28) dan Data Scientists.

## 7.1 Peran `service.py`

Docstring `crs/service.py` baris 1–4:

```python
"""What the application shows and stores, independent of the user interface.

Used by the FastAPI app (web/app.py).
"""
```

`service.py` adalah lapisan di antara mesin rekomendasi dan web:

```
web/app.py  ──►  crs/service.py  ──►  crs/recommender.py, questionnaire.py, details.py, i18n/
 (HTTP)          (isi halaman)         (perhitungan dan data)
```

Semua fungsi di sini mengembalikan kamus dan daftar Python biasa, yang oleh FastAPI langsung diubah menjadi
JSON. `service.py` tidak tahu apa pun tentang HTTP atau HTML. Karena itu, semua isinya bisa diuji tanpa
menjalankan server.

Semua fungsi memakai satu objek `Recommender` yang sama (`crs/service.py` baris 39–41):

```python
@lru_cache(maxsize=1)
def recommender():
    return Recommender()
```

Objek ini, termasuk data O*NET dan profil pekerjaan yang sudah distandarkan, dibuat sekali saat pertama
dibutuhkan, lalu dipakai ulang untuk semua siswa (Bab 6.5).

## 7.2 Kartu pekerjaan: `title()` dan `card()`

`crs/service.py` baris 44–58:

```python
def title(i):
    d = recommender().data
    return i18n.occupation_title(d.codes[i], d.titles[i])


def card(i):
    """What a result card shows for occupation row i."""
    rec = recommender()
    d = rec.data
    zone = int(d.job_zones[i])
    return {
        "code": d.codes[i], "title": title(i), "title_en": d.titles[i],
        "description": i18n.occupation_description(d.codes[i], d.descriptions[i]),
        "job_zone": zone, "job_zone_name": JOB_ZONE_NAMES[zone],
    }
```

`card(i)` menyusun informasi dasar satu pekerjaan dari nomor barisnya. Fungsi ini menjadi dasar semua
tampilan pekerjaan: kartu hasil rekomendasi, halaman detail, dan contoh karier di halaman depan.

Untuk Data Scientists (`card(119)`):

```python
{
    "code": "15-2051.00",
    "title": "Ilmuwan Data",
    "title_en": "Data Scientists",
    "description": "Mengembangkan dan menerapkan teknik atau aplikasi analitik ...",
    "job_zone": 4,
    "job_zone_name": "Job Zone 4: persiapan tinggi (umumnya S1)",
}
```

`int(d.job_zones[i])` mengubah angka NumPy menjadi `int` Python biasa, karena tipe angka NumPy tidak bisa
langsung diubah menjadi JSON.

## 7.3 Mengurutkan dan menyaring: `recommend_by_interest()`

`crs/service.py` baris 96–110:

```python
def recommend_by_interest(totals, max_job_zone=None, top_n=10):
    """Content-based filtering: rank occupations by Pearson correlation with the student's RIASEC
    scores (the method O*NET's own Interest Profiler uses). Returns result cards with a match."""
    rec = recommender()
    d = rec.data
    fit = rec.interest_fit(totals)
    order = [i for i in np.argsort(-fit) if max_job_zone is None or d.job_zones[i] <= max_job_zone]
    cards = []
    for i in order[:top_n]:
        c = card(int(i))
        c["correlation"] = round(float(fit[i]), 3)
        c["match"] = match_percent(fit[i])
        c["fit_label"] = fit_label(fit[i])
        cards.append(c)
    return cards
```

### Langkah 1: Hitung korelasi

`rec.interest_fit(totals)` menghasilkan 878 korelasi (Bab 6.5).

### Langkah 2: Urutkan dari yang paling cocok

`np.argsort(-fit)` mengembalikan **nomor-nomor baris** yang diurutkan berdasarkan nilai `fit`. `argsort`
mengurutkan dari kecil ke besar, jadi `fit` diberi tanda minus supaya urutannya terbalik, dari korelasi
tertinggi ke terendah. Untuk siswa contoh, hasilnya `[211, 119, 118, ...]`: Historians, Data Scientists,
Biostatisticians, dan seterusnya.

### Langkah 3: Saring berdasarkan rencana pendidikan

```python
if max_job_zone is None or d.job_zones[i] <= max_job_zone
```

`max_job_zone` berasal dari rencana pendidikan yang dipilih siswa sebelum tes (`crs/service.py` baris
24–30):

| Rencana pendidikan | `max_job_zone` | Pekerjaan yang boleh muncul |
|---|---|---|
| Belum tahu / tampilkan semua | `None` | semua |
| Lulus SMA/SMK lalu bekerja atau pelatihan singkat | 2 | Job Zone 2 |
| Sampai D3 atau pelatihan vokasi | 3 | Job Zone 2–3 |
| Sampai S1 | 4 | Job Zone 2–4 |
| Sampai S2/S3 | 5 | semua |

Penyaringan dilakukan **setelah** pengurutan, jadi urutan kecocokan tetap terjaga. Pekerjaan yang butuh
pendidikan lebih tinggi hanya dilewati.

Untuk siswa contoh, sepuluh besar tanpa saringan semuanya Job Zone 3–5 (Bab 6.5). Jika siswa memilih
"Sampai D3", hasilnya berubah:

| # | Pekerjaan | Kecocokan | Job Zone |
|---|---|---|---|
| 1 | Pengembang Web | 91% | 3 |
| 2 | Paralegal dan Asisten Hukum | 74% | 3 |
| 3 | Insinyur dan Arsitek Sistem Komputer | 71% | 3 |
| 4 | Teknisi Ilmu Forensik | 65% | 3 |
| 5 | Spesialis Rekam Medis | 63% | 3 |

Jika siswa memilih "Lulus SMA/SMK", kecocokan tertingginya hanya 43% (Pemrogram Mesin Perkakas CNC, Petugas
Entri Data). Artinya, pekerjaan dengan pola minat Investigative–Conventional memang hampir semuanya butuh
pendidikan tinggi. Sistem tetap menampilkan pekerjaan yang paling mendekati, dengan label kecocokan yang
jujur ("Cukup cocok").

### Langkah 4: Susun kartu untuk `top_n` teratas

Untuk sepuluh pekerjaan teratas, kartu dasar dari `card()` ditambah tiga informasi:

| Kunci | Isi | Contoh (Data Scientists) |
|---|---|---|
| `correlation` | korelasi, dibulatkan 3 angka di belakang koma | `0.979` |
| `match` | persen kecocokan (Bab 6.6) | `98` |
| `fit_label` | label kecocokan (Bab 6.6) | `"Sangat cocok"` |

## 7.4 Alasan kecocokan: `why_it_fits()`

Angka "98%" saja tidak menjelaskan **kenapa** sebuah pekerjaan cocok. `why_it_fits()` menyusun kalimat
penjelasan untuk tab "Mengapa cocok" di halaman hasil (`crs/service.py` baris 113–130):

```python
def why_it_fits(totals, code):
    """Plain statements for the "Mengapa cocok" tab, plus both RIASEC profiles as percentages."""
    d = recommender().data
    i = d.codes.index(code)
    student = {t: round(100 * totals[t] / MAX_TYPE_SCORE) for t in RIASEC}
    job = {t: round(100 * (d.riasec[i, k] - 1) / 6) for k, t in enumerate(RIASEC)}
    job_top = [RIASEC[k] for k in np.argsort(-d.riasec[i])[:3]]
    student_top = [t for t, _, _ in interest_profile(totals)[:3]]
    shared = [t for t in job_top if t in student_top and student[t] >= 50]
    reasons = [f"Minat {type_label(t)} kamu tinggi ({student[t]}%), dan pekerjaan ini juga menonjol di tipe itu."
               for t in shared]
    if not shared:
        reasons.append("Pola minatmu secara keseluruhan mirip dengan pola minat pekerjaan ini, "
                       "walaupun tipe teratasnya tidak sama.")
    areas = [questionnaire.AREA_LABELS[d.areas[j]] for j in np.argsort(-d.area_scores[i])[:3]]
    reasons.append("Bidang yang paling kuat di pekerjaan ini: " + ", ".join(areas) + ".")
    profile = [{"type": t, "label": type_label(t), "student": student[t], "job": job[t]} for t in RIASEC]
    return reasons, profile
```

### Langkah 1: Ubah kedua profil ke persen

Supaya bisa ditampilkan berdampingan, kedua profil diubah ke skala 0–100:

- **siswa**: skor / 40 × 100 (sama seperti Bab 5.6),
- **pekerjaan**: (skor − 1) / 6 × 100, karena skala OI berawal dari 1, bukan 0. Skor 1 menjadi 0% dan
  skor 7 menjadi 100%.

| Tipe | Siswa | Data Scientists |
|---|---|---|
| Realistis | 22% | 20% |
| Investigatif | 90% | 100% |
| Artistik | 42% | 27% |
| Sosial | 30% | 11% |
| Enterprising | 20% | 12% |
| Konvensional | 70% | 73% |

Tabel ini dikembalikan sebagai `profile` dan ditampilkan sebagai grafik perbandingan.

### Langkah 2: Cari tipe yang sama-sama menonjol

- `job_top`: tiga tipe tertinggi pekerjaan. Data Scientists: Investigative, Conventional, Artistic.
- `student_top`: tiga tipe tertinggi siswa. Siswa contoh: Investigative, Conventional, Artistic.
- `shared`: tipe yang ada di **kedua** daftar **dan** skor siswanya minimal 50%.

Artistic ada di kedua daftar, tapi skor siswanya hanya 42%, jadi tidak dihitung. Minat 42% tidak bisa
disebut "tinggi". Hasilnya `shared = ["Investigative", "Conventional"]`.

### Langkah 3: Susun kalimat

Untuk setiap tipe di `shared`, dibuat satu kalimat. Jika tidak ada tipe yang sama-sama menonjol, dipakai
kalimat umum bahwa polanya secara keseluruhan mirip. Hal ini bisa terjadi karena korelasi Pearson melihat
pola keenam tipe, bukan hanya tiga teratas.

### Langkah 4: Tambahkan bidang terkuat pekerjaan

Tiga area minat dengan skor tertinggi pekerjaan itu diambil dari `area_scores` (Bab 3.7), lalu
diterjemahkan dengan `AREA_LABELS` dari `crs/questionnaire.py`.

Hasil lengkap untuk siswa contoh dan Data Scientists:

```
• Minat Investigatif (Analitis) kamu tinggi (90%), dan pekerjaan ini juga menonjol di tipe itu.
• Minat Konvensional (Teratur) kamu tinggi (70%), dan pekerjaan ini juga menonjol di tipe itu.
• Bidang yang paling kuat di pekerjaan ini: Matematika dan Statistika, Teknologi Informasi, Akuntansi.
```

Area minat hanya dipakai untuk penjelasan ini dan tidak memengaruhi urutan rekomendasi.

## 7.5 Menggabungkan semuanya: `recommendation()`

Inilah fungsi yang dipanggil oleh `/api/recommend` (`crs/service.py` baris 155–170):

```python
def recommendation(answers, max_job_zone=None, top_n=10):
    """Everything the result pages need from the 60 answers (item number -> 0..4).
    A flat profile (six equal scores) has no correlation with anything, so it gets no cards."""
    try:
        totals = questionnaire.score(answers)
    except ValueError:
        ...
        return {"flat": True, "profile": profile, "cards": []}
    cards = recommend_by_interest(totals, max_job_zone, top_n)
    for c in cards:
        c["reasons"], c["comparison"] = why_it_fits(totals, c["code"])
    profile = [{"type": t, "label": label, "percent": percent} for t, label, percent in interest_profile(totals)]
    return {"flat": False, "profile": profile, "cards": cards}
```

Urutannya:

1. **Jawaban → skor** dengan `questionnaire.score()` (Bab 5.5). Jika profilnya datar, kembalikan hasil
   tanpa kartu (Bab 5.8).
2. **Skor → 10 kartu** dengan `recommend_by_interest()` (7.3).
3. **Setiap kartu diberi alasan** dengan `why_it_fits()` (7.4).
4. **Profil minat siswa** dengan `interest_profile()` (Bab 5.6).

Bentuk hasil akhirnya, yang dikirim sebagai JSON ke browser:

```python
{
    "flat": False,
    "profile": [
        {"type": "Investigative", "label": "Investigatif (Analitis)", "percent": 90},
        {"type": "Conventional", "label": "Konvensional (Teratur)", "percent": 70},
        ...
    ],
    "cards": [
        {
            "code": "19-3093.00", "title": "Sejarawan", "title_en": "Historians",
            "description": "...", "job_zone": 5, "job_zone_name": "...",
            "correlation": 0.981, "match": 98, "fit_label": "Sangat cocok",
            "reasons": ["Minat Investigatif (Analitis) kamu tinggi (90%), ...", ...],
            "comparison": [{"type": "Realistic", "label": "...", "student": 22, "job": ...}, ...],
        },
        ...   # 10 kartu
    ],
}
```

Kartu tidak memuat kompetensi, pendidikan, atau software. Detail itu baru diambil saat siswa membuka satu
pekerjaan (7.6), supaya hasil rekomendasi tetap ringan.

## 7.6 Halaman detail pekerjaan

### `occupation_detail()`

`crs/service.py` baris 61–76:

```python
def occupation_detail(code):
    """Everything the detail view shows, or None for an unknown code."""
    rec = recommender()
    d = rec.data
    if code not in d.codes:
        return None
    i = d.codes.index(code)
    detail = card(i)
    detail["skills"] = [{"name": i18n.competency(g.name), "importance": round(g.importance, 2),
                         "level": round(g.required * 7, 1)} for g in rec.competencies(i, limit=10)]
    detail["related"] = [{"code": d.codes[j], "title": title(j), "job_zone": int(d.job_zones[j])}
                         for j in d.related.get(i, [])[:10]]
    detail["education"] = [{"label": label, "percent": percent} for label, percent in details.education(code)]
    detail["work_styles"] = details.work_styles(code)
    detail["software"] = details.software(code)
    return detail
```

Fungsi ini dipanggil oleh `/api/occupations/{code}`. Jika kodenya tidak dikenal, hasilnya `None`, yang oleh
`web/app.py` diubah menjadi error 404.

Isinya adalah `card()` ditambah lima bagian:

| Kunci | Sumber | Isi |
|---|---|---|
| `skills` | `Recommender.competencies()` (Bab 6.7) | 10 kompetensi terpenting, nama diterjemahkan, level dikembalikan ke skala 0–7 |
| `related` | `OnetData.related` (Bab 3.6) | 10 pekerjaan terkait terdekat |
| `education` | `details.education()` | tingkat pendidikan yang dibutuhkan |
| `work_styles` | `details.work_styles()` | 5 gaya kerja terpenting |
| `software` | `details.software()` | maksimal 12 software |

Contoh Data Scientists:

```python
{
    ...card(119)...,
    "skills": [{"name": "Matematika", "importance": 4.52, "level": 5.7},
               {"name": "Matematika", "importance": 4.38, "level": 5.0},
               {"name": "Penalaran Deduktif", "importance": 4.25, "level": 4.8}, ...],
    "related": [{"code": "15-2041.00", "title": "Statistikawan", "job_zone": 5},
                {"code": "43-9111.00", "title": "Asisten Statistik", "job_zone": 4}, ...],
    "education": [{"label": "Sarjana (S1/D4)", "percent": 48},
                  {"label": "Magister (S2)", "percent": 44}],
    "work_styles": ["Rasa Ingin Tahu", "Ketelitian", "Inovasi", "Dapat Diandalkan", "Orientasi Prestasi"],
    "software": ["Amazon Web Services AWS software", "Apache Hadoop", "Apache Spark", "C++", "Git", ...],
}
```

"Matematika" muncul dua kali karena O*NET punya Mathematics sebagai **pengetahuan** (Knowledge) dan sebagai
**keterampilan** (EssentialSkills), dan keduanya diterjemahkan menjadi nama yang sama (Bab 4.2).

### `crs/details.py`: membaca tiga tabel tambahan

`details.py` membaca tiga file CSV yang tidak dipakai untuk perhitungan (Bab 2.3), dengan pola yang sama:
baca sekali, simpan per kode pekerjaan (`crs/details.py` baris 52–78):

```python
def _read(name):
    with open(DATA_DIR / name, encoding="utf-8") as f:
        yield from csv.DictReader(f)


@lru_cache(maxsize=1)
def _education():
    table = defaultdict(list)
    for r in _read("onet_education.csv"):
        table[r["soc_code"]].append((int(r["category"]), float(r["percent"])))
    return table


@lru_cache(maxsize=1)
def _work_styles():
    table = defaultdict(list)
    for r in _read("onet_work_styles.csv"):
        table[r["soc_code"]].append((r["style"], float(r["impact"])))
    return table


@lru_cache(maxsize=1)
def _software():
    table = defaultdict(list)
    for r in _read("onet_software.csv"):
        table[r["soc_code"]].append((r["software"], r["in_demand"] == "true", r["hot_technology"] == "true"))
    return table
```

Setiap tabel berbentuk `{kode pekerjaan: [baris, baris, ...]}`. Di tabel software, teks `"true"`/`"false"`
dari CSV diubah menjadi nilai `True`/`False`.

**Pendidikan** (`crs/details.py` baris 81–84):

```python
def education(code, min_percent=5.0, limit=4):
    """[(label, percent of workers who say this level is required)], largest first."""
    rows = sorted(_education().get(code, []), key=lambda x: -x[1])
    return [(EDUCATION_LABELS[c], round(p)) for c, p in rows if p >= min_percent][:limit]
```

Dari 12 kategori pendidikan, diambil yang dipilih **minimal 5%** pekerja, maksimal 4, dari persentase
terbesar. Untuk Data Scientists, D3 (4%) dan S3 (4%) tidak ditampilkan karena di bawah 5%, sehingga yang
tersisa S1 (48%) dan S2 (44%). Batas 5% membuang kategori yang hanya dipilih segelintir responden.

**Gaya kerja** (`crs/details.py` baris 87–90):

```python
def work_styles(code, limit=5):
    """The work styles with the largest positive impact for this occupation."""
    rows = sorted(_work_styles().get(code, []), key=lambda x: -x[1])
    return [WORK_STYLE_LABELS.get(name, name) for name, _ in rows[:limit]]
```

Lima gaya kerja dengan `impact` terbesar, diterjemahkan dengan `WORK_STYLE_LABELS`. Jika ada nama yang
belum diterjemahkan, nama Inggrisnya dipakai (`.get(name, name)`).

**Software** (`crs/details.py` baris 93–98):

```python
def software(code, limit=12):
    """Software used in the occupation that O*NET marks as in demand (else as a 'hot technology')."""
    rows = _software().get(code, [])
    in_demand = sorted(name for name, demand, _ in rows if demand)
    hot = sorted(name for name, demand, is_hot in rows if is_hot and not demand)
    return (in_demand + hot)[:limit]
```

Software yang **banyak dicari** (`in_demand`) didahulukan, lalu software yang **sedang populer**
(`hot_technology`), masing-masing urut abjad. Software tanpa kedua penanda tidak ditampilkan. Hasilnya
dibatasi 12 supaya halaman tidak terlalu panjang.

Ketiga fungsi memakai `.get(code, [])`, jadi pekerjaan yang tidak punya data di tabel tertentu cukup
mendapat daftar kosong, dan bagian itu disembunyikan di halaman.

## 7.7 Jaringan karier: `career_graph()`

`crs/service.py` baris 142–152:

```python
def career_graph(code, limit=8):
    """The same network as data for a front end that draws it itself: the centre career and the
    careers O*NET lists as related to it, closest first. None for an unknown code."""
    d = recommender().data
    if code not in d.codes:
        return None
    i = d.codes.index(code)
    related = d.related.get(i, [])[:limit]
    return {"centre": {"code": code, "title": title(i)},
            "nodes": [{"code": d.codes[j], "title": title(j), "close": rank < 5,
                       "job_zone_name": JOB_ZONE_NAMES[int(d.job_zones[j])]} for rank, j in enumerate(related)]}
```

Fungsi ini menyiapkan data untuk halaman **Jaringan Karier**, yang menggambar pekerjaan pilihan di tengah
dan pekerjaan terkaitnya di sekelilingnya. Dipanggil oleh `/api/network/{code}`.

- `centre`: pekerjaan di tengah.
- `nodes`: delapan pekerjaan terkait terdekat, urut sesuai `rank` O*NET.
- `close`: `True` untuk lima pertama, yaitu pekerjaan yang paling dekat. Di gambar, keduanya dibedakan
  secara visual. Angka lima mengikuti ukuran kelompok `Primary-Short` di data O*NET (Bab 2.3). Tapi karena
  tautan ke pekerjaan di luar 878 pekerjaan sudah dibuang (Bab 3.6), lima pertama ini tidak selalu persis
  sama dengan kelompok `Primary-Short`.

Untuk Data Scientists:

```python
{
    "centre": {"code": "15-2051.00", "title": "Ilmuwan Data"},
    "nodes": [
        {"code": "15-2041.00", "title": "Statistikawan", "close": True, ...},
        {"code": "43-9111.00", "title": "Asisten Statistik", "close": True, ...},
        {"code": "13-2099.01", "title": "Analis Kuantitatif Keuangan", "close": True, ...},
        {"code": "15-2031.00", "title": "Analis Riset Operasi", "close": True, ...},
        {"code": "19-1029.01", "title": "Ilmuwan Bioinformatika", "close": True, ...},
        {"code": "15-2099.01", "title": "Teknisi Bioinformatika", "close": False, ...},
        {"code": "15-2021.00", "title": "Matematikawan", "close": False, ...},
        {"code": "15-1221.00", "title": "Ilmuwan Riset Komputer dan Informasi", "close": False, ...},
    ],
}
```

Jaringan ini memakai data Related Occupations dari O*NET apa adanya. Hubungan antarpekerjaan **tidak**
dihitung dari minat siswa.

## 7.8 Contoh karier di halaman depan: `sample_careers()`

`crs/service.py` baris 133–139:

```python
# A few familiar careers for the landing page (O*NET-SOC codes).
SAMPLE_CAREERS = ("15-2051.00", "29-1141.00", "27-1024.00", "17-2051.00", "13-2011.00", "25-2031.00")


def sample_careers():
    codes = recommender().data.codes
    return [card(codes.index(c)) for c in SAMPLE_CAREERS if c in codes]
```

Enam pekerjaan yang familier bagi siswa SMA dipilih secara manual untuk ditampilkan di halaman depan: Ilmuwan
Data, Perawat, Desainer Grafis, Insinyur Sipil, Akuntan dan Auditor, serta Guru Sekolah Menengah Atas.
Kondisi `if c in codes` menjaga agar kode yang tidak ada di data dilewati, bukan membuat error. Daftar ini
dikirim lewat `/api/config` (Bab 9).

## 7.9 Ekspor feedback: `export_feedback()`

`crs/service.py` baris 174–188:

```python
def _to_csv(header, rows):
    out = io.StringIO()
    writer = csv.writer(out)
    writer.writerow(header)
    writer.writerows(rows)
    return out.getvalue()


def export_feedback(store):
    """CSV text with the feedback students gave on their results. No names or emails."""
    rows = [[f["result_id"], f["created_at"], f["summary"]["interest"],
             f["summary"]["top"][0]["title"], f["summary"]["top"][0]["match"],
             f["relevance"], f["ease"], f["comment"]] for f in store.all_result_feedback()]
    return _to_csv(["result_id", "created_at", "strongest_interest", "top1", "top1_match",
                    "relevance", "ease", "comment"], rows)
```

Fungsi ini menyusun semua feedback siswa menjadi teks CSV untuk diunduh tim proyek.
`io.StringIO()` adalah "file" di memori, jadi CSV ditulis tanpa membuat file sungguhan.

Kolom yang diekspor: minat terkuat, rekomendasi teratas beserta kecocokannya, dua rating, dan komentar.
**Nama dan email siswa sengaja tidak ikut**, supaya data yang dianalisis tim tidak menunjuk ke siswa
tertentu. Penyimpanan feedback dibahas di Bab 8, endpoint unduhannya di Bab 9.

## 7.10 Tes

**`tests/test_details.py`**

- `test_education_for_software_developers` (baris 7–10): untuk Software Developers, pendidikan teratas
  harus "Sarjana (S1/D4)" sebesar 85%, dan semua yang ditampilkan minimal 5%.
- `test_every_occupation_has_details_and_translated_labels` (baris 12–20): hampir semua dari 878 pekerjaan
  punya data pendidikan (O*NET 31.0 tidak punya data pendidikan untuk 9 pekerjaan), setiap pekerjaan punya
  tepat 5 gaya kerja, dan tidak ada gaya kerja yang masih berbahasa Inggris.
- `test_software_prefers_in_demand` (baris 22–26): maksimal 12 software, C++ ada di daftar lengkap Software
  Developers, dan kode yang tidak dikenal menghasilkan daftar kosong.

**`tests/test_web.py`** (lewat API, dibahas lebih lanjut di Bab 9)

- `test_recommend_ranks_by_match_and_explains` (baris 55–67): 10 kartu, urut menurun berdasarkan
  kecocokan, setiap kartu punya alasan dan perbandingan enam tipe.
- `test_education_plan_limits_the_job_zone` (baris 69–71): dengan `max_job_zone` 3, tidak ada kartu dengan
  Job Zone di atas 3.
- `test_identical_answers_give_no_cards` (baris 73–77): 60 jawaban "2" menghasilkan profil datar tanpa
  kartu.
- `test_detail_and_network` (baris 83–94): halaman detail punya kompetensi dan pekerjaan terkait, jaringan
  karier punya 8 simpul dengan 5 pertama `close`, simpul pertama jaringan sama dengan pekerjaan terkait
  pertama di halaman detail, dan kode yang tidak dikenal menghasilkan 404.

## 7.11 Ringkasan

```
                         recommendation(answers, max_job_zone)          ◄── POST /api/recommend
                                    │
     questionnaire.score() ─────────┤   jawaban → skor RIASEC
                                    │
     recommend_by_interest() ───────┤   interest_fit → urutkan → saring Job Zone → 10 kartu
         └ card(), match_percent(), fit_label()
                                    │
     why_it_fits() ─────────────────┤   alasan + perbandingan profil, per kartu
                                    │
     interest_profile() ────────────┘   profil minat siswa (persen)

     occupation_detail(code)   ◄── GET /api/occupations/{code}
         └ card() + competencies() + related + details.education / work_styles / software

     career_graph(code)        ◄── GET /api/network/{code}
     sample_careers()          ◄── GET /api/config
     export_feedback(store)    ◄── GET /api/admin/feedback.csv
```

- `service.py` mengubah angka korelasi menjadi **isi halaman** yang siap ditampilkan.
- Rencana pendidikan menyaring pekerjaan **setelah** diurutkan, lewat Job Zone.
- Alasan kecocokan dibuat dari tipe minat yang **sama-sama menonjol** dan bidang terkuat pekerjaan.
- Detail pekerjaan (kompetensi, pendidikan, gaya kerja, software, pekerjaan terkait) baru diambil saat
  siswa membuka satu pekerjaan.
