## Task 2 - Organize and Analyze Anthony's Favorite Films

Anthony sedang asyik menonton film favoritnya dari Netflix, namun seiring berjalannya waktu, koleksi filmnya semakin menumpuk. Ia pun memutuskan untuk membuat sistem agar film-film favoritnya bisa lebih terorganisir dan mudah diakses. Anthony ingin melakukan beberapa hal dengan lebih efisien dan serba otomatis.

> Film-film yang dimaksud adalah film-film yang ada di dalam file ZIP yang bisa diunduh dari **[Google Drive](https://drive.google.com/file/d/12GWsZbSH858h2HExP3x4DfWZB1jLdV-J/view?usp=drive_link)**.

Berikut adalah serangkaian tugas yang Anthony ingin capai untuk membuat pengalaman menonton filmnya jadi lebih menyenangkan:

### **a. One Click and Done!**

Pernahkah kamu merasa malas untuk mengelola file ZIP yang penuh dengan data film? Anthony merasa hal yang sama, jadi dia ingin semuanya serba instan dengan hanya satu perintah. Dengan satu perintah saja, Anthony bisa:

- Mendownload file ZIP yang berisi data film-film Netflix favoritnya.
- Mengekstrak file ZIP tersebut ke dalam folder yang sudah terorganisir.
- Menghapus file ZIP yang sudah tidak diperlukan lagi, supaya tidak memenuhi penyimpanan.

Buatlah skrip yang akan mengotomatiskan proses ini sehingga Anthony hanya perlu menjalankan satu perintah untuk mengunduh, mengekstrak, dan menghapus file ZIP.

### **b. Sorting Like a Pro**

Koleksi film Anthony semakin banyak dan dia mulai bingung mencari cara yang cepat untuk mengelompokkannya. Nah, Anthony ingin mengelompokkan film-filmnya dengan dua cara yang sangat mudah:

1. Berdasarkan huruf pertama dari judul film.
2. Berdasarkan tahun rilis (release year).

Namun, karena Anthony sudah mempelajari **multiprocessing**, dia ingin mengelompokkan kedua kategori ini secara paralel untuk menghemat waktu.

**Struktur Output:**

- **Berdasarkan Huruf Pertama Judul Film:**

  - Folder: `judul/`
  - Setiap file dinamai dengan huruf abjad atau angka, seperti `A.txt`, `B.txt`, atau `1.txt`.
  - Jika judul film tidak dimulai dengan huruf atau angka, film tersebut disimpan di file `#.txt`.

- **Berdasarkan Tahun Rilis:**
  - Folder: `tahun/`
  - Setiap file dinamai sesuai tahun rilis film, seperti `1999.txt`, `2021.txt`, dst.

Format penulisan dalam setiap file :

```
Judul Film - Tahun Rilis - Sutradara
```

Setiap proses yang berjalan akan mencatat aktivitasnya ke dalam satu file bernama **`log.txt`** dengan format:

```
[jam:menit:detik] Proses mengelompokkan berdasarkan [Abjad/Tahun]: sedang mengelompokkan untuk film [judul_film]
```

**Contoh Log:**

```
[14:23:45] Proses mengelompokkan berdasarkan Abjad: sedang mengelompokkan untuk film Avengers: Infinity War
[14:23:46] Proses mengelompokkan berdasarkan Tahun: sedang mengelompokkan untuk film Kung Fu Panda
```

### **c. The Ultimate Movie Report**

Sebagai penggemar film yang juga suka menganalisis, Anthony ingin mengetahui statistik lebih mendalam tentang film-film yang dia koleksi. Misalnya, dia ingin tahu berapa banyak film yang dirilis **sebelum tahun 2000** dan **setelah tahun 2000**.

Agar laporan tersebut mudah dibaca, Anthony ingin hasilnya disimpan dalam file **`report_ddmmyyyy.txt`**.

**Format Output dalam Laporan:**

```
i. Negara: <nama_negara>
Film sebelum 2000: <jumlah>
Film setelah 2000: <jumlah>

...
i+n. Negara: <nama_negara>
Film sebelum 2000: <jumlah>
Film setelah 2000: <jumlah>
```

Agar penggunaannya semakin mudah, Anthony ingin bisa menjalankan semua proses di atas melalui sebuah antarmuka terminal interaktif dengan pilihan menu seperti berikut:
1. Download File
2. Mengelompokkan Film
3. Membuat Report

Catatan:
- Dilarang menggunakan `system`
- Harap menggunakan thread dalam pengerjaan soal C

---

### Penyelesaian :
a. Anthony.c
- Code lengkap:
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <pthread.h>
#include <time.h>
#include <unistd.h>
#include <sys/stat.h>
#include <ctype.h>
#include <zip.h>
#include <errno.h>
#include <curl/curl.h>
#include <stdbool.h>

#define ZIP_URL "https://drive.google.com/uc?export=download&id=12GWsZbSH858h2HExP3x4DfWZB1jLdV-J"
#define ZIP_FILE "netflixData.zip"
#define DATA_FOLDER "data"
#define CSV_FILE "data/netflixData.csv"
#define MAX_LINE_LENGTH 1024
#define MAX_FIELDS 20

typedef struct {
    char country[100];
    int before_2000;
    int after_2000;
} CountryStat;

size_t write_data(void *ptr, size_t size, size_t nmemb, FILE *stream);
int download_file();
void log_message(const char* message);
void PARSE_CSV(char* line, char** fields, int max_fields);
int extract_zip();
void delete_zip();
void* organize_by_tittle(void* arg);
void* organize_by_year(void* arg);
void tampilkan_menu();
void* unduh_dan_ekstrak(void* arg);
void* kelompokkan_film(void* arg);
void* buat_laporan(void* arg);
void* generate_report(void* arg);
void proses_pilihan(int pilihan);

size_t write_data(void *ptr, size_t size, size_t nmemb, FILE *stream) {
    return fwrite(ptr, size, nmemb, stream);
}

int download_file() {
    CURL *curl;
    FILE *fp;
    CURLcode res;

    curl = curl_easy_init();
    if (!curl) {
        fprintf(stderr, "Failed to initialize curl\n");
        return -1;
    }

    fp = fopen(ZIP_FILE, "wb");
    if (!fp) {
        fprintf(stderr, "Failed to create file %s\n", ZIP_FILE);
        curl_easy_cleanup(curl);
        return -1;
    }

    curl_easy_setopt(curl, CURLOPT_URL, ZIP_URL);
    curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, write_data);
    curl_easy_setopt(curl, CURLOPT_WRITEDATA, fp);
    curl_easy_setopt(curl, CURLOPT_FOLLOWLOCATION, 1L);

    printf("Downloading file...\n");
    res = curl_easy_perform(curl);
    if (res != CURLE_OK) {
        fprintf(stderr, "Download failed: %s\n", curl_easy_strerror(res));
        fclose(fp);
        curl_easy_cleanup(curl);
        return -1;
    }

    fclose(fp);
    curl_easy_cleanup(curl);
    return 0;
}

void log_message(const char* message) {
    FILE* log = fopen("log.txt", "a");
    if (!log) {
        perror("Failed to open log file");
        return;
    }

    time_t now = time(NULL);
    struct tm *t = localtime(&now);
    fprintf(log, "[%02d:%02d:%02d] %s\n", t->tm_hour, t->tm_min, t->tm_sec, message);
    fclose(log);
}

void PARSE_CSV(char* line, char** fields, int max_fields) {
    int in_quotes = 0;
    int field_index = 0;
    char* start = line;

    for (char* p = line; *p && field_index < max_fields; p++) {
        if (*p == '"') {
            in_quotes = !in_quotes; 
        } else if (*p == ',' && !in_quotes) {
            *p = '\0';
            fields[field_index++] = start;
            start = p + 1;
        }
    }
    
    if (field_index < max_fields) {
        fields[field_index++] = start;
    }

    for (int i = 0; i < field_index; i++) {
        fields[i][strcspn(fields[i], "\r\n")] = 0;
    }
}

