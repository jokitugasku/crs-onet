# Bab 9. API Web (`web/app.py`)

Bab 5–8 membahas logika di folder `crs/`. Semua itu berupa fungsi Python yang belum bisa dipanggil dari
browser. `web/app.py` membuka fungsi-fungsi itu ke browser sebagai **API**: alamat-alamat seperti
`/api/recommend` yang bisa dipanggil lewat HTTP dan menjawab dalam bentuk JSON. File ini juga mengirim
halaman web dan file-file tampilannya.

## 9.1 Peran `web/app.py`

Docstring `web/app.py` baris 1–9:

```python
"""texin - find your career. FastAPI back end for the HTML + Tailwind front end in web/static.

    .venv/bin/uvicorn web.app:app --reload      ->  http://127.0.0.1:8000

The questionnaire runs in the browser; the server turns 60 answers into recommendations, keeps
accounts, and keeps each signed-in student's results (Riwayat). Environment variables:
    DATABASE_URL  Postgres address; without it everything goes to a local SQLite file
    ADMIN_TOKEN   password for the feedback export at /api/admin/feedback.csv
"""
```

Pembagian kerjanya:

| Dikerjakan di browser | Dikerjakan di server |
|---|---|
| menampilkan pertanyaan, menyimpan jawaban sementara, menghitung ringkasan minat | menghitung rekomendasi, mengurus akun, menyimpan riwayat dan feedback |

`web/app.py` sendiri hampir tidak berisi logika. Setiap endpoint memeriksa input, memanggil fungsi di `crs/`,
lalu mengembalikan hasilnya.

### FastAPI dan Uvicorn

- **FastAPI**: *framework* Python untuk membuat API. Fungsi Python biasa menjadi endpoint cukup dengan
  diberi dekorator seperti `@app.get(...)` atau `@app.post(...)`. Kamus yang dikembalikan fungsi otomatis
  diubah menjadi JSON.
- **Uvicorn**: server yang menjalankan aplikasi FastAPI. Perintah `uvicorn web.app:app` artinya "jalankan
  objek `app` di file `web/app.py`".

`web/app.py` baris 19–31:

```python
from crs import accounts, questionnaire, service
from crs.storage import Store

STATIC = Path(__file__).resolve().parent / "static"
ITEMS = questionnaire.items()


def setting(name):
    return os.environ.get(name) or os.environ.get("CRS_" + name)


app = FastAPI(title="texin - find your career")
_store = None
```

- `STATIC`: lokasi folder `web/static/`, berisi HTML, CSS, JavaScript, dan gambar.
- `ITEMS`: daftar 60 butir kuesioner, dibuat sekali saat server mulai.
- `setting(name)`: membaca pengaturan dari variabel lingkungan. Nama dengan awalan `CRS_` juga diterima,
  misalnya `CRS_ADMIN_TOKEN`, yang dipakai oleh tes.
- `_store`: objek `Store` (Bab 8), dibuat saat pertama dibutuhkan lewat fungsi `store()` (baris 34–39).

## 9.2 Daftar endpoint

| Metode | Alamat | Perlu masuk? | Fungsi | Subbab |
|---|---|---|---|---|
| POST | `/api/auth/register` | – | daftar akun baru, langsung masuk | 9.4 |
| POST | `/api/auth/login` | – | masuk | 9.4 |
| POST | `/api/auth/logout` | – | keluar | 9.4 |
| GET | `/api/config` | – | butir kuesioner, pilihan jawaban, rencana pendidikan, contoh karier | 9.5 |
| POST | `/api/recommend` | tidak wajib | 60 jawaban → rekomendasi | 9.5 |
| GET | `/api/occupations/{code}` | – | detail satu pekerjaan | 9.5 |
| GET | `/api/network/{code}` | – | jaringan karier satu pekerjaan | 9.5 |
| GET | `/api/state` | tidak wajib | akun yang sedang masuk, tes yang belum selesai, hasil terakhir | 9.6 |
| PUT | `/api/progress` | ya | simpan tes yang sedang dikerjakan | 9.6 |
| GET | `/api/history` | ya | daftar Riwayat | 9.6 |
| GET | `/api/history/{id}` | ya | satu hasil dari Riwayat | 9.6 |
| DELETE | `/api/history` | ya | hapus semua Riwayat | 9.6 |
| POST | `/api/feedback` | ya | simpan feedback atas satu hasil | 9.6 |
| GET | `/api/admin/feedback.csv` | token admin | unduh semua feedback | 9.7 |
| GET | `/` | – | halaman web | 9.8 |
| GET | `/static/...` | – | file CSS, JavaScript, gambar | 9.8 |

