  # Panduan Lengkap: Konfigurasi SSL/TLS Certificate pada Microsoft SQL Server

  Dokumentasi ini merangkum cara kerja enkripsi, penggunaan sertifikat (khususnya *Self-Signed Certificate*), serta langkah-langkah praktis untuk mengamankan koneksi pada **Microsoft SQL Server**.

  ---

  ![Cek hostname server](design/arsitekur.jpg)


  ## Daftar Isi
- [Panduan Lengkap: Konfigurasi SSL/TLS Certificate pada Microsoft SQL Server](#panduan-lengkap-konfigurasi-ssltls-certificate-pada-microsoft-sql-server)
  - [Daftar Isi](#daftar-isi)
  - [1. Konsep Dasar \& Protokol](#1-konsep-dasar--protokol)
  - [2. Anatomi Self-Signed Certificate](#2-anatomi-self-signed-certificate)
  - [3. Prasyarat \& Persiapan](#3-prasyarat--persiapan)
  - [4. Langkah 1: Membuat Self-Signed Certificate via PowerShell](#4-langkah-1-membuat-self-signed-certificate-via-powershell)
  - [5. Langkah 2: Memberikan Akses Service Account SQL Server](#5-langkah-2-memberikan-akses-service-account-sql-server)
  - [6. Langkah 3: Mendaftarkan Sertifikat ke SQL Server Configuration Manager](#6-langkah-3-mendaftarkan-sertifikat-ke-sql-server-configuration-manager)
  - [7. Langkah 4: Konfigurasi Sisi Klien (Client)](#7-langkah-4-konfigurasi-sisi-klien-client)
    - [Opsi A: Menggunakan Parameter `TrustServerCertificate=True` (Paling Praktis)](#opsi-a-menggunakan-parameter-trustservercertificatetrue-paling-praktis)
    - [Opsi B: Mengimpor Sertifikat ke Trusted Root Client (Tanpa Ubah Connection String)](#opsi-b-mengimpor-sertifikat-ke-trusted-root-client-tanpa-ubah-connection-string)
  - [8. Konfigrasi Pendekatan Server-Level untuk Force Enkripsi SSL SQL Server untuk semua conection](#8-konfigrasi-pendekatan-server-level-untuk-force-enkripsi-ssl-sql-server-untuk-semua-conection)
  - [9. FAQ / Tanya Jawab Seputar Enkripsi SQL Server](#9-faq--tanya-jawab-seputar-enkripsi-sql-server)
  - [10. Troubleshooting: "certificate chain was issued by an authority that is not trusted"](#10-troubleshooting-certificate-chain-was-issued-by-an-authority-that-is-not-trusted)

  ---

  ## 1. Konsep Dasar & Protokol

  Komunikasi terenkripsi antara aplikasi klien dan SQL Server menggunakan dua lapisan utama:
  * **TDS (Tabular Data Stream):** Protokol aplikasi dasar yang digunakan oleh driver Microsoft SQL Server untuk berkomunikasi dengan database engine.
  * **TLS (Transport Layer Security):** Protokol standar industri yang melapisi TDS untuk mengenkripsi seluruh data (query, parameter, hasil data, password) agar aman dari serangan *Man-in-the-Middle* (Sniffing).

  ---

  ## 2. Anatomi Self-Signed Certificate

  Berbeda dengan sertifikat dari CA (Certificate Authority) publik atau perusahaan yang memiliki hirarki ketat (Root CA $\rightarrow$ Intermediate CA $\rightarrow$ Server Cert), **Self-Signed Certificate** memiliki karakteristik berikut:
  * **Root/Intermediate CA:** Tidak ada. Komputer lokal bertindak sebagai pembuat sekaligus yang mempercayainya.
  * **Private Key:** Tersedia dan tersimpan aman di sistem operasi Windows (`Cert:\LocalMachine\My`). Berfungsi untuk mengenkripsi dan mendekripsi data.
  * **Domain / Hostname (`-DnsName`):** Ditentukan saat pembuatan sertifikat. Nama server atau IP yang Anda daftarkan di sini harus sesuai dengan yang dipanggil oleh klien pada *connection string* (kecuali jika mengabaikan validasi via parameter klien).

  ---

  ## 3. Prasyarat & Persiapan
  * Akses Administrator ke mesin server Windows tempat SQL Server diinstal.
  * PowerShell versi terbaru.
  * SQL Server Configuration Manager.

  Sebelum mulai, pastikan **hostname** dan **IP address** server sudah sesuai. Catat nilainya karena akan dipakai pada parameter `-DnsName` di Langkah 1.

  ![Cek hostname server](screenshoot/1-check-host.png)

  ![Cek IP address server](screenshoot/2-check-ip.png)

  ---

  ## 4. Langkah 1: Membuat Self-Signed Certificate via PowerShell

  1. Buka **PowerShell** sebagai **Administrator**.
  2. Jalankan skrip berikut (sesuaikan nama komputer, FQDN, atau IP server Anda pada parameter `-DnsName`):

  ![Membuat self-signed certificate via PowerShell](screenshoot/3-create-certificate.png)

  ```powershell
  New-SelfSignedCertificate `
      -CertStoreLocation "Cert:\LocalMachine\My" `
      -DnsName "Dev01", "localhost", "192.168.100.35" `
      -KeySpec KeyExchange `
      -TextExtension @("2.5.29.37={text}1.3.6.1.5.5.7.3.1") `
      -FriendlyName "SQLServerSelfSignedCert"
  ```

  -TextExtension @("2.5.29.37={text}1.3.6.1.5.5.7.3.1")?

  Parameter ini digunakan untuk menambahkan EKU (Enhanced Key Usage) atau ekstensi teks khusus ke dalam sertifikat.

  Mari kita bedah artinya:
  2.5.29.37: Ini adalah Object Identifier (OID) standar untuk ekstensi Enhanced Key Usage.
  1.3.6.1.5.5.7.3.1: Ini adalah OID spesifik untuk Server Authentication (Autentikasi Server).

  Mengapa ini wajib untuk SQL Server?

  Secara default, jika Anda membuat sertifikat self-signed tanpa parameter ini, Windows/PowerShell bisa jadi membuat sertifikat umum yang tidak dikhususkan untuk mengamankan server web/database. 

  Dengan menambahkan baris -TextExtension, kita memaksa sertifikat tersebut untuk mendeklarasikan: "Sertifikat ini sah dan dirancang khusus untuk digunakan sebagai identitas server (Server Authentication)"

  ---

  ## 5. Langkah 2: Memberikan Akses Service Account SQL Server

  Agar layanan SQL Server bisa membaca sertifikat tersebut, Anda perlu memberikan izin akses (*Read*):

  1. Tekan tombol `Windows + R`, ketik `certlm.msc`, lalu tekan **Enter**.
  2. Arahkan ke folder **Personal** > **Certificates**.
  3. Cari sertifikat dengan *Friendly Name* `SQLServerSelfSignedCert`.
  4. Klik kanan $\rightarrow$ **All Tasks** $\rightarrow$ **Manage Private Keys...**
  5. Klik **Add**, lalu masukkan akun yang menjalankan layanan SQL Server (default umumnya: `NT Service\MSSQLSERVER` atau `Network Service`).
  6. Berikan izin **Read**, lalu klik **Apply** dan **OK**.

  **Buka `certlm.msc`** lalu arahkan ke **Personal** > **Certificates** untuk menemukan sertifikat `SQLServerSelfSignedCert`:

  ![Membuka Certificate Manager (certlm.msc)](screenshoot/4-memberikan-akses-service-account.png)

  **Klik kanan sertifikat** $\rightarrow$ **All Tasks** $\rightarrow$ **Manage Private Keys...**:

  ![Menu All Tasks pada sertifikat](screenshoot/5-memberikan-akses-service-account-certifcate-2.png)

  ![Properties sertifikat](screenshoot/6-memberikan-akses-service-account-certifcate-properties.png)

  **Manage Private Keys** — tambahkan akun service SQL Server, berikan izin **Read**:

  ![Manage Private Keys - dialog permissions](screenshoot/7-memberikan-akses-service-account-certifcate-manage-private-key.png)

  ![Menambahkan service account](screenshoot/8-memberikan-akses-service-account-certifcate-manage-private-key-2.png)

  ![Memberikan izin Read untuk service account](screenshoot/9-memberikan-akses-service-account-certifcate-manage-private-key-3.png)

  ---

  ## 6. Langkah 3: Mendaftarkan Sertifikat ke SQL Server Configuration Manager

  1. Buka aplikasi **SQL Server Configuration Manager**.
  2. Di panel kiri, pilih **SQL Server Network Configuration** $\rightarrow$ klik kanan pada **Protocols for [Nama Instance Anda]** $\rightarrow$ pilih **Properties**.
  3. Pindah ke tab **Certificate**.
  4. Pilih sertifikat `SQLServerSelfSignedCert` dari dropdown yang tersedia.
  5. Klik **Apply** dan **OK**.
  6. **Restart Layanan:** Masuk ke **SQL Server Services**, klik kanan pada SQL Server instance Anda, lalu pilih **Restart**.

  **Buka SQL Server Configuration Manager** $\rightarrow$ **SQL Server Network Configuration** $\rightarrow$ **Protocols for MSSQLSERVER**:

  ![SQL Server Network Configuration - Protocols for MSSQLSERVER](screenshoot/10-mendaftarkan-sertifikat-ke-SQL-Server-Configuration-Manager.png)

  **Pilih tab Certificate** lalu pilih `SQLServerSelfSignedCert` dari dropdown:

  ![Memilih sertifikat pada tab Certificate](screenshoot/11-mendaftarkan-sertifikat-ke-SQL-Server-Configuration-Manager-2.png)

  **Verifikasi detail sertifikat** (Issued By/To: Dev01, masa berlaku):

  ![Verifikasi detail sertifikat](screenshoot/12-mendaftarkan-sertifikat-ke-SQL-Server-Configuration-Manager-3.png)

  > ⚠️ **Catatan:** Sertifikat ini adalah *self-signed* / CA Root yang belum dipercaya (`This CA Root certificate is not trusted`). Ini normal dan akan diselesaikan pada Langkah 4 di sisi klien.

  **Restart layanan SQL Server** setelah menerapkan sertifikat:

  ![Restart layanan SQL Server (MSSQLSERVER)](screenshoot/13-mendaftarkan-sertifikat-ke-SQL-Server-Configuration-Manager-resttart.png)

  ---

  ## 7. Langkah 4: Konfigurasi Sisi Klien (Client)

  Karena menggunakan *Self-Signed Certificate*, ada dua pendekatan yang bisa diambil pada sisi aplikasi klien:

  ### Opsi A: Menggunakan Parameter `TrustServerCertificate=True` (Paling Praktis)
  Tambahkan `TrustServerCertificate=True` di dalam *connection string* aplikasi Anda. 
  * *Contoh (.NET):*
    ```text
    Server=NAMA_KOMPUTER;Database=MyDb;User Id=sa;Password=myPassword;TrustServerCertificate=True;
    ```
  * **Catatan:** Enkripsi **tetap berjalan 100%**, namun klien mengabaikan validasi apakah sertifikat tersebut berasal dari CA resmi atau apakah nama domainnya cocok.

  **Uji koneksi dari SQL Server Management Studio (SSMS)** dengan **Trust Server Certificate di-uncheck** — koneksi berhasil karena sertifikat sudah terpasang di server:

  ![Test koneksi SSMS dengan Trust Server Certificate di-uncheck](screenshoot/24-test-dari-client-menggunakan-sqlserver-studio-trust-certificate-di-uncheck.png)

  ![Berhasil login dari SSMS](screenshoot/25-test-dari-client-menggunakan-sqlserver-studio-trust-certificate-di-uncheck-berhasil-login.png)

  ![Verifikasi enkripsi aktif setelah login](screenshoot/25-test-dari-client-menggunakan-sqlserver-studio-trust-certificate-di-uncheck-berhasil-login-dan-check-enkripsi.png)

  **Uji dari aplikasi (ASP / ODBC Driver 17)** menggunakan `Encrypt=yes;` pada connection string:

  ![Test koneksi dari aplikasi ASP dengan Encrypt=yes](screenshoot/26-test-dari-client-menggunakan-aplikasi-asp.png)

  ### Opsi B: Mengimpor Sertifikat ke Trusted Root Client (Tanpa Ubah Connection String)
  Jika kebijakan keamanan melarang penggunaan `TrustServerCertificate=True`:
  1. **Export dari Server:** Buka `certlm.msc` di server, klik kanan sertifikat $\rightarrow$ **All Tasks** $\rightarrow$ **Export** (Format: DER encoded binary X.509 .CER).
  2. **Import ke Klien:** Pindahkan file `.cer` ke komputer klien, buka, klik **Install Certificate** $\rightarrow$ **Local Machine** $\rightarrow$ Pilih **Trusted Root Certification Authorities**.

  **Langkah 1 — Export CA Root dari server** (klik kanan sertifikat $\rightarrow$ **All Tasks** $\rightarrow$ **Export**):

  ![Export CA Root dari server](screenshoot/14-export-ca-root-dari-server-to-client.png)

  ![Export Wizard - pilih format DER encoded binary X.509 (.CER)](screenshoot/15-export-ca-root-dari-server-to-client-no-private-key.png)

  ![Export - tidak menyertakan private key](screenshoot/16-export-ca-root-dari-server-to-client-no-private-key-2.png)

  ![Menentukan lokasi file .cer hasil export](screenshoot/17-export-ca-root-dari-server-to-client-no-private-key-3.png)

  ![Export berhasil diselesaikan](screenshoot/18-export-ca-root-dari-server-to-client-no-private-key-4.png)

  **Langkah 2 — Import CA Root ke komputer klien.** Buka file `.cer` $\rightarrow$ **Install Certificate** $\rightarrow$ **Local Machine**:

  ![Memulai wizard Import Certificate di klien](screenshoot/19-import-ca-root-ke-client.png)

  ![Memilih Store Location: Local Machine](screenshoot/20-import-ca-root-ke-client-2.png)

  ![Memilih Trusted Root Certification Authorities](screenshoot/21-import-ca-root-ke-client-3.png)

  ![Konfirmasi pemasangan sertifikat](screenshoot/21-import-ca-root-ke-client-4.png)

  ![Import sertifikat ke Trusted Root berhasil](screenshoot/22-import-ca-root-ke-client-success.png)

  **Verifikasi:** Buka sertifikat di **Trusted Root Certification Authorities** untuk memastikan statusnya sudah *trusted*:

  ![Verifikasi Certificate Information - sertifikat sudah trusted](screenshoot/23-import-ca-root-ke-client-success-check-certificate-information.png)

  ---



  ## 8. Konfigrasi Pendekatan Server-Level untuk Force Enkripsi SSL SQL Server untuk semua conection

  Memastikan seluruh jalur komunikasi data antara aplikasi (seperti IIS/ASP, Azure Data Studio, dll.) dan SQL Server terlindungi oleh enkripsi SSL/TLS.
  Pendekatan Server-Level: Mengaktifkan kebijakan Force Encryption di server agar pemaksaan enkripsi ditangani langsung oleh SQL Server.

  Keuntungan: Mencegah modifikasi kode secara manual pada banyak aplikasi lama (legacy app) karena server secara otomatis menuntut jalur aman untuk setiap koneksi yang masuk.

  **Konfigurasi Server di SQL Server Configuration Manager**

  1.1 Buka SQL Server Configuration Manager pada server database.
  1.2 Masuk ke menu SQL Server Network Configuration > Protocols for MSSQLSERVER (klik kanan lalu pilih Properties).
  1.3 Pada tab Certificate, pilih sertifikat server yang telah diimpor sebelumnya.
  1.4 Pada tab Flags, ubah opsi Force Encryption dari No menjadi Yes.

  (screenshoot/29-config-sqlserver-force-engkripsi-for-client.png)

  **Verifikasi**
  Untuk memastikan bahwa seluruh koneksi yang masuk ke database benar-benar telah terenkripsi secara otomatis oleh server, jalankan query verifikasi berikut melalui SQL Server Management Studio (SSMS) atau Azure Data Studio:

  (screenshoot/30-config-sqlserver-force-engkripsi-for-client.png)


  ## 9. FAQ / Tanya Jawab Seputar Enkripsi SQL Server

  * **Q: Apakah enkripsi tetap aktif jika menggunakan `TrustServerCertificate=True`?**
    * **A:** Ya, tetap aktif sepenuhnya. Data Anda tetap terlindungi dari penyadapan jaringan (sniffing) karena proses *TLS Handshake* dan enkripsi trafik tetap berjalan.
  * **Q: Apa fungsi `-DnsName` saat membuat sertifikat?**
    * **A:** Sebagai identitas atau domain server yang dicocokkan oleh klien saat koneksi. Jika Anda tidak menggunakan `TrustServerCertificate=True`, nama server pada *connection string* wajib sama persis dengan salah satu entri di `-DnsName` tersebut.

  ---

  ## 10. Troubleshooting: "certificate chain was issued by an authority that is not trusted"

  Jika CA Root **belum** diimpor ke komputer klien (Opsi B belum dijalankan), koneksi akan gagal dengan error berikut:

  ![Error SSMS: certificate chain was issued by an authority that is not trusted](screenshoot/27-test-dari-client-apabila-ca-root-belum-ada.png)

  ![Error aplikasi ASP: SSL Provider - certificate chain is not trusted](screenshoot/28-test-dari-client-apabila-ca-root-belum-ada-dari-aplikasi.png)

  **Penyebab & Solusi:**
  * **Penyebab:** Sertifikat server adalah *self-signed* dan CA Root-nya belum dipercaya oleh klien.
  * **Solusi 1:** Impor CA Root ke **Trusted Root Certification Authorities** di klien (lihat [Opsi B](#opsi-b-mengimpor-sertifikat-ke-trusted-root-client-tanpa-ubah-connection-string)).
  * **Solusi 2:** Tambahkan `TrustServerCertificate=True` pada *connection string* (atau `Encrypt=yes;` + `TrustServerCertificate=yes;` untuk ODBC) — lihat [Opsi A](#opsi-a-menggunakan-parameter-trustservercertificatetrue-paling-praktis).