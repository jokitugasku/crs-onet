# Bab 10. Tampilan (`web/static/`)

Bab ini membahas apa yang berjalan di **browser**: halaman yang dilihat siswa, cara berpindah halaman,
cara jawaban disimpan, dan cara browser berbicara dengan API di Bab 9.

## 10.1 Gambaran umum

Tampilan texin adalah **aplikasi satu halaman** (*single-page application*). Server hanya mengirim satu file
HTML, `index.html`. Semua halaman lain (beranda, tes, hasil, riwayat, dan seterusnya) **digambar oleh
JavaScript** di dalam halaman itu. Saat siswa berpindah halaman, browser tidak memuat ulang halaman baru dari
server; JavaScript hanya mengganti isi halaman.

Kodenya ditulis dengan JavaScript biasa yang dipecah menjadi **modul** (`import`/`export`), tanpa *framework*
seperti React atau Vue. Tampilannya memakai **Tailwind CSS**.

| File | Isi |
|---|---|
| `index.html` | satu-satunya halaman HTML, wadah kosong untuk aplikasi |
| `tailwind.css` | CSS hasil build dari `web/tailwind.css` |
| `js/main.js` | titik awal: daftar halaman, aturan akses, dan semua *event listener* |
| `js/router.js` | berpindah halaman lewat alamat `#/...` |
| `js/state.js` | data yang dipegang browser: konfigurasi, akun, tes yang sedang dikerjakan |
| `js/sync.js` | menyelaraskan data di browser dengan data di server |
| `js/api.js` | memanggil API server (Bab 9.9) |
| `js/components/ui.js` | potongan kecil yang dipakai di mana-mana: ikon, tombol, bar, kolom isian |
| `js/components/shell.js` | kerangka halaman: menu samping, penanda langkah, kartu masuk/daftar |
| `js/components/charts.js` | diagram radar profil minat |
| `js/views/*.js` | satu file untuk setiap halaman atau kelompok halaman |

Prinsip utamanya ditulis di komentar pembuka `js/main.js` baris 1–16:

```javascript
/* texin - find your career. Entry point: routes, guards and the one set of event listeners.
   ...
   Views return HTML strings. Elements ask for behaviour with data attributes:
     data-act="name"      runs actions.name()            data-nav="route"   side menu
     data-answer="0..4"   answers the open question      data-tab="n"       detail tab
     data-pick="code"     selects a career in the graph  data-star="x:n"    star rating
     data-scroll="id"     scrolls the landing page       data-open="id"    opens an earlier result
     data-peek="id"       shows or hides a password
           <form data-form="name">  runs forms.name() */
```

Dua hal penting dari komentar ini:

1. **Setiap halaman adalah fungsi yang mengembalikan teks HTML.** Fungsi itu tidak memasang *event* apa pun.
2. **Tombol meminta perilakunya lewat atribut `data-...`**, misalnya `data-act="logout"`. Satu *listener* di
   `main.js` membaca atribut itu dan menjalankan fungsi yang sesuai (10.6).

## 10.2 `index.html` dan Tailwind CSS

`web/static/index.html` baris 20–32:

```html
  <body>
    <div id="app"><p class="p-8 text-center text-mute">Memuat…</p></div>
    <div
      id="toast"
      class="pointer-events-none fixed inset-x-0 bottom-6 z-50 flex justify-center px-4"
    ></div>
    <noscript
      ><p style="padding: 2rem; text-align: center">
        Aplikasi ini membutuhkan JavaScript.
      </p></noscript
    >
    <script type="module" src="/static/js/main.js"></script>
  </body>
```

- `#app`: wadah tempat semua halaman digambar. Isinya "Memuat…" sampai JavaScript selesai dimuat.
- `#toast`: wadah pesan singkat yang muncul di bawah layar, misalnya "Email atau password salah".
- `<script type="module">`: memuat `main.js` sebagai modul. `main.js` lalu mengimpor modul-modul lainnya.

### Tailwind CSS

Tailwind adalah CSS berbasis *utility class*: tampilan ditulis langsung sebagai nama kelas di HTML, misalnya
`rounded-xl bg-brand-500 px-4 py-3`. Warna merek texin didefinisikan di `web/tailwind.css` baris 7–20:

```css
@theme {
  --font-sans:
    "Plus Jakarta Sans", ui-sans-serif, system-ui, -apple-system, "Segoe UI",
    sans-serif;
  --color-brand-50: #eff5ff;
  ...
  --color-brand-500: #2f80ff;
  ...
  --color-ink: #0f1e3d;
  --color-mute: #5b6b8a;
```

Dari sini muncul kelas seperti `bg-brand-500` (biru texin), `text-ink` (teks utama), dan `text-mute` (teks
sekunder). Perintah `npm run css` memindai semua file di `web/static/`, mencari nama kelas yang dipakai,
lalu menulis CSS yang dibutuhkan saja ke `web/static/tailwind.css`. File hasil build itu ikut di-commit,
jadi server tidak butuh Node.js.

## 10.3 Data di browser: `js/state.js`

`js/state.js` baris 1–33:

```javascript
/* What the app knows: the server's configuration, who is signed in, and the test in progress.
   The account and the history live on the server. The test in progress and the last result are
   kept in this browser (localStorage) so a reload does not lose them. */
const KEY = "texin-v1";

export let CONFIG = null;
export const setConfig = (config) => {
  CONFIG = config;
};

const blank = () => ({
  user: null,
  owner: null, // id of the account the test below belongs to
  plan: 0,
  answers: {},
  pos: 0,
  result: null,
  feedbackSent: false,
});
export let S = blank();
try {
  S = Object.assign(blank(), JSON.parse(localStorage.getItem(KEY) || "{}"));
  ...
} catch (e) {
  /* private window */
}
export const save = () => {
  try {
    localStorage.setItem(KEY, JSON.stringify(S));
  } catch (e) {
    /* not available */
  }
};
```

Browser memegang dua objek:

- **`CONFIG`**: isi `/api/config` (Bab 9.5), yaitu butir kuesioner, pilihan jawaban, rencana pendidikan,
  dan lainnya. Diisi sekali saat halaman dibuka dan tidak berubah.
- **`S`** (*state*): kondisi siswa saat ini.

| Isi `S` | Arti |
|---|---|
| `user` | siswa yang sedang masuk (`{id, name, email}`), atau `null` |
| `owner` | ID akun pemilik tes di bawah ini |
| `plan` | rencana pendidikan yang dipilih (nomor urut di `CONFIG.plans`) |
| `answers` | jawaban sejauh ini, `{nomor butir: 0..4}` |
| `pos` | posisi pertanyaan yang sedang dibuka (0–59) |
| `result` | hasil rekomendasi terakhir (Bab 7.5) |
| `feedbackSent` | apakah feedback untuk hasil itu sudah dikirim |

`S` disimpan ke **localStorage** browser setiap kali berubah (`save()`). Karena itu, memuat ulang halaman
tidak menghapus jawaban siswa. Pemanggilan `localStorage` dibungkus `try`, karena di beberapa mode browser
(misalnya jendela privat tertentu) penyimpanan ini tidak tersedia. Aplikasi tetap jalan; hanya saja datanya
tidak tersimpan saat halaman dimuat ulang.

### Tes milik siapa

`js/state.js` baris 35–49:

```javascript
export function setUser(user) {
  if (!user || S.owner !== user.id) S = blank();
  S.user = user;
  S.owner = user ? user.id : null;
  save();
}

export function sessionExpired() {
  S.user = null;
  save();
}
```

Satu browser bisa dipakai beberapa siswa, misalnya di laboratorium komputer sekolah. Karena itu, setiap tes
dicatat pemiliknya (`owner`). Jika yang masuk adalah akun lain, semua data di browser dikosongkan, sehingga
siswa kedua tidak melihat jawaban siswa pertama.

Bedakan dua cara sesi berakhir:

- **Siswa menekan Keluar** → `setUser(null)` → semua dikosongkan.
- **Sesi kedaluwarsa** → `sessionExpired()` → hanya `user` yang dikosongkan; jawaban tetap disimpan,
  menunggu pemiliknya masuk lagi.

### Fungsi bantu

`js/state.js` baris 59–67:

```javascript
export const NEEDS_RESULT = ["hasil", "karier", "jaringan", "feedback"];
export const answered = () => Object.keys(S.answers).length;
export const total = () => CONFIG.items.length;
export const hasResult = () =>
  Boolean(S.result && S.result.cards && S.result.cards.length);
```

