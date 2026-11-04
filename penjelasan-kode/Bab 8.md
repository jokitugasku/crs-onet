# Bab 8. Penyimpanan dan Akun (`crs/storage.py`, `crs/accounts.py`)

Bab 2–7 membahas perhitungan rekomendasi, yang tidak butuh basis data sama sekali. Bab ini membahas bagian
yang **menyimpan**: akun siswa, sesi masuk, tes yang sedang dikerjakan, hasil tes (Riwayat), dan feedback.

Ada dua file dengan tugas berbeda:

| File | Tugas | Menyentuh basis data? |
|---|---|---|
| `crs/accounts.py` | memeriksa email, mengacak (*hash*) password, membuat token masuk, membatasi percobaan masuk | tidak |
| `crs/storage.py` | membaca dan menulis tabel di basis data | ya |

`web/app.py` memakai keduanya bersama-sama. Endpoint-nya dibahas di Bab 9; bab ini fokus pada isi kedua
file tersebut.

## 8.1 Dua jenis basis data, satu kode

Docstring `crs/storage.py` baris 1–9:

```python
"""Storage for accounts, the test a student is in the middle of, results and feedback.

Two backends behind the same functions:
  - SQLite file (default, data/crs.sqlite3; override with CRS_DB) for local use
  - Postgres, when a DATABASE_URL is given, for hosting where local files do not survive restarts

An account holds a name, an email and a password hash. Each finished test is kept under its
account so the student can see it again (Riwayat), with the feedback given on it.
"""
```

| | SQLite | Postgres |
|---|---|---|
| Dipakai saat | menjalankan di komputer sendiri | dipasang daring (Vercel + Neon) |
| Lokasi data | file `data/crs.sqlite3` | server basis data terpisah |
| Dipilih jika | `DATABASE_URL` tidak diisi | `DATABASE_URL` diisi |

Postgres dibutuhkan saat dipasang daring karena Vercel tidak menyimpan file. Setiap kali server dimulai
ulang, file lokal hilang, jadi akun dan riwayat siswa juga akan hilang jika disimpan di SQLite.

Pilihan basis data ditentukan di `crs/storage.py` baris 72–74:

```python
    @property
    def backend(self):
        return "postgres" if self.database_url else "sqlite"
```

`web/app.py` membuat satu objek `Store` saat pertama dibutuhkan (`web/app.py` baris 34–39):

```python
def store():
    """Opened on first use."""
    global _store
    if _store is None:
        _store = Store(setting("DATABASE_URL"))
    return _store
```

## 8.2 Tabel-tabel basis data

Struktur tabel ditulis sebagai daftar perintah SQL di `crs/storage.py` baris 21–55. Setiap perintah memakai
`CREATE TABLE IF NOT EXISTS`, jadi tabel dibuat otomatis saat aplikasi pertama kali dijalankan, dan
dibiarkan jika sudah ada.

```
users ──┬──< logins            (satu akun bisa masuk di beberapa perangkat)
        ├──< results ──── result_feedback   (satu hasil punya paling banyak satu feedback)
        └─── progress          (satu akun punya paling banyak satu tes yang sedang dikerjakan)
```

### `users`: akun

```sql
CREATE TABLE IF NOT EXISTS users (
    id            TEXT PRIMARY KEY,
    name          TEXT NOT NULL,
    email         TEXT NOT NULL UNIQUE,
    password_hash TEXT NOT NULL,
    created_at    TEXT NOT NULL
)
```

Yang disimpan hanya nama, email, dan **hash** password (8.4), bukan password aslinya. `UNIQUE` pada email
membuat satu email hanya bisa didaftarkan sekali.

### `logins`: sesi masuk

```sql
CREATE TABLE IF NOT EXISTS logins (
    token_hash  TEXT PRIMARY KEY,
    user_id     TEXT NOT NULL REFERENCES users(id),
    created_at  TEXT NOT NULL,
    expires_at  TEXT NOT NULL
)
```

Satu baris untuk setiap kali siswa masuk. Yang disimpan adalah **hash token**, bukan tokennya (8.5).
`REFERENCES users(id)` menandai bahwa `user_id` harus merujuk ke akun yang ada.