Metode HTTP menunjukkan jenis tindakannya: **GET** untuk membaca, **POST** untuk mengirim data atau
menjalankan proses, **PUT** untuk menyimpan atau mengganti data, dan **DELETE** untuk menghapus.

## 9.3 Memeriksa input dengan Pydantic

Data yang dikirim browser tidak boleh langsung dipercaya. Bentuk setiap input ditulis sebagai kelas
**Pydantic** (`web/app.py` baris 42–69):

```python
class Answers(BaseModel):
    answers: list[int] = Field(min_length=len(ITEMS), max_length=len(ITEMS))   # item 1..60, each 0..4
    max_job_zone: int | None = Field(default=None, ge=2, le=5)


class Register(BaseModel):
    name: str = Field(min_length=1, max_length=60)
    email: str = Field(max_length=254)
    password: str = Field(min_length=accounts.MIN_PASSWORD, max_length=128)


class Login(BaseModel):
    email: str = Field(max_length=254)
    password: str = Field(max_length=128)
    remember: bool = False


class Progress(BaseModel):
    answers: dict[int, int] = Field(default_factory=dict, max_length=len(ITEMS))   # item number -> 0..4
    pos: int = Field(default=0, ge=0, lt=len(ITEMS))
    plan: int = Field(default=0, ge=0, lt=len(service.EDUCATION_PLANS))


class Feedback(BaseModel):
    result_id: str = Field(max_length=64)   # one of the signed-in student's results
    relevance: int = Field(ge=1, le=5)
    ease: int = Field(ge=1, le=5)
    comment: str | None = Field(default=None, max_length=500)
```

Jika sebuah endpoint menerima parameter bertipe salah satu kelas ini, FastAPI otomatis memeriksa JSON yang
masuk. Jika tidak sesuai, permintaan langsung ditolak dengan kode **422** sebelum fungsi endpoint dijalankan.

| Aturan | Arti |
|---|---|
| `list[int]`, `min_length=60, max_length=60` | jawaban harus daftar berisi tepat 60 bilangan bulat |
| `int \| None`, `ge=2, le=5` | Job Zone boleh kosong, atau 2 sampai 5 |
| `min_length=accounts.MIN_PASSWORD` | password minimal 8 karakter |
| `max_length=...` | batas panjang teks, supaya tidak ada yang mengirim teks raksasa |
| `ge=1, le=5` | rating 1–5 |

Batas yang memakai `len(ITEMS)`, `accounts.MIN_PASSWORD`, dan `len(service.EDUCATION_PLANS)` diambil dari
modul aslinya, bukan ditulis ulang sebagai angka. Jika jumlah butir atau rencana pendidikan diubah, batas
ini ikut berubah.

Beberapa pemeriksaan tidak bisa ditulis sebagai aturan Pydantic, misalnya "setiap jawaban 0–4" atau "email
harus berbentuk email". Pemeriksaan seperti itu dilakukan di dalam fungsi endpoint, dengan kode kesalahan
yang sama (422).

## 9.4 Endpoint akun

Cara kerja password, token, dan cookie sudah dibahas di Bab 8.4–8.6. Di sini dibahas bagaimana endpoint
merangkainya.

### Mengenali siswa: `current_user` dan `require_user`

