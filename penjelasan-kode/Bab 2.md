# Bab 2. Data yang Dipakai

Semua rekomendasi texin dihitung dari data O*NET 31.0. Bab ini menjelaskan dari mana data itu berasal,
bagaimana data resmi O*NET diubah menjadi file CSV di folder `data/`, apa isi tiap file, dan arti skala
angka di dalamnya. Cara file-file ini dimuat ke program dibahas di Bab 3.

Sepanjang bab ini dipakai satu contoh pekerjaan: **Data Scientists** (kode `15-2051.00`).

## 2.1 Sumber data: O*NET 31.0

O*NET (Occupational Information Network) adalah basis data pekerjaan milik Departemen Tenaga Kerja
Amerika Serikat (USDOL/ETA). Untuk setiap pekerjaan, O*NET mencatat antara lain:

- deskripsi pekerjaan dan tingkat persiapan yang dibutuhkan (Job Zone),
- profil minat RIASEC dan area minat yang lebih spesifik,
- kompetensi: kemampuan (Abilities), pengetahuan (Knowledge), dan keterampilan (Skills),
- pendidikan yang dibutuhkan, gaya kerja, dan software yang dipakai,
- daftar pekerjaan lain yang terkait.

Setiap pekerjaan punya kode **O*NET-SOC** berbentuk `XX-XXXX.XX`, misalnya `15-2051.00` untuk Data
Scientists. Kode ini menjadi kunci yang menghubungkan semua file di folder `data/`.

texin memakai **O*NET 31.0** dalam format **RDF N-Triples** (file `.nt`). Data ini dipakai di bawah lisensi
CC BY 4.0 dan O*NET Tools Developer License, sehingga aplikasi wajib menampilkan atribusi. Teks
atribusinya ada di `crs/service.py` baris 31–37 dan tampil di bagian bawah setiap halaman.

## 2.2 Dari RDF ke CSV: `scripts/flatten_rdf.py`

### Kenapa perlu diubah

Di format RDF, data disimpan sebagai **triple**: tiga bagian, yaitu *subjek – predikat – objek*. Satu
angka rating tidak tersimpan dalam satu baris, tapi tersebar di beberapa triple yang terhubung lewat
sebuah simpul perantara. Pola ini dijelaskan di docstring `scripts/flatten_rdf.py` baris 1–10:

```python
"""Flatten the official O*NET 31.0 RDF (N-Triples) into tidy CSV tables.

The RDF models every rating as an intermediate node:

    Occupation --hasRating--> <Domain>Rating --refersTo--> Element
                                             --refersTo--> Scale
                                             --dataValue-> float

This script collapses that pattern into one row per
(occupation, element, scale) and also extracts the Related Occupations links.
```

Artinya, untuk mengetahui "skor Investigative Data Scientists adalah 6,98", kita harus menyambungkan:

1. pekerjaan Data Scientists **punya rating** X (`hasRating`),
2. rating X **merujuk ke** elemen *Investigative* (`refersTo`),
3. rating X **merujuk ke** skala *OI* (`refersTo`),
4. rating X **bernilai** 6,98 (`dataValue`).

Bentuk ini sulit dipakai untuk perhitungan. Script `flatten_rdf.py` "meratakan" pola tersebut menjadi
tabel biasa dengan satu baris per kombinasi (pekerjaan, elemen, skala):

```
soc_code,title,domain,element_id,element_name,scale_id,value
15-2051.00,Data Scientists,CareerInterestTypes,1.B.1.b,Investigative,OI,6.98
```

Script ini dijalankan sekali saja, di komputer pengembang, sebelum aplikasi dipakai. Hasil CSV-nya
ikut di-commit, jadi server tidak perlu membaca RDF sama sekali. File RDF mentahnya sendiri (folder
`data/db_31_0_nt/`) tidak ada di repo.

### Lokasi input dan output

`scripts/flatten_rdf.py` baris 22–24:

```python
ROOT = Path(__file__).resolve().parent.parent
RDF_DIR = ROOT / "data" / "db_31_0_nt"
OUT_DIR = ROOT / "data"
```