### `results`: hasil tes (Riwayat)

```sql
CREATE TABLE IF NOT EXISTS results (
    id            TEXT PRIMARY KEY,
    user_id       TEXT NOT NULL REFERENCES users(id),
    created_at    TEXT NOT NULL,
    max_job_zone  INTEGER,
    answers       TEXT NOT NULL,
    summary       TEXT NOT NULL
)
```

| Kolom | Isi |
|---|---|
| `max_job_zone` | rencana pendidikan yang dipilih saat tes; kosong (`NULL`) jika "tampilkan semua" |
| `answers` | 60 jawaban, disimpan sebagai teks JSON, misalnya `[1, 4, 2, ...]` |
| `summary` | ringkasan untuk daftar Riwayat: minat terkuat dan tiga rekomendasi teratas |

Contoh `summary`:

```json
{"interest": "Investigatif (Analitis)",
 "top": [{"title": "Sejarawan", "match": 98}, {"title": "Ilmuwan Data", "match": 98}, {"title": "Ahli Biostatistika", "match": 95}]}
```

**Hasil rekomendasi lengkapnya tidak disimpan.** Yang disimpan hanya jawaban dan rencana pendidikan. Saat
siswa membuka hasil lama, rekomendasinya **dihitung ulang** dari jawaban itu (`web/app.py` baris 159–166):

```python
def _saved_result(user, result_id):
    """A stored result rebuilt for the result pages, or None when it is not this user's."""
    saved = store().get_result(user["id"], result_id) if result_id else None
    if saved is None:
        return None
    result = service.recommendation({n + 1: v for n, v in enumerate(saved["answers"])}, saved["max_job_zone"])
    ...
```

Cara ini membuat basis data tetap kecil, karena satu hasil lengkap memuat 10 kartu dengan deskripsi,
alasan, dan perbandingan profil. Konsekuensinya, jika data O*NET atau cara perhitungannya diubah, hasil
lama juga ikut berubah saat dibuka lagi. Hanya `summary` yang tetap seperti saat tes dikerjakan.

### `result_feedback`: feedback atas hasil

```sql
CREATE TABLE IF NOT EXISTS result_feedback (
    result_id   TEXT PRIMARY KEY REFERENCES results(id),
    relevance   INTEGER NOT NULL,
    ease        INTEGER NOT NULL,
    comment     TEXT,
    created_at  TEXT NOT NULL
)
```

`result_id` menjadi `PRIMARY KEY`, artinya satu hasil hanya bisa punya satu feedback. `relevance` dan `ease`
adalah dua rating bintang 1–5; `comment` boleh kosong.

### `progress`: tes yang sedang dikerjakan

```sql
CREATE TABLE IF NOT EXISTS progress (
    user_id     TEXT PRIMARY KEY REFERENCES users(id),
    data        TEXT NOT NULL,
    updated_at  TEXT NOT NULL
)
```

Satu baris per akun. `data` berisi JSON: jawaban sejauh ini, posisi pertanyaan, dan rencana pendidikan.
Contoh:

```json
{"answers": {"1": 3, "2": 0}, "pos": 2, "plan": 4}
```

Tabel ini membuat siswa bisa berhenti di tengah tes, lalu melanjutkannya di perangkat lain. Barisnya
dihapus saat tes selesai.

### Format waktu dan ID

Semua kolom waktu disimpan sebagai teks ISO 8601 dalam zona waktu UTC (`crs/storage.py` baris 58–59):

```python
def _now():
    return datetime.now(timezone.utc).isoformat(timespec="seconds")
```

Contohnya `2026-11-04T02:32:28+00:00`. Format ini bisa dibandingkan langsung sebagai teks: waktu yang lebih
awal selalu "lebih kecil" secara abjad. Sifat ini dipakai untuk mengecek sesi yang kedaluwarsa (8.5).