`NEEDS_RESULT` adalah halaman yang hanya bisa dibuka setelah ada hasil rekomendasi.

## 10.4 Berpindah halaman: `js/router.js` dan `main.js`

### Alamat berbentuk `#/...`

`js/router.js`:

```javascript
/* Hash routes: "#/hasil", "#/karier/15-2051.00". main.js registers the function that draws the page. */
let draw = () => {};
export const onRender = (fn) => {
  draw = fn;
};
export const render = () => draw();
/** `replace` keeps the current page out of the Back history (used for pages that only pass through). */
export const go = (route, replace = false) => {
  if (replace) return location.replace("#/" + route);
  location.hash = "#/" + route;
};
export const currentRoute = () => location.hash.replace(/^#\/?/, "").split("/");
```

Halaman dibedakan dari bagian alamat setelah tanda `#`, disebut *hash*. Contohnya:

| Alamat | Halaman | Argumen |
|---|---|---|
| `/#/` | beranda (landing) | – |
| `/#/tes` | pengarahan tes | – |
| `/#/hasil` | hasil rekomendasi | – |
| `/#/karier/15-2051.00` | detail karier | `15-2051.00` |

Mengubah *hash* tidak membuat browser meminta halaman baru ke server, tapi tetap tercatat di riwayat browser.
Tombol Kembali dan Maju di browser pun tetap bekerja.

- `go(route)`: pindah halaman. Dengan `replace = true`, halaman saat ini diganti dan **tidak** tercatat di
  riwayat browser. Cara ini dipakai untuk halaman perantara seperti layar analisis, supaya tombol Kembali
  tidak menjalankan analisis lagi.
- `currentRoute()`: memecah *hash* menjadi nama halaman dan argumennya, misalnya
  `"karier/15-2051.00"` → `["karier", "15-2051.00"]`.
- `render()`: menggambar ulang halaman. Fungsi penggambarnya didaftarkan oleh `main.js` lewat `onRender`.
  Dengan pola ini, `router.js` tidak perlu mengimpor `main.js`, jadi kedua modul tidak saling mengimpor.

### Daftar halaman

`js/main.js` baris 43–58:

```javascript
const VIEWS = {
  "": landing.viewLanding,
  masuk: auth.viewMasuk,
  daftar: auth.viewDaftar,
  dasbor: viewDasbor,
  tes: quiz.viewTes,
  kuis: quiz.viewKuis,
  ringkasan: quiz.viewRingkasan,
  analisis: quiz.viewAnalisis,
  hasil: results.viewHasil,
  karier: results.viewKarier,
  jaringan: network.viewJaringan,
  feedback: feedback.viewFeedback,
  riwayat: history.viewRiwayat,
};
const PUBLIC = ["", "masuk", "daftar"];
```

Setiap nama halaman dipetakan ke fungsi yang menggambarnya. Hanya tiga halaman yang bisa dibuka tanpa masuk
(`PUBLIC`): beranda, masuk, dan daftar.

### Menggambar halaman dan aturan akses

`js/main.js` baris 83–104:

```javascript
onRender(() => {
  if (!CONFIG) return; // a hash change before the first load finished
  const [route, arg] = currentRoute();
  if (!VIEWS[route]) return go(S.user ? "dasbor" : "");
  if (!PUBLIC.includes(route) && !S.user) return go("masuk");
  if (NEEDS_RESULT.includes(route) && !hasResult()) return go("dasbor");
  if ((route === "ringkasan" || route === "analisis") && answered() < total())
    return go("tes", true);
  if (route === "analisis" && hasResult()) return go("hasil", true); // already analysed
  const arrived = location.hash !== lastHash;
  lastHash = location.hash;
  if (arrived && route === "karier") results.resetTab();
  app.innerHTML = VIEWS[route](arg);
  if (route === "kuis") quiz.paintQuiz();
  if (arrived) {
    const page = app.querySelector("[data-page]");
    if (page) page.classList.add("animate-rise");
    window.scrollTo(0, 0);
    if (route === "analisis") quiz.runAnalysis();
  }
});
```

Sebelum halaman digambar, dicek dulu apakah siswa boleh membukanya:

| Kondisi | Diarahkan ke |
|---|---|
| nama halaman tidak dikenal | dasbor (jika sudah masuk) atau beranda |
| halaman non-publik, belum masuk | halaman masuk |
| halaman hasil, belum ada hasil | dasbor |
| ringkasan/analisis, jawaban belum lengkap | pengarahan tes |
| analisis, padahal hasil sudah ada | hasil |

Jika lolos, fungsi halamannya dipanggil dan hasil HTML-nya dimasukkan ke `#app`.

`arrived` membedakan **membuka** halaman dengan **menggambar ulang** halaman yang sama. Misalnya, memilih
bintang di halaman feedback menggambar ulang halaman yang sama. Animasi masuk, gulir ke atas, dan
menjalankan analisis hanya dilakukan saat halaman benar-benar baru dibuka.

Aturan akses ini hanya untuk kenyamanan tampilan. Keamanan yang sebenarnya ada di server: data milik siswa
tetap tidak bisa diambil tanpa sesi yang sah (Bab 9.4).

### Memulai aplikasi

`js/main.js` baris 135 dan 144–152:

```javascript
window.addEventListener("hashchange", render);
...
Promise.all([api("/api/config"), api("/api/state")])
  .then(([config, state]) => {
    setConfig(config);
    applyState(state);
    render();
  })
  .catch(() => {
    app.innerHTML = `<p class="p-8 text-center text-mute">Tidak bisa memuat aplikasi. Muat ulang halaman.</p>`;
  });
```

Saat halaman dibuka, konfigurasi dan kondisi siswa diminta **bersamaan** (`Promise.all`), lalu halaman
pertama digambar. Setelah itu, setiap perubahan *hash* menggambar ulang halaman.

## 10.5 Menangani klik dan formulir

`js/main.js` baris 59–79 dan 106–125:

```javascript
const actions = {
  ...shell.actions,
  ...quiz.actions,
  ...history.actions,
};
const forms = { ...auth.forms, ...feedback.forms };
// data attribute -> what a click on it does
const CLICKS = {
  act: (name) => actions[name](),
  nav: shell.onNav,
  answer: (value) => quiz.onAnswer(Number(value)),
  tab: results.onTab,
  pick: network.onPick,
  star: feedback.onStar,
  scroll: landing.onScroll,
  open: history.onOpen,
  peek: auth.onPeek,
};
const CLICKABLE = Object.keys(CLICKS)
  .map((key) => `[data-${key}]`)
  .join(",");
...
document.addEventListener("click", (event) => {
  const el = event.target.closest(CLICKABLE);
  if (!el) return;
  const key = Object.keys(CLICKS).find((k) => el.dataset[k] !== undefined);
  CLICKS[key](el.dataset[key], event);
});

let submitting = false;

document.addEventListener("submit", async (event) => {
  event.preventDefault();
  const handler = forms[event.target.dataset.form];
  if (!handler || submitting) return; // a second click while the first is still on its way
  submitting = true;
  try {
    await handler();
  } finally {
    submitting = false;
  }
});
```

Halaman digambar ulang dengan mengganti seluruh isi `#app`. Jika setiap tombol diberi *event listener*
sendiri, semuanya harus dipasang ulang setiap kali halaman digambar. Karena itu dipakai **satu listener** di
`document` (*event delegation*):

1. Saat ada klik, cari elemen terdekat yang punya salah satu atribut `data-...` yang dikenal
   (`closest(CLICKABLE)`).
2. Lihat atribut mana yang dimilikinya, lalu jalankan fungsi yang sesuai dengan nilai atributnya.

Contoh: tombol `<button data-answer="3">` menjalankan `quiz.onAnswer(3)`, dan tombol
`<button data-act="logout">` menjalankan `actions.logout()`.

Formulir ditangani dengan cara yang sama: `<form data-form="masuk">` menjalankan `forms.masuk()`.
`preventDefault()` mencegah browser mengirim formulir dengan cara biasa, yang akan memuat ulang halaman.
`submitting` mencegah formulir terkirim dua kali jika tombol diklik berulang saat permintaan pertama belum
selesai.

## 10.6 Sesi berakhir

`js/main.js` baris 137–142:

```javascript
onUnauthorized(() => {
  if (!S.user) return;
  sessionExpired(); // the test stays in this browser for when its owner signs in again
  toast("Sesi masukmu berakhir. Silakan masuk lagi.");
  go("masuk");
});
```