Script membaca file `.nt` dari `data/db_31_0_nt/` dan menulis CSV ke `data/`.

### Domain rating yang diambil

`scripts/flatten_rdf.py` baris 27–35:

```python
DOMAINS = [
    "CareerInterestTypesRating",
    "SpecificInterestAreasRating",
    "AbilitiesRating",
    "EssentialSkillsRating",
    "TransferableSkillsRating",
    "KnowledgeRating",
    "WorkActivitiesRating",
]
```

Setiap nama di sini sama dengan nama file `.nt` (misalnya `CareerInterestTypesRating.nt`). Ada tujuh
domain yang diambil, tapi aplikasi hanya memakai enam: `WorkActivities` ikut tersimpan di CSV, namun
tidak dibaca oleh `crs/data.py`.

### Membaca satu baris triple

Setiap baris file N-Triples berbentuk `<subjek> <predikat> <objek> .` atau
`<subjek> <predikat> "nilai teks" .`. Baris ini dipecah dengan regular expression
(`scripts/flatten_rdf.py` baris 37):

```python
TRIPLE = re.compile(r'^<([^>]+)> <([^>]+)> (?:<([^>]+)>|"((?:[^"\\]|\\.)*)"\S*) \.$')
```

Empat grup yang ditangkap:

| Grup | Isi |
|---|---|
| 1 | subjek (alamat IRI di dalam `< >`) |
| 2 | predikat (IRI) |
| 3 | objek, jika objeknya IRI lain |
| 4 | objek, jika objeknya teks/angka di dalam tanda kutip (literal) |

Fungsi `triples()` membaca file baris per baris dengan regex itu (`scripts/flatten_rdf.py` baris 40–47):

```python
def triples(filename):
    """Yield (subject, predicate local name, object IRI or None, literal or None)."""
    with open(RDF_DIR / filename, encoding="utf-8") as f:
        for line in f:
            m = TRIPLE.match(line.rstrip("\n"))
            if m:
                s, p, o, lit = m.groups()
                yield s, p.rsplit("/", 1)[-1].rsplit("#", 1)[-1], o, lit
```

Baris terakhir memotong predikat menjadi nama pendeknya saja. IRI yang panjang seperti
`http://.../hasRating` menjadi `hasRating`, sehingga mudah dibandingkan di langkah berikutnya. Fungsi
ini memakai `yield`, jadi file yang besar dibaca sedikit demi sedikit tanpa harus dimuat sekaligus ke memori.

### Langkah 1: Kamus elemen dan skala

`scripts/flatten_rdf.py` baris 51–57:

```python
    elements, scales = {}, {}
    for s, p, _, lit in triples("Element.nt"):
        if p in ("elementID", "elementName"):
            elements.setdefault(s, {})[p] = lit
    for s, p, _, lit in triples("Scale.nt"):
        if p == "scaleID":
            scales[s] = lit
```

Hasilnya dua kamus: IRI elemen → ID dan namanya (misalnya `1.B.1.b`, `Investigative`), dan IRI skala →
kode skalanya (misalnya `OI`). Kamus ini dipakai nanti untuk menerjemahkan IRI di rating menjadi nama yang
bisa dibaca.

### Langkah 2: Pekerjaan dan pemilik tiap simpul

`scripts/flatten_rdf.py` baris 59–75:

```python
    occupations = {}    # iri -> {"code":..., "title":...}
    rating_owner = {}   # rating iri -> occupation iri
    related_owner = {}  # linkage iri -> occupation iri
    task_owner = {}     # task iri -> occupation iri
    for s, p, o, lit in triples("Occupation.nt"):
        if p == "onetSOCCode":
            occupations.setdefault(s, {})["code"] = lit
        elif p == "title":
            occupations.setdefault(s, {})["title"] = lit
        elif p == "description":
            occupations.setdefault(s, {})["description"] = lit
        elif p == "hasRating":
            rating_owner[o] = s
        elif p == "hasRelatedOccupation":
            related_owner[o] = s
        elif p == "hasTask":
            task_owner[o] = s
```