ID akun dan ID hasil dibuat dengan `secrets.token_urlsafe(12)` (baris 141 dan 169), yaitu teks acak 16
karakter seperti `kq2X9fN0aTzL3bPw`. ID acak tidak bisa ditebak, berbeda dengan nomor urut 1, 2, 3, ...

## 8.3 Menjalankan query di dua basis data

### Satu gaya query untuk keduanya

`crs/storage.py` baris 62–63:

```python
class Store:
    """Queries are written with '?' placeholders and work on both backends."""
```

Semua query ditulis dengan tanda `?` sebagai tempat nilai, gaya yang dipakai SQLite. Postgres memakai `%s`,
jadi tanda itu diganti sebelum query dijalankan (baris 127):

```python
                    cur = self._postgres().execute(sql.replace("?", "%s"), params)
```

Nilai **selalu** dikirim terpisah dari teks query (`params`), tidak pernah digabung langsung ke teks SQL.
Dengan cara ini, isi yang diketik siswa (nama, email, komentar) tidak bisa disalahgunakan untuk menjalankan
perintah SQL lain (*SQL injection*).

### `_run()`: satu pintu untuk semua query

`crs/storage.py` baris 112–134:

```python
    def _run(self, sql, params=(), fetch=None):
        """fetch: None (write), 'one' or 'all'."""
        if self.backend == "sqlite":
            conn = self._sqlite()
            try:
                cur = conn.execute(sql, params)
                rows = cur.fetchone() if fetch == "one" else cur.fetchall() if fetch == "all" else None
                conn.commit()
                return rows
            finally:
                conn.close()
        import psycopg
        with self._lock:
            for attempt in (1, 2):   # a hosted connection can drop while the app is idle: reconnect once
                try:
                    cur = self._postgres().execute(sql.replace("?", "%s"), params)
                    return cur.fetchone() if fetch == "one" else cur.fetchall() if fetch == "all" else None
                except psycopg.OperationalError:
                    if self._pg is not None:
                        self._pg.close()
                    self._pg = None
                    if attempt == 2:
                        raise
```

Semua method `Store` lainnya memanggil `_run`, jadi perbedaan SQLite dan Postgres hanya diurus di satu
tempat ini. Parameter `fetch` menentukan hasilnya: `None` untuk perintah tulis, `"one"` untuk satu baris,
`"all"` untuk semua baris.

- **SQLite**: setiap query membuka koneksi baru lalu menutupnya lagi. Koneksi ke file lokal sangat cepat,
  jadi cara ini aman dan sederhana.
- **Postgres**: satu koneksi dipakai ulang, karena membuka koneksi ke server di internet butuh waktu.
  `self._lock` memastikan hanya satu query yang memakai koneksi itu dalam satu waktu. Jika koneksinya
  terputus (sering terjadi pada server daring yang lama tidak dipakai), koneksi ditutup, dibuka ulang, dan
  query dicoba **sekali lagi**.

### Membuka SQLite

`crs/storage.py` baris 77–98:

```python
    def sqlite_path(self):
        return Path(os.environ.get("CRS_DB") or self._fallback or DEFAULT_DB)

    def _sqlite(self):
        path = self.sqlite_path()
        try:
            conn = sqlite3.connect(path)
            conn.execute("BEGIN IMMEDIATE")   # takes the write lock: fails here if the file cannot be written
            conn.rollback()
        except sqlite3.OperationalError:
            # Read-only or missing folder on the host: keep the app usable with a temporary file.
            if os.environ.get("CRS_DB") or self._fallback:
                raise
            self._fallback = Path(tempfile.gettempdir()) / "crs.sqlite3"
            return self._sqlite()
        conn.row_factory = sqlite3.Row
        if self._ready_for != path:
            for ddl in TABLES:
                conn.execute(ddl)
            conn.commit()
            self._ready_for = path
        return conn
```

1. **Lokasi file**: dari variabel lingkungan `CRS_DB` jika diisi (dipakai oleh tes), lalu file cadangan
   jika ada, lalu `data/crs.sqlite3`.
