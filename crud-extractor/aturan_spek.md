# Aturan Spek yang Bisa Diuji (SPESIFIKASI.md)

Catatan pribadi buat nyocokin kode dengan spek. Spek = kebenaran mutlak. Kalau kode atau komentar beda dari spek, spek yang menang.

Cara pakai: tiap baris = satu aturan yang bisa dites. Kolom **Status** diisi pas ngecek kode (✔ sesuai, ✘ bug, ? belum dicek).

---

## Bagian 2: Lexer

| No | Bagian | Aturan | Contoh SQL | Hasil yang diharapkan | Status |
|---|---|---|---|---|---|
| L1 | 2.1 | Tiap token nyimpen baris dan kolom awal, mulai dari 1 | `SELECT a` | token `SELECT` di baris 1 kolom 1 | ? |
| L2 | 2.1 | Posisi tetap benar setelah newline | `SELECT a\n  FROM t` | `FROM` di baris 2 kolom 3 | ? |
| L3 | 2.2 | Keyword tidak peka huruf besar/kecil | `select`, `Select`, `SELECT` | ketiganya KEYWORD | ? |
| L4 | 2.2 | Nilai token keyword selalu huruf besar | `select` | KEYWORD bernilai `SELECT` | ? |
| L5 | 2.2 | Kata yang cuma diawali keyword tetap IDENT | `selection`, `fromage` | IDENT, bukan KEYWORD | ? |
| L6 | 2.2 | Quoted identifier di luar lingkup | `"nama"`, `[nama]` | tidak perlu didukung | ? |
| L7 | 2.3 | Komentar `--` sampai akhir baris, tidak jadi token | `-- hapus\nSELECT` | token pertama `SELECT` di baris 2 | ? |
| L8 | 2.3 | Komentar blok `/* ... */` boleh lintas baris | `/* a\n b */ SELECT` | token pertama `SELECT` di baris 2 | ? |
| L9 | 2.3 | Komentar blok boleh berisi `*` | `/* a * b */ SELECT` | komentar utuh dibuang | ? |
| L10 | 2.3 | Komentar blok tidak bersarang | `/* a /* b */ c */` | berhenti di `*/` pertama (bersarang di luar lingkup) | ? |
| L11 | 2.3 | Komentar blok tidak ditutup = LexError | `/* tidak ditutup` | LexError | ? |
| L12 | 2.4 | String diapit kutip tunggal, nilainya tanpa kutip | `'LUNAS'` | STRING bernilai `LUNAS` | ? |
| L13 | 2.4 | Dua kutip `''` di dalam string = satu karakter kutip | `'O''Brien'` | satu STRING bernilai `O'Brien` | ? |
| L14 | 2.4 | String tidak ditutup = LexError | `'abc` | LexError | ? |
| L15 | 2.5 | Operator: `= <> != < <= > >= + - /` jadi token OP | `a >= 1` | OP bernilai `>=` | ? |
| L16 | 2.5 | `*` jadi token STAR, bukan OP | `*` | STAR | ? |
| L17 | 2.5 | Maximal munch: ambil operator terpanjang yang cocok | `a<>b`, `a<=b`, `a!=b` | masing-masing satu token | ? |
| L18 | 2.6 | Parameter `?` atau `:nama` (huruf, angka, garis bawah) | `? :id :batas_awal` | tiga PARAM | ? |
| L19 | 2.6 | Angka bulat atau desimal | `10`, `2.5` | NUMBER bernilai `10` dan `2.5` | ? |
| L20 | 2.7 | Karakter lain = LexError dengan baris dan kolom | `SELECT\n$` | LexError di baris 2 | ? |
| L21 | 2.7 | LexError menghentikan pemrosesan file itu | file berisi `$` | `lex_error` terisi, tidak ada statement | ? |

---

## Bagian 3: Grammar dan Parser

### 3.1 Presedensi (terendah ke tertinggi)

| Tingkat | Operator | Asosiatif |
|---|---|---|
| 1 | `OR` | kiri |
| 2 | `AND` | kiri |
| 3 | `NOT` (prefiks) | kanan |
| 4 | perbandingan, `IN`, `LIKE`, `BETWEEN`, `IS NULL` | tidak berantai |
| 5 | `+`, `-` | kiri |
| 6 | `*`, `/` | kiri |
| 7 | `-` (prefiks) | kanan |

### Aturan yang bisa dites