Dari `Occupation.nt` diambil kode, judul, dan deskripsi tiap pekerjaan. Selain itu dicatat juga **arah
baliknya**: setiap simpul rating, simpul pekerjaan terkait, dan simpul tugas dimiliki oleh pekerjaan mana.
Kamus `rating_owner` inilah yang nanti dipakai untuk menjawab "rating X ini milik pekerjaan apa?".

### Langkah 3: Job Zone, lalu tulis `onet_occupations.csv`

`scripts/flatten_rdf.py` baris 77–86:

```python
    zone_number = {s: lit for s, p, _, lit in triples("JobZone.nt") if p == "jobZone"}
    for s, p, o, _ in triples("JobZoneRating.nt"):
        if p == "refersTo" and o in zone_number and s in rating_owner:
            occupations[rating_owner[s]]["job_zone"] = zone_number[o]
    with open(OUT_DIR / "onet_occupations.csv", "w", newline="", encoding="utf-8") as f:
        w = csv.writer(f)
        w.writerow(["soc_code", "title", "job_zone", "description"])
        for occ in sorted(occupations.values(), key=lambda x: x["code"]):
            w.writerow([occ["code"], occ["title"], occ.get("job_zone", ""), occ["description"]])
```

Job Zone juga tersimpan sebagai rating: rating Job Zone merujuk ke sebuah simpul Job Zone yang memuat
nomornya. Pekerjaan yang tidak punya Job Zone tetap ditulis, dengan kolom `job_zone` kosong.

### Langkah 4: Rating, lalu tulis `onet_ratings.csv`

`scripts/flatten_rdf.py` baris 92–110:

```python
        for domain in DOMAINS:
            ratings = {}
            for s, p, o, lit in triples(domain + ".nt"):
                r = ratings.setdefault(s, {})
                if p == "dataValue":
                    r["value"] = lit
                elif p == "refersTo":
                    if o in elements:
                        r["element"] = o
                    elif o in scales:
                        r["scale"] = o
            for iri, r in ratings.items():
                occ = occupations.get(rating_owner.get(iri))
                if not occ or "element" not in r or "scale" not in r or "value" not in r:
                    continue
                el = elements[r["element"]]
                w.writerow([occ["code"], occ["title"], domain.removesuffix("Rating"),
                            el["elementID"], el["elementName"], scales[r["scale"]], r["value"]])
                n_rows += 1
```

Inilah inti "perataan":

1. Untuk setiap file domain, kumpulkan semua triple per simpul rating: nilainya (`dataValue`), elemen yang
   dirujuk, dan skala yang dirujuk. Karena kedua rujukan memakai predikat yang sama (`refersTo`),
   pembedanya adalah apakah objeknya ada di kamus `elements` atau `scales`.
2. Untuk setiap simpul rating, cari pemiliknya lewat `rating_owner`.
3. Simpul yang tidak lengkap (tidak punya pemilik, elemen, skala, atau nilai) dilewati.
4. Tulis satu baris CSV. Akhiran `Rating` dihapus dari nama domain, jadi `CareerInterestTypesRating`
   ditulis sebagai `CareerInterestTypes`.

### Langkah 5: Pekerjaan terkait, lalu tulis `onet_related_occupations.csv`

`scripts/flatten_rdf.py` baris 114–132:

```python
    links = {}
    for s, p, o, lit in triples("RelatedOccupationLinkage.nt"):
        l = links.setdefault(s, {})
        if p == "refersTo":
            l["target"] = o
        elif p == "relatednessTier":
            l["tier"] = lit
        elif p == "relatedIndex":
            l["rank"] = lit
    ...
        for iri, l in links.items():
            src = occupations.get(related_owner.get(iri))
            dst = occupations.get(l.get("target"))
            if src and dst:
                w.writerow([src["code"], dst["code"], l.get("tier", ""), l.get("rank", "")])
```

Polanya sama seperti rating: setiap tautan adalah simpul perantara yang merujuk ke pekerjaan tujuan,
dengan tingkat keterkaitan (`tier`) dan urutannya (`rank`). Asal tautannya dicari lewat `related_owner`.

### Langkah 6: Detailed Work Activities (tidak dipakai)