`web/app.py` baris 73–89:

```python
COOKIE = "texin_session"


def current_user(request: Request):
    """The signed-in user as {id, name, email}, or None."""
    token = request.cookies.get(COOKIE)
    if not token:
        return None
    row = store().get_login_user(accounts.token_hash(token))
    return {"id": row["id"], "name": row["name"], "email": row["email"]} if row else None


def require_user(request: Request):
    user = current_user(request)
    if user is None:
        raise HTTPException(401, "Masuk dulu untuk melanjutkan")
    return user
```

- `current_user`: untuk endpoint yang **boleh** dipakai tanpa masuk. Hasilnya data siswa, atau `None`.
- `require_user`: untuk endpoint yang **wajib** masuk. Jika belum masuk, permintaan dihentikan dengan kode
  **401**.

### Daftar: `POST /api/auth/register`

`web/app.py` baris 100–111:

```python
@app.post("/api/auth/register")
def register(body: Register, request: Request, response: Response):
    name, email = body.name.strip(), accounts.clean_email(body.email)
    if not name:
        raise HTTPException(422, "Nama perlu diisi")
    if email is None:
        raise HTTPException(422, "Alamat email tidak valid")
    user_id = store().create_user(name, email, accounts.hash_password(body.password))
    if user_id is None:
        raise HTTPException(409, "Email ini sudah terdaftar. Silakan masuk.")
    _sign_in(request, response, user_id, remember=True)
    return {"id": user_id, "name": name, "email": email}
```

1. Rapikan nama dan email. Nama yang isinya hanya spasi ditolak.
2. Buat akun dengan password yang sudah di-hash.
3. Jika email sudah terdaftar, tolak dengan kode **409** (*Conflict*).
4. Langsung masukkan siswa dengan "Ingat saya" aktif, jadi siswa tidak perlu masuk lagi setelah mendaftar.

### Masuk: `POST /api/auth/login`

`web/app.py` baris 114–125:

```python
@app.post("/api/auth/login")
def login(body: Login, request: Request, response: Response):
    email = accounts.clean_email(body.email) or ""
    if accounts.too_many_failures(email):
        raise HTTPException(429, "Terlalu banyak percobaan. Coba lagi dalam 15 menit.")
    user = store().get_user_by_email(email) if email else None
    if not accounts.check_login(user, body.password):
        accounts.record_failure(email)
        raise HTTPException(401, "Email atau password salah")   # the same message for both, on purpose
    accounts.clear_failures(email)
    _sign_in(request, response, user["id"], body.remember)
    return {"id": user["id"], "name": user["name"], "email": user["email"]}
```

Urutannya sengaja seperti ini:

1. **Cek pemblokiran dulu**, sebelum memeriksa password. Jika sudah 10 kali salah, permintaan ditolak
   dengan kode **429** (*Too Many Requests*), bahkan jika password-nya kali ini benar.
2. **Periksa password.** Jika salah, catat kegagalannya. Pesannya sama untuk email yang tidak terdaftar
   maupun password yang salah (Bab 8.4).
3. **Jika benar**, hapus catatan kegagalan dan buat sesi.

Kedua endpoint mengembalikan nama dan email siswa, tidak pernah hash password-nya.

### Keluar: `POST /api/auth/logout`

`web/app.py` baris 128–134:

```python
@app.post("/api/auth/logout")
def logout(request: Request, response: Response):
    token = request.cookies.get(COOKIE)
    if token:
        store().delete_login(accounts.token_hash(token))
    response.delete_cookie(COOKIE, path="/")
    return {"ok": True}
```

Sesi dihapus dari basis data **dan** cookie dihapus dari browser. Menghapus di basis data itu penting:
cookie yang sudah disalin ke tempat lain pun tidak berlaku lagi.

## 9.5 Endpoint inti rekomendasi

### `GET /api/config`