Jika server menjawab 401 (Bab 9.9), artinya sesi siswa sudah berakhir. Siswa diarahkan ke halaman masuk,
tapi jawaban tesnya tetap disimpan di browser.

## 10.7 Menyelaraskan browser dan server: `js/sync.js`

Data tes yang sedang dikerjakan ada di **dua tempat**: localStorage browser (supaya cepat dan tahan dimuat
ulang) dan tabel `progress` di server (supaya bisa dilanjutkan di perangkat lain). `sync.js` menjaga keduanya
tetap sama.

### Mengirim progres ke server

`js/sync.js` baris 43–59:

```javascript
let timer = null;

/** Send the test in progress to the server: at most once per `delay`, and always the latest state. */
export function pushProgress(delay = 700) {
  if (!S.user) return;
  if (delay === 0) clearTimeout(timer);
  else if (timer) return; // one is already on its way
  timer = setTimeout(() => {
    timer = null;
    if (!S.user) return;
    api(
      "/api/progress",
      { answers: S.answers, pos: S.pos, plan: S.plan },
      "PUT",
    ).catch(() => {}); // the browser copy still holds it; the next change tries again
  }, delay);
}
```

Fungsi ini dipanggil setiap kali siswa menjawab. Supaya server tidak dibanjiri permintaan saat siswa menjawab
cepat, pengiriman ditunda 700 milidetik, dan selama penundaan itu panggilan berikutnya diabaikan. Saat
akhirnya dikirim, yang dikirim selalu kondisi **terbaru** (`S.answers` dibaca saat itu, bukan saat fungsi
dipanggil). Jika pengiriman gagal, tidak apa-apa: salinan di browser masih ada, dan jawaban berikutnya akan
mencoba lagi.

### Menerima kondisi dari server

`js/sync.js` baris 10–39:

```javascript
export function applyState(state) {
  if (!state.user) {
    if (S.user) sessionExpired();
    return;
  }
  setUser(state.user);
  const progress = state.progress;
  const latest = state.result;
  if (progress && (S.result || count(progress.answers) >= answered())) {
    // a test is under way on the server, at least as far along as the one here
    S.answers = progress.answers;
    S.pos = progress.pos;
    S.plan = progress.plan;
    S.result = null;
  } else if (progress) {
    pushProgress(0); // this browser is further along: tell the server
  } else if (answered() === total() && !S.result) {
    // all answered, waiting on the summary page: nothing to take over
  } else {
    const sameResult =
      S.result && latest && S.result.result_id === latest.result_id;
    if (!sameResult) S.answers = {};
    if (!sameResult) S.pos = 0;
    S.result = latest;
    S.feedbackSent = Boolean(latest && latest.feedback_given);
  }
  save();
}
```

Dipanggil saat halaman dibuka dan setelah siswa masuk. Aturannya:

| Kondisi | Tindakan |
|---|---|
| server: belum masuk | jika browser mengira sudah masuk, tandai sesi berakhir |
| server punya tes yang **sama jauh atau lebih jauh** | ambil dari server |
| browser punya tes yang **lebih jauh** | kirim ke server |
| semua butir sudah dijawab, menunggu di halaman ringkasan | biarkan |
| server tidak punya tes yang sedang dikerjakan | tes di browser sudah selesai atau dimulai ulang di perangkat lain; ambil hasil terakhir dari server |

Dengan aturan ini, siswa bisa mulai tes di ponsel, lalu melanjutkannya di laptop dari pertanyaan yang sama.

## 10.8 Komponen bersama

### `js/components/ui.js`

**Mencegah HTML berbahaya** (baris 2–3):

```javascript
export const h = (text) => String(text ?? "").replace(/[&<>"']/g, (c) => (
  { "&": "&amp;", "<": "&lt;", ">": "&gt;", '"': "&quot;", "'": "&#39;" }[c]));
```

Karena halaman disusun sebagai teks HTML, setiap teks yang berasal dari data (nama siswa, judul pekerjaan,
komentar) harus di-*escape* lebih dulu. Fungsi `h()` mengganti karakter seperti `<` menjadi `&lt;`. Tanpa ini,
siswa yang mendaftar dengan nama `<script>...</script>` bisa membuat kode itu berjalan di halaman
(*cross-site scripting*). Hampir setiap teks di semua halaman melewati `h()`.