| No | Bagian | Aturan | Contoh SQL | Hasil yang diharapkan | Status |
|---|---|---|---|---|---|
| P1 | 3.1 | `AND` lebih kuat dari `OR` | `a = 1 OR b = 2 AND c = 3` | `OR(a=1, AND(b=2, c=3))` | ? |
| P2 | 3.1 | `AND` dan `OR` asosiatif kiri | `a = 1 AND b = 2 AND c = 3` | `AND(AND(a=1, b=2), c=3)` | ? |
| P3 | 3.1 | `NOT` lebih kuat dari `AND`, lebih lemah dari perbandingan | `NOT a = 1 AND b = 2` | `AND(NOT(a=1), b=2)` | ? |
| P4 | 3.1 | `NOT` prefiks asosiatif kanan | `NOT NOT a = 1` | `NOT(NOT(a=1))` | ? |
| P5 | 3.1 | Perbandingan tidak berantai | `a = 1 = 2` | error sintaks (atau tidak dibaca sebagai rantai) | ? |
| P6 | 3.1 | `+` dan `-` asosiatif kiri | `a - b - c` | `(a - b) - c` | ? |
| P7 | 3.1 | `*` dan `/` lebih kuat dari `+` `-` | `a + b * c` | `a + (b * c)` | ? |
| P8 | 3.1 | Minus prefiks paling kuat | `-a * b` | `(-a) * b` | ? |
| P9 | 3.2 | `*` di awal select_list = semua kolom | `SELECT * FROM t` | `Star` | ? |
| P10 | 3.2 | `*` di dalam ekspresi = perkalian | `SELECT harga * qty FROM t` | `BinaryOp` dengan op `*` | ? |
| P11 | 3.2 | `(` diikuti SELECT = subquery (di FROM dan di primary) | `FROM (SELECT ...) x`, `WHERE a = (SELECT ...)` | `DerivedTable` / `Subquery` | ? |
| P12 | 3.2 | `BETWEEN a AND b`: `AND` milik BETWEEN | `x BETWEEN 1 AND 5 AND y = 2` | `AND(Between(x,1,5), y=2)` | ? |
| P13 | 3 | `NOT IN`, `NOT LIKE`, `NOT BETWEEN` | `a NOT IN (1,2)` | `InExpr` negated | ? |
| P14 | 3 | `IS NULL` dan `IS NOT NULL` | `a IS NOT NULL` | `IsNull` negated | ? |
| P15 | 3 | `IN` bisa berisi daftar atau SELECT | `IN (1,2)`, `IN (SELECT ...)` | `values` atau `query` terisi | ? |
| P16 | 3 | `EXISTS ( select_stmt )` | `EXISTS (SELECT 1 FROM t)` | `Exists` | ? |
| P17 | 3 | JOIN: `JOIN`, `INNER JOIN`, `LEFT JOIN`, `LEFT OUTER JOIN` | `a LEFT OUTER JOIN b ON ...` | kind `LEFT` | ? |
| P18 | 3 | JOIN tanpa kata depan = INNER | `a JOIN b ON ...` | kind `INNER` | ? |
| P19 | 3 | Alias dengan atau tanpa `AS` (kolom dan tabel) | `FROM pelanggan AS p`, `FROM pesanan o` | alias `p` dan `o` | ? |
| P20 | 3 | `FROM` boleh beberapa tabel dipisah koma | `FROM a, b` | dua sumber | ? |
| P21 | 3 | `ORDER BY` dengan `ASC`/`DESC`, default `ASC` | `ORDER BY a DESC, b` | `DESC`, `ASC` | ? |
| P22 | 3 | `GROUP BY` berisi daftar ekspresi | `GROUP BY a, b` | dua ekspresi | ? |
| P23 | 3 | `SELECT DISTINCT` | `SELECT DISTINCT a FROM t` | `distinct=True` | ? |
| P24 | 3 | Fungsi: `COUNT(*)`, `COUNT(DISTINCT x)`, `SUM(a)` | `COUNT(*)` | `FunctionCall` star=True, nama huruf besar | ? |
| P25 | 3 | Nama tabel boleh berskema dan dinormalisasi huruf kecil | `CRM.Pelanggan` | `crm.pelanggan` | ? |
| P26 | 3 | INSERT: daftar kolom opsional | `INSERT INTO t VALUES (1)` | `columns = []` | ? |
| P27 | 3 | INSERT ... VALUES boleh banyak baris | `VALUES (1,'a'), (2,'b')` | 2 baris | ? |
| P28 | 3 | INSERT ... SELECT didukung (T1) | `INSERT INTO a (id) SELECT id FROM b` | `Insert.query` berisi `Select` | ? |
| P29 | 3 | UPDATE: alias opsional, `SET` banyak assignment, `WHERE` opsional | `UPDATE t AS x SET a = 1, b = 2` | alias `x`, 2 assignment | ? |
| P30 | 3 | DELETE: alias opsional, `WHERE` opsional | `DELETE FROM sesi s WHERE ...` | alias `s` | ? |
| P31 | 3 | `;` kosong diabaikan, statement terakhir boleh tanpa `;` | `;; SELECT a FROM t` | 1 statement, tanpa error | ? |
| P32 | 3 | Nama kolom boleh berkualifikasi | `p.nama` | `qualifier=p, name=nama` (huruf kecil) | ? |