Baris 135–144 menulis `onet_dwa.csv` (aktivitas kerja rinci per pekerjaan). File ini **tidak ada** di
folder `data/` dan tidak dipakai di mana pun di aplikasi.

### File yang tidak dibuat oleh script ini

`flatten_rdf.py` hanya menghasilkan `onet_occupations.csv`, `onet_ratings.csv`, dan
`onet_related_occupations.csv`. Tiga file lain di `data/`, yaitu `onet_education.csv`,
`onet_work_styles.csv`, dan `onet_software.csv`, berasal dari proses lain yang script-nya belum ada di repo.
Isinya tetap dijelaskan di subbab 2.3.

## 2.3 Isi tiap file CSV

| File | Baris data | Jumlah pekerjaan | Dipakai oleh |
|---|---|---|---|
| `onet_occupations.csv` | 1.016 | 1.016 | `crs/data.py` |
| `onet_ratings.csv` | 374.471 | 923 | `crs/data.py` |
| `onet_related_occupations.csv` | 18.460 | 923 | `crs/data.py` |
| `onet_education.csv` | 10.812 | 901 | `crs/details.py` |
| `onet_work_styles.csv` | 18.711 | 891 | `crs/details.py` |
| `onet_software.csv` | 31.821 | 923 | `crs/details.py` |

Jumlah pekerjaan di tiap file berbeda karena O*NET tidak melengkapi semua data untuk semua pekerjaan. Hanya
pekerjaan yang datanya lengkap yang dipakai untuk rekomendasi, yaitu **878 pekerjaan** (Bab 3).

### `onet_occupations.csv`: daftar pekerjaan

Kolom: `soc_code, title, job_zone, description`.

```
15-2051.00,Data Scientists,4,"Develop and implement a set of techniques or analytics applications to
transform raw data into meaningful information using data-oriented programming languages and
visualization software. ..."
```

Dari 1.016 pekerjaan, 93 tidak punya Job Zone (kolomnya kosong). Judul dan deskripsi di sini masih dalam
bahasa Inggris; versi Indonesianya ada di `crs/i18n/pekerjaan.csv` (Bab 4).

### `onet_ratings.csv`: skor tiap pekerjaan

Kolom: `soc_code, title, domain, element_id, element_name, scale_id, value`. Ini file terbesar dan
terpenting, karena memuat semua angka yang dipakai untuk menghitung kecocokan dan kompetensi.

| Domain | Jumlah elemen | Skala | Dipakai untuk |
|---|---|---|---|
| `CareerInterestTypes` | 6 tipe RIASEC + 3 kode *high-point* | OI, IH | **perhitungan kecocokan** (hanya OI) |
| `SpecificInterestAreas` | 41 area minat | OI, DS | alasan "Mengapa cocok" (hanya OI) |
| `Abilities` | 52 | IM, LV | daftar kompetensi |
| `Knowledge` | 33 | IM, LV | daftar kompetensi |
| `EssentialSkills` | 10 | IM, LV | daftar kompetensi |
| `TransferableSkills` | 25 | IM, LV | daftar kompetensi |
| `WorkActivities` | 41 | IM, LV | tidak dipakai |

Empat domain kompetensi berjumlah 52 + 33 + 10 + 25 = **120 kompetensi**.

Contoh profil RIASEC Data Scientists:

```
15-2051.00,Data Scientists,CareerInterestTypes,1.B.1.a,Realistic,OI,2.17
15-2051.00,Data Scientists,CareerInterestTypes,1.B.1.b,Investigative,OI,6.98
15-2051.00,Data Scientists,CareerInterestTypes,1.B.1.c,Artistic,OI,2.61
15-2051.00,Data Scientists,CareerInterestTypes,1.B.1.d,Social,OI,1.66
15-2051.00,Data Scientists,CareerInterestTypes,1.B.1.e,Enterprising,OI,1.71
15-2051.00,Data Scientists,CareerInterestTypes,1.B.1.f,Conventional,OI,5.39
```