int extract_zip() {
    struct zip *archive;
    struct zip_file *file;
    struct zip_stat stat;
    char buf[100];
    int err;

    if (mkdir(DATA_FOLDER, 0777) == -1 && errno != EEXIST) {
        perror("Failed to create data directory");
        return -1;
    }

    if ((archive = zip_open(ZIP_FILE, 0, &err)) == NULL) {
        zip_error_to_str(buf, sizeof(buf), err, errno);
        fprintf(stderr, "Error opening zip: %s\n", buf);
        return -1;
    }

    int entries = zip_get_num_entries(archive, 0);
    for (int i = 0; i < entries; i++) {
        if (zip_stat_index(archive, i, 0, &stat) == -1) {
            fprintf(stderr, "Failed to stat file at index %d\n", i);
            continue;
        }

        char path[256];
        snprintf(path, sizeof(path), "%s/%s", DATA_FOLDER, stat.name);

        char *dir = strdup(path);
        char *last_slash = strrchr(dir, '/');
        if (last_slash) {
            *last_slash = '\0';
            mkdir(dir, 0777);
        }
        free(dir);

        if ((file = zip_fopen_index(archive, i, 0)) == NULL) {
            fprintf(stderr, "Failed to open file %s in zip\n", stat.name);
            continue;
        }

        FILE *out = fopen(path, "wb");
        if (!out) {
            fprintf(stderr, "Failed to create file %s\n", path);
            zip_fclose(file);
            continue;
        }

        int bytes_read;
        while ((bytes_read = zip_fread(file, buf, sizeof(buf))) > 0) {
            fwrite(buf, 1, bytes_read, out);
        }

        fclose(out);
        zip_fclose(file);
    }

    zip_close(archive);
    return 0;
}

void delete_zip() {
    if (remove(ZIP_FILE) == -1) {
        perror("Failed to delete zip file");
    }
}

void* organize_by_tittle(void* arg) {
    FILE* f = fopen(CSV_FILE, "r");
    if (!f) pthread_exit(NULL);

    mkdir("judul", 0777);
    char line[1024];
    fgets(line, sizeof(line), f);

    while (fgets(line, sizeof(line), f)) {
        char* fields[10];
        PARSE_CSV(line, fields, 10);
        char* title = fields[0];
        char* director = fields[1];
        char* country = fields[2];
        char* year_str = fields[3];
        if (!title || !year_str || !director) continue;

        year_str[strcspn(year_str, "\r\n")] = 0;

        char logmsg[256];
        snprintf(logmsg, sizeof(logmsg), "Proses mengelompokkan berdasarkan Abjad: sedang mengelompokkan untuk film %s", title);
        log_message(logmsg);

        char fname[20];
        char first = title[0];
        if (isalpha(first)) {
            snprintf(fname, sizeof(fname), "judul/%c.txt", toupper(first));
        } else if (isdigit(first)) {
            snprintf(fname, sizeof(fname), "judul/%c.txt", first);
        } else {
            snprintf(fname, sizeof(fname), "judul/#.txt");
        }

        FILE* out = fopen(fname, "a");
        if (out) {
            fprintf(out, "%s - %s - %s\n", title, year_str, director);
            fclose(out);
        }
    }
    fclose(f);
    pthread_exit(NULL);
}

void* organize_by_year(void* arg) {
    FILE* f = fopen(CSV_FILE, "r");
    if (!f) pthread_exit(NULL);

    mkdir("tahun", 0777);
    char line[1024];
    fgets(line, sizeof(line), f);

    while (fgets(line, sizeof(line), f)) {
        char* fields[10];
        PARSE_CSV(line, fields, 10);
        char* title = fields[0];
        char* director = fields[1];
        char* country = fields[2];
        char* year_str = fields[3];        
        if (!title || !year_str || !director) continue;

        year_str[strcspn(year_str, "\r\n")] = 0;

        char logmsg[256];
        snprintf(logmsg, sizeof(logmsg), "Proses mengelompokkan berdasarkan Tahun: sedang mengelompokkan untuk film %s", title);
        log_message(logmsg);

        char fname[64];
        snprintf(fname, sizeof(fname), "tahun/%s.txt", year_str);

        FILE* out = fopen(fname, "a");
        if (out) {
            fprintf(out, "%s - %s - %s\n", title, year_str, director);
            fclose(out);
        }
    }
    fclose(f);
    pthread_exit(NULL);
}

void tampilkan_menu() {
    printf("\n===== FILM ANTHONY =====\n");
    printf("1. Unduh dan Ekstrak ZIP\n");
    printf("2. Kelompokkan Film\n");
    printf("3. Buat Laporan\n");
    printf("0. Keluar\n");
    printf("Pilihan: ");
}

void* unduh_dan_ekstrak(void* arg) {
    if (download_file() != 0) {
        pthread_exit(NULL);
    }

    printf("Ekstrak isi ZIP...\n");
    if (extract_zip() == 0) {
        printf("Berhasil diekstrak ke folder %s.\n", DATA_FOLDER);
        delete_zip();
        printf("File ZIP dihapus untuk menghemat ruang.\n");
    } else {
        printf("Gagal mengekstrak ZIP.\n");
    }
    pthread_exit(NULL);
}

void* kelompokkan_film(void* arg) {
    pthread_t t1, t2;
    pthread_create(&t1, NULL, organize_by_tittle, NULL);
    pthread_create(&t2, NULL, organize_by_year, NULL);
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    printf("Film berhasil dikelompokkan berdasarkan abjad dan tahun.\n");
    pthread_exit(NULL);
}

void* buat_laporan(void* arg) {
    pthread_t tid;
    pthread_create(&tid, NULL, generate_report, NULL);
    pthread_join(tid, NULL);
    pthread_exit(NULL);
}

void* generate_report(void* arg) {
    FILE* f = fopen(CSV_FILE, "r");
    if (!f) {
        perror("Failed to open CSV file");
        pthread_exit(NULL);
    }

    CountryStat stats[100] = {0};
    int stat_count = 0;

    char line[MAX_LINE_LENGTH];
    if (!fgets(line, sizeof(line), f)) {
        fclose(f);
        pthread_exit(NULL);
    }

    while (fgets(line, sizeof(line), f)) {
        char* fields[MAX_FIELDS] = {0};
        PARSE_CSV(line, fields, MAX_FIELDS);
        
        if (!fields[0] || !fields[1] || !fields[2] || !fields[3]) continue;

        int year = atoi(fields[3]);
        bool found = false;

        for (int i = 0; i < stat_count; i++) {
            if (strcmp(stats[i].country, fields[2]) == 0) {
                found = true;
                if (year < 2000) stats[i].before_2000++;
                else stats[i].after_2000++;
                break;
            }
        }

        if (!found && stat_count < 100) {
            strncpy(stats[stat_count].country, fields[2], sizeof(stats[0].country) - 1);
            if (year < 2000) {
                stats[stat_count].before_2000 = 1;
                stats[stat_count].after_2000 = 0;
            } else {
                stats[stat_count].before_2000 = 0;
                stats[stat_count].after_2000 = 1;
            }
            stat_count++;
        }
    }
    fclose(f);

    time_t now = time(NULL);
    struct tm *t = localtime(&now);
    char report_name[50];
    strftime(report_name, sizeof(report_name), "report_%d%m%Y.txt", t);

    FILE* report = fopen(report_name, "w");
    if (report) {
        for (int i = 0; i < stat_count; i++) {
            fprintf(report, "%d. Negara: %s\n", i+1, stats[i].country);
            fprintf(report, "Film sebelum 2000: %d\n", stats[i].before_2000);
            fprintf(report, "Film setelah 2000: %d\n\n", stats[i].after_2000);
        }
        fclose(report);
        printf("Report successfully created: %s\n", report_name);
    } else {
        perror("Failed to create report");
    }

    pthread_exit(NULL);
}