---

## Bagian 4: Resolver

| No | Bagian | Aturan | Contoh SQL | Hasil yang diharapkan | Status |
|---|---|---|---|---|---|
| R1 | 4.1 | Scope baru untuk tiap SELECT (termasuk subquery), UPDATE, DELETE | `SELECT ... WHERE x IN (SELECT ...)` | dua scope, yang dalam punya induk | ? |
| R2 | 4.1 | INSERT tidak bikin scope; SELECT di dalamnya bikin scope sendiri | `INSERT INTO a SELECT ...` | scope hanya dari SELECT | ? |
| R3 | 4.2 | Nama di scope = alias, atau bagian terakhir nama tabel kalau tanpa alias | `FROM crm.pelanggan` | nama scope `pelanggan` | ? |
| R4 | 4.2 | Subquery di FROM masuk dengan aliasnya; kolomnya resolve ke `<subquery alias>` | `FROM (SELECT ...) x` lalu `x.id` | `resolved = "<subquery x>"` | ? |
| R5 | 4.2 | Subquery di FROM tanpa alias = warning | `FROM (SELECT ...)` | 1 warning | ? |
| R6 | 4.2 | Nama dobel di scope yang sama = error | `FROM a p, b p` | 1 error | ? |
| R7 | 4.3 | Kolom berkualifikasi dicari dari scope terdalam ke luar | `SELECT p.nama FROM pelanggan p` | `resolved = "pelanggan"` | ? |
| R8 | 4.3 | Shadowing: alias di subquery menutupi alias luar yang sama | `... x ... IN (SELECT x.id FROM pesanan x)` | `x` di dalam resolve ke `pesanan` | ? |
| R9 | 4.3 | Correlated subquery: subquery bisa merujuk alias luar | `EXISTS (SELECT 1 FROM b WHERE b.id = p.id)` | `p` resolve ke tabel luar | ? |
| R10 | 4.3 | Qualifier tidak ditemukan di mana pun = error `alias 'x' tidak terdefinisi` | `SELECT q.nama FROM pelanggan p` | 1 error | ? |
| R11 | 4.3 | Tanpa kualifikasi: hanya scope **terdalam** yang dilihat | `SELECT nama FROM pelanggan` | `resolved = "pelanggan"` (T2) | ? |
| R12 | 4.3 | Tanpa kualifikasi, lebih dari satu tabel = warning `kolom 'nama' ambigu tanpa skema`, `resolved` tetap `None` | `SELECT nama FROM a, b` | 1 warning, `resolved=None` | ? |
| R13 | 4.3 | Tanpa kualifikasi, tidak ada tabel = warning | kolom tanpa tabel di scope | 1 warning | ? |
| R14 | 4.3 | Hasil disimpan di `ColumnRef.resolved` (nama tabel lengkap dengan skema) | `crm.pelanggan c` lalu `c.id` | `resolved = "crm.pelanggan"` | ? |
| R15 | 4.4 | Subquery di FROM di-resolve sebelum scope SELECT pembungkusnya | `FROM a, (SELECT ... ) x` | subquery tidak lihat tabel `a`, tapi lihat scope luar | ? |
| R16 | 4.5 | Tiap statement berdiri sendiri, tidak ada scope tersisa | `DELETE FROM sesi s ...; SELECT s.id FROM pengguna u ...` | `s` di statement kedua = error tidak terdefinisi | ? |
| R17 | 4.6 | Diagnostik berbentuk `Diagnostic(level, message, line)`, level `"error"` atau `"warning"` | | dua nilai level itu saja | ? |

---

## Bagian 5: Aturan CRUD

| Statement | Tabel target | Tabel lain (subquery, sumber SELECT) |
|---|---|---|
| SELECT | semua tabel di FROM/JOIN: R | R |
| INSERT ... VALUES | C | R |
| INSERT ... SELECT | C | tabel sumber: R |
| UPDATE | U | R (subquery di SET maupun WHERE) |
| DELETE | D | R |