2. **Cek bisa ditulis**: `BEGIN IMMEDIATE` mencoba mengambil izin tulis, lalu langsung dibatalkan. Jika
   folder tidak bisa ditulis, aplikasi pindah ke file sementara di folder temp sistem, supaya tetap bisa
   jalan walaupun datanya tidak permanen.
3. **`row_factory = sqlite3.Row`**: baris hasil query bisa dibaca dengan nama kolom, misalnya
   `row["email"]`, sama seperti di Postgres (`dict_row`, baris 106).
4. **Buat tabel sekali**: tabel dibuat saat file pertama dibuka. `_ready_for` mencatat file mana yang sudah
   disiapkan, jadi langkah ini tidak diulang di setiap query.

### Membuka Postgres

`crs/storage.py` baris 100–110:

```python
    def _postgres(self):
        import psycopg
        from psycopg.rows import dict_row
        if self._pg is None or self._pg.closed:
            # prepare_threshold=None: hosted Postgres is often reached through a transaction pooler,
            # which does not support server-side prepared statements.
            self._pg = psycopg.connect(self.database_url, autocommit=True, row_factory=dict_row,
                                       prepare_threshold=None, connect_timeout=15)
            for ddl in TABLES:
                self._pg.execute(ddl)
        return self._pg
```

`psycopg` baru diimpor di dalam fungsi ini, jadi paket itu tidak perlu ada jika hanya memakai SQLite.
`autocommit=True` membuat setiap perintah langsung tersimpan, sama seperti `conn.commit()` di SQLite.

## 8.4 Password: `hash_password` dan `verify_password`

Docstring `crs/accounts.py` baris 1–5:

```python
"""Accounts: checking what a student types, hashing passwords, and login tokens.

Passwords are never stored: only a salted scrypt hash is. A login is a random token kept in an
HttpOnly cookie; the database holds only its SHA-256, so a leaked database cannot be used to log in.
"""
```

### Kenapa password di-hash

Password **tidak pernah disimpan**. Yang disimpan adalah hasil *hash*-nya: deretan karakter yang dihitung
dari password, tapi tidak bisa dibalik menjadi password lagi. Saat masuk, password yang diketik di-hash
dengan cara yang sama, lalu hasilnya dibandingkan. Jika basis data bocor, password asli siswa tetap aman.

### Membuat hash

`crs/accounts.py` baris 14 dan 28–32:

```python
SCRYPT_N, SCRYPT_R, SCRYPT_P = 2 ** 14, 8, 1
...
def hash_password(password):
    salt = secrets.token_bytes(16)
    digest = hashlib.scrypt(password.encode(), salt=salt, n=SCRYPT_N, r=SCRYPT_R, p=SCRYPT_P)
    return "scrypt${}${}${}${}${}".format(SCRYPT_N, SCRYPT_R, SCRYPT_P, base64.b64encode(salt).decode(),
                                         base64.b64encode(digest).decode())
```

1. **Salt**: 16 byte acak, berbeda untuk setiap akun. Dua siswa dengan password yang sama akan punya hash
   yang berbeda, sehingga tidak bisa dicocokkan dengan daftar hash yang sudah dihitung sebelumnya.
2. **scrypt**: algoritma hash yang sengaja dibuat **lambat dan butuh banyak memori**. Bagi pengguna biasa,
   perbedaannya tidak terasa. Bagi penyerang yang mencoba jutaan password, cara ini membuat percobaannya
   sangat mahal. Parameter `n`, `r`, dan `p` mengatur seberapa berat perhitungannya.
3. **Hasil**: semua bagian digabung menjadi satu teks dengan pemisah `$`:

```
scrypt$16384$8$1$4pJh17TKphvrfUOAHcJ3lQ==$1tG0XP3eVYcPXrRc27vPAwmO5eEG+RdMWPF7VTkFT1O4...
│      │     │ │ │                        └─ hash (Base64)
│      │     │ │ └─ salt (Base64)
│      │     │ └─ p
│      │     └─ r
│      └─ n
└─ nama algoritma
```

Parameter dan salt ikut disimpan, jadi password tetap bisa diperiksa walaupun parameternya nanti diubah
untuk akun-akun baru.

### Memeriksa password

