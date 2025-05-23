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

## B. Petunjuk Penggunaan
### 1. Buat direktori
```
mkdir relics mount_dir
```
### 2. Kompilasi program
```
gcc baymax.c `pkg-config fuse3 --cflags --libs` -o baymax_fs
```
### 3. Jalankan filesystem
```
./baymax_fs mount_dir
```
### 4. Uji Fungsi
- Gabung file:
```
cat mount_dir/Baymax.jpeg > out.jpg
```
- Lihat log:
```
cat activity.log
```
- Tulis file baru:
```
echo "Hello Baymax" > mount_dir/hero.txt
ls relics/hero.txt.*
```
- Hapus file:
```
rm mount_dir/hero.txt
```
- Unmount
```
fusermount3 -u mount_dir
```
## C. Kode Revisi (All)
```
#define FUSE_USE_VERSION 31

#include <fuse3/fuse.h>
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <fcntl.h>
#include <sys/stat.h>
#include <dirent.h>
#include <stdlib.h>
#include <unistd.h>
#include <time.h>

#define MAX_FRAG_SIZE 1024
#define MAX_FRAG_COUNT 1024

#define ROOT_DIR "."
#define RELICS_DIR "relics"
#define LOG_FILE "activity.log"
#define TMP_DIR "/tmp/fuse_frag_"

void tulis_log(const char *pesan) {
    FILE *log = fopen(LOG_FILE, "a");
    if (!log) return;

    time_t t = time(NULL);
    struct tm *tm = localtime(&t);
    char waktu[64];
    strftime(waktu, sizeof(waktu), "%Y-%m-%d %H:%M:%S", tm);

    fprintf(log, "[%s] %s\n", waktu, pesan);
    fclose(log);
}

static int dapetin_info(const char *path, struct stat *stbuf, struct fuse_file_info *fi) {
    memset(stbuf, 0, sizeof(struct stat));

    if (strcmp(path, "/") == 0) {
        stbuf->st_mode = S_IFDIR | 0755;
        stbuf->st_nlink = 2;
        return 0;
    }

    const char *base = path + 1;
    char fragpath[256];
    size_t total = 0;
    int found = 0;
    for (int i = 0; i < MAX_FRAG_COUNT; i++) {
        snprintf(fragpath, sizeof(fragpath), "%s/%s.%03d", RELICS_DIR, base, i);
        FILE *f = fopen(fragpath, "rb");
        if (!f) break;
        found = 1;
        fseek(f, 0, SEEK_END);
        total += ftell(f);
        fclose(f);
    }

    if (found) {
        stbuf->st_mode = S_IFREG | 0444;
        stbuf->st_nlink = 1;
        stbuf->st_size = total;
        return 0;
    }

    return -ENOENT;
}

static int tampilkan_isi_dir(const char *path, void *buf, fuse_fill_dir_t filler, off_t offset,
                              struct fuse_file_info *fi, enum fuse_readdir_flags flags) {
    filler(buf, ".", NULL, 0, 0);
    filler(buf, "..", NULL, 0, 0);

    DIR *dp = opendir(RELICS_DIR);
    if (!dp) return 0;
    struct dirent *de;
    char listed[1024][256];
    int listed_count = 0;

    while ((de = readdir(dp)) != NULL) {
        char *dot = strrchr(de->d_name, '.');
        if (dot && strlen(dot) == 4 && dot[1] >= '0' && dot[1] <= '9') {
            char name[256];
            strncpy(name, de->d_name, dot - de->d_name);
            name[dot - de->d_name] = '\0';

            int already = 0;
            for (int i = 0; i < listed_count; i++) {
                if (strcmp(listed[i], name) == 0) {
                    already = 1;
                    break;
                }
            }
            if (!already) {
                filler(buf, name, NULL, 0, 0);
                strncpy(listed[listed_count++], name, 256);
            }
        }
    }
    closedir(dp);
    return 0;
}

static int baca_dalem(const char *path, char *buf, size_t size, off_t offset, struct fuse_file_info *fi) {
    const char *base = path + 1;
    char fragpath[256];
    size_t total = 0;
    char *temp = malloc(MAX_FRAG_SIZE * MAX_FRAG_COUNT);
    if (!temp) return -ENOMEM;

    for (int i = 0;; i++) {
        snprintf(fragpath, sizeof(fragpath), "%s/%s.%03d", RELICS_DIR, base, i);
        FILE *f = fopen(fragpath, "rb");
        if (!f) break;
        size_t len = fread(temp + total, 1, MAX_FRAG_SIZE, f);
        total += len;
        fclose(f);
    }

    if (offset >= total) {
        free(temp);
        return 0;
    }

    if (offset + size > total) size = total - offset;
    memcpy(buf, temp + offset, size);
    free(temp);

    char logmsg[256];
    snprintf(logmsg, sizeof(logmsg), "READ: %s", base);
    tulis_log(logmsg);

    return size;
}

static int buka_file(const char *path, struct fuse_file_info *fi) {
    return 0;
}

static int buat_file(const char *path, mode_t mode, struct fuse_file_info *fi) {
    char tmp[256];
    snprintf(tmp, sizeof(tmp), "%s%s", TMP_DIR, path + 1);
    FILE *f = fopen(tmp, "wb");
    if (!f) return -EACCES;
    fclose(f);
    return 0;
}

static int tulis_file(const char *path, const char *buf, size_t size, off_t offset, struct fuse_file_info *fi) {
    char tmp[256];
    snprintf(tmp, sizeof(tmp), "%s%s", TMP_DIR, path + 1);
    FILE *f = fopen(tmp, "ab");
    if (!f) return -EIO;
    fwrite(buf, 1, size, f);
    fclose(f);
    return size;
}

static int lepas_file(const char *path, struct fuse_file_info *fi) {
    const char *base = path + 1;
    char tmp[256];
    snprintf(tmp, sizeof(tmp), "%s%s", TMP_DIR, base);
    FILE *f = fopen(tmp, "rb");
    if (!f) return 0;

    int frag = 0;
    char buf[MAX_FRAG_SIZE];
    size_t len;
    char logmsg[1024] = {0};
    snprintf(logmsg, sizeof(logmsg), "WRITE: %s -> ", base);

    while ((len = fread(buf, 1, MAX_FRAG_SIZE, f)) > 0) {
        char fragpath[256];
        snprintf(fragpath, sizeof(fragpath), "%s/%s.%03d", RELICS_DIR, base, frag);
        FILE *out = fopen(fragpath, "wb");
        if (!out) break;
        fwrite(buf, 1, len, out);
        fclose(out);

        char fragname[64];
        snprintf(fragname, sizeof(fragname), "%s.%03d", base, frag);
        strcat(logmsg, fragname);
        frag++;
        if (!feof(f)) strcat(logmsg, ", ");
    }

    tulis_log(logmsg);
    fclose(f);
    remove(tmp);
    return 0;
}

static int hapus_file(const char *path) {
    const char *base = path + 1;
    int i = 0;
    char fragpath[256];
    while (1) {
        snprintf(fragpath, sizeof(fragpath), "%s/%s.%03d", RELICS_DIR, base, i);
        if (access(fragpath, F_OK) != 0) break;
        remove(fragpath);
        i++;
    }
    if (i > 0) {
        char logmsg[256];
        snprintf(logmsg, sizeof(logmsg), "DELETE: %s.000 - %s.%03d", base, base, i - 1);
        tulis_log(logmsg);
    }
    return 0;
}

static const struct fuse_operations ops = {
    .getattr    = dapetin_info,
    .readdir    = tampilkan_isi_dir,
    .open       = buka_file,
    .create     = buat_file,
    .read       = baca_dalem,
    .write      = tulis_file,
    .release    = lepas_file,
    .unlink     = hapus_file,
};

int main(int argc, char *argv[]) {
    return fuse_main(argc, argv, &ops, NULL);
}
```
