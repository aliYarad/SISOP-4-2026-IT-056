# SISOP-4-2026-IT-056
## Laporan Resmi
**Praktikum Sistem Operasi 2026 Modul 4 - Filesystem & Docker**  

---
**Nama : Aliya Rahmadina**  
**NRP : 5027251056**   

---
### _Soal 1_
#### _Deskripsi Soal_
Pada soal 1 diperintahkan untuk membuat program FUSE `kenz_rescue.c` yang menerima argumen `<source_directory>` dan `<mount_directory>` sekaligus membuat file virtual `tujuan.txt`.

#### _Penjelasan Kode_
Menyimpan path absolut dari direktori sumber (amba_files/) yang akan di-mirror.
```
static char source_dir[PATH_MAX];
```
Nama file virtual yang hanya ada di mount point, tidak di disk asli.
```
#define VIRTUAL_FILE "tujuan.txt"
```
Menentukan jumlah file log yang akan dibaca untuk menyusun koordinat.
```                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             kenz_rescue.c
#define NUM_SRC_FILES 7
```
Fungsi helper untuk membangun path lengkap ke file di direktori sumber dengan cara menggabungkan `source_dir` dan path relatif dari FUSE.
```
static void build_source_path(char *fpath, const char *path)
{
    snprintf(fpath, PATH_MAX, "%s%s", source_dir, path);
}
```
`build_virtual_content` membangun isi file `tujuan.txt` secara dinamis (on-the-fly) dengan membaca fragmen KOORD: dari setiap file 1.txt sampai 7.txt secara berurutan, lalu menggabungkannya menjadi satu baris output.
```
static char *build_virtual_content(size_t *out_size)
{
    const char *prefix = "Tujuan Mas Amba: ";
    size_t prefix_len  = strlen(prefix);

    size_t total  = prefix_len;
    char  *result = malloc(prefix_len + 1);
    if (!result) { *out_size = 0; return NULL; }
    memcpy(result, prefix, prefix_len);

    for (int i = 1; i <= NUM_SRC_FILES; i++) {
        char fpath[PATH_MAX * 2];
        snprintf(fpath, sizeof(fpath), "%s/%d.txt", source_dir, i);

        FILE *f = fopen(fpath, "r");
        if (!f) continue;

        char line[1024];
        while (fgets(line, sizeof(line), f)) {
            char *koord_pos = strstr(line, "KOORD: ");
            if (koord_pos) {
                char *value = koord_pos + strlen("KOORD: ");

                size_t vlen = strlen(value);
                while (vlen > 0 && (value[vlen-1] == '\n' || value[vlen-1] == '\r'))
                    vlen--;

                size_t add_len = vlen;
                if (total > prefix_len) {
                    result = realloc(result, total + add_len + 2);
                    if (!result) { fclose(f); *out_size = 0; return NULL; }
                    result[total] = ' ';
                    total += 1;
                } else {
                    result = realloc(result, total + add_len + 1);
                    if (!result) { fclose(f); *out_size = 0; return NULL; }
                }

                memcpy(result + total, value, vlen);
                total += vlen;
                break;
            }
        }
        fclose(f);
    }

    result = realloc(result, total + 2);
    if (result) {
        result[total] = '\n';
        total += 1;
        result[total] = '\0';
    }

    *out_size = total;
    return result;
}
```
Callback FUSE yang dipanggil setiap kali sistem perlu mengambil metadata (seperti ukuran, permission, tipe file) dari sebuah path. Setara dengan syscall `stat()`.
```
static int kenz_getattr(const char *path, struct stat *stbuf,
                        struct fuse_file_info *fi)
{
    (void) fi;
    memset(stbuf, 0, sizeof(struct stat));

    if (strcmp(path, "/") == 0) {
        stbuf->st_mode  = S_IFDIR | 0755;
        stbuf->st_nlink = 2;
        return 0;
    }

    if (strcmp(path, "/" VIRTUAL_FILE) == 0) {
        size_t vsize = 0;
        char  *vcontent = build_virtual_content(&vsize);
        if (vcontent) free(vcontent);

        stbuf->st_mode  = S_IFREG | 0444;
        stbuf->st_nlink = 1;
        stbuf->st_size  = (off_t)vsize;
        return 0;
    }

    char fpath[PATH_MAX];
    build_source_path(fpath, path);

    int res = lstat(fpath, stbuf);
    if (res == -1)
        return -errno;

    stbuf->st_mode &= ~(S_IWUSR | S_IWGRP | S_IWOTH);

    return 0;
}
```
Callback FUSE yang dipanggil saat direktori di-listing (misal `ls`). Mengisi buffer dengan daftar nama file yang ada di mount point.
```
static int kenz_readdir(const char *path, void *buf, fuse_fill_dir_t filler,
                        off_t offset, struct fuse_file_info *fi,
                        enum fuse_readdir_flags flags)
{
    (void) offset;
    (void) fi;
    (void) flags;

    if (strcmp(path, "/") != 0)
        return -ENOENT;

    filler(buf, ".",  NULL, 0, 0);
    filler(buf, "..", NULL, 0, 0);

    DIR *dp = opendir(source_dir);
    if (!dp)
        return -errno;

    struct dirent *de;
    while ((de = readdir(dp)) != NULL) {
        if (strcmp(de->d_name, ".")  == 0) continue;
        if (strcmp(de->d_name, "..") == 0) continue;

        struct stat st;
        memset(&st, 0, sizeof(st));
        st.st_ino  = de->d_ino;
        st.st_mode = de->d_type << 12;
        filler(buf, de->d_name, &st, 0, 0);
    }
    closedir(dp);

    struct stat vst;
    memset(&vst, 0, sizeof(vst));
    vst.st_mode = S_IFREG | 0444;
    filler(buf, VIRTUAL_FILE, &vst, 0, 0);

    return 0;
}
```
Callback FUSE yang dipanggil saat sebuah file akan dibuka. Berfungsi sebagai validasi sebelum operasi `read` dilakukan.
```
static int kenz_open(const char *path, struct fuse_file_info *fi)
{
    if ((fi->flags & O_ACCMODE) != O_RDONLY)
        return -EACCES;

    if (strcmp(path, "/" VIRTUAL_FILE) == 0)
        return 0;

    char fpath[PATH_MAX];
    build_source_path(fpath, path);

    int fd = open(fpath, O_RDONLY);
    if (fd == -1)
        return -errno;

    close(fd);
    return 0;
}
```
Callback FUSE yang dipanggil saat isi file dibaca, misalnya `cat`.
```
static int kenz_read(const char *path, char *buf, size_t size,
                     off_t offset, struct fuse_file_info *fi)
{
    (void) fi;

    if (strcmp(path, "/" VIRTUAL_FILE) == 0) {
        size_t   vsize    = 0;
        char    *vcontent = build_virtual_content(&vsize);

        if (!vcontent)
            return -ENOMEM;

        if ((size_t)offset >= vsize) {
            free(vcontent);
            return 0;
        }

        size_t bytes_available = vsize - (size_t)offset;
        size_t bytes_to_copy   = (size < bytes_available) ? size : bytes_available;

        memcpy(buf, vcontent + offset, bytes_to_copy);
        free(vcontent);
        return (int)bytes_to_copy;
    }

    char fpath[PATH_MAX];
    build_source_path(fpath, path);

    int fd = open(fpath, O_RDONLY);
    if (fd == -1)
        return -errno;

    int res = pread(fd, buf, size, offset);
    if (res == -1)
        res = -errno;

    close(fd);
    return res;
}

static const struct fuse_operations kenz_oper = {
    .getattr = kenz_getattr,
    .readdir = kenz_readdir,
    .open    = kenz_open,
    .read    = kenz_read,
};
```
Memvalidasi dua argumen diberikan yaitu`<source_directory>` dan `<mount_directory>` dan memanggil `fuse_main()` dengan struct `kenz_oper` yang berisi pointer ke keempat callback.
```
int main(int argc, char *argv[])
{
    if (argc < 3) {
        fprintf(stderr, "Usage: %s <source_directory> <mount_directory>\n",
                argv[0]);
        return 1;
    }

    if (realpath(argv[1], source_dir) == NULL) {
        perror("realpath");
        return 1;
    }

    for (int i = 1; i < argc - 1; i++) {
        argv[i] = argv[i + 1];
    }
    argc--;

    umask(0);
    return fuse_main(argc, argv, &kenz_oper, NULL);
}
```

#### _Output_
<img width="1471" height="233" alt="Screenshot 2026-05-17 164705" src="https://github.com/user-attachments/assets/fa0e9f50-f5b7-490f-9148-8011d5f9f72f" />
<img width="1460" height="47" alt="Screenshot 2026-05-17 165012" src="https://github.com/user-attachments/assets/9d6df7a9-bcff-4fc4-a69c-72389348cba8" />
<img width="1468" height="206" alt="Screenshot 2026-05-17 164928" src="https://github.com/user-attachments/assets/763d8977-9b70-4f67-b7e0-1693ee232917" />
<img width="1471" height="95" alt="Screenshot 2026-05-17 164853" src="https://github.com/user-attachments/assets/e52a3d47-e979-4f83-9ebb-d6f7d47494d4" />
<img width="1475" height="210" alt="Screenshot 2026-05-17 164817" src="https://github.com/user-attachments/assets/8316366d-9032-44a3-9897-fd100b6b15b0" />