void proses_pilihan(int pilihan) {
    pthread_t tid;

    switch (pilihan) {
        case 1:
            pthread_create(&tid, NULL, unduh_dan_ekstrak, NULL);
            pthread_join(tid, NULL);
            break;
        case 2:
            pthread_create(&tid, NULL, kelompokkan_film, NULL);
            pthread_join(tid, NULL);
            break;
        case 3:
            pthread_create(&tid, NULL, buat_laporan, NULL);
            pthread_join(tid, NULL);
            break;
        case 0:
            curl_global_cleanup();
            exit(0);
        default:
            printf("Pilihan tidak valid.\n");
    }
}

int main() {
    curl_global_init(CURL_GLOBAL_DEFAULT);

    int pilihan;
    while (1) {
        tampilkan_menu();
        scanf("%d", &pilihan);
        proses_pilihan(pilihan);
    }

    curl_global_cleanup();
    return 0;
}
```
### Penjelasan:
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <pthread.h>
#include <time.h>
#include <unistd.h>
#include <sys/stat.h>
#include <ctype.h>
#include <zip.h>
#include <errno.h>
#include <curl/curl.h>
#include <stdbool.h>
```
Ada sejumlah library yang digunakan untuk menyelesaikan soal ini yaitu :
- `#include <stdio.h>` berfungsi untuk input-output standar seperti printf, scanf, fprintf, dll.
- `#include <stdlib.h>` berfungsi sebagai manajemen memori.
- `#include <string.h>` berfungsi sebagai library yang memanipulasi string.
- `#include <pthread.h>` berfungsi untuk menggunakan threading (proses berjalan paralel).
- `#include <time.h>` library yang digunakan untuk menangani waktu dan tanggal.
- `#include <unistd.h>` berfungsi sebagai fungsi-fungsi sistem dasar unix/linux seperti `sleep`, `usleep`, `fork`, `exec`.
- `#include <sys/stat.h>` berfungsi untuk memanipulasi file dan direktori.
- `#include <ctype.h>` untuk fungsi yang berhubungan dengan karakter.
- `#include <zip.h>` digunakan untuk mengolah dan mengakses file ZIP.
- `#include <errno.h>` untuk menangani error (kesalahan) yang terjadi di sistem operasi.
- `#include <curl/curl.h>` digunakan untuk melakukan HTTP Request (download file, upload data ke server, dll.).
- `#include <stdbool.h>` digunakan sebagai penggunaan tipe data boolean di C.

```c
#define ZIP_URL "https://drive.google.com/uc?export=download&id=12GWsZbSH858h2HExP3x4DfWZB1jLdV-J"
#define ZIP_FILE "netflixData.zip"
#define DATA_FOLDER "data"
#define CSV_FILE "data/netflixData.csv"
#define MAX_LINE_LENGTH 1024
#define MAX_FIELDS 20
```
Kemudian di sini ada beberapa konstanta di mana konstanta `#define ZIP_URL "https://drive.google.com/uc?export=download&id=12GWsZbSH858h2HExP3x4DfWZB1jLdV-J"` itu berfungsi untuk mendownload file ZIP dari URL yang sudah dicantumkan, `#define ZIP_FILE "netflixData.zip"` berfungsi sebagai nama dari file ZIP yang sudah didownload, `#define DATA_FOLDER "data"` pada konstanta ini berfungsi sebagai pemberian nama folder untuk menyimpan file ZIP yang telah diekstrak yaitu netflixData.csv, `#define CSV_FILE "data/netflixData.csv"` ini adalah konstanta untuk menyimpan path dari file netflixData.zip, `#define MAX_LINE_LENGTH 1024` konstanta yang menjadi batasan panjang saat membaca file, `#define MAX_FIELDS 20` pada konstanta ini mirip seperti konstanta sebelumnya, yaitu untuk menentukan batasan jumlah kolom (field) dalam satu baris CSV.

```c
typedef struct {
    char country[100];
    int before_2000;
    int after_2000;
} CountryStat;
```
Struck CountryStat ini dugunakan untuk menyimpan statistik dari film menggunakan beberapa variabel didalamnya. `char country[100]` digunakan untuk menyimpan nama negara dengan panjang maksimal 100 karakter, `int before_2000` digunakan untuk menyimpan jumlah film yang rilis sebelum tahun 200, dan `int after_2000` digunakan untuk menyimpan jumlah film yang rilis setelah tahun 2000.


```c
size_t write_data(void *ptr, size_t size, size_t nmemb, FILE *stream);
int download_file();
void log_message(const char* message);
void PARSE_CSV(char* line, char** fields, int max_fields);
int extract_zip();
void delete_zip();
void* organize_by_tittle(void* arg);
void* organize_by_year(void* arg);
void tampilkan_menu();
void* unduh_dan_ekstrak(void* arg);
void* kelompokkan_film(void* arg);
void* buat_laporan(void* arg);
void* generate_report(void* arg);
void proses_pilihan(int pilihan);
```
Ini ada bentuk deklarasi fungsi prototype di dalam program C, bagian ini dugunakan untuk memberitahu compiler bahwa fungsi-fungsi ini ada, sehingga jika nanti kode dijalankan akan menghindari yang namanya kompiler error.

```c
size_t write_data(void *ptr, size_t size, size_t nmemb, FILE *stream) {
    return fwrite(ptr, size, nmemb, stream);
}
```
Bagian ini berfungsi untuk menulis data hasil download ke dalam file.
`curl` di sini digunakan untuk mendownload file (`netflixData.zip`), curl akan nerima data sedikit-sedikit (kayak aliran). Nah, write_data ini dipanggil setiap ada data baru, supaya data itu langsung ditulis ke file lokal.
Penjelasan bagian fungsi `write_data` sendiri adalah sebagai berikut :
- `void *ptr` : Pointer ke data yang mau ditulis (data hasil download dari internet).
-  `size_t size` : Ukuran tiap 1 data (dalam byte).
-  `size_t nmemb` : Jumlah data yang mau ditulis.
-  `FILE *stream` : file tujuan tempat data ini mau ditulis (`netflixData.zip`).

```c
return fwrite(ptr, size, nmemb, stream);
```
Kemudian bagian ini juga memiliki penjelasan tersendiri yakni :
Fungsi `fwrite` digunakan untuk menulis data ke file. Alurnya yaitu mengambil data dari `ptr`, setelah itu banyak byte dari data itu sebesar `size` x `nmemb`, kemudian tulis ke file tujuan (`stream`).

```c
int download_file() {
    CURL *curl;
    FILE *fp;
    CURLcode res;

    curl = curl_easy_init();
    if (!curl) {
        fprintf(stderr, "Failed to initialize curl\n");
        return -1;
    }

    fp = fopen(ZIP_FILE, "wb");
    if (!fp) {
        fprintf(stderr, "Failed to create file %s\n", ZIP_FILE);
        curl_easy_cleanup(curl);
        return -1;
    }

    curl_easy_setopt(curl, CURLOPT_URL, ZIP_URL);
    curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, write_data);
    curl_easy_setopt(curl, CURLOPT_WRITEDATA, fp);
    curl_easy_setopt(curl, CURLOPT_FOLLOWLOCATION, 1L);

    printf("Downloading file...\n");
    res = curl_easy_perform(curl);
    if (res != CURLE_OK) {
        fprintf(stderr, "Download failed: %s\n", curl_easy_strerror(res));
        fclose(fp);
        curl_easy_cleanup(curl);
        return -1;
    }

    fclose(fp);
    curl_easy_cleanup(curl);
    return 0;
}
```
Fungsi `download_file` ini digunakan untuk mendownload file ZIP dari link URL dan menyimpannya ke laptop.
`CURL *curl` digunakan untuk handle koneksi internet (CURL), `FILE *fp` untuk handle file output (ZIP), `CURLcode res`  untuk menyimpan hasil dari proses download. Kemudian ada `curl = curl_easy_init();` digunakan sebagai inisialisasi CURL.
```c
if (!curl) {
        fprintf(stderr, "Failed to initialize curl\n");
        return -1;
    }
```
Jika CURL gagal diinisialisasi, maka akan keluar dari fungsi dengan error.
```c
fp = fopen(ZIP_FILE, "wb");
```
Bagian ini digunakan untuk membuka file baru di laptop (`netflixData.zip`) untuk ditulis dalam mode biner.