`crs/accounts.py` baris 35–43:

```python
def verify_password(password, stored):
    try:
        _, n, r, p, salt, digest = stored.split("$")
        expected = base64.b64decode(digest)
        actual = hashlib.scrypt(password.encode(), salt=base64.b64decode(salt), n=int(n), r=int(r), p=int(p),
                                dklen=len(expected))
    except (ValueError, AttributeError):
        return False
    return hmac.compare_digest(actual, expected)
```

Teks hash dipecah lagi menjadi bagian-bagiannya, password yang diketik di-hash dengan salt dan parameter
yang sama, lalu kedua hash dibandingkan. Jika teks hash rusak, hasilnya `False`, tidak error.

`hmac.compare_digest` dipakai sebagai ganti `==`. Perbandingan `==` berhenti di karakter pertama yang
berbeda, jadi waktunya sedikit berbeda tergantung seberapa mirip hasilnya. `compare_digest` selalu butuh
waktu yang sama, sehingga tidak membocorkan informasi lewat waktu.

### Email yang tidak terdaftar

`crs/accounts.py` baris 46–52:

```python
# Checked when the email is unknown, so a wrong email takes as long as a wrong password.
_DUMMY_HASH = hash_password(secrets.token_urlsafe(16))


def check_login(user, password):
    """True when the password fits. `user` is a row with password_hash, or None."""
    return verify_password(password, user["password_hash"] if user else _DUMMY_HASH) and user is not None
```

Jika email tidak terdaftar (`user` bernilai `None`), password tetap diperiksa terhadap hash palsu
`_DUMMY_HASH`. Hasilnya pasti `False`, tapi waktunya sama dengan memeriksa password sungguhan. Tanpa ini,
email yang tidak terdaftar akan langsung ditolak dengan cepat, dan penyerang bisa menebak email mana yang
terdaftar dari lamanya jawaban server.

Untuk alasan yang sama, pesan kesalahannya juga selalu sama, "Email atau password salah"
(`web/app.py` baris 122). Pesan itu tidak menyebut mana yang salah.

### Memeriksa email

`crs/accounts.py` baris 18 dan 22–25:

```python
_EMAIL = re.compile(r"^[^@\s]+@[^@\s]+\.[^@\s]+$")
...
def clean_email(email):
    """Lower-cased, trimmed email, or None when it does not look like one."""
    email = (email or "").strip().lower()
    return email if len(email) <= 254 and _EMAIL.match(email) else None
```

Email dirapikan (spasi di awal/akhir dibuang, huruf dijadikan kecil), lalu dicek bentuknya: ada `@`, ada
titik setelahnya, dan tidak ada spasi. Dengan huruf kecil, `Sari@Example.com` dan `sari@example.com`
dianggap email yang sama.

## 8.5 Sesi masuk: token dan cookie

Setelah siswa berhasil masuk, server perlu mengenali siswa itu di setiap permintaan berikutnya tanpa meminta
password lagi. Untuk itu dipakai **token**.

### Membuat token

`crs/accounts.py` baris 55–68:

```python
def new_token():
    """(token for the cookie, its hash for the database)."""
    token = secrets.token_urlsafe(32)
    return token, token_hash(token)


def token_hash(token):
    return hashlib.sha256(token.encode()).hexdigest()


def expiry(remember):
    """ISO time after which a login stops working."""
    life = timedelta(days=LOGIN_DAYS) if remember else timedelta(hours=12)
    return (datetime.now(timezone.utc) + life).isoformat(timespec="seconds")
```

- **Token**: 32 byte acak (43 karakter), misalnya `LgIGkZN8LA6SOocVJ4ulroQJ0k7zGO56yGOvRqFAAnA`. Token ini
  dikirim ke browser sebagai cookie.
- **Hash token**: SHA-256 dari token. Hanya ini yang disimpan di tabel `logins`. Jika basis data bocor,
  hash token tidak bisa dipakai untuk masuk, karena server selalu meminta token aslinya.
- **Masa berlaku**: 30 hari jika siswa mencentang "Ingat saya", 12 jam jika tidak.