Fungsi lain di file ini:

| Fungsi | Baris | Menggambar |
|---|---|---|
| `toast(message)` | 5–10 | pesan singkat di bawah layar, hilang sendiri setelah 3,2 detik |
| `icon(name)` | 32–33 | ikon SVG dari kumpulan garis di `ICONS` |
| `logo()` | 35–36 | logo texin |
| `ring(percent)` | 38–48 | lingkaran persen kecocokan |
| `bar(percent)` | 50–51 | bar horizontal |
| `Field(...)` | 54–64 | kolom isian berlabel dengan ikon; kolom password diberi tombol tampilkan/sembunyikan |
| `CheckList`, `Meter`, `Note` | 70–75 | daftar centang, bar berlabel, kotak catatan |

**Lingkaran kecocokan** (`ring`, baris 38–48) memakai trik SVG: lingkaran digambar dengan garis putus-putus
(`stroke-dasharray`) sepanjang kelilingnya, lalu digeser (`stroke-dashoffset`) sebesar bagian yang tidak
terisi. Untuk kecocokan 98%, hanya 2% keliling yang kosong.

### `js/components/shell.js`

- **`appShell(active, content)`** (baris 13–41): kerangka semua halaman setelah masuk, berisi menu samping,
  kepala halaman dengan nama siswa, isi halaman, dan kaki halaman berisi atribusi O*NET. Menu yang butuh
  hasil ditandai terkunci sebelum tes selesai.
- **`stepper(current)`** (baris 43–51): penanda empat langkah tes: Pengarahan → Isi Kuis → Analisis → Hasil.
- **`authShell(content)`** (baris 54–60): kartu di tengah layar untuk halaman masuk dan daftar.
- **`onNav(route)`** (baris 76–79): klik menu. Jika menu terkunci, muncul pesan "Selesaikan tes minat karier
  dulu."

### `js/components/charts.js`: diagram radar

`js/components/charts.js` baris 4–20 menggambar profil minat sebagai segi enam, satu sudut untuk setiap tipe
RIASEC:

```javascript
export function radar(rows) {
  const cx = 150, cy = 140, R = 96;
  const point = (k, value) => {
    const angle = -Math.PI / 2 + k * Math.PI / 3;
    return [cx + R * value * Math.cos(angle), cy + R * value * Math.sin(angle)];
  };
  ...
  const shape = polygon((k) => Math.max(0.04, rows[k].percent / 100));
```

Tipe ke-*k* diletakkan pada sudut *k* × 60°, dimulai dari atas (−90°). Jarak titik dari pusat sebanding dengan
persen minatnya. Titik-titik itu lalu disambung menjadi poligon biru. `Math.max(0.04, ...)` menjaga agar tipe
dengan 0% tetap terlihat sedikit, tidak menempel di pusat.

## 10.9 Alur halaman

```
Beranda ──► Masuk / Daftar ──► Dasbor ──► Tes (pengarahan) ──► Kuis (60 butir)
                                                                   │
                                                                   ▼
             Riwayat ◄──┐                         Ringkasan Minat (radar)
                        │                                          │
 Jaringan Karier ◄── Hasil Rekomendasi ◄──────────── Analisis ◄────┘
        │                │       │
        └──► Detail ◄────┘       └──► Feedback
             Karier
```

### Beranda (`views/landing.js`)

Halaman publik yang memperkenalkan texin: bagian hero, fitur, penjelasan, cara kerja, contoh karier dari
`CONFIG.samples`, dan ajakan mulai (baris 175–176). Tombol "Mulai Sekarang" membuka halaman tes jika siswa
sudah masuk, atau halaman masuk jika belum (baris 52). Klik menu di atas menggulir ke bagian yang dituju tanpa
mengubah *hash*, karena *hash* dipakai router (baris 179–182).

### Masuk dan Daftar (`views/auth.js`)

Formulir masuk dan daftar memeriksa input di browser lebih dulu: email harus berbentuk email, password
minimal 8 karakter, dan konfirmasi password harus sama (baris 58–81). Jika lolos, data dikirim ke
`/api/auth/login` atau `/api/auth/register`. Setelah berhasil, kondisi siswa diambil dari server dan siswa
dibawa ke dasbor (baris 48–56):