```c
if (!fp) {
        fprintf(stderr, "Failed to create file %s\n", ZIP_FILE);
        curl_easy_cleanup(curl);
        return -1;
    }
```
Jika file tidak bisa dibuat, maka hentikan proses dan bersihkan resource CURL.

```c
curl_easy_setopt(curl, CURLOPT_URL, ZIP_URL);
```
Bagian ini berfungsi untuk memberi tahu CURL, URL mana yang akan di-download.

```c
curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, write_data);
```
Bagian ini untuk set fungsi `write_data` sebagai cara menulis data hasil download ke file.

```c
curl_easy_setopt(curl, CURLOPT_WRITEDATA, fp);
```
Bagian ini berfungsi untuk memberitahu CURL supaya menulis hasil download ke file fp (netflixData.zip).

```c
 curl_easy_setopt(curl, CURLOPT_FOLLOWLOCATION, 1L);
```
Bagian ini berfungsi Jika URL yang mau diakses itu mengarahkan ke URL lain, CURL akan otomatis mengikutinya.

```c
printf("Downloading file...\n");
```
Pesan `Downloading file...` akan dicetak ke terminal supaya user tahu jika sedang melakukan download file.

```c
res = curl_easy_perform(curl);
```
Bagian ini akan memulai download file sesuai semua setting tadi.

```c
if (res != CURLE_OK) {
        fprintf(stderr, "Download failed: %s\n", curl_easy_strerror(res));
        fclose(fp);
        curl_easy_cleanup(curl);
        return -1;
    }
```
Jika download file gagal, maka akan menampilkan pesan error dan tutup semua resource.

```c
fclose(fp);
```
Bagian ini digunakan untuk menutup file setelah selesai download supaya isinya benar-benar tersimpan.

```c
curl_easy_cleanup(curl);
```
Bagian ini digunakan untuk membersihkan CURL dari memori.
```c
return 0;
```
Jika berjalan dengan lancar maka akan mengembalikan angka 0, yang berarti fungsi sukses.



```c
void log_message(const char* message) {
    FILE* log = fopen("log.txt", "a");
    if (!log) {
        perror("Failed to open log file");
        return;
    }

    time_t now = time(NULL);
    struct tm *t = localtime(&now);
    fprintf(log, "[%02d:%02d:%02d] %s\n", t->tm_hour, t->tm_min, t->tm_sec, message);
    fclose(log);
}
```
Fungsi `log_message` ini berfungsi untuk  mencatat pesan ke dalam file log.txt, dengan timestamp (jam-menit-detik).
- Beriikut ini adalah penjelasan lebih dalam:
```c
void log_message(const char* message)
```
Parameter dari fungsi `log_message` ini adalah teks yang mau dicatat di file log.
```c
FILE* log = fopen("log.txt", "a");
```
Bagian ini digunakan untuk membuka file `log.txt` dalam mode append (`"a"` artinya tambah di akhir file, tidak menghapus isinya), jika `log.txt` belum ada, otomatis dibuat.

```c
if (!log) {
        perror("Failed to open log file");
        return;
    }
```
Bagian ini menjelaskan jika file gagal dibuka, akan muncul pesan error dan fungsi akan langsung keluar (return).

```c
time_t now = time(NULL);
```
Ini digunakan untuk mengambil waktu saat ini.

```c
struct tm *t = localtime(&now);
```
Bagian ini akan mengubah waktu (now) menjadi format waktu lokal yang bisa dibaca manusia, misalnya jam 13:45:20.

```c
fprintf(log, "[%02d:%02d:%02d] %s\n", t->tm_hour, t->tm_min, t->tm_sec, message);
```
Bagian ini berfungsi menulis file ke `log.txt` dengan format seperti `[13:45:20] Download berhasil` dan pada simbol `%02d` artinya angka ditulis 2 digit, kalau kurang ditambah 0 di depannya (contoh: 05, 09).

```c
fclose(log);
```
Bagian ini digunakan untuk menutup file setelah selesai menulis supaya tidak terjadi error atau kehilangan data.

```c
void PARSE_CSV(char* line, char** fields, int max_fields) {
    int in_quotes = 0;
    int field_index = 0;
    char* start = line;

    for (char* p = line; *p && field_index < max_fields; p++) {
        if (*p == '"') {
            in_quotes = !in_quotes; 
        } else if (*p == ',' && !in_quotes) {
            *p = '\0';
            fields[field_index++] = start;
            start = p + 1;
        }
    }
    
    if (field_index < max_fields) {
        fields[field_index++] = start;
    }

    for (int i = 0; i < field_index; i++) {
        fields[i][strcspn(fields[i], "\r\n")] = 0;
    }
}
```

