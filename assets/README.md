# Sisop-4-2025-IT32
- Ananda Fitri Wibowo (5027241057)
- Raihan Fahri Ghazali (5027241061)

| No | Nama                   | NRP         |
|----|------------------------|-------------|
| 1  | Ananda Fitri Wibowo    | 5027241057  |
| 2  | Raihan Fahri Ghazali   | 5027241061  |

# Soal 2
Solved by. 057_Ananda Fitri Wibowo
## A. Image Server
Server ini berfungsi untuk decrypt file dan ngirim filenya ke client.
### Define
- Di sini saya define port 8080 buat koneksi server dan client.
- Log file untuk nyimpen file log.
- Database untuk folder database yang isinya file-file jpeg hasil decrypt.
```
#define PORT 8080
#define LOG_FILE "server/server.log"
#define DATABASE_DIR "server/database/"
#define BUFFER_SIZE 8192
```

