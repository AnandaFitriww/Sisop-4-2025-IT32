# Sisop-4-2025-IT32
- Ananda Fitri Wibowo (5027241057)
- Raihan Fahri Ghazali (5027241061)

| No | Nama                   | NRP         |
|----|------------------------|-------------|
| 1  | Ananda Fitri Wibowo    | 5027241057  |
| 2  | Raihan Fahri Ghazali   | 5027241061  |

# Soal 1
Solverd by. 061_Raihan Fahri Ghazali

## A. Unduh dan Ekstrasi File ZIP Anomali 
Pada modul kali ini kita diharuskan untuk menggunakan FUSE dan harus :
- Mendownload file
- zip file
- unzip file

### Download Zip  File
Fungsi ini digunakan untuk mengunduh file dari link Google Drive yang telah disediakan, dengan menggunakan perintah "wget" yang dijalankan melalui shell.


void download_zip() {
    char command[512];
    snprintf(command, sizeof(command), "wget \"%s\" -O %s", ZIP_URL, ZIP_NAME);
    system(command);
}


### Ekstrak Zip File
Setelah file sudah ter download. Seluruh file beserta semua isinya di ekstrak dan menghasilkan satu file zip terbaru. 

void extract_zip() {
    char command[256];
    snprintf(command, sizeof(command), "unzip -o -q %s -d .", ZIP_NAME);
    system(command);
}


### Hapus zip File 
Bagian ini digunakan untuk menghapus file Zip setelah proses ektraksi selesai, sehingga hanya menyisakan hasil ekstraknya saja.

void delete_zip() {
    remove(ZIP_NAME);
}


Semua fungsi ini dipanggil di dalam fungsi main

if (access("anomali/1.txt", F_OK) != 0) {
    download_zip();
    extract_zip();
    delete_zip();
}


## B. Detect format and conver Hexadecimal to Image
Sekarang kita diharuskan mengubah format teks awal yang awalnya adalah String Hexadecimal dan mengubahnya menjadi file image. 


### Function string Hex to Image 
Fungsi ini awalannya membuka file .txt berisi string hex lalu mengambil 2 karakter sekaligus.
Mengubah 2 karakter hexadesimal menjadi 1 byte dan menghasilkan file output PNG. Lalu mengakhirinya dengan menutup file .txt 

void convert_hex_to_image(const char *abs_input_path, const char *filename) {
    FILE *in = fopen(abs_input_path, "r");
    if (!in) return;

    create_directory(IMAGE_DIR);

    time_t raw_time = time(NULL);
    struct tm *timeinfo = localtime(&raw_time);

    char tanggal[16], waktu_file[16], waktu_log[16];
    strftime(tanggal, sizeof(tanggal), "%Y-%m-%d", timeinfo);
    strftime(waktu_file, sizeof(waktu_file), "%H-%M-%S", timeinfo); // untuk filename
    strftime(waktu_log, sizeof(waktu_log), "%H:%M:%S", timeinfo);   // untuk log

    char base_name[128];
    strncpy(base_name, filename, sizeof(base_name));
    char *dot = strrchr(base_name, '.');
    if (dot) *dot = '\0';

    char output_file[512];
    snprintf(output_file, sizeof(output_file), "%s/%s_image_%s_%s.png", IMAGE_DIR, base_name, tanggal, waktu_file);

    FILE *out = fopen(output_file, "wb");
    if (!out) {
        fclose(in);
        return;
    }

    int c1, c2;
    while ((c1 = fgetc(in)) != EOF && (c2 = fgetc(in)) != EOF) {
        if (!isxdigit(c1) || !isxdigit(c2)) continue;
        unsigned char byte = parse_byte((char)c1, (char)c2);
        fwrite(&byte, sizeof(unsigned char), 1, out);
    }

    fclose(in);
    fclose(out);

    FILE *log = fopen(LOG_FILE_PATH, "a");
    if (log) {
        fprintf(log, "[%s][%s]: Successfully converted hexadecimal text %s to %s.\n",
                tanggal, waktu_log, filename, strrchr(output_file, '/') + 1);
        fclose(log);
    }
}

Fuse akan memanggil "fuse_open()" saat pengguna membuka file di mount point

static int fuse_open(const char *path, struct fuse_file_info *fi) {
    char full_path[512];
    snprintf(full_path, sizeof(full_path), "%s%s", source_dir, path);

    int res = open(full_path, fi->flags);
    if (res == -1) return -errno;
    close(res);

    // ✅ Inilah bagian yang memicu konversi hexadecimal → image
    const char *filename = strrchr(path, '/') ? strrchr(path, '/') + 1 : path;
    if (strstr(filename, ".txt")) {
        convert_hex_to_image(full_path, filename);  // ← dipanggil di sini!
    }

    return 0;
}


## C. Naming the Converted Image File 
Bagian ini mengatur penamaan file hasil konversi dari string hexadecimal menjadi gambar, yang harus mengikuti format yang telah ditetapkan agar konsisten dan mudah dilacak.

### Membaca jenis file Image
Code pada bagian ini bertugas untuk membaca dan memproses penamaan file hasil konversi gambar.



char output_file[512];
snprintf(output_file, sizeof(output_file), "%s/%s_image_%s_%s.png",
         IMAGE_DIR, base_name, tanggal, waktu_file);



### Menyesuaikan format tanggal dan waktu 
Bagian kode ini digunakan untuk mengambil tanggal dan waktu saat ini, lalu memformatnya sesuai dengan format yang diinginkan untuk keperluan penamaan file atau pencatatan log.



strftime(tanggal, sizeof(tanggal), "%Y-%m-%d", timeinfo);
strftime(waktu_file, sizeof(waktu_file), "%H-%M-%S", timeinfo);  // ← untuk nama file



### Menggambungkan semua 
Code pada bagian ini bertugas untuk menggabungkan elemen-elemen yang dibutuhkan, yaitu nama file string, tanggal, bulan, tahun, serta waktu saat proses dijalankan, menjadi satu format penamaan yang sesuai.

snprintf(output_file, sizeof(output_file), "%s/%s_image_%s_%s.png", IMAGE_DIR, base_name, tanggal, waktu_file);



## D. Make conversion.log

### Library path log
Berikut adalah library yang digunakan untuk mendefinisikan path log dan memastikan program dapat dijalankan dengan benar.

#define LOG_FILE_PATH "anomali/conversion.log"


### Membuat conversion.log 
Diharapkan isi dari file conversion.log mengikuti format yang telah ditentukan, seperti contoh berikut:

[2025-05-11][18:35:26]: Successfully converted hexadecimal text 1.txt to 1_image_2025-05-11_18:35:26.png.


Bagian kode ini bertanggung jawab untuk membuat dan mengisi file conversion.log, yang mencatat aktivitas konversi secara otomatis dengan format log yang sesuai instruksi soal.


FILE *log = fopen(LOG_FILE_PATH, "a");  // ← Membuka atau membuat conversion.log
if (log) {
    fprintf(log, "[%s][%s]: Successfully converted hexadecimal text %s to %s.\n",
            tanggal, waktu_log, filename, strrchr(output_file, '/') + 1);  // ← Menulis isi log
    fclose(log);  // ← Menutup file log
}



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
