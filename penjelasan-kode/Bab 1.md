# Bab 1. Gambaran Umum Sistem

Bab ini menjelaskan apa yang dikerjakan texin, metode yang dipakai, dan bagaimana data mengalir dari
file O*NET sampai menjadi rekomendasi di layar. Penjelasan rinci tiap bagian ada di bab-bab berikutnya;
di sini cukup peta besarnya.

## 1.1 Tujuan dan metode

texin adalah sistem rekomendasi karier untuk siswa SMA. Siswa menjawab 60 pertanyaan tentang aktivitas
kerja yang mungkin ia sukai, lalu sistem:

1. menghitung **profil minat** siswa pada enam tipe minat RIASEC,
2. membandingkan profil itu dengan profil minat **878 pekerjaan** dari basis data O*NET 31.0,
3. menampilkan **10 pekerjaan paling cocok**, lengkap dengan persentase kecocokan, alasan kecocokan,
   kompetensi yang dibutuhkan, pendidikan, dan jaringan karier terkait.

### Model minat RIASEC

RIASEC (model Holland) membagi minat kerja menjadi enam tipe:

| Kode | Tipe | Di aplikasi | Contoh aktivitas |
|---|---|---|---|
| R | Realistic | Realistis (Praktis) | memperbaiki mesin, bekerja dengan alat |
| I | Investigative | Investigatif (Analitis) | meneliti, menganalisis data |
| A | Artistic | Artistik (Kreatif) | menulis, mendesain, bermusik |
| S | Social | Sosial (Membantu Orang) | mengajar, merawat, membimbing |
| E | Enterprising | Enterprising (Kepemimpinan) | memimpin, menjual, berbisnis |
| C | Conventional | Konvensional (Teratur) | mengelola arsip, menghitung, mendata |

Label Indonesia dan nama panggilannya didefinisikan di `crs/service.py` baris 15–19:

```python
RIASEC_LABELS = {"Realistic": "Realistis", "Investigative": "Investigatif", "Artistic": "Artistik",
                 "Social": "Sosial", "Enterprising": "Enterprising", "Conventional": "Konvensional"}
# Short everyday names for the six interest types, shown next to the Indonesian label.
RIASEC_NICKNAMES = {"Realistic": "Praktis", "Investigative": "Analitis", "Artistic": "Kreatif",
                    "Social": "Membantu Orang", "Enterprising": "Kepemimpinan", "Conventional": "Teratur"}
```

Baik siswa maupun pekerjaan sama-sama digambarkan dengan enam angka ini. Karena keduanya memakai
"bahasa" yang sama, keduanya bisa dibandingkan langsung.

### Content-Based Filtering dengan Pearson Correlation

Metode rekomendasinya adalah **Content-Based Filtering**: rekomendasi dibuat dari kemiripan *isi*
(profil minat siswa vs. profil minat pekerjaan), bukan dari perilaku pengguna lain.

Ukuran kemiripannya adalah **Pearson Correlation**, cara yang juga dipakai O*NET Interest Profiler
resmi. Korelasi mengukur apakah **pola** naik-turun enam skor siswa mirip dengan pola enam skor
pekerjaan, tanpa peduli skalanya. Skor siswa berskala 0–40, sedangkan skor pekerjaan berskala 1–7;
korelasi tetap bisa membandingkan keduanya.

- Hasil korelasi berkisar −1 sampai 1.
- Di layar, nilai ini ditampilkan sebagai persen "Kecocokan" (korelasi 0,85 → 85%). Korelasi negatif
  ditampilkan 0%.

Kompetensi (keterampilan, pengetahuan, kemampuan) **tidak memengaruhi urutan** rekomendasi. Kompetensi
hanya ditampilkan sebagai informasi: apa yang perlu dikembangkan siswa untuk menuju pekerjaan itu.
Hal ini tertulis di docstring `crs/recommender.py` baris 1–9:

```python
"""Career recommendation for high-school students, based on O*NET 31.0.

Content-based filtering on interests: an occupation's fit is the Pearson correlation between the
student's six RIASEC scores and the occupation's RIASEC profile, the method O*NET's own Interest
Profiler uses to link a profile to occupations.

Competencies do not affect the ranking. They are reported per occupation: what the student would
need to develop to get there.
"""
```

Rumus dan cara menghitungnya dibahas di Bab 6.

## 1.2 Alur besar

Urutan dari data mentah sampai rekomendasi tampil:

```
 data/*.csv (O*NET 31.0)
        │  dimuat sekali saat server mulai            crs/data.py
        ▼
 OnetData: 878 pekerjaan × 6 skor RIASEC,
 41 area minat, 120 kompetensi, pekerjaan terkait
        │
        │            ┌───────────────────────────────────────────────┐
        │            │ Browser: siswa menjawab 60 butir (0..4)       │  web/static/js/views/quiz.js
        │            └───────────────────────┬───────────────────────┘
        │                                    │ POST /api/recommend
        │                                    ▼
        │                         web/app.py  recommend()
        │                                    │
        │                                    ▼
        │                     crs/service.py  recommendation()
        │                                    │
        │            ┌───────────────────────┴───────────────────────┐
        │            ▼                                               │
        │   crs/questionnaire.py  score()                            │
        │   60 jawaban → 6 skor RIASEC (0..40)                       │
        │            │                                               │
        ▼            ▼                                               │
   crs/recommender.py  interest_fit()                                │
   korelasi Pearson siswa vs. 878 pekerjaan                          │
        │                                                            │
        ▼                                                            │
   crs/service.py  recommend_by_interest()  → 10 teratas             │
                   why_it_fits()            → alasan kecocokan  ◄────┘
        │
        ▼
 JSON hasil → browser menampilkan halaman Hasil, Detail Karier, Jaringan Karier
```