`web/app.py` baris 194–207:

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
        "samples": [{"title": c["title"], "description": c["description"], "job_zone_name": c["job_zone_name"]}
                    for c in service.sample_careers()],
        "attribution": service.ATTRIBUTION,
    }
```

Dipanggil sekali saat halaman dibuka (Bab 1.2, langkah 2). Isinya semua yang dibutuhkan browser untuk
menjalankan kuesioner sendiri:

| Kunci | Isi | Dari |
|---|---|---|
| `prompt` | kalimat pengantar setiap butir | Bab 5.2 |
| `responses` | lima pilihan jawaban | Bab 5.2 |
| `types` | enam tipe RIASEC dan labelnya | Bab 1.1 |
| `items` | 60 butir (nomor, tipe, teks Indonesia) | Bab 5.4 |
| `plans` | lima rencana pendidikan dan Job Zone-nya | Bab 7.3 |
| `samples` | enam contoh karier untuk halaman depan | Bab 7.8 |
| `attribution` | teks atribusi O*NET | Bab 2.1 |

Teks Inggris butir kuesioner (`text_en`) tidak ikut dikirim, karena browser tidak membutuhkannya.

### `POST /api/recommend`

`web/app.py` baris 210–229:

```python
@app.post("/api/recommend")
def recommend(body: Answers, request: Request):
    if any(not 0 <= value <= 4 for value in body.answers):
        raise HTTPException(422, "Setiap jawaban harus antara 0 dan 4")
    result = service.recommendation({n + 1: v for n, v in enumerate(body.answers)}, body.max_job_zone)
    result["result_id"] = None
    result["feedback_given"] = False
    user = current_user(request)
    if user and result["cards"]:
        summary = {"interest": result["profile"][0]["label"],
                   "top": [{"title": c["title"], "match": c["match"]} for c in result["cards"][:3]]}
        latest = store().latest_result(user["id"])
        if latest and latest["answers"] == body.answers and latest["max_job_zone"] == body.max_job_zone:
            # the same test sent again (a double click, or Back then forward): not a new history entry
            result["result_id"] = latest["id"]
            result["feedback_given"] = store().get_result_feedback(latest["id"]) is not None
        else:
            result["result_id"] = store().add_result(user["id"], body.max_job_zone, body.answers, summary)
        store().delete_progress(user["id"])   # the test is finished
    return result
```

Endpoint terpenting di aplikasi ini. Langkah-langkahnya:

1. **Periksa jawaban.** Pydantic sudah memastikan ada tepat 60 bilangan bulat; di sini dicek setiap
   nilainya 0–4.
2. **Ubah bentuk jawaban.** Browser mengirim daftar `[1, 4, 2, ...]` sesuai urutan butir. `service`
   membutuhkan kamus `{nomor butir: nilai}`, jadi daftar itu diubah menjadi `{1: 1, 2: 4, 3: 2, ...}`.
   `n + 1` karena nomor butir dimulai dari 1, sedangkan posisi di daftar dimulai dari 0.
3. **Hitung rekomendasi** dengan `service.recommendation()` (Bab 7.5).
4. **Simpan ke Riwayat, jika siswa sedang masuk dan hasilnya tidak datar.** Ringkasan untuk daftar Riwayat
   disusun dari minat terkuat dan tiga kartu teratas (Bab 8.2).
5. **Cegah hasil ganda.** Jika jawaban dan rencana pendidikannya sama persis dengan hasil terakhir siswa,
   misalnya karena tombol diklik dua kali atau siswa menekan Kembali lalu Maju di browser, hasil lama yang
   dipakai, bukan membuat entri Riwayat baru.
6. **Hapus progres**, karena tesnya sudah selesai.

Endpoint ini tetap bisa dipakai tanpa masuk. Dalam hal itu, rekomendasi dihitung seperti biasa tapi tidak
disimpan (`result_id` bernilai `None`). Di tampilan saat ini, halaman tes hanya bisa dibuka setelah masuk.

### `GET /api/occupations/{code}` dan `GET /api/network/{code}`

`web/app.py` baris 242–255:

```python
@app.get("/api/occupations/{code}")
def get_occupation(code: str):
    detail = service.occupation_detail(code)
    if detail is None:
        raise HTTPException(404, "Pekerjaan tidak ditemukan")
    return detail