```javascript
async function enter(path, body) {
  try {
    await api(path, body);
    await loadState(); // the account, a test in progress and the latest result
    go("dasbor");
  } catch (error) {
    toast(error.message);
  }
}
```

Password hanya dikirim sekali lewat formulir dan tidak pernah disimpan di browser (komentar baris 1).

### Dasbor (`views/dashboard.js`)

Ringkasan kondisi siswa: status tes, minat terkuat, rekomendasi teratas, dan satu tombol untuk langkah
berikutnya. Tombolnya menyesuaikan kondisi: "Ke Tes Minat Karier", "Lanjutkan Tes", atau "Lihat Hasil
Rekomendasi" (baris 15–31).

### Tes dan Kuis (`views/quiz.js`)

**Pengarahan** (`viewTes`, baris 21–49): aturan mengerjakan tes dan pilihan rencana pendidikan.

**Kuis** (`viewKuis`, baris 73–87): satu pertanyaan per layar, dengan lima tombol jawaban dan bar kemajuan.

Supaya halaman tidak berkedip setiap kali siswa menjawab, halaman kuis **tidak** digambar ulang seluruhnya.
Hanya bagian yang berubah yang diperbarui (komentar baris 51–52):

- `onAnswer(value)` (baris 128–143): simpan jawaban, kirim progres, tandai pilihan, lalu **pindah ke
  pertanyaan berikutnya secara otomatis setelah 280 milidetik**, supaya siswa sempat melihat pilihannya.
- `paintQuiz()` (baris 90–108): perbarui nomor pertanyaan, persen terjawab, bar kemajuan, dan tombol.
- `swapQuestion(direction)` (baris 113–125): animasi kartu pertanyaan bergeser ke kiri atau kanan.
- `onKeydown(event)` (baris 306–312): tombol 1–5 untuk menjawab, panah kiri/kanan untuk pindah pertanyaan.

**Ringkasan Minat** (`viewRingkasan`, baris 180–205): diagram radar dan persen enam tipe, dihitung di browser
(Bab 5.7). Jika profilnya datar, muncul peringatan dan tombol Ulangi Kuis.

**Analisis** (`viewAnalisis` dan `runAnalysis`, baris 207–266): layar perantara yang mengirim jawaban ke
`/api/recommend`. Daftar langkah di layar ini (baris 207–212) adalah **animasi**, ditampilkan satu per satu
dengan jeda 550 milidetik:

```javascript
const ANALYSIS = [
  "Memetakan profil minat kamu",
  "Membandingkan dengan 878 profil pekerjaan O*NET",
  "Menghitung kecocokan dengan Pearson Correlation",
  "Menyusun rekomendasi karier terbaik",
];
```

Perhitungan sebenarnya berlangsung di server dalam satu permintaan dan biasanya selesai jauh lebih cepat.
Permintaan sudah dikirim sejak awal, dan langkah terakhir menunggu jawabannya (baris 237). Setelah hasil
diterima, siswa dibawa ke halaman hasil dengan `go(..., true)`, supaya tombol Kembali tidak membuka layar
analisis lagi (baris 261).

### Hasil Rekomendasi dan Detail Karier (`views/results.js`)

**Hasil** (`viewHasil`, baris 17–43): sepuluh kartu, masing-masing berisi judul, deskripsi, label kecocokan,
Job Zone, dan lingkaran persen kecocokan. Di atasnya ada catatan bahwa kecocokan adalah korelasi Pearson dan
hasilnya bahan eksplorasi, bukan keputusan.

**Detail Karier** (`viewKarier`, baris 54–111): empat tab.

| Tab | Isi | Dari |
|---|---|---|
| Deskripsi | deskripsi, Job Zone, pendidikan yang dibutuhkan | `/api/occupations` |
| Mengapa Cocok | alasan dan perbandingan profil RIASEC siswa vs. pekerjaan | kartu di `S.result` (Bab 7.4) |
| Skill yang Dibutuhkan | 10 kompetensi, gaya kerja, software | `/api/occupations` |
| Karier Serupa | 10 pekerjaan terkait | `/api/occupations` |