Enam angka inilah yang nanti dikorelasikan dengan enam skor siswa. Data Scientists sangat tinggi di
Investigative (6,98) dan cukup tinggi di Conventional (5,39), jadi pekerjaan ini akan cocok untuk siswa
yang suka meneliti dan bekerja dengan data secara teratur.

Contoh area minat dan kompetensi:

```
15-2051.00,Data Scientists,SpecificInterestAreas,...,Mathematics/Statistics,OI,6.85
15-2051.00,Data Scientists,SpecificInterestAreas,...,Information Technology,OI,5.81
15-2051.00,Data Scientists,EssentialSkills,2.A.1.e,Mathematics,IM,4.38
15-2051.00,Data Scientists,EssentialSkills,2.A.1.e,Mathematics,LV,5.00
15-2051.00,Data Scientists,Knowledge,2.C.4.a,Mathematics,IM,4.52
15-2051.00,Data Scientists,Knowledge,2.C.4.a,Mathematics,LV,5.68
```

Setiap kompetensi muncul dua kali: sekali dengan skala IM (seberapa penting) dan sekali dengan skala LV
(seberapa tinggi levelnya). Ada juga nama elemen yang sama di domain berbeda, misalnya "Mathematics" ada
di Knowledge dan di EssentialSkills. Karena itu, kompetensi selalu dikenali dari pasangan
(domain, nama), bukan dari namanya saja.

### `onet_related_occupations.csv`: pekerjaan terkait

Kolom: `soc_code, related_soc_code, tier, rank`.

```
15-2051.00,15-2041.00,Primary-Short,1
15-2051.00,43-9111.00,Primary-Short,2
15-2051.00,13-2099.01,Primary-Short,3
```

Setiap pekerjaan punya 20 pekerjaan terkait, diurutkan dari yang paling dekat (`rank` 1–20). Kolom `tier`
membaginya menjadi tiga kelompok: `Primary-Short` (5 terdekat), `Primary-Long` (5 berikutnya), dan
`Supplemental` (10 sisanya). Data ini dipakai untuk halaman Jaringan Karier.

### `onet_education.csv`: pendidikan yang dibutuhkan

Kolom: `soc_code, category, percent`. Setiap pekerjaan punya 12 baris, satu untuk setiap kategori
pendidikan. `percent` adalah persentase pekerja di pekerjaan itu yang menyatakan bahwa tingkat pendidikan
tersebut dibutuhkan.

Contoh Data Scientists (kategori yang bukan 0%):

| Kategori | Label di aplikasi | Persen |
|---|---|---|
| 5 | Diploma (D2/D3) | 4 |
| 6 | Sarjana (S1/D4) | 48 |
| 8 | Magister (S2) | 44 |
| 11 | Doktor (S3) | 4 |

Label Indonesia untuk 12 kategori ini ada di `crs/details.py` baris 12–25.

### `onet_work_styles.csv`: gaya kerja

Kolom: `soc_code, style, impact`. Ada 21 gaya kerja, misalnya *Attention to Detail* atau *Integrity*.
`impact` menunjukkan seberapa besar pengaruh gaya kerja itu terhadap keberhasilan di pekerjaan tersebut;
nilainya di data ini berkisar −1,42 sampai 3,0. Lima gaya kerja teratas Data Scientists: Intellectual
Curiosity (3,0), Attention to Detail (2,99), Innovation (2,42), Dependability (2,36), dan Achievement
Orientation (2,14).

Label Indonesianya ada di `crs/details.py` baris 27–49.

### `onet_software.csv`: software yang dipakai

Kolom: `soc_code, software, in_demand, hot_technology`.

```
15-2051.00,Alteryx software,false,true
15-2051.00,Amazon Redshift,false,true
15-2051.00,Amazon Simple Storage Service S3,false,false
```

Dua kolom terakhir adalah penanda dari O*NET: `in_demand` (banyak dicari di lowongan kerja) dan
`hot_technology` (teknologi yang sedang banyak dipakai). Aplikasi hanya menampilkan software yang punya
salah satu dari dua penanda ini.

## 2.4 Arti skala angka

