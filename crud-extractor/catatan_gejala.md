# Catatan Gejala

Catatan pribadi (ga usah di-commit). Isinya hal-hal yang kelihatan aneh waktu jalanin test, CLI, dan `--ast`. Kolom aturan spek diisi pakai nomor di `aturan_spek.md` (misal L17, E5).

## Gejala dari test (`python -m pytest`)

Hasil: 5 failed, 25 passed.

| Test gagal | Penyebab kasar | Jenis |
|---|---|---|
| `test_komentar_blok` | Lexer nolak `/* ... */` | TODO T3 |
| `test_insert_select` | Parser nolak `INSERT ... SELECT` | TODO T1 |
| `test_kolom_tanpa_qualifier_satu_tabel` | `resolved` masih `None` | TODO T2 |
| `test_keyword_huruf_kecil` | `select` kebaca IDENT, bukan KEYWORD | Bug |
| `test_matrix_laporan_penjualan` | Parse error di baris 6 kolom 41 | Bug |

## Gejala dari CLI (`python -m crud.cli data/sample/...`)

| No | Gejala | File / input pemicu | Aturan spek yang kayaknya dilanggar | Akar masalah (dugaan, belum pasti) |
|---|---|---|---|---|
| 1 | Parse error di `<>`, tanda `>` dibaca berdiri sendiri | `laporan_penjualan.sql` baris 6 |  | |
| 2 | Seluruh file gagal karena komentar blok | `pemeliharaan.sql` | L8 (TODO T3) | |
| 3 | Error sintaks baris 4 (disengaja), belum jelas apakah statement setelahnya tetap diproses | `campuran_error.sql` | E5, belum dicek | |
| 4 | Nol diagnostik, curiga ada alias yang lolos | `audit.sql` | | |
| 5 | | | | |
| 6 | | | | |

## Gejala dari `--ast`

| No | Gejala | File / input pemicu | Aturan spek yang kayaknya dilanggar | Akar masalah (dugaan, belum pasti) |
|---|---|---|---|---|
| 1 | | | | |
| 2 | | | | |

## Gejala dari `evaluate.py`

| No | Gejala | Catatan |
|---|---|---|
| 1 | | |

## Cek isi `crud_matrix.csv`

- Tabel yang muncul di baris `campuran_error.sql`:
- File yang tidak punya baris sama sekali:

## 3 aturan yang menurut gue paling gampang dilanggar

1.
2.
3.
