# Bab 5. Kuesioner Minat (`crs/questionnaire.py`)

Bab 2–4 menjelaskan sisi **pekerjaan**: setiap pekerjaan punya profil enam angka RIASEC dari O*NET. Bab ini
menjelaskan sisi **siswa**: bagaimana 60 jawaban kuesioner diubah menjadi enam angka RIASEC yang sama,
sehingga keduanya bisa dibandingkan di Bab 6.

## 5.1 Sumber kuesioner: O*NET Interest Profiler Short Form

Kuesioner texin adalah **O*NET Interest Profiler Short Form**, alat ukur minat resmi dari O*NET yang
bisa dipakai pengembang di bawah O*NET Tools Developer License. Isinya 60 butir. Setiap butir adalah satu
**aktivitas kerja**, misalnya "Develop a new medicine" atau "Teach a high-school class", dan setiap butir
mengukur satu tipe RIASEC. Setiap tipe punya **10 butir**.

Penjelasan ini ada di docstring `crs/questionnaire.py` baris 1–9:

```python
"""Interest questionnaire: the 60 items of the O*NET Interest Profiler Short Form, in Indonesian.

Each item is a work activity. The student answers on a five-point scale
(0 = sangat tidak suka ... 4 = sangat suka). The score of a RIASEC type is the sum of
its 10 items, so every type ranges from 0 to 40.

The Indonesian wording is an unvalidated translation made for this project. The O*NET Tools
Developer License asks that adapted versions be validated for their audience before release:
https://www.onetcenter.org/license_toolsdev.html
```

Terjemahan Indonesia ke-60 butir dibuat **dengan bantuan AI** dan belum divalidasi (lihat Bab 4.6).
Baris 11–16 docstring yang sama berisi teks atribusi yang diwajibkan lisensi tersebut.

Memakai alat ukur resmi O*NET punya keuntungan penting: butir-butirnya memang dirancang untuk
menghasilkan skor RIASEC yang **sejalan dengan profil RIASEC pekerjaan di basis data O*NET**. Jadi skor
siswa dan skor pekerjaan benar-benar mengukur hal yang sama.

## 5.2 Pilihan jawaban

`crs/questionnaire.py` baris 22:

```python
RESPONSES = ["Sangat tidak suka", "Tidak suka", "Ragu-ragu", "Suka", "Sangat suka"]  # scored 0..4
```

Siswa menjawab setiap butir dengan salah satu dari lima pilihan. **Posisi** pilihan di daftar ini
menjadi nilainya:

| Pilihan | Nilai | Tombol di layar |
|---|---|---|
| Sangat tidak suka | 0 | 1 |
| Tidak suka | 1 | 2 |
| Ragu-ragu | 2 | 3 |
| Suka | 3 | 4 |
| Sangat suka | 4 | 5 |

Di layar, tombolnya diberi nomor 1–5 supaya lebih mudah dibaca, dan siswa juga bisa menekan angka 1–5 di
keyboard. Yang dikirim ke server tetap nilai 0–4 (`web/static/js/views/quiz.js` baris 309):

```javascript
  if (event.key >= "1" && event.key <= "5") onAnswer(Number(event.key) - 1);
```

Setiap butir didahului kalimat pengantar yang sama (`crs/service.py` baris 20–21):

```python
ITEM_PROMPT = ("Bayangkan kamu melakukan kegiatan berikut sebagai pekerjaan. "
               "Seberapa suka kamu melakukannya?")
```

## 5.3 Daftar 60 butir

Butir-butirnya disimpan di kamus `_ITEMS` (`crs/questionnaire.py` baris 25–104). Kuncinya tipe RIASEC,
isinya daftar 10 pasangan **(teks asli bahasa Inggris, terjemahan Indonesia)**:

```python
_ITEMS = {
    "Realistic": [
        ("Build kitchen cabinets", "Membuat lemari dapur"),
        ("Lay brick or tile", "Memasang batu bata atau ubin"),
        ("Repair household appliances", "Memperbaiki peralatan rumah tangga"),
        ...
    ],
    "Investigative": [
        ("Develop a new medicine", "Mengembangkan obat baru"),
        ("Study ways to reduce water pollution", "Meneliti cara mengurangi pencemaran air"),
        ...
    ],
    ...
}
```

Teks Inggris disimpan berdampingan dengan terjemahannya supaya setiap terjemahan bisa dicek terhadap butir
aslinya. Yang ditampilkan ke siswa hanya teks Indonesia.