Penjelasan tiap langkah:

### Langkah 1: Memuat data O*NET

Saat server pertama kali membutuhkan data, `crs/data.py` membaca file CSV di folder `data/` dan
menyusunnya menjadi matriks NumPy. Dari 1.016 pekerjaan di `onet_occupations.csv`, hanya pekerjaan yang
datanya lengkap (skor minat, area minat, kompetensi, dan Job Zone) yang dipakai, sehingga tersisa 878
pekerjaan. Fungsi `load()` diberi `@lru_cache`, jadi data hanya dibaca sekali selama server berjalan
(`crs/data.py` baris 42–44):

```python
@lru_cache(maxsize=1)
def load(data_dir=DATA_DIR):
    """Occupations that lack interest or competency ratings are left out (878 remain in O*NET 31.0)."""
```

Dibahas di Bab 2 (isi data) dan Bab 3 (cara memuatnya).

### Langkah 2: Browser mengambil konfigurasi

Saat halaman dibuka, browser meminta daftar pertanyaan, pilihan jawaban, dan data lain lewat
`/api/config`, sekaligus status akun lewat `/api/state` (`web/static/js/main.js` baris 144–149):

```javascript
Promise.all([api("/api/config"), api("/api/state")])
  .then(([config, state]) => {
    setConfig(config);
    applyState(state);
    render();
  })
```

Server menjawabnya di `web/app.py` baris 194–207. Isinya antara lain 60 butir pertanyaan, lima pilihan
jawaban, enam tipe RIASEC, dan pilihan rencana pendidikan:

```python
@app.get("/api/config")
def config():
    return {
        "prompt": service.ITEM_PROMPT,
        "responses": questionnaire.RESPONSES,
        "types": [{"type": t, "label": service.RIASEC_LABELS[t], "nickname": service.RIASEC_NICKNAMES[t]}
                  for t in service.RIASEC],
        "items": [{"number": it.number, "type": it.riasec, "text": it.text_id}
                  for it in ITEMS],
        "plans": [{"label": label, "max_job_zone": zone} for label, zone in service.EDUCATION_PLANS.items()],
        ...
    }
```

Artinya, kuesioner dijalankan sepenuhnya di browser; server baru bekerja lagi setelah semua butir dijawab.

### Langkah 3: Siswa menjawab kuesioner

Siswa menjawab 60 butir, masing-masing dengan lima pilihan yang bernilai 0–4
(`crs/questionnaire.py` baris 22):

```python
RESPONSES = ["Sangat tidak suka", "Tidak suka", "Ragu-ragu", "Suka", "Sangat suka"]  # scored 0..4
```

Siswa juga memilih rencana pendidikan (misalnya "Sampai S1"), yang nanti dipakai untuk menyaring
pekerjaan berdasarkan Job Zone (`crs/service.py` baris 24–30):

```python
EDUCATION_PLANS = {
    "Belum tahu / tampilkan semua": None,
    "Lulus SMA/SMK lalu bekerja atau pelatihan singkat": 2,
    "Sampai D3 atau pelatihan vokasi": 3,
    "Sampai S1": 4,
    "Sampai S2/S3": 5,
}
```

### Langkah 4: Jawaban dikirim ke server

Setelah selesai, browser mengirim 60 jawaban dan batas Job Zone ke `/api/recommend`
(`web/static/js/views/quiz.js` baris 227–232):

```javascript
export async function runAnalysis() {
  const answers = CONFIG.items.map((it) => S.answers[it.number]);
  const request = api("/api/recommend", {
    answers,
    max_job_zone: CONFIG.plans[S.plan].max_job_zone,
  });
```

Server memeriksa bahwa setiap jawaban bernilai 0–4, lalu meneruskannya ke `service.recommendation()`.
Jika siswa sedang masuk ke akunnya, ringkasan hasil juga disimpan sebagai riwayat
(`web/app.py` baris 210–229):

```python
@app.post("/api/recommend")
def recommend(body: Answers, request: Request):
    if any(not 0 <= value <= 4 for value in body.answers):
        raise HTTPException(422, "Setiap jawaban harus antara 0 dan 4")
    result = service.recommendation({n + 1: v for n, v in enumerate(body.answers)}, body.max_job_zone)
    ...
    user = current_user(request)
    if user and result["cards"]:
        ...
            result["result_id"] = store().add_result(user["id"], body.max_job_zone, body.answers, summary)
        store().delete_progress(user["id"])   # the test is finished
    return result
```