@app.get("/api/network/{code}")
def get_network(code: str):
    graph = service.career_graph(code)
    if graph is None:
        raise HTTPException(404, "Pekerjaan tidak ditemukan")
    return graph
```

`{code}` di alamat adalah **parameter path**: untuk `/api/occupations/15-2051.00`, FastAPI mengisi
parameter `code` dengan `"15-2051.00"`. Isi jawabannya dari `service` (Bab 7.6 dan 7.7). Jika kodenya tidak
dikenal, `service` mengembalikan `None`, yang di sini diubah menjadi kode **404** (*Not Found*).

## 9.6 Riwayat, progres, dan feedback

### `GET /api/state`: kondisi siswa saat halaman dibuka

`web/app.py` baris 170–177:

```python
@app.get("/api/state")
def state(request: Request):
    """Who is signed in, the test they are in the middle of, and their latest result."""
    user = current_user(request)
    if user is None:
        return {"user": None, "progress": None, "result": None}
    return {"user": user, "progress": store().get_progress(user["id"]),
            "result": _saved_result(user, store().latest_result_id(user["id"]))}
```

Dipanggil bersamaan dengan `/api/config` saat halaman dibuka. Dengan satu permintaan, browser tahu tiga hal:
siapa yang sedang masuk, tes yang belum selesai (jika ada), dan hasil terakhir. Karena data ini diambil dari
server, siswa yang masuk di perangkat lain langsung melihat kondisi yang sama.

`_saved_result` (baris 159–166) menghitung ulang hasil dari jawaban yang tersimpan (Bab 8.2), lalu
menambahkan ID hasil, waktunya, dan apakah feedback sudah diberikan.

### `PUT /api/progress`: menyimpan tes yang sedang dikerjakan

`web/app.py` baris 180–190:

```python
@app.put("/api/progress")
def put_progress(body: Progress, request: Request):
    user = require_user(request)
    if any(not 1 <= n <= len(ITEMS) or not 0 <= v <= 4 for n, v in body.answers.items()):
        raise HTTPException(422, "Jawaban tidak valid")
    if body.answers:
        store().save_progress(user["id"], {"answers": {str(n): v for n, v in body.answers.items()},
                                           "pos": body.pos, "plan": body.plan})
    else:
        store().delete_progress(user["id"])   # started over
    return {"ok": True}
```

Browser memanggil endpoint ini setiap kali siswa menjawab atau pindah pertanyaan. Nomor butir harus 1–60
dan nilainya 0–4. Jika daftar jawabannya kosong, artinya siswa memulai ulang tes, sehingga progresnya
dihapus.

Kunci nomor butir diubah menjadi teks (`str(n)`) sebelum disimpan, karena JSON hanya mengizinkan kunci
berupa teks. Dengan begitu, data yang dibaca kembali selalu berbentuk sama.

### Riwayat

`web/app.py` baris 138–156:

```python
@app.get("/api/history")
def history(request: Request):
    user = require_user(request)
    return {"results": [{"id": r["id"], "at": r["created_at"], **r["summary"]}
                        for r in store().list_results(user["id"])]}


@app.get("/api/history/{result_id}")
def history_result(result_id: str, request: Request):
    result = _saved_result(require_user(request), result_id)
    if result is None:
        raise HTTPException(404, "Hasil tidak ditemukan")
    return result


@app.delete("/api/history")
def clear_history(request: Request):
    store().delete_results(require_user(request)["id"])
    return {"ok": True}