Contoh dua butir pertama tiap tipe:

| Tipe | Butir (Indonesia) | Butir asli |
|---|---|---|
| Realistic | Membuat lemari dapur | Build kitchen cabinets |
| | Memasang batu bata atau ubin | Lay brick or tile |
| Investigative | Mengembangkan obat baru | Develop a new medicine |
| | Meneliti cara mengurangi pencemaran air | Study ways to reduce water pollution |
| Artistic | Menulis buku atau naskah drama | Write books or plays |
| | Memainkan alat musik | Play a musical instrument |
| Social | Mengajari seseorang rangkaian latihan olahraga | Teach an individual an exercise routine |
| | Membantu orang yang memiliki masalah pribadi atau emosional | Help people with personal or emotional problems |
| Enterprising | Membeli dan menjual saham dan obligasi | Buy and sell stocks and bonds |
| | Mengelola toko ritel | Manage a retail store |
| Conventional | Membuat lembar kerja (spreadsheet) dengan perangkat lunak komputer | Develop a spreadsheet using computer software |
| | Memeriksa ketepatan catatan atau formulir | Proofread records or forms |

Nama `_ITEMS` diawali garis bawah karena kamus ini hanya dipakai di dalam modul. Modul lain mengambil
butir lewat fungsi `items()`.

## 5.4 Kelas `Item` dan fungsi `items()`

`crs/questionnaire.py` baris 152–167:

```python
@dataclass(frozen=True)
class Item:
    number: int   # 1..60, the order in which items are shown
    riasec: str
    text_en: str
    text_id: str


def items():
    """The 60 items, interleaved so that consecutive items come from different RIASEC types."""
    result = []
    for position in range(10):
        for riasec in RIASEC:
            en, idn = _ITEMS[riasec][position]
            result.append(Item(len(result) + 1, riasec, en, idn))
    return result
```

Setiap butir menjadi objek `Item` dengan empat atribut: nomor urut, tipe RIASEC, teks Inggris, dan teks
Indonesia. `frozen=True` membuat objeknya tidak bisa diubah setelah dibuat, jadi tidak ada bagian program
yang bisa mengubah isi butir tanpa sengaja.

### Urutan butir diselang-seling

`items()` **tidak** menampilkan 10 butir Realistic dulu, lalu 10 butir Investigative, dan seterusnya.
Perulangan luarnya adalah posisi (0–9) dan perulangan dalamnya adalah tipe, sehingga butir dari keenam
tipe muncul bergantian:

| Nomor | Tipe | Butir |
|---|---|---|
| 1 | Realistic | Membuat lemari dapur |
| 2 | Investigative | Mengembangkan obat baru |
| 3 | Artistic | Menulis buku atau naskah drama |
| 4 | Social | Mengajari seseorang rangkaian latihan olahraga |
| 5 | Enterprising | Membeli dan menjual saham dan obligasi |
| 6 | Conventional | Membuat lembar kerja (spreadsheet) ... |
| 7 | Realistic | Memasang batu bata atau ubin |
| 8 | Investigative | Meneliti cara mengurangi pencemaran air |
| … | … | … |
| 60 | Conventional | Mencap, menyortir, dan membagikan surat ... |

Polanya: butir nomor *n* berasal dari tipe ke-((*n* − 1) mod 6) dalam urutan RIASEC. Dengan urutan
seperti ini, siswa tidak menjawab sepuluh aktivitas sejenis berturut-turut. Jawaban jadi lebih mencerminkan
minat terhadap tiap aktivitas, bukan terbawa jawaban sebelumnya.

Nomor butir (`len(result) + 1`) dimulai dari 1. Nomor inilah yang dipakai sebagai kunci jawaban di seluruh
program: jawaban siswa disimpan sebagai `{nomor butir: nilai}`.

### Butir dikirim ke browser

Daftar butir dikirim ke browser lewat `/api/config` (`web/app.py` baris 201–202), hanya dengan nomor,
tipe, dan teks Indonesianya:

```python
        "items": [{"number": it.number, "type": it.riasec, "text": it.text_id}
                  for it in ITEMS],
```

Setelah itu, seluruh kuesioner berjalan di browser. Server baru dihubungi lagi saat semua butir sudah
dijawab.

## 5.5 Menghitung skor: fungsi `score()`