Token cukup di-hash dengan SHA-256, tidak perlu scrypt. Token sudah acak dan sangat panjang, jadi tidak
mungkin ditebak, berbeda dengan password yang dibuat manusia.

### Alur masuk

Fungsi-fungsi di atas dirangkai di `web/app.py` (dibahas lengkap di Bab 9):

```
Siswa mengirim email + password
      │
      ▼
accounts.clean_email()              rapikan email
accounts.too_many_failures()        diblokir? → 429
store.get_user_by_email()           cari akun
accounts.check_login()              cocokkan password (scrypt)
      │  salah → accounts.record_failure() → 401
      ▼  benar
accounts.new_token()                token + hash-nya
store.add_login(user_id, hash, accounts.expiry(remember))
      │
      ▼
Cookie "texin_session" = token      dikirim ke browser
```

Cookie dipasang di `web/app.py` baris 92–97:

```python
def _sign_in(request, response, user_id, remember):
    token, hashed = accounts.new_token()
    store().add_login(user_id, hashed, accounts.expiry(remember))
    https = request.url.scheme == "https" or request.headers.get("x-forwarded-proto") == "https"
    response.set_cookie(COOKIE, token, httponly=True, samesite="lax", secure=https, path="/",
                        max_age=accounts.LOGIN_DAYS * 86400 if remember else None)
```

| Pengaturan cookie | Arti |
|---|---|
| `httponly=True` | JavaScript di halaman tidak bisa membaca cookie ini, jadi token tidak bisa dicuri lewat skrip |
| `samesite="lax"` | cookie tidak dikirim dari permintaan yang dibuat situs lain |
| `secure=https` | saat situs memakai HTTPS, cookie hanya dikirim lewat HTTPS |
| `max_age` | 30 hari dengan "Ingat saya"; tanpa itu, cookie hilang saat browser ditutup |

### Mengenali siswa di setiap permintaan

`web/app.py` baris 76–82 dan `crs/storage.py` baris 159–162:

```python
def current_user(request: Request):
    """The signed-in user as {id, name, email}, or None."""
    token = request.cookies.get(COOKIE)
    if not token:
        return None
    row = store().get_login_user(accounts.token_hash(token))
    ...
```

```python
    def get_login_user(self, token_hash):
        """The user behind a login token, or None when it is unknown or expired."""
        return self._run("SELECT users.id, users.name, users.email FROM logins JOIN users ON users.id = logins.user_id "
                         "WHERE logins.token_hash = ? AND logins.expires_at > ?", (token_hash, _now()), "one")
```

Token dari cookie di-hash, lalu dicari di tabel `logins`. Kondisi `expires_at > ?` membuat sesi yang sudah
kedaluwarsa tidak dikenali lagi. Perbandingan ini bisa dilakukan langsung pada teks karena format waktunya
ISO (8.2). `JOIN` dengan tabel `users` langsung mengambil nama dan email siswa. Kolom `password_hash`
sengaja tidak diambil.

### Merapikan dan keluar

`crs/storage.py` baris 154–165:

```python
    def add_login(self, user_id, token_hash, expires_at):
        self._run("DELETE FROM logins WHERE expires_at < ?", (_now(),))   # tidy up while we are here
        self._run("INSERT INTO logins (token_hash, user_id, created_at, expires_at) VALUES (?, ?, ?, ?)",
                  (token_hash, user_id, _now(), expires_at))
    ...
    def delete_login(self, token_hash):
        self._run("DELETE FROM logins WHERE token_hash = ?", (token_hash,))
```

Setiap kali ada yang masuk, sesi-sesi yang sudah kedaluwarsa ikut dihapus, jadi tabel tidak terus membesar.
Saat siswa keluar, baris sesinya dihapus, sehingga cookie yang disalin ke tempat lain pun tidak bisa dipakai
lagi.

## 8.6 Membatasi percobaan masuk

`crs/accounts.py` baris 17, 19, dan 71–88:

```python
MAX_FAILURES, FAILURE_WINDOW = 10, 15 * 60   # wrong passwords allowed per email, per 15 minutes
_failures = {}             # email -> [time of each recent wrong password]
...
def too_many_failures(email):
    recent = [t for t in _failures.get(email, []) if time.time() - t < FAILURE_WINDOW]
    _failures[email] = recent
    return len(recent) >= MAX_FAILURES


def record_failure(email):
    if len(_failures) > 10_000:   # do not let a flood of made-up emails fill the memory
        for key in [k for k, times in _failures.items() if time.time() - times[-1] >= FAILURE_WINDOW]:
            del _failures[key]
    _failures.setdefault(email, []).append(time.time())


def clear_failures(email=None):
    if email is None:
        _failures.clear()
    else:
        _failures.pop(email, None)
```

Tujuannya mencegah penyerang mencoba ribuan password untuk satu email.

- `_failures` mencatat **waktu** setiap password salah, per email.
- `too_many_failures`: hitung percobaan salah dalam 15 menit terakhir. Jika sudah 10 kali, percobaan
  berikutnya ditolak dengan pesan "Terlalu banyak percobaan. Coba lagi dalam 15 menit." Catatan yang lebih
  lama dari 15 menit dibuang.
- `record_failure`: catat satu percobaan salah. Jika ada lebih dari 10.000 email di catatan, email yang
  sudah lama tidak dicoba dibuang dulu, supaya memori server tidak penuh karena email karangan.
- `clear_failures`: hapus catatan setelah berhasil masuk. Tanpa argumen, semua catatan dihapus (dipakai
  oleh tes).

Catatan ini disimpan di **memori server**, bukan di basis data. Akibatnya, catatan hilang saat server
dimulai ulang. Jika server berjalan di beberapa mesin sekaligus, seperti yang bisa terjadi di Vercel, setiap
mesin juga punya catatannya sendiri. Untuk keperluan presentasi, perlindungan ini sudah cukup, tapi bukan
pembatas yang benar-benar ketat.

## 8.7 Method `Store` lainnya

### Akun

`crs/storage.py` baris 137–152:

```python
    def create_user(self, name, email, password_hash):
        """Return the new user's id, or None when the email is already registered."""
        if self.get_user_by_email(email) is not None:
            return None
        user_id = secrets.token_urlsafe(12)
        try:
            self._run("INSERT INTO users (id, name, email, password_hash, created_at) VALUES (?, ?, ?, ?, ?)",
                      (user_id, name, email, password_hash, _now()))
        except Exception as error:   # two registrations of one email at the same moment
            if type(error).__name__ in ("IntegrityError", "UniqueViolation"):
                return None
            raise
        return user_id
```

Email dicek dulu. Pemeriksaan ini belum cukup jika dua pendaftaran dengan email yang sama datang di saat
yang persis bersamaan. Untuk kasus itu, `UNIQUE` di tabel menolak yang kedua, dan kesalahan itu ditangkap
lalu dikembalikan sebagai `None`. `IntegrityError` adalah nama kesalahannya di SQLite, `UniqueViolation`
di Postgres.

### Hasil tes

| Method | Baris | Fungsi |
|---|---|---|
| `add_result` | 168–173 | simpan hasil baru, kembalikan ID-nya |
| `list_results` | 175–179 | semua hasil milik akun, terbaru lebih dulu, untuk halaman Riwayat |
| `get_result` | 181–187 | satu hasil, **hanya jika milik akun itu** |
| `latest_result` | 189–195 | hasil terbaru beserta jawabannya |
| `latest_result_id` | 197–200 | ID hasil terbaru saja |
| `delete_results` | 202–205 | hapus semua hasil dan feedback milik akun |

`get_result` selalu mencari dengan dua syarat (baris 183):

```python
        r = self._run("SELECT * FROM results WHERE id = ? AND user_id = ?", (result_id, user_id), "one")
```

Jadi siswa tidak bisa membuka hasil siswa lain walaupun mengetahui ID-nya.

`delete_results` menghapus feedback lebih dulu, baru hasilnya (baris 203–205), karena feedback merujuk ke
hasil. Jika urutannya terbalik, akan tertinggal feedback yang merujuk ke hasil yang sudah tidak ada.