| No | Bagian | Aturan | Contoh SQL | Hasil yang diharapkan | Status |
|---|---|---|---|---|---|
| C1 | 5 | SELECT: semua tabel FROM/JOIN = R | `SELECT ... FROM a JOIN b ON ...` | `{a: R, b: R}` | ? |
| C2 | 5 | INSERT ... VALUES: target = C | `INSERT INTO log (id) VALUES (1)` | `{log: C}` | ? |
| C3 | 5 | INSERT ... SELECT: target C, sumber R | `INSERT INTO arsip SELECT ... FROM pesanan` | `{arsip: C, pesanan: R}` | ? |
| C4 | 5 | UPDATE: target = U | `UPDATE produk SET stok = stok - 1` | `{produk: U}` | ? |
| C5 | 5 | Tabel di subquery **selalu R**, apa pun statement luarnya | `UPDATE pesanan SET ... WHERE id IN (SELECT id FROM pelanggan)` | `{pesanan: U, pelanggan: R}` | ? |
| C6 | 5 | Subquery di bagian SET pada UPDATE juga R | `UPDATE a SET x = (SELECT y FROM b)` | `{a: U, b: R}` | ? |
| C7 | 5 | DELETE: target = D, subquery = R | `DELETE FROM alamat WHERE id NOT IN (SELECT id FROM pelanggan)` | `{alamat: D, pelanggan: R}` | ? |
| C8 | 5 | Nama tabel huruf kecil lengkap dengan skema | `CRM.Pelanggan` | `crm.pelanggan` | ? |
| C9 | 5 | Gabungan operasi selalu urut C, R, U, D | tabel yang kena U dan R | `RU`, bukan `UR` | ? |
| C10 | 5 | Statement yang gagal di-parse tidak dihitung | file dengan 1 statement rusak | tabelnya tidak muncul | ? |
| C11 | 1 | Hasil per file = gabungan semua statement | `laporan_penjualan.sql` | `arsip_pesanan: C`, `pelanggan: RU`, `pesanan: RD` | ? |

---

## Bagian 6: Error sintaks

| No | Bagian | Aturan | Contoh SQL | Hasil yang diharapkan | Status |
|---|---|---|---|---|---|
| E1 | 6 | Format pesan: `Baris L, kolom K: diharapkan X, ditemukan Y` | `SELECT id\nFROM\n;` | pesan menunjuk baris 3 | ? |
| E2 | 6 | Posisi error = token tempat error terdeteksi | | `ParseError.line` dan `.col` sesuai token itu | ? |
| E3 | 6 | Panic mode: buang token sampai **dan termasuk** `;` berikutnya | `SELECT id nama, FROM pesanan; UPDATE ...` | parsing lanjut dari statement setelah `;` | ? |
| E4 | 6 | Tiap statement rusak = **tepat satu** error | 1 statement rusak | `len(errors) == 1` | ? |
| E5 | 6 | Statement valid setelah error tetap diproses | `campuran_error.sql` | `produk: U` dan `keranjang: D` tetap muncul | ? |

---

## Bagian 7: API publik dan batasan (jangan diubah)

- `crud.lexer`: `tokenize`, `Token`, `TokenType`, `LexError`, `KEYWORDS`
- `crud.parser`: `parse_sql`, `Parser`, `ParseError` (atribut `line`, `col`)
- `crud.ast_nodes`: semua kelas node dan nama field-nya, `iter_nodes`, `pretty`
- `crud.resolver`: `Resolver().resolve_script(script)` mengembalikan list `Diagnostic`
- `crud.analyzer`: `analyze_statement(stmt)` mengembalikan `{tabel: {operasi}}`, `column_usage(stmt)`
- `crud.pipeline`: `analyze_source(text)` mengembalikan `FileResult` (`statements`, `parse_errors`, `diagnostics`, `lex_error`, `matrix()`)

Batasan arsitektur:
- Urutan pipeline tetap: lexer, parser, resolver, analyzer.
- Resolver = pass terpisah setelah parsing. Parser tidak boleh resolusi.
- Analyzer kerja di atas AST, tidak baca teks SQL mentah dan tidak pakai regex.
- Tidak ada skema database. Semua keputusan dari teks query.
- `baseline/regex_extractor.py` tidak boleh diubah.

---

## Bagian 8: Format keluaran

| No | Aturan | Status |
|---|---|---|
| O1 | `crud_matrix.csv`: `file,<tabel1>,<tabel2>,...`, satu baris per file | ? |
| O2 | `--detail`: `file,baris,jenis,tabel,operasi`, satu baris per (statement, tabel). `baris` = baris keyword awal statement | ? |
| O3 | `--kolom`: `file,baris,tabel,kolom,operasi` | ? |
| O4 | `data/ground_truth.csv`: `file,baris,tabel,operasi`, **satu huruf operasi per baris** (`RU` ditulis dua baris) | ? |

---

## Bagian 9: Di luar lingkup (ga perlu didukung)

CTE (`WITH`), `UNION`, `MERGE`, DDL, quoted identifier, `t.*`, window function, alias kolom di `ORDER BY`, komentar bersarang, stored procedure, trigger, view.

---

## Isi sendiri: 3 aturan yang menurut gue paling gampang dilanggar kode

1.
2.
3.

## Hal yang belum gue ngerti dari spek (tanyain nanti)

-