```

- **Daftar Riwayat** hanya berisi ID, waktu, dan ringkasan setiap hasil. `**r["summary"]` menggabungkan
  isi ringkasan (`interest` dan `top`) langsung ke kamus yang sama.
- **Satu hasil** dihitung ulang lengkap. Jika hasil itu bukan milik siswa yang sedang masuk, jawabannya
  404, bukan 403. Dengan begitu, server tidak memberi tahu bahwa ID itu ada.
- **Hapus Riwayat** menghapus semua hasil dan feedback milik siswa itu saja.

### `POST /api/feedback`

`web/app.py` baris 232–239:

```python
@app.post("/api/feedback")
def post_feedback(body: Feedback, request: Request):
    """Kept with the student's own result."""
    user = require_user(request)
    if store().get_result(user["id"], body.result_id) is None:
        raise HTTPException(404, "Hasil tidak ditemukan")
    store().save_result_feedback(body.result_id, body.relevance, body.ease, (body.comment or "").strip())
    return {"saved": True}
```

Sebelum disimpan, dicek bahwa hasil yang dinilai memang milik siswa itu. Komentar dirapikan dari spasi di
awal dan akhir. Jika feedback dikirim lagi untuk hasil yang sama, isinya diganti (Bab 8.7).

## 9.7 Ekspor feedback untuk tim proyek

`web/app.py` baris 259–265:

```python
@app.get("/api/admin/feedback.csv")
def export_feedback(token: str = ""):
    expected = setting("ADMIN_TOKEN")
    if not expected or not token or not hmac.compare_digest(token, expected):
        raise HTTPException(404, "Not Found")   # do not reveal that the export exists
    return Response(service.export_feedback(store()), media_type="text/csv; charset=utf-8",
                    headers={"Content-Disposition": 'attachment; filename="feedback.csv"'})
```

Endpoint ini dipakai tim proyek untuk mengunduh feedback, dengan alamat
`/api/admin/feedback.csv?token=<ADMIN_TOKEN>`. Parameter `token: str = ""` yang tidak ada di alamat path
otomatis dibaca FastAPI dari bagian `?token=...`.

- Jika `ADMIN_TOKEN` tidak diisi di server, ekspor **mati sama sekali**.
- Token dibandingkan dengan `hmac.compare_digest` (Bab 8.4).
- Jika token salah, jawabannya **404**, sama seperti alamat yang tidak ada, sehingga orang luar tidak tahu
  bahwa alamat ekspor ini ada.
- `Content-Disposition: attachment` membuat browser langsung mengunduh hasilnya sebagai file
  `feedback.csv`, bukan menampilkannya.

Isi CSV-nya dibahas di Bab 7.9. Nama dan email siswa tidak ikut diekspor.

## 9.8 Menyajikan halaman dan file statis

`web/app.py` baris 268–283:

```python
@app.get("/")
def index():
    return FileResponse(STATIC / "index.html", headers={"Cache-Control": "no-cache"})


class FreshStatic(StaticFiles):
    """Static files that browsers re-check on every load (a cheap 304 when unchanged), so a new
    version of the site is never mixed with scripts cached from the old one."""

    async def get_response(self, path, scope):
        response = await super().get_response(path, scope)
        response.headers["Cache-Control"] = "no-cache"
        return response


