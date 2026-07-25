# Novel Rekomendasi Harian

Digest email otomatis berisi peringkat 10 novel teratas dari empat situs berbeda, terkirim tiap pagi. Dibangun di atas n8n.

---

## Problem

Saya membaca web novel setiap hari, dan tiap pagi selalu mengulang ritual yang sama: buka satu per satu situs peringkat, bandingkan mana yang naik, tebak mana yang layak dibaca. Repetitif, memakan waktu, dan yang paling mengganggu — saya sering membaca daftar yang sama persis dengan kemarin tanpa sadar, karena peringkat jarang berubah drastis.

Tiga masalah yang ingin saya selesaikan:

1. **Repetitif.** Empat situs dibuka manual setiap hari.
2. **Tidak ada memori.** Tidak ada cara tahu mana yang benar-benar judul baru versus yang sudah nangkring seminggu.
3. **Bahasa dan genre.** Sebagian besar situs berbahasa Inggris, dan ada genre yang selalu saya filter tapi harus dilakukan manual di tiap situs.

---

## Solution

Satu workflow n8n terjadwal yang mengambil, menyaring, membandingkan, menerjemahkan, lalu mengirimkan hasilnya.

```
Schedule 07:00 WITA
        │
        ├─→ WTR Lab        (3 halaman, JSON dari __NEXT_DATA__)
        ├─→ Royal Road     (HTML parsing)
        ├─→ NovelUpdates   (via proxy scrape.do — diblokir Cloudflare)
        └─→ MeioNovel      (3 halaman + fetch detail per novel)
                │
                ▼
        Gabung & saring genre
                │
                ▼
        Gemini 2.5 Flash ── terjemahkan + ringkas sinopsis
                │
                ▼
        Baca riwayat (Google Sheets) ── hitung BARU / naik / turun
                │
                ├─→ Email digest HTML (Gmail)
                └─→ Simpan snapshot hari ini (Google Sheets)
```

### Keputusan teknis

**Riset sumber sebelum menulis satu baris kode.** Saya probe sebelas situs lebih dulu untuk memetakan mana yang bisa diakses, sebelum memutuskan arsitektur. Hasilnya mengubah rencana awal secara signifikan.

| Situs | Status | Keputusan |
|---|---|---|
| wtr-lab.com | Tembus langsung | Dipakai — data JSON terstruktur di `__NEXT_DATA__` |
| royalroad.com | Tembus langsung | Dipakai — HTML bersih dan konsisten |
| novelupdates.com | Cloudflare 403 | Dipakai lewat proxy — genre paling lengkap, sepadan dengan biayanya |
| meionovel.id | Tembus langsung | Dipakai — konten sudah berbahasa Indonesia |
| webnovel.com | Cloudflare 403 | Dibuang — lihat Batasan |
| scribblehub, ranobes, lightnovelworld, novelfull, sakuranovel, novelgo | 403 / mati | Tidak dipakai |

**Memilih sumber yang sudah menyelesaikan masalah, bukan menambah lapisan untuk menyelesaikannya.** meionovel.id menyajikan judul, genre, dan sinopsis yang sudah berbahasa Indonesia. Untuk sumber ini tidak ada terjemahan sama sekali — lebih akurat sekaligus lebih murah daripada menerjemahkan hasil bahasa Inggris.

**Menyaring pada tag bahasa Inggris, menampilkan dalam bahasa Indonesia.** Filter genre bekerja pada nama asli dari situs (`Harem`, `Yaoi`, `Shounen Ai`) karena pencocokannya andal. Yang tampil ke pembaca sudah diterjemahkan lewat kamus tetap, bukan mesin penerjemah, supaya nama genre tidak pernah meleset.

**Mengambil lebih banyak dari yang dibutuhkan.** Filter genre membuang sekitar 40–57% kandidat, jadi tiap sumber mengambil 2–3 kali lipat dari target. Hasil akhirnya tetap genap sepuluh, bukan sisa seadanya.

**Degradasi bertahap, bukan gagal total.** Setiap sumber dan panggilan LLM diset `continueRegularOutput`. Kalau proxy habis kuota atau Gemini down, email tetap terkirim dengan sumber yang berhasil, ditambah kotak peringatan yang menyebut sumber mana yang tidak terbaca hari ini.

---

## Result

Berjalan otomatis tiap pagi 07:00 WITA. Satu eksekusi ~67 detik.

**40 judul per hari** — sepuluh dari masing-masing empat sumber, sudah tersaring genre.

Isi email:

- **Pilihan Hari Ini** — satu judul dengan rating tertinggi di antara pendatang baru
- **Badge BARU** untuk judul yang belum pernah masuk top 10
- **▲ / ▼** perubahan peringkat dibanding hari sebelumnya
- **"hari ke-N"** untuk judul yang bertahan lama, supaya ketahuan mana yang cuma lewat
- **Chip genre** berbahasa Indonesia
- **Sinopsis** 2–3 kalimat bahasa Indonesia hasil ringkasan Gemini
- Rating, jumlah bab, pembaca, dan tautan langsung