Penyimpanan dibahas di Bab 8, endpoint API di Bab 9.

### Langkah 5: Jawaban menjadi skor RIASEC

Setiap butir milik satu tipe RIASEC, dan setiap tipe punya 10 butir. Skor satu tipe adalah jumlah
jawaban dari 10 butirnya, jadi berkisar 0–40 (`crs/questionnaire.py` baris 176–181):

```python
    totals = dict.fromkeys(RIASEC, 0)
    for it in all_items:
        value = answers[it.number]
        ...
        totals[it.riasec] += value
```

Contoh hasilnya: `{"Realistic": 12, "Investigative": 34, "Artistic": 20, "Social": 28,
"Enterprising": 15, "Conventional": 18}`.

Jika keenam skor sama persis (misalnya semua dijawab "Ragu-ragu"), profil dianggap datar dan tidak bisa
dicocokkan, sama seperti aturan profiler resmi O*NET. Dibahas di Bab 5.

### Langkah 6: Menghitung kecocokan dengan semua pekerjaan

`Recommender.interest_fit()` menghitung korelasi Pearson antara enam skor siswa dan enam skor
**setiap** pekerjaan sekaligus, dalam satu perkalian matriks (`crs/recommender.py` baris 51–58):

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

Hasilnya adalah 878 angka korelasi, satu per pekerjaan. Dibahas di Bab 6.

### Langkah 7: Mengurutkan dan menyusun hasil

`recommend_by_interest()` mengurutkan pekerjaan dari korelasi tertinggi, membuang pekerjaan yang Job
Zone-nya melebihi rencana pendidikan siswa, lalu mengambil 10 teratas (`crs/service.py` baris 101–110):

```python
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

Lalu `recommendation()` menambahkan alasan kecocokan dan perbandingan profil ke setiap kartu
(`crs/service.py` baris 166–170):

```python
    cards = recommend_by_interest(totals, max_job_zone, top_n)
    for c in cards:
        c["reasons"], c["comparison"] = why_it_fits(totals, c["code"])
    profile = [{"type": t, "label": label, "percent": percent} for t, label, percent in interest_profile(totals)]
    return {"flat": False, "profile": profile, "cards": cards}
```

Dibahas di Bab 7.

### Langkah 8: Menampilkan hasil

Browser menerima JSON berisi profil minat dan 10 kartu rekomendasi, lalu menampilkannya di halaman
Hasil Rekomendasi. Saat siswa membuka satu karier, browser meminta detailnya lewat
`/api/occupations/{code}` (kompetensi, pendidikan, gaya kerja, software) dan jaringan kariernya lewat
`/api/network/{code}`. Dibahas di Bab 7 dan Bab 10.

## 1.3 Struktur folder

```
crs/
├── data/                       Data O*NET 31.0 (CSV) dan basis data lokal
│   ├── onet_occupations.csv        daftar pekerjaan: kode, judul, deskripsi, Job Zone
│   ├── onet_ratings.csv            skor tiap pekerjaan: minat, area minat, kompetensi
│   ├── onet_related_occupations.csv  pekerjaan yang saling terkait
│   ├── onet_education.csv          tingkat pendidikan yang dibutuhkan
│   ├── onet_work_styles.csv        gaya kerja
│   ├── onet_software.csv           software yang dipakai
│   └── crs.sqlite3                 akun dan hasil tes (lokal, tidak di-commit)
│
├── crs/                        Logika sistem (Python), tidak bergantung pada tampilan
│   ├── data.py                     memuat CSV menjadi matriks           → Bab 3
│   ├── i18n/                       terjemahan Indonesia                 → Bab 4
│   ├── questionnaire.py            60 butir dan penghitungan skor       → Bab 5
│   ├── recommender.py              korelasi Pearson dan kompetensi      → Bab 6
│   ├── service.py                  menyusun hasil untuk ditampilkan     → Bab 7
│   ├── details.py                  pendidikan, gaya kerja, software     → Bab 7
│   ├── accounts.py                 password, token, batas login         → Bab 8
│   └── storage.py                  basis data SQLite/Postgres           → Bab 8
│
├── web/                        Aplikasi web
│   ├── app.py                      server FastAPI dan endpoint API      → Bab 9
│   ├── tailwind.css                sumber CSS
│   └── static/                     yang dikirim ke browser              → Bab 10
│       ├── index.html
│       ├── tailwind.css                CSS hasil build
│       ├── img/
│       └── js/                         router, state, komponen, satu file per halaman
│
├── tests/                      Tes otomatis                             → Bab 11
├── requirements.txt            paket Python: numpy, fastapi, uvicorn, psycopg
├── pyproject.toml              pengaturan untuk Vercel
└── package.json                perintah build CSS Tailwind
```

Pembagiannya berlapis: `crs/` berisi seluruh logika rekomendasi dan bisa dipakai tanpa web sama sekali,
`web/app.py` hanya menjadi jembatan antara browser dan `crs/`, dan `web/static/` hanya mengurus tampilan.
Karena itu, perhitungan rekomendasi bisa diuji langsung lewat `tests/` tanpa menjalankan server.