app.mount("/static", FreshStatic(directory=STATIC), name="static")
```

- **`/`** mengirim `index.html`, satu-satunya halaman HTML. Semua halaman lain (tes, hasil, riwayat, dan
  seterusnya) digambar oleh JavaScript di dalam halaman ini (Bab 10).
- **`/static/...`** mengirim file apa pun dari folder `web/static/`, misalnya `/static/js/main.js` atau
  `/static/img/texin-putih.png`. `app.mount` memasang `StaticFiles` bawaan FastAPI di alamat itu.

### Kenapa `no-cache`

Browser biasanya menyimpan salinan (*cache*) file JavaScript dan CSS supaya halaman lebih cepat dibuka.
Masalahnya, setelah situs diperbarui, browser bisa memakai sebagian file lama dan sebagian file baru. Karena
file JavaScript texin saling mengimpor, campuran ini bisa membuat halaman rusak.

`Cache-Control: no-cache` **tidak** berarti "jangan simpan". Artinya: "simpan, tapi tanyakan dulu ke server
setiap kali akan dipakai". Jika file tidak berubah, server cukup menjawab **304** (*Not Modified*) tanpa
mengirim ulang isinya, jadi tetap cepat. `FreshStatic` adalah turunan `StaticFiles` yang menambahkan header
ini ke setiap file.

Rute `/` ditulis sebelum `app.mount`, tapi urutannya tidak berpengaruh karena keduanya melayani alamat yang
berbeda.

## 9.9 Kode status dan pesan kesalahan

| Kode | Arti | Contoh di texin |
|---|---|---|
| 200 | berhasil | semua permintaan yang berhasil |
| 401 | belum masuk / gagal masuk | `require_user`, password salah |
| 404 | tidak ditemukan | kode pekerjaan tidak dikenal, hasil milik orang lain, token admin salah |
| 409 | bentrok | email sudah terdaftar |
| 422 | input tidak valid | jawaban di luar 0–4, email tidak valid, password terlalu pendek |
| 429 | terlalu banyak percobaan | 10 kali password salah dalam 15 menit |

`HTTPException(kode, pesan)` menghentikan endpoint dan mengirim JSON `{"detail": "pesan"}`. Semua pesan
ditulis dalam bahasa Indonesia supaya bisa langsung ditampilkan ke siswa.

Di browser, semua panggilan API melewati satu fungsi (`web/static/js/api.js` baris 9–30):

```javascript
export async function api(path, body, method) {
  const options = { credentials: "same-origin" };
  if (body !== undefined) {
    options.method = "POST";
    options.headers = { "Content-Type": "application/json" };
    options.body = JSON.stringify(body);
  }
  if (method) options.method = method;
  const response = await fetch(path, options);
  if (!response.ok) {
    const detail = await response.json().catch(() => ({}));
    const error = new Error(
      typeof detail.detail === "string"
        ? detail.detail
        : "Terjadi kesalahan. Coba lagi.",
    );
    error.status = response.status;
    if (response.status === 401 && !path.startsWith("/api/auth/")) unauthorized();
    throw error;
  }
  return response.json();
}
```

- `credentials: "same-origin"` membuat cookie sesi selalu ikut terkirim.
- Jika server menjawab dengan kesalahan, pesan `detail`-nya diambil dan ditampilkan ke siswa.
- Kesalahan yang dibuat otomatis oleh Pydantic punya `detail` berupa daftar, bukan teks, sehingga yang
  tampil adalah pesan umum "Terjadi kesalahan. Coba lagi." Halaman masuk dan daftar sudah memeriksa input
  lebih dulu di browser (misalnya panjang password), jadi kasus ini jarang dilihat siswa.
- Kode **401** di luar endpoint akun berarti sesi siswa sudah berakhir. Browser lalu mengarahkan siswa ke
  halaman masuk (Bab 10).

## 9.10 Dipasang di Vercel

`pyproject.toml`:

```toml
[project]
name = "texin"
...
requires-python = ">=3.12"
dependencies = [
    "numpy",
    "fastapi",
    "uvicorn",
    "psycopg[binary]",
]