| Skala | Nama | Rentang | Arti | Dipakai |
|---|---|---|---|---|
| **OI** | Occupational Interest | 1–7 | Seberapa kuat pekerjaan ini berkaitan dengan suatu tipe atau area minat. 1 = sangat lemah, 7 = sangat kuat. | Ya |
| **IH** | Interest High-Point | 0–6 | Kode tipe RIASEC teratas pekerjaan (1 = R, 2 = I, …, 6 = C; 0 = tidak ada). Data Scientists: pertama 2 (Investigative), kedua 6 (Conventional), ketiga 0. | Tidak |
| **DS** | (area minat) | 0–8 | Di data, area dengan OI tertinggi mendapat nilai 1, 2, dan seterusnya; area lain 0. Data Scientists: Mathematics/Statistics = 1, Information Technology = 2. | Tidak |
| **IM** | Importance | 1–5 | Seberapa penting kompetensi itu untuk pekerjaan. 3 = "Important". | Ya |
| **LV** | Level | 0–7 | Seberapa tinggi tingkat kompetensi yang dibutuhkan. | Ya |

Beberapa catatan:

- **OI** adalah skala yang menjadi dasar rekomendasi. Skala siswa (0–40) dan skala pekerjaan (1–7) berbeda,
  tapi itu tidak masalah karena korelasi Pearson hanya membandingkan pola, bukan besar angkanya (Bab 6).
- **IM dan LV** dipakai bersama: kompetensi dianggap penting jika IM ≥ 3, dan level yang dibutuhkan
  ditampilkan sebagai LV dibagi 7 (`crs/recommender.py` baris 16 dan 20).
- **IH dan DS** ikut tersimpan di CSV, tapi tidak dibaca oleh aplikasi.

### Job Zone

Job Zone menunjukkan seberapa banyak persiapan (pendidikan, pengalaman, pelatihan) yang dibutuhkan sebuah
pekerjaan. Di O*NET 31.0, Job Zone 1 dan 2 digabung, sehingga data hanya berisi nilai 2–5.

| Job Zone | Jumlah pekerjaan | Label di aplikasi |
|---|---|---|
| 2 | 337 | sedikit persiapan (SMA/SMK atau pelatihan singkat) |
| 3 | 208 | persiapan menengah (pelatihan vokasi / D3) |
| 4 | 226 | persiapan tinggi (umumnya S1) |
| 5 | 152 | persiapan sangat tinggi (umumnya S2/S3) |
| kosong | 93 | tidak dipakai |

Labelnya didefinisikan di `crs/data.py` baris 14–19:

```python
JOB_ZONE_NAMES = {
    2: "Job Zone 1-2: sedikit persiapan (SMA/SMK atau pelatihan singkat)",
    3: "Job Zone 3: persiapan menengah (pelatihan vokasi / D3)",
    4: "Job Zone 4: persiapan tinggi (umumnya S1)",
    5: "Job Zone 5: persiapan sangat tinggi (umumnya S2/S3)",
}
```

Job Zone dipakai untuk menyaring rekomendasi sesuai rencana pendidikan yang dipilih siswa. Misalnya, siswa
yang memilih "Sampai D3 atau pelatihan vokasi" hanya mendapat pekerjaan dengan Job Zone ≤ 3.

## 2.5 Ringkasan

```
O*NET 31.0 (RDF N-Triples, data/db_31_0_nt/, tidak di-commit)
   │
   │  scripts/flatten_rdf.py          (dijalankan sekali oleh pengembang)
   ▼
onet_occupations.csv ─────────┐
onet_ratings.csv ─────────────┼──►  crs/data.py     → matriks untuk rekomendasi  (Bab 3)
onet_related_occupations.csv ─┘
onet_education.csv ───────────┐
onet_work_styles.csv ─────────┼──►  crs/details.py  → halaman detail karier      (Bab 7)
onet_software.csv ────────────┘
```

- Dasar rekomendasi hanya **enam angka OI per pekerjaan** (profil RIASEC).
- Area minat dipakai untuk menyusun alasan kecocokan.
- Kompetensi, pendidikan, gaya kerja, software, dan pekerjaan terkait hanya **ditampilkan**, tidak ikut
  menentukan urutan rekomendasi.
