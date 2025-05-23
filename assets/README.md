# Sisop-4-2025-IT32
- Ananda Fitri Wibowo (5027241057)
- Raihan Fahri Ghazali (5027241061)

| No | Nama                   | NRP         |
|----|------------------------|-------------|
| 1  | Ananda Fitri Wibowo    | 5027241057  |
| 2  | Raihan Fahri Ghazali   | 5027241061  |

# Soal 2
Solved by. 057_Ananda Fitri Wibowo
## A. Virtual Filesystem (FUSE)
Filesystem ini berfungsi untuk menyatukan pecahan-pecahan file menjadi satu file utuh secara virtual, sekaligus memecah file baru yang dibuat ke dalam potongan 1 KB saat ditulis. Sistem juga mencatat semua aktivitas ke dalam 'activity.log'.
### Define
- Pecahan file disimpan di folder relics/ dalam format .000, .001, dst.
- Folder mount_dir/ digunakan untuk mount virtual filesystem.
- Log aktivitas dicatat di activity.log.
```
#define MAX_FRAG_SIZE 1024
#define MAX_FRAG_COUNT 1024
#define ROOT_DIR "."
#define RELICS_DIR "relics"
#define LOG_FILE "activity.log"
#define TMP_DIR "/tmp/fuse_frag_"
```
### Void tulis_log
Untuk mencatat aktivitas ke activity.log seperti READ, WRITE, DELETE dalam format timestamp.
```
void tulis_log(const char *pesan) {
    FILE *log = fopen(LOG_FILE, "a");
    ...
}
```
### FUSE getattr: dapetin_info
Mengembalikan info file atau direktori yang diminta:
- Jika direktori /, maka di-set sebagai folder.
- Jika file ditemukan pecahannya di relics/, maka dijumlahkan ukurannya dan dikembalikan info file biasa.
```
static int dapetin_info(const char *path, struct stat *stbuf, ...)
```
### FUSE readdir: tampilkan_isi_dir
Menampilkan isi mount dir (hanya file virtual):
- Membaca semua pecahan di relics/
- Menampilkan nama file utuh tanpa duplikasi (misal Baymax.jpeg saja, meskipun ada .000 sampai .013).
```
static int tampilkan_isi_dir(const char *path, void *buf, ...)
```
### FUSE read: baca_dalem
Membaca file dari pecahan:
- Menggabungkan seluruh fragmen [namafile].000, .001, dst dari relics/
- Mengisi buffer sesuai permintaan offset dan size
- Menulis log READ: [namafile]
```
static int baca_dalem(const char *path, char *buf, ...)
```
### FUSE create: buat_file
Membuat file baru di folder temp /tmp/fuse_frag_[namafile] untuk ditulis secara sementara.
```
static int buat_file(const char *path, mode_t mode, ...)
```
### FUSE write: tulis_file
Menulis isi ke file sementara, belum langsung dipecah saat write, hanya ditambahkan ke file tmp.
```
static int tulis_file(const char *path, const char *buf, ...)
```
### FUSE release: lepas_file
Saat file selesai ditulis (ditutup), isinya akan:
- Dibaca dari file sementara
- Dipecah ke file 1 KB .000, .001, dst di relics/
- Ditulis log WRITE: [namafile] -> namafile.000, namafile.001, ...
```
static int lepas_file(const char *path, ...)
```
### FUSE unlink: hapus_file
Menghapus file:
- Menghapus semua pecahan [namafile].000 hingga akhir
- Mencatat aktivitas DELETE: ... ke log
```
static int hapus_file(const char *path)
```
### FUSE ops & main
Mendaftarkan semua operasi ke FUSE dan menjalankan filesystem mount-nya.
```
static const struct fuse_operations ops = { ... };
int main(int argc, char *argv[]) {
    return fuse_main(argc, argv, &ops, NULL);
}
```