`crs/questionnaire.py` baris 170–185:

```python
def score(answers):
    """answers: {item number: 0..4} for all 60 items -> {RIASEC type: 0..40}."""
    all_items = items()
    missing = [it.number for it in all_items if it.number not in answers]
    if missing:
        raise ValueError(f"Butir belum dijawab: {missing}")
    totals = dict.fromkeys(RIASEC, 0)
    for it in all_items:
        value = answers[it.number]
        if value not in range(len(RESPONSES)):
            raise ValueError(f"Jawaban butir {it.number} harus 0..{len(RESPONSES) - 1}, bukan {value!r}")
        totals[it.riasec] += value
    if len(set(totals.values())) == 1:
        # Same rule as the official profiler: a flat profile carries no information about interests.
        raise ValueError("Semua skor minat sama, sehingga profil tidak bisa dicocokkan. Coba isi ulang kuesioner.")
    return totals
```

Fungsi ini bekerja dalam empat langkah.

### Langkah 1: Semua butir harus dijawab

```python
    missing = [it.number for it in all_items if it.number not in answers]
    if missing:
        raise ValueError(f"Butir belum dijawab: {missing}")
```

Jika ada butir yang belum dijawab, fungsi berhenti dengan pesan kesalahan yang menyebut nomor butirnya.

### Langkah 2: Siapkan wadah skor

```python
    totals = dict.fromkeys(RIASEC, 0)
```

Hasilnya `{"Realistic": 0, "Investigative": 0, ..., "Conventional": 0}`, urut sesuai `RIASEC`.

### Langkah 3: Jumlahkan jawaban per tipe

```python
    for it in all_items:
        value = answers[it.number]
        if value not in range(len(RESPONSES)):
            raise ValueError(...)
        totals[it.riasec] += value
```

Untuk setiap butir, nilai jawabannya ditambahkan ke skor tipe RIASEC butir itu. Sebelumnya nilai dicek
harus salah satu dari 0, 1, 2, 3, atau 4. `range(len(RESPONSES))` sama dengan `range(5)`, jadi batasnya
ikut berubah jika jumlah pilihan jawaban diubah.

Karena setiap tipe punya 10 butir dan setiap butir bernilai 0–4, **skor setiap tipe berkisar 0–40**.

### Langkah 4: Tolak profil datar

```python
    if len(set(totals.values())) == 1:
        raise ValueError("Semua skor minat sama, sehingga profil tidak bisa dicocokkan. ...")
```

`set(totals.values())` mengumpulkan nilai-nilai skor yang berbeda. Jika isinya hanya satu nilai, berarti
keenam skor sama persis. Contohnya siswa yang menjawab "Ragu-ragu" di semua butir, sehingga semua skornya 20.

Profil seperti ini disebut **profil datar** dan ditolak karena dua alasan:

1. **Tidak ada informasinya.** Profil datar tidak menunjukkan tipe mana yang lebih disukai.
2. **Tidak bisa dihitung.** Korelasi Pearson (Bab 6) membagi dengan simpangan baku skor siswa. Jika semua
   skornya sama, simpangan bakunya nol, sehingga korelasinya tidak terdefinisi.

Aturan ini sama dengan aturan O*NET Interest Profiler resmi, seperti dicatat di komentar baris 183.

### Contoh perhitungan

Misalkan seorang siswa menjawab:

| Tipe | Jawaban 10 butirnya | Skor |
|---|---|---|
| Realistic | 1, 0, 2, 1, 1, 0, 2, 1, 0, 1 | 9 |
| Investigative | 4, 3, 4, 4, 3, 4, 3, 4, 3, 4 | 36 |
| Artistic | 2, 1, 2, 3, 1, 2, 1, 2, 2, 1 | 17 |
| Social | 1, 1, 2, 1, 2, 1, 1, 0, 1, 2 | 12 |
| Enterprising | 1, 0, 1, 2, 1, 0, 1, 1, 0, 1 | 8 |
| Conventional | 3, 3, 2, 3, 3, 4, 2, 3, 3, 2 | 28 |

Maka `score()` mengembalikan:

```python
{"Realistic": 9, "Investigative": 36, "Artistic": 17, "Social": 12, "Enterprising": 8, "Conventional": 28}
```

Pola siswa ini (Investigative paling tinggi, lalu Conventional) mirip dengan pola Data Scientists di Bab 2
(I = 6,98, C = 5,39). Seberapa mirip keduanya dihitung di Bab 6.

