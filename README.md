# Organize and Analyze Anthony's Favorite Films

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
-Code lengkap:
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
