# Bagaimana Caranya untuk mengintegrasikan KernelSU ke kernel non GKI?

KernelSU dapat diintegrasikan ke kernel non GKI, dan saat ini sudah di-backport ke 4.14, dan juga dapat dijalankan pada kernel di bawah 4.14.

Karena fragmentasi kernel non GKI, kami tidak memiliki cara yang seragam untuk membangunnya, sehingga kami tidak dapat menyediakan gambar boot non GKI. Tetapi Anda dapat membangun kernel sendiri dengan KernelSU yang terintegrasi.

Pertama, Anda harus dapat membangun kernel yang dapat di-boot dari kode sumber kernel, jika kernel tersebut tidak open source, maka akan sulit untuk menjalankan KernelSU untuk perangkat Anda.

Jika Anda dapat membuat kernel yang dapat di-booting, ada dua cara untuk mengintegrasikan KernelSU ke kode sumber kernel:

1. Secara otomatis dengan `kprobe`
2. Secara manual

## Integrasikan dengan kprobe

KernelSU menggunakan kprobe untuk melakukan hook kernel, jika *kprobe* berjalan dengan baik pada kernel Anda, maka disarankan untuk menggunakan cara ini.

Pertama, tambahkan KernelSU ke dalam berkas kernel source tree:

```sh
curl -LSs "https://raw.githubusercontent.com/tiann/KernelSU/main/kernel/setup.sh" | bash -
```

Kemudian, Anda harus memeriksa apakah *kprobe* diaktifkan dalam konfigurasi kernel Anda, jika tidak, tambahkan konfigurasi ini ke dalamnya:

```
CONFIG_KPROBES=y
CONFIG_HAVE_KPROBES=y
CONFIG_KPROBE_EVENTS=y
```

Dan build kernel Anda lagi, KernelSU seharusnya bekerja dengan baik.

etapi jika Anda mengalami boot loop saat mengintegrasikan KernelSU, itu mungkin *kprobe rusak di kernel Anda*, Anda harus memperbaiki bug kprobe atau menggunakan cara kedua.

## Memodifikasi sumber kernel secara manual

Jika kprobe tidak dapat bekerja pada kernel Anda (mungkin karena bug di upstream atau kernel di bawah 4.8), maka Anda dapat mencoba cara ini:

Pertama, tambahkan KernelSU ke dalam direktori kernel source tree:

```sh
curl -LSs "https://raw.githubusercontent.com/tiann/KernelSU/main/kernel/setup.sh" | bash -
```

Kemudian, tambahkan panggilan KernelSU ke source kernel, berikut ini adalah patch yang dapat dirujuk:

```diff
diff --git a/fs/exec.c b/fs/exec.c
index ac59664eaecf..bdd585e1d2cc 100644
--- a/fs/exec.c
+++ b/fs/exec.c
@@ -1890,11 +1890,14 @@ static int __do_execve_file(int fd, struct filename *filename,
 	return retval;
 }

+extern int ksu_handle_execveat(int *fd, struct filename **filename_ptr, void *argv,
+			void *envp, int *flags);
 static int do_execveat_common(int fd, struct filename *filename,
 			      struct user_arg_ptr argv,
 			      struct user_arg_ptr envp,
 			      int flags)
 {
+	ksu_handle_execveat(&fd, &filename, &argv, &envp, &flags);
 	return __do_execve_file(fd, filename, argv, envp, flags, NULL);
 }

diff --git a/fs/open.c b/fs/open.c
index 05036d819197..965b84d486b8 100644
--- a/fs/open.c
+++ b/fs/open.c
@@ -348,6 +348,8 @@ SYSCALL_DEFINE4(fallocate, int, fd, int, mode, loff_t, offset, loff_t, len)
 	return ksys_fallocate(fd, mode, offset, len);
 }

+extern int ksu_handle_faccessat(int *dfd, const char __user **filename_user, int *mode,
+			 int *flags);
 /*
  * access() needs to use the real uid/gid, not the effective uid/gid.
  * We do this by temporarily clearing all FS-related capabilities and
@@ -355,6 +357,7 @@ SYSCALL_DEFINE4(fallocate, int, fd, int, mode, loff_t, offset, loff_t, len)
  */
 long do_faccessat(int dfd, const char __user *filename, int mode)
 {
+	ksu_handle_faccessat(&dfd, &filename, &mode, NULL);
 	const struct cred *old_cred;
 	struct cred *override_cred;
 	struct path path;
diff --git a/fs/read_write.c b/fs/read_write.c
index 650fc7e0f3a6..55be193913b6 100644
--- a/fs/read_write.c
+++ b/fs/read_write.c
@@ -434,10 +434,14 @@ ssize_t kernel_read(struct file *file, void *buf, size_t count, loff_t *pos)
 }
 EXPORT_SYMBOL(kernel_read);

+extern int ksu_handle_vfs_read(struct file **file_ptr, char __user **buf_ptr,
+			size_t *count_ptr, loff_t **pos);
 ssize_t vfs_read(struct file *file, char __user *buf, size_t count, loff_t *pos)
 {
 	ssize_t ret;

+	ksu_handle_vfs_read(&file, &buf, &count, &pos);
+
 	if (!(file->f_mode & FMODE_READ))
 		return -EBADF;
 	if (!(file->f_mode & FMODE_CAN_READ))
diff --git a/fs/stat.c b/fs/stat.c
index 376543199b5a..82adcef03ecc 100644
--- a/fs/stat.c
+++ b/fs/stat.c
@@ -148,6 +148,8 @@ int vfs_statx_fd(unsigned int fd, struct kstat *stat,
 }
 EXPORT_SYMBOL(vfs_statx_fd);

+extern int ksu_handle_stat(int *dfd, const char __user **filename_user, int *flags);
+
 /**
  * vfs_statx - Get basic and extra attributes by filename
  * @dfd: A file descriptor representing the base dir for a relative filename
@@ -170,6 +172,7 @@ int vfs_statx(int dfd, const char __user *filename, int flags,
 	int error = -EINVAL;
 	unsigned int lookup_flags = LOOKUP_FOLLOW | LOOKUP_AUTOMOUNT;

+	ksu_handle_stat(&dfd, &filename, &flags);
 	if ((flags & ~(AT_SYMLINK_NOFOLLOW | AT_NO_AUTOMOUNT |
 		       AT_EMPTY_PATH | KSTAT_QUERY_FLAGS)) != 0)
 		return -EINVAL;
```

Anda harus menemukan empat fungsi dalam kernel source:

1. do_faccessat, usually in `fs/open.c`
2. do_execveat_common, usually in `fs/exec.c`
3. vfs_read, usually in `fs/read_write.c`
4. vfs_statx, usually in `fs/stat.c`

Terakhir, edit `KernelSU/kernel/ksu.c` dan beri komentar pada `enable_sucompat()` lalu build kernel Anda lagi, KernelSU akan bekerja dengan baik.