## 5.6 Mengubah skor menjadi persen

Skor 0–40 sulit dibaca siswa, jadi di layar skor ditampilkan sebagai persen dari skor maksimal
(`crs/service.py` baris 22–23 dan 84–87):

```python
ITEMS_PER_TYPE = 10
MAX_TYPE_SCORE = 40   # ten items, each answered 0..4
...
def interest_profile(totals):
    """[(type, label, percent)] from RIASEC totals (0..40 each), strongest first."""
    rows = [(t, type_label(t), round(100 * totals[t] / MAX_TYPE_SCORE)) for t in RIASEC]
    return sorted(rows, key=lambda r: -r[2])
```

Untuk contoh di atas:

| Tipe | Skor | Persen | Label di layar |
|---|---|---|---|
| Investigative | 36 | 90% | Investigatif (Analitis) |
| Conventional | 28 | 70% | Konvensional (Teratur) |
| Artistic | 17 | 42% | Artistik (Kreatif) |
| Social | 12 | 30% | Sosial (Membantu Orang) |
| Realistic | 9 | 22% | Realistis (Praktis) |
| Enterprising | 8 | 20% | Enterprising (Kepemimpinan) |

Skor 17 sebenarnya 42,5% dan skor 9 sebenarnya 22,5%. Fungsi `round()` di Python membulatkan angka
,5 ke bilangan **genap** terdekat (*banker's rounding*), sehingga hasilnya 42 dan 22, bukan 43 dan 23.

Daftarnya diurutkan dari persen tertinggi (`key=lambda r: -r[2]`), sehingga minat terkuat muncul paling
atas. Label diambil dari `type_label()` (`crs/service.py` baris 80–81), yang menggabungkan nama Indonesia
dengan nama panggilannya.

Persen ini **hanya untuk tampilan**. Perhitungan kecocokan di Bab 6 memakai skor 0–40 aslinya. Hasilnya
sebenarnya sama saja, karena korelasi Pearson tidak terpengaruh perubahan skala.

## 5.7 Ringkasan minat di browser

Setelah butir ke-60 dijawab, siswa melihat halaman **Hasil Minat Kamu** sebelum analisis dimulai. Halaman
ini dihitung langsung di browser dengan rumus yang sama, tanpa menghubungi server
(`web/static/js/views/quiz.js` baris 161–178):

```javascript
function localProfile() {
  const sums = Object.fromEntries(CONFIG.types.map((t) => [t.type, 0]));
  CONFIG.items.forEach((it) => {
    sums[it.type] += S.answers[it.number] ?? 0;
  });
  const rows = CONFIG.types.map((t) => ({
    type: t.type,
    label: typeLabel(t.type),
    short: t.nickname,
    percent: Math.round((100 * sums[t.type]) / 40),
  }));
  const flat = new Set(Object.values(sums)).size === 1;
  ...
}
```

Isinya sama dengan `score()` dan `interest_profile()` di Python: jumlahkan jawaban per tipe, bagi 40,
kalikan 100. Pemeriksaan profil datarnya juga sama (`new Set(...).size === 1`).

Ada satu perbedaan kecil: `Math.round()` di JavaScript selalu membulatkan ,5 **ke atas**, sedangkan
`round()` di Python membulatkannya ke bilangan genap (subbab 5.6). Akibatnya, untuk skor ganjil, persen di
halaman ini bisa selisih 1% dengan persen di halaman Hasil Rekomendasi, yang memakai hasil dari server.
Pada contoh di atas, Artistic tampil 43% di sini tetapi 42% di halaman hasil.

Jika profilnya datar, halaman ini langsung menampilkan peringatan dan tombol **Ulangi Kuis** sebagai
pengganti tombol **Lanjut ke Analisis** (`web/static/js/views/quiz.js` baris 182–184 dan 202). Jadi siswa
sudah tahu masalahnya sebelum jawabannya dikirim ke server.

Halaman kuesioner di browser dibahas lebih lengkap di Bab 10.

## 5.8 Profil datar di server

Walaupun browser sudah mencegahnya, server tetap memeriksa profil datar, karena permintaan ke server bisa
saja datang dari luar tampilan web. `service.recommendation()` menangkap kesalahan dari `score()` dan
mengembalikan hasil tanpa rekomendasi (`crs/service.py` baris 158–165):

```python
    try:
        totals = questionnaire.score(answers)
    except ValueError:
        totals = {t: 0 for t in RIASEC}
        for item in questionnaire.items():
            totals[item.riasec] += answers[item.number]
        profile = [{"type": t, "label": label, "percent": percent} for t, label, percent in interest_profile(totals)]
        return {"flat": True, "profile": profile, "cards": []}
```

Profil minat tetap dihitung ulang dan dikirim supaya bisa ditampilkan, tapi `cards` dikosongkan dan
`flat` bernilai `True`.

## 5.9 Tes kuesioner

Tesnya ada di `tests/test_questionnaire.py`.

**60 butir, 10 per tipe, tidak ada yang kembar** (baris 8–13):

```python
    def test_sixty_items_ten_per_type(self):
        items = questionnaire.items()
        self.assertEqual([it.number for it in items], list(range(1, 61)))
        self.assertEqual(Counter(it.riasec for it in items), dict.fromkeys(RIASEC, 10))
        self.assertEqual(len({it.text_en for it in items}), 60)
        self.assertEqual(len({it.text_id for it in items}), 60)
```

Nomor butir harus 1 sampai 60 berurutan, setiap tipe tepat 10 butir, dan tidak boleh ada dua butir dengan
teks yang sama, baik teks Inggris maupun Indonesianya.

**Penjumlahan skor benar** (baris 15–19):

```python
    def test_scoring(self):
        answers = {it.number: 4 if it.riasec == "Investigative" else 1 for it in questionnaire.items()}
        scores = questionnaire.score(answers)
        self.assertEqual(scores["Investigative"], 40)
        self.assertEqual(scores["Artistic"], 10)
```

Semua butir Investigative dijawab 4 (10 × 4 = 40), butir lainnya dijawab 1 (10 × 1 = 10).

**Jawaban tidak lengkap, tidak valid, atau datar ditolak** (baris 21–28):

```python
    def test_rejects_incomplete_invalid_and_flat_answers(self):
        full = {it.number: 2 for it in questionnaire.items()}
        with self.assertRaises(ValueError):
            questionnaire.score({1: 3})
        with self.assertRaises(ValueError):
            questionnaire.score({**full, 5: 9})
        with self.assertRaises(ValueError):
            questionnaire.score(full)
```

Tiga kasus: hanya satu butir dijawab, satu jawaban bernilai 9 (di luar 0–4), dan semua butir dijawab 2
(profil datar).

**Skor kuesioner menghasilkan rekomendasi yang masuk akal** (baris 33–37):

```python
    def test_quiz_scores_feed_the_recommender(self):
        answers = {it.number: 4 if it.riasec == "Artistic" else 0 for it in questionnaire.items()}
        top = service.recommend_by_interest(questionnaire.score(answers), top_n=5)
        self.assertTrue(all(r["fit_label"] in ("Sangat cocok", "Cocok") for r in top))
        self.assertGreaterEqual(top[0]["match"], 61)
```

Siswa yang hanya menyukai aktivitas Artistic harus mendapat lima rekomendasi teratas berlabel "Sangat
cocok" atau "Cocok", dengan kecocokan teratas minimal 61%. Tes ini menghubungkan Bab 5 dengan Bab 6:
skor dari kuesioner memang bisa langsung dipakai oleh mesin rekomendasi.

## 5.10 Ringkasan

```
60 butir O*NET Interest Profiler Short Form (diselang-seling R, I, A, S, E, C)
      │  siswa menjawab di browser, tiap butir 0..4
      ▼
{1: 1, 2: 4, 3: 2, ..., 60: 3}                         jawaban per nomor butir
      │  questionnaire.score()
      │    1. semua butir harus dijawab
      │    2. setiap jawaban harus 0..4
      │    3. jumlahkan per tipe
      │    4. tolak jika keenam skor sama
      ▼
{"Realistic": 9, "Investigative": 36, ..., "Conventional": 28}     skor 0..40
      │
      ├──► interest_profile()  → persen, untuk ditampilkan
      └──► Bab 6: dikorelasikan dengan profil RIASEC 878 pekerjaan
```

- Kuesioner memakai alat ukur resmi O*NET, sehingga skor siswa sejalan dengan profil pekerjaan.
- Skor satu tipe = jumlah jawaban 10 butirnya, berkisar 0–40.
- Profil datar ditolak, baik di browser maupun di server.