Fungsi `PARSE_CSV` digunakan untuk memecah satu baris CSV menjadi beberapa kolom (fields) berdasarkan tanda koma (,), dengan memperhatikan tanda kutip (") jika ada. Berikut ini adalah penjelasan lebih lanjut:
```c
void PARSE_CSV(char* line, char** fields, int max_fields)
```
`line` adalah 1 baris data CSV, `fields` array untuk menyimpan hasil pecahan kolom, `max_fields` jumlah maksimum kolom yang bisa diproses.

```c
int in_quotes = 0;
int field_index = 0;
char* start = line;
```
Veriabel `in_quotes` digunakan untuk mengecek apakah sekarang lagi di dalam tanda kutip atau tidak, variabel `field_index` untuk indeks kolom saat ini (misal kolom ke-0, ke-1, ke-2...), dan variabel `start` untuk pointer ke awal kolom yang sedang dibaca.
```c
for (char* p = line; *p && field_index < max_fields; p++)
```
Bagian ini loop karakter per karakter di dalam line, akan berhenti kalau sudah habis (*p == '\0') atau sudah maksimal jumlah kolom (field_index < max_fields).

```c
if (*p == '"') {
    in_quotes = !in_quotes; 
}
```
Kalau ketemu tanda kutip ("), bolak-balik status `in_quotes`.

```c
else if (*p == ',' && !in_quotes) {
    *p = '\0';
    fields[field_index++] = start;
    start = p + 1;
}
```
Jika ketemu koma dan lagi di luar tanda kutip:
- Ubah koma itu menjadi \0 (mengakhiri string kolom).
- Simpan alamat `start` ke `fields`.
- Lanjut `start` ke karakter berikutnya (setelah koma).

```c
if (field_index < max_fields) {
        fields[field_index++] = start;
    }
```
Setelah selesai loop, simpan sisa data yang terakhir ke `fields`.

```c
for (int i = 0; i < field_index; i++) {
    fields[i][strcspn(fields[i], "\r\n")] = 0;
}
```
Bersihkan karakter newline (\n) atau carriage return (\r) di akhir setiap field, jadi setiap kolom benar-benar bersih dari karakter aneh.

```c
int extract_zip() {
    struct zip *archive;
    struct zip_file *file;
    struct zip_stat stat;
    char buf[100];
    int err;

    if (mkdir(DATA_FOLDER, 0777) == -1 && errno != EEXIST) {
        perror("Failed to create data directory");
        return -1;
    }

    if ((archive = zip_open(ZIP_FILE, 0, &err)) == NULL) {
        zip_error_to_str(buf, sizeof(buf), err, errno);
        fprintf(stderr, "Error opening zip: %s\n", buf);
        return -1;
    }

    int entries = zip_get_num_entries(archive, 0);
    for (int i = 0; i < entries; i++) {
        if (zip_stat_index(archive, i, 0, &stat) == -1) {
            fprintf(stderr, "Failed to stat file at index %d\n", i);
            continue;
        }

        char path[256];
        snprintf(path, sizeof(path), "%s/%s", DATA_FOLDER, stat.name);

        char *dir = strdup(path);
        char *last_slash = strrchr(dir, '/');
        if (last_slash) {
            *last_slash = '\0';
            mkdir(dir, 0777);
        }
        free(dir);

        if ((file = zip_fopen_index(archive, i, 0)) == NULL) {
            fprintf(stderr, "Failed to open file %s in zip\n", stat.name);
            continue;
        }

        FILE *out = fopen(path, "wb");
        if (!out) {
            fprintf(stderr, "Failed to create file %s\n", path);
            zip_fclose(file);
            continue;
        }

        int bytes_read;
        while ((bytes_read = zip_fread(file, buf, sizeof(buf))) > 0) {
            fwrite(buf, 1, bytes_read, out);
        }

        fclose(out);
        zip_fclose(file);
    }

    zip_close(archive);
    return 0;
}
```

Fungsi ini digunakan untuk mengekstrak file ZIP (`netflixData.zip`) ke dalam folder data/. Jadi, isinya di-buka lalu file-file di dalamnya diekstrak keluar jadi file biasa di sistem.
Berikut adalah penjelasan per bagiannya:
```c
struct zip *archive;
struct zip_file *file;
struct zip_stat stat;
char buf[100];
int err;
```
- `archive`: pointer untuk file ZIP yang dibuka.
- `file`: pointer untuk file yang ada di dalam ZIP.
- `stat`: info tentang file di dalam ZIP (nama, ukuran, dll).
- `buf`: buffer kecil untuk membaca data.
- `err`: menyimpan kode error jika ada.

```c
if (mkdir(DATA_FOLDER, 0777) == -1 && errno != EEXIST) {
        perror("Failed to create data directory");
        return -1;
    }
```
Bagian ini untuk membuat folder data/ untuk mengekstrak file, jika folder sudah ada maka lanjut saja. Jika gagal buat folder karena alasan lain, langsung gagal (return -1).

```c
if ((archive = zip_open(ZIP_FILE, 0, &err)) == NULL) {
        zip_error_to_str(buf, sizeof(buf), err, errno);
        fprintf(stderr, "Error opening zip: %s\n", buf);
        return -1;
    }
```
Bagian ini akan membuka file ZIP, jika gagal cetak pesan error dan keluar dari fungsi.
```c
int entries = zip_get_num_entries(archive, 0);
```
Bagian ini akan menghitung ada berapa jumlah file di dc
alam ZIP.
```c
for (int i = 0; i < entries; i++) {
```
Ini akan looping dari file pertama sampai file terakhir di dalam ZIP.

```c
if (zip_stat_index(archive, i, 0, &stat) == -1) {
    fprintf(stderr, "Failed to stat file at index %d\n", i);
    continue;
}
```
Untuk cek informasi file (nama file, ukuran file, dll), Jika gagal cek file ini, skip ke file berikutnya (continue).

```c
char path[256];
snprintf(path, sizeof(path), "%s/%s", DATA_FOLDER, stat.name);
```
Ini digunakan menyiapkan path untuk menyimpan file, dengan menggabungkan nama folder data/ dengan nama file yang akan diekstrak.

```c
char *dir = strdup(path);
char *last_slash = strrchr(dir, '/');
if (last_slash) {
    *last_slash = '\0';
    mkdir(dir, 0777);
}
free(dir);
```
Bagian ini untuk membuat folder jika diperlukan, Jika file ada di dalam subfolder maka bikin folder dulu sebelum membuat file-nya.
- `strdup` = duplikat string.
- `strrchr(dir, '/')` = cari / terakhir (folder terakhir).

```c
if ((file = zip_fopen_index(archive, i, 0)) == NULL) {
    fprintf(stderr, "Failed to open file %s in zip\n", stat.name);
    continue;
}
```
Ini akan membuka file di dalam ZIP untuk dibaca.

```c
FILE *out = fopen(path, "wb");
if (!out) {
    fprintf(stderr, "Failed to create file %s\n", path);
    zip_fclose(file);
    continue;
}
```
Bagian ini untuk membuka file output yaitu membuka file hasil di komputer untuk ditulisi (wb = write binary), Jika gagal tutup file dari ZIP dan lanjut ke file berikutnya.

```c
int bytes_read;
while ((bytes_read = zip_fread(file, buf, sizeof(buf))) > 0) {
    fwrite(buf, 1, bytes_read, out);
}
```

Bagian ini akan menyalin isi file yaitu membaca isi file dari ZIP (`zip_fread`) sedikit-sedikit (pakai `buf`) lalu tulis ke file hasil (`fwrite`). Ini seperti mengopi file dari ZIP ke file lokal.

```c
fclose(out);
zip_fclose(file);
```
Setelah selesai menulis satu file, tutup file output dan tutup file dari ZIP.

```c
zip_close(archive);
```
Setelah semua file selesai, tutup file ZIP.

```c
return 0;
```
Menandakan kode sukses dijalankan.

```c
void delete_zip() {
    if (remove(ZIP_FILE) == -1) {
        perror("Failed to delete zip file");
    }
}
```
Fungsi `delete_zip()` digunakan untuk menghapus file ZIP setelah selesai digunakan (misalnya setelah proses ekstraksi selesai), agar file tidak memenuhi penyimpanan. Berikut adalah penjelasannya:

```c
    if (remove(ZIP_FILE) == -1) {
```
Fungsi remove() mencoba menghapus file yang namanya disimpan dalam ZIP_FILE (`"netflixData.zip"`), Jika gagal (return -1), maka masuk ke blok if.

```c
        perror("Failed to delete zip file");
```
Bagian ini yang akan menampilkan pesan error ke terminal, fungsi perror() otomatis menambahkan detail error dari sistem.


```c
void* organize_by_tittle(void* arg) {
    FILE* f = fopen(CSV_FILE, "r");
    if (!f) pthread_exit(NULL);

    mkdir("judul", 0777);
    char line[1024];
    fgets(line, sizeof(line), f);

    while (fgets(line, sizeof(line), f)) {
        char* fields[10];
        PARSE_CSV(line, fields, 10);
        char* title = fields[0];
        char* director = fields[1];
        char* country = fields[2];
        char* year_str = fields[3];
        if (!title || !year_str || !director) continue;

        year_str[strcspn(year_str, "\r\n")] = 0;

        char logmsg[256];
        snprintf(logmsg, sizeof(logmsg), "Proses mengelompokkan berdasarkan Abjad: sedang mengelompokkan untuk film %s", title);
        log_message(logmsg);

        char fname[20];
        char first = title[0];
        if (isalpha(first)) {
            snprintf(fname, sizeof(fname), "judul/%c.txt", toupper(first));
        } else if (isdigit(first)) {
            snprintf(fname, sizeof(fname), "judul/%c.txt", first);
        } else {
            snprintf(fname, sizeof(fname), "judul/#.txt");
        }

        FILE* out = fopen(fname, "a");
        if (out) {
            fprintf(out, "%s - %s - %s\n", title, year_str, director);
            fclose(out);
        }
    }
    fclose(f);
    pthread_exit(NULL);
}
```
Fungsi organize_by_tittle bertugas untuk mengelompokkan data film berdasarkan huruf pertama judul film, lalu menyimpannya ke dalam file teks sesuai abjadnya.
Berikut adalah penjelasannya:
```c
void* organize_by_tittle(void* arg)
```
Fungsi ini adalah fungsi thread (`void*` untuk argumen dan hasil) agar proses pengelompokan ini bisa berjalan paralel.

```c
FILE* f = fopen(CSV_FILE, "r");
if (!f) pthread_exit(NULL);
```
Ini untuk membuka file CSV untuk dibaca (`CSV_FILE = "data/netflixData.csv"`) dan jika gagal (misalnya file tidak ada), maka thread keluar.

```c
mkdir("judul", 0777);
```
Membuat folder bernama judul (jika belum ada), di sini adalah tempat menyimpan file txt berdasarkan urutan judul film.
```c
char line[1024];
fgets(line, sizeof(line), f);
```
Digunakan untuk membaca satu baris dari CSV untuk melewati header.

```c
while (fgets(line, sizeof(line), f)) {
```
Bagian ini digunakan untuk membaca baris-baris berikutnya (isi data film) satu per satu.

```c
char* fields[10];
PARSE_CSV(line, fields, 10);
```
Memanggil fungsi `PARSE_CSV` untuk memecah satu baris CSV menjadi array field (judul, sutradara, negara, tahun, dll).

```c
char* title = fields[0];
char* director = fields[1];
char* country = fields[2];
char* year_str = fields[3];
```
Varibel ini digunakan untuk menyimpan field seperti judul, sutradara, negara, dan tahun.

```c
if (!title || !year_str || !director) continue;
```

Jika ada field penting yang kosong, baris ini dilewati.

```c
year_str[strcspn(year_str, "\r\n")] = 0;
```
Ini digunakan untuk menghapus karakter \r atau \n dari akhir tahun, supaya tidak ikut ditulis ke file.

```c
char logmsg[256];
snprintf(logmsg, sizeof(logmsg), "Proses mengelompokkan berdasarkan Abjad: sedang mengelompokkan untuk film %s", title);
log_message(logmsg);
```
Membuat pesan log untuk mencatat bahwa program sedang mengelompokkan film berdasarkan abjad / judul yang akan disimpan dalam `log.txt`.

```c
char fname[20];
char first = title[0];
if (isalpha(first)) {
    snprintf(fname, sizeof(fname), "judul/%c.txt", toupper(first));
} else if (isdigit(first)) {
    snprintf(fname, sizeof(fname), "judul/%c.txt", first);
} else {
    snprintf(fname, sizeof(fname), "judul/#.txt");
}
```
Ini digunakan untuk menentukan nama file txt pada folder judul berdasarkan karakter pertamanya.

```c
FILE* out = fopen(fname, "a");
if (out) {
    fprintf(out, "%s - %s - %s\n", title, year_str, director);
    fclose(out);
}
```
Membuka file tujuan dengan mode append (a), lalu menuliskan informasi film, yaitu `Judul Film - Tahun Rilis - Sutradara`
```c
fclose(f);
pthread_exit(NULL);
```

Menutup file CSV dan keluar dari thread setelah selesai.

```c
void* organize_by_year(void* arg) {
    FILE* f = fopen(CSV_FILE, "r");
    if (!f) pthread_exit(NULL);

    mkdir("tahun", 0777);
    char line[1024];
    fgets(line, sizeof(line), f);

    while (fgets(line, sizeof(line), f)) {
        char* fields[10];
        PARSE_CSV(line, fields, 10);
        char* title = fields[0];
        char* director = fields[1];
        char* country = fields[2];
        char* year_str = fields[3];        
        if (!title || !year_str || !director) continue;

        year_str[strcspn(year_str, "\r\n")] = 0;

        char logmsg[256];
        snprintf(logmsg, sizeof(logmsg), "Proses mengelompokkan berdasarkan Tahun: sedang mengelompokkan untuk film %s", title);
        log_message(logmsg);

        char fname[64];
        snprintf(fname, sizeof(fname), "tahun/%s.txt", year_str);

        FILE* out = fopen(fname, "a");
        if (out) {
            fprintf(out, "%s - %s - %s\n", title, year_str, director);
            fclose(out);
        }
    }
    fclose(f);
    pthread_exit(NULL);
}
```

Fungsi organize_by_year bertugas mengelompokkan data film dari file CSV ke dalam file teks berdasarkan tahun rilisnya, lalu menyimpannya ke dalam folder bernama `tahun.txt`.

```c
void* organize_by_year(void* arg)
```
Fungsi ini dibuat untuk dijalankan secara paralel (multithreading). `arg` tidak digunakan, tapi harus tetap ada sesuai format fungsi thread (void* → pointer umum).

```c
FILE* f = fopen(CSV_FILE, "r");
if (!f) pthread_exit(NULL);
```
Membuka file CSV (`data/netflixData.csv`) dalam mode baca (`r`). Jika file tidak bisa dibuka, thread langsung keluar (`pthread_exit(NULL)`).

```c
mkdir("tahun", 0777);
```
Membuat folder baru bernama tahun untuk menyimpan file berdasarkan tahun, mode `0777` memberi izin penuh untuk semua pengguna.

```c
char line[1024];
fgets(line, sizeof(line), f);
```
Membaca baris pertama dari file CSV.

```c
while (fgets(line, sizeof(line), f)) {
```
Untuk membaca setiap baris berikutnya (isi data film) hingga habis.

```c
char* fields[10];
PARSE_CSV(line, fields, 10);
```
Memanggil fungsi `PARSE_CSV` untuk memecah baris menjadi bagian-bagian yang sudah diformat.

```c
char* title = fields[0];
char* director = fields[1];
char* country = fields[2];
char* year_str = fields[3];
```
Untuk menyimpan bagian data yang dibutuhkan ke dalam variabel.

```c
if (!title || !year_str || !director) continue;
```
Jika salah satu field kosong, baris tersebut dilewati.

```c
year_str[strcspn(year_str, "\r\n")] = 0;
```
Menghapus karakter newline (`\n` atau `\r`) di akhir string tahun agar tidak mengacaukan nama file.

```c
char logmsg[256];
snprintf(logmsg, sizeof(logmsg), "Proses mengelompokkan berdasarkan Tahun: sedang mengelompokkan untuk film %s", title);
log_message(logmsg);
```

Bagian ini digunakan untuk menuliskan log ke file `log.txt` untuk memberi tahu bahwa film sedang diproses.

```c
char fname[64];
snprintf(fname, sizeof(fname), "tahun/%s.txt", year_str);
```
Nama file ditentukan dari tahun film, misalnya: `tahun/2021.txt`

```c
FILE* out = fopen(fname, "a");
if (out) {
    fprintf(out, "%s - %s - %s\n", title, year_str, director);
    fclose(out);
}
```
Untuk membuka file tahun dalam mode append (`a`) agar bisa menambahkan data baru tanpa menghapus data sebelumnya dan menulis data film ke dalam file tersebut.

```c
fclose(f);
pthread_exit(NULL);
```
Menutup file CSV setelah semua baris selesai diproses dan mengakhiri thread.

```c
void tampilkan_menu() {
    printf("\n===== FILM ANTHONY =====\n");
    printf("1. Unduh dan Ekstrak ZIP\n");
    printf("2. Kelompokkan Film\n");
    printf("3. Buat Laporan\n");
    printf("0. Keluar\n");
    printf("Pilihan: ");
}
```
Fungsi `tampilkan_menu()` digunakan untuk menampilkan menu utama program di terminal agar pengguna bisa memilih apa yang ingin dilakukan (misalnya mengunduh file, mengelompokkan film, dll).

```c
void* unduh_dan_ekstrak(void* arg) {
    if (download_file() != 0) {
        pthread_exit(NULL);
    }

    printf("Ekstrak isi ZIP...\n");
    if (extract_zip() == 0) {
        printf("Berhasil diekstrak ke folder %s.\n", DATA_FOLDER);
        delete_zip();
        printf("File ZIP dihapus untuk menghemat ruang.\n");
    } else {
        printf("Gagal mengekstrak ZIP.\n");
    }
    pthread_exit(NULL);
}
```

Fungsi `unduh_dan_ekstrak()` bertugas untuk mengunduh file ZIP, mengekstraknya, lalu menghapus file ZIP-nya setelah selesai. Ini dijalankan sebagai thread agar proses berjalan secara paralel dengan proses lain.
Berikut ini adalah penjelasan lebih detailnya:
```c
void* unduh_dan_ekstrak(void* arg)
```
Fungsi ini didefinisikan sebagai fungsi thread (`void*`) dan menerima `arg` meskipun tidak digunakan (standar untuk `pthread_create`).

```c
if (download_file() != 0) {
    pthread_exit(NULL);
}
```
Menjalankan fungsi download_file() untuk mengunduh ZIP dari internet. Jika gagal (`!= 0`), maka thread dihentikan langsung dengan `pthread_exit(NULL)`.

```c
printf("Ekstrak isi ZIP...\n");
```
Akan mencetak pesan `Ekstrak isi ZIP...` yang menandakan psroses ekstrak file dimulai.

```c
if (extract_zip() == 0) {
```
Mengekstrak file ZIP yang sudah diunduh menggunakan fungsi `extract_zip()`. Kemudian mengecek apakah proses ekstraksi berhasil (`== 0` artinya sukses).

```c
printf("Berhasil diekstrak ke folder %s.\n", DATA_FOLDER);
```
Jika ekstraksi berhasil, tampilkan pesan sukses ke folder tujuan.

```c
delete_zip();
printf("File ZIP dihapus untuk menghemat ruang.\n");
```
Setelah berhasil diekstrak, file ZIP akan dihapus dari penyimpanan menggunakan fungsi `delete_zip()`.

```c
} else {
    printf("Gagal mengekstrak ZIP.\n");
}
```
Jika ekstraksi gagal, tampilkan pesan `Gagal mengekstrak ZIP.`.

```c
pthread_exit(NULL);
```
Menandakan bahwa thread sudah selesai.

```c
void* kelompokkan_film(void* arg) {
    pthread_t t1, t2;
    pthread_create(&t1, NULL, organize_by_tittle, NULL);
    pthread_create(&t2, NULL, organize_by_year, NULL);
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    printf("Film berhasil dikelompokkan berdasarkan abjad dan tahun.\n");
    pthread_exit(NULL);
}
```

Fungsi `kelompokkan_film()` digunakan untuk mengelompokkan data film secara paralel berdasarkan dua kriteria, yakni judul (abjad) dan tahun rilis. Fungsi ini menggunakan multithreading dengan pthread untuk melakukan kedua pengelompokan tersebut secara bersamaan (paralel), sehingga prosesnya bisa lebih cepat.
Berikut adalah penjelasan lebih lengkapnya:

```c
void* kelompokkan_film(void* arg)
```
Fungsi ini didefinisikan sebagai fungsi thread (`void* return` dan `void* arg` parameter) agar bisa dijalankan secara paralel menggunakan `pthread_create`.


```c
pthread_t t1, t2;
```
Mendeklarasikan dua variabel `pthread_t` yaitu `t1` dan `t2`, yang akan menyimpan ID thread yang dibuat.

```c
pthread_create(&t1, NULL, organize_by_tittle, NULL);
```
- Membuat thread pertama yang menjalankan fungsi organize_by_tittle (mengelompokkan film berdasarkan huruf awal judul).
- Parameter kedua (`NULL`) berarti menggunakan pengaturan default.
- Parameter terakhir `NULL` menunjukkan tidak ada data khusus yang dikirim ke fungsi tersebut.

```c
pthread_create(&t2, NULL, organize_by_year, NULL);
```
Membuat thread kedua untuk menjalankan fungsi `organize_by_year` (mengelompokkan film berdasarkan tahun rilis).

```c
pthread_join(t1, NULL);
pthread_join(t2, NULL);
```
Menunggu kedua thread (`t1` dan `t2`) selesai sebelum melanjutkan.

```c
printf("Film berhasil dikelompokkan berdasarkan abjad dan tahun.\n");
```
Pesan ini memberitahu bahwa tandanya pengelompokkan film sudah dilakukan.

```c
pthread_exit(NULL);
```
Menyelesaikan fungsi thread ini. Jika tidak ada `pthread_exit`, maka thread bisa selesai terlalu cepat dan menghentikan program utama.

```c
void* buat_laporan(void* arg) {
    pthread_t tid;
    pthread_create(&tid, NULL, generate_report, NULL);
    pthread_join(tid, NULL);
    pthread_exit(NULL);
}
```
Fungsi `buat_laporan` adalah fungsi thread yang bertugas untuk membuat laporan dengan menjalankan thread lain (`generate_report`) secara paralel. Berikut adalah penjelasan lebih lengkapnya:
```c
void* buat_laporan(void* arg)
```
Ini mendefinisikan fungsi `buat_laporan` sebagai fungsi untuk thread, yang menerima argumen `void* arg`. `void*` digunakan agar fungsi ini kompatibel dengan `pthread_create`.

```c
pthread_t tid;
```
Mendeklarasikan variabel tid bertipe `pthread_t`, yaitu variabel untuk menyimpan ID dari thread baru yang akan dibuat.

```c
pthread_create(&tid, NULL, generate_report, NULL);
```
Membuat thread baru untuk menjalankan fungsi `generate_report`.
- `&tid`: pointer ke variabel untuk menyimpan ID thread yang dibuat.
- `NULL`: pakai default thread attributes.
-   generate_report  : nama fungsi yang akan dijalankan di thread baru.
-   `NULL`: tidak ada argumen yang dikirim ke `generate_report`.


```c
pthread_join(tid, NULL);
```
Bagian ini akan menunggu sampai thread generate_report selesai dijalankan. Tanpa ini, thread `buat_laporan` bisa selesai duluan sebelum `generate_report` selesai.

```c
pthread_exit(NULL);
```
Menandakan bahwa thread `buat_laporan` telah selesai dijalankan dengan normal. `NULL` menunjukkan bahwa tidak ada nilai yang dikembalikan ke thread yang menunggu.

```c
void* generate_report(void* arg) {
    FILE* f = fopen(CSV_FILE, "r");
    if (!f) {
        perror("Failed to open CSV file");
        pthread_exit(NULL);
    }

    CountryStat stats[100] = {0};
    int stat_count = 0;

    char line[MAX_LINE_LENGTH];
    if (!fgets(line, sizeof(line), f)) {
        fclose(f);
        pthread_exit(NULL);
    }

    while (fgets(line, sizeof(line), f)) {
        char* fields[MAX_FIELDS] = {0};
        PARSE_CSV(line, fields, MAX_FIELDS);
        
        if (!fields[0] || !fields[1] || !fields[2] || !fields[3]) continue;

        int year = atoi(fields[3]);
        bool found = false;

        for (int i = 0; i < stat_count; i++) {
            if (strcmp(stats[i].country, fields[2]) == 0) {
                found = true;
                if (year < 2000) stats[i].before_2000++;
                else stats[i].after_2000++;
                break;
            }
        }

        if (!found && stat_count < 100) {
            strncpy(stats[stat_count].country, fields[2], sizeof(stats[0].country) - 1);
            if (year < 2000) {
                stats[stat_count].before_2000 = 1;
                stats[stat_count].after_2000 = 0;
            } else {
                stats[stat_count].before_2000 = 0;
                stats[stat_count].after_2000 = 1;
            }
            stat_count++;
        }
    }
    fclose(f);

    time_t now = time(NULL);
    struct tm *t = localtime(&now);
    char report_name[50];
    strftime(report_name, sizeof(report_name), "report_%d%m%Y.txt", t);

    FILE* report = fopen(report_name, "w");
    if (report) {
        for (int i = 0; i < stat_count; i++) {
            fprintf(report, "%d. Negara: %s\n", i+1, stats[i].country);
            fprintf(report, "Film sebelum 2000: %d\n", stats[i].before_2000);
            fprintf(report, "Film setelah 2000: %d\n\n", stats[i].after_2000);
        }
        fclose(report);
        printf("Report successfully created: %s\n", report_name);
    } else {
        perror("Failed to create report");
    }

    pthread_exit(NULL);
}
```
Fungsi `generate_report()` adalah fungsi yang menganalisis data film dari file CSV dan membuat laporan statistik jumlah film per negara yang dirilis sebelum dan sesudah tahun 2000. Laporan ini kemudian disimpan ke file teks dengan nama berdasarkan tanggal saat ini.
Berikut adalah penjelasan lebih detail:
```c
FILE* f = fopen(CSV_FILE, "r");
if (!f) {
    perror("Failed to open CSV file");
    pthread_exit(NULL);
}
```
Bagian ini akan membuka file CSV tempat data film disimpan. Jika gagal, cetak error dan keluar dari thread.

```c
CountryStat stats[100] = {0};
int stat_count = 0;
```
- `stats[100]`: array untuk menyimpan statistik maksimal 100 negara.
- `stat_count`: menghitung berapa negara unik yang sudah masuk ke statistik.

```c
char line[MAX_LINE_LENGTH];
if (!fgets(line, sizeof(line), f)) {
    fclose(f);
    pthread_exit(NULL);
}
```
Bagian ini digunakan untuk mengambil baris pertama, lalu abaikan.

```c
while (fgets(line, sizeof(line), f)) {
    char* fields[MAX_FIELDS] = {0};
    PARSE_CSV(line, fields, MAX_FIELDS);

    if (!fields[0] || !fields[1] || !fields[2] || !fields[3]) continue;
```
- Membaca setiap baris CSV.
- Pisahkan kolomnya ke `fields[]` menggunakan makro `PARSE_CSV`.
- Lewati baris jika ada fiels yang kosong (judul, sutradara, negara, tahun).


```c
    int year = atoi(fields[3]);
    bool found = false;

    for (int i = 0; i < stat_count; i++) {
        if (strcmp(stats[i].country, fields[2]) == 0) {
            found = true;
            if (year < 2000) stats[i].before_2000++;
            else stats[i].after_2000++;
            break;
        }
    }
```
- Ubah kolom tahun (`fields[3]`) ke integer.
- Periksa apakah negara (`fields[2]`) sudah ada di array statistik.
- Jika ada, tambahkan hitungannya ke before_2000 atau after_2000.

```c
    if (!found && stat_count < 100) {
        strncpy(stats[stat_count].country, fields[2], sizeof(stats[0].country) - 1);
        if (year < 2000) {
            stats[stat_count].before_2000 = 1;
            stats[stat_count].after_2000 = 0;
        } else {
            stats[stat_count].before_2000 = 0;
            stats[stat_count].after_2000 = 1;
        }
        stat_count++;
    }
}
fclose(f);
```
Jika negara belum ada di array maka tambahkan nama negaranya ke array, inisialisasi hitungan film berdasarkan tahun, tambah total `stat_count`.

```c
time_t now = time(NULL);
struct tm *t = localtime(&now);
char report_name[50];
strftime(report_name, sizeof(report_name), "report_%d%m%Y.txt", t);
```
Ambil waktu saat ini kemudian format nama file laporan menjadi `report_DDMMYYYY.txt`.

```c
FILE* report = fopen(report_name, "w");
if (report) {
    for (int i = 0; i < stat_count; i++) {
        fprintf(report, "%d. Negara: %s\n", i+1, stats[i].country);
        fprintf(report, "Film sebelum 2000: %d\n", stats[i].before_2000);
        fprintf(report, "Film setelah 2000: %d\n\n", stats[i].after_2000);
    }
    fclose(report);
    printf("Report successfully created: %s\n", report_name);
} else {
    perror("Failed to create report");
}
```
- Buka file laporan untuk ditulis.
- Tulis daftar negara, jumlah film sebelum dan sesudah 2000.
- Cetak pesan sukses jika berhasil dibuat.

```c
pthread_exit(NULL);
```
Ini menandakan bahwa thread `generate_report` sudah selesai dijalankan.


```c
void proses_pilihan(int pilihan) {
    pthread_t tid;

    switch (pilihan) {
        case 1:
            pthread_create(&tid, NULL, unduh_dan_ekstrak, NULL);
            pthread_join(tid, NULL);
            break;
        case 2:
            pthread_create(&tid, NULL, kelompokkan_film, NULL);
            pthread_join(tid, NULL);
            break;
        case 3:
            pthread_create(&tid, NULL, buat_laporan, NULL);
            pthread_join(tid, NULL);
            break;
        case 0:
            curl_global_cleanup();
            exit(0);
        default:
            printf("Pilihan tidak valid.\n");
    }
}
```

Fungsi proses_pilihan berfungsi untuk menangani pilihan menu dari pengguna dan menjalankan proses yang sesuai dengan pilihan tersebut menggunakan thread (pthread).
Berikut adalah penjelasannya:

```c
pthread_t tid;
```
Mendeklarasikan variabel `tid` yang bertipe `pthread_t`, digunakan untuk menyimpan ID thread yang akan dibuat.
```c
case 1:
    pthread_create(&tid, NULL, unduh_dan_ekstrak, NULL);
    pthread_join(tid, NULL);
    break;
```
- Jika pengguna memilih `1`, program:
  - Membuat thread baru untuk menjalankan fungsi `unduh_dan_ekstrak`.
  - `pthread_join` menunggu hingga thread tersebut selesai.

```c
case 2:
    pthread_create(&tid, NULL, kelompokkan_film, NULL);
    pthread_join(tid, NULL);
    break;
```
- Jika user memilih `2`, maka program menjalankan fungsi `kelompokkan_film` dalam thread.

```c
case 3:
    pthread_create(&tid, NULL, buat_laporan, NULL);
    pthread_join(tid, NULL);
    break;
```
Jika user memilih `3`, program menjalankan fungsi `buat_laporan`.

```c
case 0:
    curl_global_cleanup();
    exit(0);
```
Jika user memilih `0`, program membersihkan resource yang digunakan oleh libcurl (`curl_global_cleanup`).

```c
int main() {
    curl_global_init(CURL_GLOBAL_DEFAULT);

    int pilihan;
    while (1) {
        tampilkan_menu();
        scanf("%d", &pilihan);
        proses_pilihan(pilihan);
    }

    curl_global_cleanup();
    return 0;
}
```
Ini adalah pengendali program yang akan menjalankan fungsi utama program. Berikut adalah penjelasannya:

### Foto Hasil Output :
![image](https://github.com/user-attachments/assets/0d0abdf6-3897-45a2-bfe6-46f2a7e81020)
![image](https://github.com/user-attachments/assets/e413761f-bc9a-41f0-8b57-3e338ea6a24c)
![image](https://github.com/user-attachments/assets/2f09d828-2b93-4e9c-97af-14e980584f50)
![image](https://github.com/user-attachments/assets/5a373805-9aa7-4317-8609-5c1ae444ab66)
![image](https://github.com/user-attachments/assets/1df74eb9-28e2-4b4f-8ed5-fce0365f193f)
![image](https://github.com/user-attachments/assets/a2b5da17-ed19-487f-80e4-642f06fc2be9)
![image](https://github.com/user-attachments/assets/90bdbed4-5016-44d2-a6da-487322f83be3)
![image](https://github.com/user-attachments/assets/9b243e9c-e6e9-46d0-b855-dc3dfb474557)
![image](https://github.com/user-attachments/assets/de099695-a560-415e-af5d-b13ae0d19a16)
![image](https://github.com/user-attachments/assets/b58a46ed-55fc-4e49-844f-770361ff5613)
![image](https://github.com/user-attachments/assets/989fddbe-8f57-47c4-af98-b30da7a8a771)

---