Detail diambil dari server saat pertama dibuka, lalu disimpan di `detailCache` (baris 52), jadi membuka
pekerjaan yang sama lagi tidak meminta ke server lagi. Selama menunggu, halaman menampilkan "Memuat detail
karier…", lalu digambar ulang setelah data tiba (baris 55–70).

Tab "Mengapa Cocok" hanya terisi untuk pekerjaan yang termasuk sepuluh rekomendasi siswa. Pekerjaan yang
dibuka dari daftar karier serupa tidak punya perbandingan, dan halaman menjelaskan hal itu (baris 88).

### Jaringan Karier (`views/network.js`)

Satu pekerjaan di tengah dan delapan pekerjaan terkait di sekelilingnya, dari `/api/network` (Bab 7.7).
Posisinya dihitung di lingkaran (baris 31–34):

```javascript
  const place = (k) => {
    const angle = -Math.PI / 2 + (k * 2 * Math.PI) / g.nodes.length;
    return [50 + 39 * Math.cos(angle), 50 + 39 * Math.sin(angle)];
  };
```

Pekerjaan ke-*k* diletakkan pada sudut *k* × (360° / 8), dalam satuan persen dari lebar kotak gambar, jadi
gambarnya menyesuaikan ukuran layar. Garis ke lima pekerjaan terdekat (`close`) lebih tebal. Pekerjaan yang
juga ada di rekomendasi siswa diberi warna berbeda. Mengklik pekerjaan menampilkan keterangannya, dan
"Jadikan pusat" memindahkan pekerjaan itu ke tengah, sehingga siswa bisa terus menjelajah.

### Feedback (`views/feedback.js`)

Dua rating bintang ("Seberapa sesuai rekomendasi ini?" dan "Seberapa mudah aplikasi ini digunakan?") dan
komentar opsional, dikirim ke `/api/feedback` untuk hasil yang sedang dibuka. Kedua rating wajib diisi
(baris 58–59). Kolom komentar mengingatkan siswa untuk tidak menulis nama atau kontak (baris 44).

### Riwayat (`views/history.js`)

Tabel hasil-hasil sebelumnya dari `/api/history`: tanggal, minat terkuat, dan tiga rekomendasi teratas.
Tombol "Lihat" membuka hasil itu lagi di halaman hasil (`onOpen`, baris 77–86). Tombol "Hapus riwayat"
menghapus semuanya lewat `DELETE /api/history` (baris 88–98).

## 10.10 Tes

`tests/test_web.py`, `test_page_and_assets_are_served_without_the_wrong_algorithm_name` (baris 245–259),
memeriksa sisi tampilan:

- halaman `/` terkirim dan memuat `tailwind.css` serta `main.js`,
- file JavaScript, CSS, dan gambar terkirim dengan header `no-cache`,
- teks JavaScript menyebut "Pearson Correlation" dan tidak menyebut "PageRank",
- setiap `import` di setiap file JavaScript menunjuk ke file yang benar-benar ada. Jika ada modul yang
  dipindah atau diganti nama tanpa memperbarui `import`-nya, tes ini gagal.

Interaksi di browser (klik, animasi, tampilan) tidak diuji otomatis.

## 10.11 Ringkasan

```
index.html ──► main.js
                 ├─ GET /api/config + /api/state  →  CONFIG, S (state.js, sync.js)
                 ├─ hashchange → render()
                 │     ├─ cek aturan akses
                 │     └─ #app.innerHTML = VIEWS[route](arg)       views/*.js → teks HTML
                 │                                                  components/ → potongan bersama
                 ├─ click  → data-act / data-answer / ...  → fungsi di views
                 ├─ submit → data-form                     → forms (masuk, daftar, feedback)
                 └─ api.js → fetch ke server (Bab 9)
```

- Satu halaman HTML; semua halaman lain digambar JavaScript berdasarkan **hash** di alamat.
- Setiap halaman adalah fungsi yang mengembalikan **teks HTML**; perilaku tombol diminta lewat atribut
  `data-...` dan ditangani **satu listener**.
- Tes yang sedang dikerjakan disimpan di **localStorage** dan di **server**; `sync.js` menyelaraskan
  keduanya.
- Semua teks dari data di-*escape* dengan `h()` supaya aman.
- Layar analisis adalah animasi; perhitungan sebenarnya dilakukan server dalam satu permintaan.