[tool.vercel]
entrypoint = "web.app:app"
```

Vercel membaca `entrypoint` untuk tahu objek aplikasi mana yang dijalankan, yaitu `app` di `web/app.py`,
sama seperti perintah `uvicorn` saat dijalankan di komputer sendiri. Daftar paketnya harus sama dengan
`requirements.txt`. Di Vercel, variabel `DATABASE_URL` wajib diisi supaya data disimpan di Postgres
(Bab 8.1).

## 9.11 Tes API

Tesnya ada di `tests/test_web.py`. Setiap tes memakai `TestClient` dari FastAPI, yang memanggil endpoint
seperti browser tanpa perlu menjalankan server sungguhan. Sebelum setiap tes, `setUp` (baris 29–39)
menyiapkan basis data SQLite sementara yang baru, jadi tes tidak saling memengaruhi dan tidak menyentuh data
asli.

| Tes | Baris | Yang diperiksa |
|---|---|---|
| `test_config_has_the_sixty_items` | 46–53 | 60 butir, 5 pilihan jawaban, 6 tipe, 6 contoh karier |
| `test_recommend_ranks_by_match_and_explains` | 55–67 | 10 kartu urut menurun, ada alasan; tamu yang tidak masuk tidak menyimpan apa pun |
| `test_education_plan_limits_the_job_zone` | 69–71 | saringan Job Zone bekerja |
| `test_identical_answers_give_no_cards` | 73–77 | profil datar tanpa kartu |
| `test_bad_answers_are_rejected` | 79–81 | 59 jawaban atau jawaban bernilai 5 → 422 |
| `test_detail_and_network` | 83–94 | detail dan jaringan karier; kode tidak dikenal → 404 |
| `test_register_signs_in_and_never_stores_the_password` | 99–114 | email dirapikan; cookie `HttpOnly` dan `SameSite`; basis data hanya berisi hash password dan hash token |
| `test_register_rejects_bad_input_and_duplicate_email` | 116–123 | password pendek, email tidak valid, nama kosong → 422; email kembar (beda huruf besar-kecil) → 409 |
| `test_login_logout_and_wrong_password` | 125–145 | pesan salah sama untuk email tak dikenal dan password salah; "Ingat saya" memberi cookie 30 hari; cookie lama tidak berlaku setelah keluar |
| `test_repeated_wrong_passwords_are_blocked_for_that_email` | 147–154 | 10 kali salah → 401, ke-11 → 429, password benar pun ditolak |
| `test_history_belongs_to_the_account` | 156–188 | hasil datar tidak disimpan; kiriman ganda tidak membuat entri baru; siswa lain tidak bisa melihat atau menghapus Riwayat orang lain |
| `test_state_follows_the_account_to_another_device` | 190–215 | progres dan hasil terakhir terlihat di perangkat lain; progres hilang setelah tes selesai atau dimulai ulang |
| `test_feedback_is_kept_with_the_students_own_result` | 217–243 | feedback hanya untuk hasil sendiri; dikirim ulang → diganti; ekspor CSV butuh token yang benar dan tidak memuat nama/email |
| `test_page_and_assets_are_served_without_the_wrong_algorithm_name` | 245–259 | halaman dan file statis terkirim dengan `no-cache`; semua `import` di JavaScript menunjuk ke file yang ada |

Tes terakhir juga memastikan kode JavaScript menyebut "Pearson Correlation" dan **tidak** menyebut
"PageRank", metode yang sempat dipertimbangkan di awal proyek tetapi tidak dipakai.

## 9.12 Ringkasan

```
Browser (web/static/js/api.js)
   │  fetch + cookie texin_session
   ▼
web/app.py
   ├─ Pydantic           periksa bentuk input            → 422
   ├─ current_user /     kenali siswa dari cookie        → 401
   │  require_user
   ├─ endpoint           panggil crs/ ...
   │     ├─ accounts     password, token, pembatasan
   │     ├─ service      rekomendasi, detail, jaringan, ekspor
   │     └─ store()      basis data
   └─ JSON / file        kirim kembali ke browser
```

- `web/app.py` adalah **jembatan tipis**: periksa input, panggil `crs/`, kirim hasil.
- Input diperiksa dua lapis: **Pydantic** untuk bentuknya, dan pemeriksaan tambahan di dalam endpoint.
- Data milik siswa (Riwayat, progres, feedback) hanya bisa diakses oleh **akun pemiliknya**.
- Hasil rekomendasi yang sama tidak disimpan dua kali.
- File tampilan dikirim dengan `no-cache` supaya browser tidak mencampur versi lama dan baru.