Masalah "daftar yang sama tiap hari" selesai secara terukur, terverifikasi lewat dua eksekusi berturut-turut: run pertama menandai semua 20 judul sebagai BARU, run kedua menandai 0 baru dan seluruhnya berubah jadi "hari ke-2".

Filter genre juga terbukti bekerja, bukan sekadar terpasang. Peringkat #1 NovelUpdates saat pengujian adalah judul bergenre Harem — hilang dari hasil, digantikan judul berikutnya. Di MeioNovel, 8 dari 14 kandidat tersaring.

**Biaya operasional: nol.** Semuanya berjalan di kuota gratis:

| Layanan | Kuota | Pemakaian |
|---|---|---|
| scrape.do | 1.000 kredit/bulan | ~300 |
| Gemini 2.5 Flash | free tier | 1 panggilan batch/hari |
| Google Sheets & Gmail | — | dalam batas normal |

---

## Cara kerjanya

### Riwayat dan perbandingan

Google Sheets menyimpan satu baris per novel per hari: `run_date, source, rank, key, title, url, rating, views, chapters, genres`.

Tiap kali berjalan, workflow membaca seluruh riwayat, mencari tanggal terakhir sebelum hari ini, lalu membandingkan peringkat. Kolom `key` (`sumber|id`) yang membuat pencocokan tetap akurat meski judul berubah.

Zona waktu berpengaruh di sini: `run_date` dihitung pada `Asia/Makassar`. Zona waktu yang salah membuat eksekusi dini hari tercatat sebagai tanggal kemarin dan merusak perhitungan BARU.

### Terjemahan

Seluruh sinopsis dikirim dalam satu panggilan batch ke Gemini, bukan satu panggilan per novel. Instruksinya tegas: 2–3 kalimat utuh, dilarang berhenti di tengah kalimat, dilarang memakai elipsis, dan pertahankan istilah khas seperti *kultivasi*, *xianxia*, *LitRPG*.

### Penanganan error

Workflow terpisah `Error Handler - Universal` terpasang sebagai error workflow. Saat ada kegagalan, ia mengirim email berisi nama workflow, node yang gagal, pesan error, dan tautan langsung ke halaman eksekusi — tanpa perlu membuka n8n untuk tahu ada yang rusak.

---

## Batasan yang diketahui

**Filter genre WTR Lab tidak sepenuhnya presisi.** Situs ini menyimpan genre sebagai ID angka dan tidak memublikasikan peta angka-ke-nama di mana pun — saya telusuri lewat halaman novel, novel-finder, sitemap, locale alternatif, dan 47 berkas JS-nya. Filter untuk sumber ini berjalan lewat *tag* (Reverse Harem, Slave Harem, Shounen-Ai Subplot, dan sejenisnya). Novel yang hanya berlabel genre pokok "Harem" tanpa tag terkait masih bisa lolos.

**Webnovel dibuang, bukan gagal dipasang.** Halaman peringkatnya hanya memuat satu kategori luas (`Urban`, `Fantasy`), tanpa genre detail dan tanpa sinopsis. Filter sungguhan menuntut pengambilan halaman tiap buku lewat proxy berbayar — sekitar 3.000 kredit/bulan melawan kuota gratis 1.000. Nilainya tidak sepadan.

**Sinopsis NovelUpdates tidak tersedia** di halaman peringkat, dan menampilkannya butuh permintaan proxy tambahan per novel.

**Tiga dari 40 entri masih menampilkan mojibake** (`Countâ€™s` alih-alih `Count's`) pada judul dari sumber tertentu. Tiga pendekatan perbaikan sudah dicoba dan gagal; akar masalahnya belum ditemukan. Dampaknya kosmetik — judul tetap terbaca dan tautannya berfungsi.

---

## Menjalankan sendiri

Prasyarat: instance n8n, akun Google, dan dua API key gratis.

1. Buat spreadsheet riwayat dengan header:
   `run_date, source, rank, key, title, url, rating, views, chapters, genres`
2. Siapkan kredensial n8n — Gmail (OAuth2), Google Sheets (OAuth2), dan dua **Query Auth**:

   | Credential | Nama parameter | Sumber |
   |---|---|---|
   | Scrape.do | `token` | [scrape.do](https://scrape.do) — 1.000 kredit/bulan gratis |
   | Gemini | `key` | [Google AI Studio](https://aistudio.google.com/apikey) |

3. Impor [`novel-digest.sanitized.json`](novel-digest.sanitized.json) lewat **Workflows → Import from File**
4. Cari `GANTI_` di dalam workflow dan isi dengan nilai sendiri
5. Set timezone workflow — n8n default mengikuti UTC
6. Publish, lalu pasang error workflow

API key tidak disimpan di dalam node mana pun. Semuanya lewat sistem credential n8n, sehingga tidak ikut terbawa saat workflow diekspor. Berkas `novel-digest.sanitized.json` di repo ini sudah dibersihkan dari ID spreadsheet, ID kredensial, dan alamat email.

---

## Tumpukan teknologi

n8n · Google Gemini 2.5 Flash · Google Sheets · Gmail API · scrape.do

## Lisensi

[MIT](LICENSE) — © 2026 Agung Tri Mahmudi