### Feedback

`crs/storage.py` baris 208–212:

```python
    def save_result_feedback(self, result_id, relevance, ease, comment):
        self._run("INSERT INTO result_feedback (result_id, relevance, ease, comment, created_at) "
                  "VALUES (?, ?, ?, ?, ?) ON CONFLICT (result_id) DO UPDATE SET relevance = excluded.relevance, "
                  "ease = excluded.ease, comment = excluded.comment, created_at = excluded.created_at",
                  (result_id, relevance, ease, comment or None, _now()))
```

`ON CONFLICT ... DO UPDATE` (disebut *upsert*): jika hasil itu belum punya feedback, buat baru; jika sudah,
ganti isinya. Siswa bisa memperbaiki feedback-nya tanpa membuat baris ganda. Perintah ini didukung SQLite
maupun Postgres dengan bentuk yang sama.

`all_result_feedback` (baris 217–223) mengambil semua feedback beserta `summary` hasilnya, **tanpa** nama
dan email, untuk ekspor CSV (Bab 7.9).

### Tes yang sedang dikerjakan

`save_progress` (baris 226–229) memakai *upsert* yang sama, sehingga setiap akun hanya punya satu baris
progres yang terus diperbarui. `get_progress` (231–233) membacanya, dan `delete_progress` (235–236)
menghapusnya saat tes selesai atau dimulai ulang.

## 8.8 Tes

**`tests/test_storage.py`** memakai satu kumpulan tes (`StorageContract`, baris 17–77) yang dijalankan untuk
**kedua** basis data. Dengan cara ini, SQLite dan Postgres dijamin berperilaku sama.

| Tes | Baris | Yang diperiksa |
|---|---|---|
| `test_accounts_and_logins` | 22–36 | email tidak bisa didaftarkan dua kali; sesi kedaluwarsa tidak dikenali; sesi yang dihapus tidak berlaku lagi |
| `test_progress_is_one_per_user_and_replaced` | 38–46 | progres kedua menggantikan yang pertama, bukan menambah baris |
| `test_results_and_feedback_belong_to_their_user` | 48–77 | hasil hanya bisa dibuka pemiliknya; feedback kedua menggantikan yang pertama; ekspor CSV berisi data yang benar; menghapus hasil orang lain tidak menyentuh hasil sendiri |

Tes khusus SQLite (baris 80–108) memeriksa bahwa aplikasi pindah ke file sementara jika folder `data/`
tidak bisa ditulis. Tes khusus Postgres (baris 111–122) memeriksa bahwa koneksi yang terputus tersambung
ulang otomatis. Tes Postgres hanya berjalan jika variabel `TEST_DATABASE_URL` diisi dengan alamat basis data
uji; jika tidak, tes itu dilewati.

**`tests/test_web.py`** menguji alur akun lewat API, misalnya bahwa password tidak pernah tersimpan dalam
bentuk asli, password yang salah berulang kali diblokir, dan riwayat hanya bisa dibuka pemiliknya. Tes-tes
ini dibahas di Bab 9.

## 8.9 Ringkasan

| Yang disimpan | Tabel | Catatan |
|---|---|---|
| Nama, email, hash password | `users` | password asli tidak pernah disimpan (scrypt + salt) |
| Sesi masuk | `logins` | hanya hash token (SHA-256); berlaku 12 jam atau 30 hari |
| Tes yang sedang dikerjakan | `progress` | satu per akun, dihapus saat tes selesai |
| Hasil tes | `results` | jawaban + ringkasan; rekomendasi dihitung ulang saat dibuka |
| Feedback | `result_feedback` | satu per hasil; diekspor tanpa nama dan email |
| Percobaan masuk yang salah | memori server | 10 kali per email per 15 menit |

- Satu kode `Store` melayani **SQLite** (lokal) dan **Postgres** (daring).
- Semua nilai dikirim terpisah dari teks SQL, sehingga aman dari *SQL injection*.
- Setiap data hasil dan feedback hanya bisa diakses oleh **akun pemiliknya**.
