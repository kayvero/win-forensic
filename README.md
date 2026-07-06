# Windows Forensics 1 — TryHackMe Writeup (Lengkap)

**Room:** https://tryhackme.com/room/windowsforensics1
**Artikel acuan:** *TryHackMe: Windows Forensics 1 - Detailed Write-Up* oleh Cindy (Shunxian) Ou

> Catatan pribadi lengkap per-task, ditambah penjelasan konsep & tools secara lebih rinci (Windows Registry, akuisisi, tools EZtools, artefak evidence-of-execution, USB forensics). Windows jadi OS yang paling sering muncul di kasus forensik karena penggunaannya luas — makin paham struktur registry & artefaknya, makin cepat bisa merekonstruksi aktivitas user di sistem.

---

## Task 1 — Introduction to Windows Forensics

**Apa itu Computer Forensics?**
Computer forensics adalah bagian dari Digital Forensics yang fokus mengumpulkan bukti aktivitas yang dilakukan di komputer. Penerapannya luas: dari ranah hukum (mendukung/membantah hipotesis dalam kasus pidana/perdata) sampai ranah privat (investigasi internal perusahaan, analisis insiden & intrusi keamanan).

**Contoh kasus nyata — BTK Serial Killer:**
Salah satu contoh terkenal soal peran digital forensics dalam kasus kriminal adalah kasus pembunuh berantai BTK. Kasus ini sempat mandek bertahun-tahun sampai akhirnya sang pelaku mulai "memancing" polisi lewat surat-surat. Titik baliknya adalah saat ia mengirim floppy disk ke stasiun berita lokal, yang kemudian diserahkan ke polisi sebagai barang bukti. Dari disk itu, penyidik berhasil memulihkan dokumen Word yang sudah dihapus — dan dari metadata dokumen tersebut (beserta bukti pendukung lain), polisi berhasil melacak dan menangkap pelakunya.

**Kenapa fokus ke Windows?**
Microsoft Windows menguasai sekitar 80% pangsa pasar desktop OS — baik di kalangan personal maupun enterprise. Karena dominasinya ini, kemampuan melakukan analisis forensik di Windows jadi skill yang sangat penting buat siapa pun yang berkecimpung di Digital Forensics.

**Konsep Forensic Artifact:**
"Artifact" dalam forensik adalah potongan informasi yang jadi bukti aktivitas manusia — analoginya kayak sidik jari, kancing baju yang lepas, atau alat yang dipakai di TKP fisik: semuanya digabungkan buat merekonstruksi cerita bagaimana suatu kejadian terjadi. Di komputer, artefak ini berupa jejak-jejak kecil aktivitas yang biasanya tersimpan di lokasi-lokasi yang jarang disentuh user awam — dan justru di situlah nilai forensiknya.

**"Apakah komputer saya memata-matai saya?"**
Poin menarik yang diangkat room ini: banyak fitur tracking di Windows sebenarnya **bukan** dirancang untuk tujuan mengintai, melainkan untuk meningkatkan pengalaman pengguna (personalisasi desktop, riwayat file biar gampang diakses lagi, dst). Tapi justru data personalisasi itulah yang dimanfaatkan penyidik forensik sebagai artefak investigasi. Jadi Windows memang "mengingat" banyak hal soal aktivitas kita — tapi awalnya untuk kenyamanan pengguna, bukan pengawasan.

**Q: Apa Desktop Operating System yang paling banyak dipakai saat ini?**
**A:** Microsoft Windows

---

## Task 2 — Windows Registry and Forensics

**Apa itu Windows Registry?**
Windows Registry adalah kumpulan database yang menyimpan data konfigurasi sistem — bisa soal hardware, software, maupun info user, termasuk file yang baru-baru dipakai, program yang dijalankan, atau device yang pernah terhubung. Dari sudut pandang forensik, semua data ini sangat berharga.

Registry bisa dilihat lewat `regedit.exe`, utility bawaan Windows. Registry tersusun dari **Keys** (folder yang terlihat di regedit) dan **Values** (data yang tersimpan di dalam key tersebut). Gabungan key, subkey, dan value yang disimpan dalam satu file fisik di disk disebut **Registry Hive**.

### Struktur Root Key Registry

Registry di semua sistem Windows terdiri dari 5 root key berikut:

| Root Key | Singkatan | Deskripsi |
|---|---|---|
| **HKEY_CURRENT_USER** | HKCU | Berisi root konfigurasi untuk user yang sedang login saat itu — termasuk folder pribadi, warna layar, pengaturan Control Panel. Terhubung ke profil user tersebut. |
| **HKEY_USERS** | HKU | Berisi semua profil user yang sedang aktif di-load di komputer. `HKEY_CURRENT_USER` sebenarnya adalah subkey dari `HKEY_USERS`. |
| **HKEY_LOCAL_MACHINE** | HKLM | Berisi info konfigurasi yang spesifik untuk komputer itu sendiri (berlaku untuk user manapun). |
| **HKEY_CLASSES_ROOT** | HKCR | Subkey dari `HKLM\Software`. Menyimpan info supaya program yang tepat terbuka saat kita membuka suatu file lewat Explorer. |
| **HKEY_CURRENT_CONFIG** | HKCC | Berisi info profil hardware yang sedang dipakai komputer saat startup. |

**Detail menarik soal HKEY_CLASSES_ROOT:** sejak Windows 2000, info asosiasi file sebenarnya disimpan di **dua tempat sekaligus** — `HKLM\Software\Classes` (setting default, berlaku untuk semua user) dan `HKCU\Software\Classes` (override yang cuma berlaku untuk user yang sedang login). `HKEY_CLASSES_ROOT` itu sendiri sebenarnya cuma **"gabungan tampilan" (merged view)** dari dua sumber tadi, termasuk juga buat kompatibilitas program versi Windows lama. Implikasinya: kalau mau mengubah setting untuk user yang sedang aktif, perubahan harus dilakukan di `HKCU\Software\Classes`, bukan langsung di `HKCR`. Sedangkan untuk mengubah default (semua user), harus lewat `HKLM\Software\Classes`.

**Q: Apa singkatan dari HKEY_LOCAL_MACHINE?**
**A:** `HKLM`

---

## Task 3 — Accessing Registry Hives Offline

Kalau sedang analisis sistem live, registry bisa langsung diakses lewat `regedit.exe`. Tapi kalau yang kita punya cuma **disk image**, kita harus tahu di mana lokasi fisik hive-hive tersebut di disk.

### 5 Hive Utama

Mayoritas hive terletak di `C:\Windows\System32\Config`:

| Hive | Di-mount ke |
|---|---|
| DEFAULT | `HKEY_USERS\DEFAULT` |
| SAM | `HKEY_LOCAL_MACHINE\SAM` |
| SECURITY | `HKEY_LOCAL_MACHINE\Security` |
| SOFTWARE | `HKEY_LOCAL_MACHINE\Software` |
| SYSTEM | `HKEY_LOCAL_MACHINE\System` |

### Hive Berisi Info User

Selain 5 hive di atas, ada 2 hive lagi yang berisi info spesifik per-user, lokasinya di profil user (`C:\Users\<username>\` — untuk Windows 7 ke atas):

- **NTUSER.DAT** — di-mount ke `HKEY_CURRENT_USER` saat user login. Lokasinya langsung di `C:\Users\<username>\`.
- **USRCLASS.DAT** — di-mount ke `HKEY_CURRENT_USER\Software\CLASSES`. Lokasinya di `C:\Users\<username>\AppData\Local\Microsoft\Windows`.

Keduanya **hidden by default**.

### AmCache Hive

Ada satu hive penting lagi di luar daftar tadi: **AmCache**, lokasinya di `C:\Windows\AppCompat\Programs\Amcache.hve`. Windows membuat hive ini khusus buat menyimpan info program yang baru-baru dijalankan di sistem.

### Transaction Logs & Registry Backup

**Transaction Log** bisa dianggap sebagai "jurnal perubahan" dari suatu hive. Windows sering memakai transaction log ini saat menulis data ke hive — artinya, transaction log kadang menyimpan **perubahan paling terbaru** yang bahkan belum sempat masuk ke file hive utamanya. Transaction log disimpan sebagai file `.LOG` di folder yang sama dengan hive-nya, nama file sama tapi ekstensinya `.LOG` (contoh: transaction log untuk hive SAM ada di `C:\Windows\System32\Config\SAM.LOG`). Kadang ada lebih dari satu log, jadi ekstensinya bisa `.LOG1`, `.LOG2`, dst. Sangat disarankan buat selalu ikut memeriksa transaction log saat melakukan registry forensics, bukan cuma hive-nya saja.

**Registry Backup** itu kebalikannya — backup dari hive-hive di `C:\Windows\System32\Config`, disalin otomatis ke folder `C:\Windows\System32\Config\RegBack` setiap **10 hari sekali**. Ini lokasi yang bagus buat dicek kalau dicurigai ada registry key yang baru-baru ini dihapus atau diubah.

**Q: Di mana lokasi 5 hive utama (DEFAULT, SAM, SECURITY, SOFTWARE, SYSTEM)?**
**A:** `C:\Windows\System32\Config`

**Q: Di mana lokasi hive AmCache?**
**A:** `C:\Windows\AppCompat\Programs\Amcache.hve`

---

## Task 4 — Data Acquisition

Saat melakukan forensik, kita akan berhadapan dengan salah satu dari dua kondisi: sistem yang masih hidup (live) atau image yang sudah diambil dari sistem tersebut. Demi akurasi, praktik yang direkomendasikan adalah meng-image sistem atau menyalin data yang dibutuhkan, lalu melakukan analisis di atas salinan itu — proses inilah yang disebut **data acquisition**.

Walau registry bisa dilihat lewat Registry Editor, cara yang benar secara forensik adalah mengambil salinan datanya dulu baru dianalisis. Masalahnya, kalau kita coba menyalin hive langsung dari `%WINDIR%\System32\Config`, itu **tidak bisa dilakukan begitu saja** karena file tersebut sedang dipakai/dikunci sistem (*restricted file*).

### Tools untuk Akuisisi

**1. KAPE (Kroll Artifact Parser and Extractor)**
Tool akuisisi & analisis data live. Utamanya berbasis command-line, tapi tersedia juga versi GUI. KAPE dipakai buat menargetkan & menarik artefak tertentu — termasuk registry hive — secara cepat dan terstruktur.

**2. Autopsy**
Bisa mengakuisisi data baik dari sistem live maupun disk image. Caranya: setelah data source ditambahkan, navigasi ke lokasi file yang mau diambil, klik kanan, pilih opsi **"Extract File(s)"**.

**3. FTK Imager**
Mirip Autopsy — file bisa diekstrak dengan cara mount disk image atau drive-nya dulu di FTK Imager, lalu gunakan opsi **"Export files"**. Ada juga opsi lain khusus untuk sistem live: **"Obtain Protected Files"**, yang memungkinkan ekstraksi semua hive registry sekaligus ke lokasi pilihan kita. Catatan penting: opsi ini **tidak** ikut menyalin `Amcache.hve`, padahal file itu sering krusial buat investigasi bukti program yang terakhir dieksekusi — jadi harus diambil manual secara terpisah.

Di room ini, data akuisisinya sudah disediakan lewat VM yang di-attach (hasil koleksi KAPE), jadi kita tidak perlu melakukan akuisisi sendiri.

---

## Task 5 — Exploring Windows Registry

Setelah hive berhasil diekstrak, kita butuh tool untuk membukanya layaknya di Registry Editor — karena Registry Editor bawaan cuma bisa baca registry sistem yang sedang live, tidak bisa me-load file hive lepasan.

### Perbandingan Tools

**Registry Viewer (AccessData)**
Tampilan antarmukanya mirip Windows Registry Editor. Ada dua keterbatasan: cuma bisa me-load **satu hive** dalam satu waktu, dan **tidak** memperhitungkan transaction log.

**Eric Zimmerman's Registry Explorer**
Bisa me-load **banyak hive sekaligus**, dan bisa menggabungkan data dari transaction log ke dalam hive untuk menghasilkan hive yang lebih "bersih" dengan data yang lebih up-to-date. Ada juga fitur **Bookmarks** yang isinya key-key registry penting yang biasa dicari penyidik forensik — jadi analis bisa langsung menuju key/value yang relevan tanpa harus navigasi manual.

**RegRipper**
Utility yang menerima hive sebagai input, lalu mengeluarkan laporan (dalam bentuk text file) berisi data yang diekstrak dari key-key dan value-value penting secara forensik, ditampilkan berurutan. Tersedia dalam bentuk CLI maupun GUI. Kekurangannya: **tidak** memperhitungkan transaction log — jadi kalau mau hasil lebih akurat, hive harus digabungkan dulu dengan transaction log-nya lewat Registry Explorer, baru hasilnya dikirim ke RegRipper.

Room ini akan fokus memakai **Registry Explorer** beserta tools Eric Zimmerman lainnya.

---

## Task 6 — System Information and System Accounts

Langkah pertama forensik biasanya adalah memastikan info dasar sistem yang sedang dianalisis.

### OS Version

Kalau kita cuma punya data triage, versi OS bisa ditentukan lewat key berikut:
```
SOFTWARE\Microsoft\Windows NT\CurrentVersion
```

### Current Control Set

Hive yang berisi data konfigurasi sistem untuk mengontrol proses startup disebut **Control Set**. Umumnya ada dua di hive SYSTEM: `ControlSet001` dan `ControlSet002`. Dalam kebanyakan kasus (tapi tidak selalu), `ControlSet001` menunjuk ke Control Set yang dipakai sistem saat boot, dan `ControlSet002` adalah *last known good configuration*.

Windows membuat Control Set **volatile** saat mesin hidup, disebut `CurrentControlSet` (`HKLM\SYSTEM\CurrentControlSet`). Untuk info sistem yang paling akurat, inilah hive yang harus dijadikan acuan. Untuk tahu Control Set mana yang sedang jadi `CurrentControlSet`, cek value berikut:
```
SYSTEM\Select\Current
```
Sedangkan last known good configuration bisa dicek di:
```
SYSTEM\Select\LastKnownGood
```

Penting untuk memastikan info ini di awal analisis, karena banyak artefak forensik lain yang nantinya diambil dari Control Set ini.

### Computer Name

Penting untuk memastikan Computer Name saat forensik, supaya yakin kita menganalisis mesin yang memang seharusnya:
```
SYSTEM\CurrentControlSet\Control\ComputerName\ComputerName
```

### Time Zone Information

Demi akurasi, penting untuk tahu timezone komputer yang diinvestigasi — ini membantu memahami kronologi kejadian:
```
SYSTEM\CurrentControlSet\Control\TimeZoneInformation
```
Info timezone ini penting karena sebagian data di komputer punya timestamp dalam UTC/GMT, sementara yang lain dalam waktu lokal. Mengetahui timezone lokal membantu menyusun timeline yang akurat saat menggabungkan data dari berbagai sumber.

### Network Interfaces dan Riwayat Jaringan

Key berikut memberi daftar network interface pada mesin yang diinvestigasi:
```
SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\Interfaces
```
Tiap interface direpresentasikan sebagai subkey dengan identifier unik (GUID), berisi value terkait konfigurasi TCP/IP interface tersebut — termasuk IP address, DHCP IP address, Subnet Mask, DNS Server, dll. Info ini penting untuk memastikan kita memang sedang melakukan forensik pada mesin yang seharusnya.

Riwayat jaringan yang pernah terhubung ke mesin bisa dicek di:
```
SOFTWARE\Microsoft\Windows NT\CurrentVersion\NetworkList\Signatures\Unmanaged
SOFTWARE\Microsoft\Windows NT\CurrentVersion\NetworkList\Signatures\Managed
```
Key ini menyimpan jaringan yang pernah terhubung beserta waktu terakhir kali terhubung — ditandai dari **last write time** key registry tersebut.

### Autostart Programs (Autoruns)

Key-key berikut berisi info program atau command yang otomatis berjalan saat user login:
```
NTUSER.DAT\Software\Microsoft\Windows\CurrentVersion\Run
NTUSER.DAT\Software\Microsoft\Windows\CurrentVersion\RunOnce
SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce
SOFTWARE\Microsoft\Windows\CurrentVersion\policies\Explorer\Run
SOFTWARE\Microsoft\Windows\CurrentVersion\Run
```

Sedangkan info soal service ada di:
```
SYSTEM\CurrentControlSet\Services
```
Kalau value dari key **Start** di dalamnya adalah `0x02`, artinya service tersebut akan otomatis start saat boot.

*Relevansi forensik:* autorun/service ini sering jadi tempat favorit malware atau tool persistence "menempel" — jadi bagian ini penting dicek kalau mencurigai adanya program asing yang berjalan otomatis tanpa sepengetahuan user.

### SAM Hive dan Info User

Hive SAM menyimpan info akun user, info login, dan info grup, lokasinya terutama di:
```
SAM\Domains\Account\Users
```
Info yang tersimpan di sini meliputi: RID (Relative Identifier) user, jumlah kali login, waktu login terakhir, waktu login gagal terakhir, waktu ganti password terakhir, masa berlaku password, kebijakan password, hint password, serta grup yang diikuti user tersebut.

### Soal & Jawaban

**Q: Berapa Current Build Number mesin yang diinvestigasi?**
**A:** `19044`

**Q: ControlSet mana yang berisi last known good configuration?**
**A:** `1` (ControlSet001)

**Q: Apa Computer Name-nya?**
**A:** `THM-4n6`

**Q: Berapa value dari TimeZoneKeyName?**
**A:** `Pakistan Standard Time`

**Q: Berapa DHCP IP address-nya?**
**A:** `192.168.100.58`

**Q: Berapa RID akun Guest User?**
**A:** `501`
*Konteks tambahan:* RID adalah angka unik di ekor SID (Security Identifier) tiap akun. Ada beberapa RID yang "baku"/reserved, contohnya `500` = Administrator dan `501` = Guest, sedangkan akun buatan user biasanya punya RID di atas 1000. Ini berguna untuk cepat mengidentifikasi jenis akun tanpa harus menebak dari namanya.

---

## Task 7 — Usage or Knowledge of Files/Folders

Task ini fokus ke artefak yang menunjukkan user **tahu/pernah berinteraksi** dengan suatu file atau folder — beda dengan "evidence of execution" (bukti program itu benar-benar dijalankan) di Task 8.

### Recent Files

Windows menyimpan daftar file yang baru-baru dibuka untuk tiap user (ini yang muncul di Windows Explorer sebagai "recently used files"), disimpan di hive NTUSER pada lokasi:
```
NTUSER.DAT\Software\Microsoft\Windows\CurrentVersion\Explorer\RecentDocs
```
Registry Explorer mengurutkan data ini dengan file **Most Recently Used (MRU)** ditampilkan paling atas.

Menariknya, di key ini juga ada subkey berdasarkan ekstensi file (`.pdf`, `.jpg`, `.docx`, dst), yang menyimpan info file terakhir yang dipakai untuk ekstensi tertentu. Misalnya, kalau ingin fokus mencari PDF terakhir yang dibuka, tinggal cek:
```
NTUSER.DAT\Software\Microsoft\Windows\CurrentVersion\Explorer\RecentDocs\.pdf
```
Registry Explorer juga menampilkan waktu terakhir dibuka (Last Opened time) dari tiap file.

### Office Recent Files

Mirip dengan Recent Docs bawaan Explorer, Microsoft Office juga menyimpan daftar dokumen yang baru-baru dibuka. Lokasinya juga di hive NTUSER:
```
NTUSER.DAT\Software\Microsoft\Office\VERSION
```
Nomor versi berbeda-beda untuk tiap rilis Office (contoh: `15.0` untuk Office 2013), sehingga key lengkapnya bisa berbentuk `NTUSER.DAT\Software\Microsoft\Office\15.0\Word`.

Mulai dari Office 365, Microsoft mengaitkan lokasi ini dengan Live ID user, sehingga recent file-nya bisa ditemukan di:
```
NTUSER.DAT\Software\Microsoft\Office\VERSION\UserMRU\LiveID_####\FileMRU
```
Lokasi ini juga menyimpan path lengkap dari file yang baru-baru dipakai.

### ShellBags

Setiap kali user membuka suatu folder, folder itu terbuka dengan layout tertentu (ukuran ikon, tampilan list/detail, dst) yang bisa disesuaikan sesuai preferensi user, dan bisa berbeda-beda antar folder. Info soal "shell" Windows ini tersimpan dan bisa dipakai untuk mengidentifikasi file/folder yang paling baru diakses. Karena setting ini spesifik per-user, lokasinya ada di hive user:
```
USRCLASS.DAT\Local Settings\Software\Microsoft\Windows\Shell\Bags
USRCLASS.DAT\Local Settings\Software\Microsoft\Windows\Shell\BagMRU
NTUSER.DAT\Software\Microsoft\Windows\Shell\BagMRU
NTUSER.DAT\Software\Microsoft\Windows\Shell\Bags
```
Registry Explorer tidak menampilkan info ShellBags dengan cukup jelas. Untuk ini, dipakai tool lain dari suite Eric Zimmerman: **ShellBag Explorer**, yang tinggal diarahkan ke hive yang sudah diekstrak, dan otomatis mem-parsing datanya jadi format yang lebih mudah dibaca.

### Open/Save dan LastVisited Dialog MRUs

Saat kita membuka atau menyimpan file, muncul dialog box yang menanyakan lokasi. Windows "mengingat" lokasi terakhir yang dipakai untuk buka/simpan file — artinya jejak ini bisa dipakai untuk melacak file yang baru-baru diakses lewat dialog tersebut. Key-nya:
```
NTUSER.DAT\Software\Microsoft\Windows\CurrentVersion\Explorer\ComDlg32\OpenSavePIDlMRU
NTUSER.DAT\Software\Microsoft\Windows\CurrentVersion\Explorer\ComDlg32\LastVisitedPidlMRU
```

### Windows Explorer Address/Search Bars

Cara lain untuk melacak aktivitas user adalah lewat path yang pernah diketik di address bar Windows Explorer, atau pencarian yang pernah dilakukan:
```
NTUSER.DAT\Software\Microsoft\Windows\CurrentVersion\Explorer\TypedPaths
NTUSER.DAT\Software\Microsoft\Windows\CurrentVersion\Explorer\WordWheelQuery
```

### Soal & Jawaban

**Q: Kapan EZtools dibuka?**
**A:** `2021-12-01 13:00:34`

**Q: Jam berapa terakhir "My Computer" diinteraksi?**
**A:** `2021-12-01 13:06:47`

**Q: Apa absolute path file yang dibuka pakai notepad.exe?**
**A:** `C:\Program Files\Amazon\Ec2ConfigService\Settings`

**Q: Kapan file itu dibuka?**
**A:** `2021-11-30 10:56:19`

---

## Task 8 — Evidence of Execution

Bagian yang paling penting dipahami detail karena ada beberapa artefak yang mirip-mirip tapi fungsinya beda.

### UserAssist

Windows melacak aplikasi yang dijalankan lewat Windows Explorer (untuk keperluan statistik) di registry key UserAssist. Key ini menyimpan info program yang dijalankan, waktu peluncuran, dan berapa kali dijalankan. **Catatan penting:** program yang dijalankan lewat command line **tidak** akan tercatat di UserAssist. Key ini ada di hive NTUSER, dipetakan ke GUID masing-masing user:
```
NTUSER.DAT\Software\Microsoft\Windows\Currentversion\Explorer\UserAssist\{GUID}\Count
```

### ShimCache (AppCompatCache)

ShimCache adalah mekanisme untuk melacak kompatibilitas aplikasi dengan OS, dan sebagai efek sampingnya, ia melacak **semua** aplikasi yang dijalankan di mesin. Tujuan utamanya sebenarnya untuk memastikan kompatibilitas mundur (backward compatibility) aplikasi. Nama lainnya adalah **Application Compatibility Cache (AppCompatCache)**. Lokasinya di hive SYSTEM:
```
SYSTEM\CurrentControlSet\Control\Session Manager\AppCompatCache
```
ShimCache menyimpan nama file, ukuran file, dan waktu terakhir modifikasi dari file executable.

Registry Explorer **tidak** mem-parsing data ShimCache dalam format yang mudah dibaca manusia — untuk ini dipakai tool lain dari EZtools: **AppCompatCache Parser**. Tool ini menerima hive SYSTEM sebagai input, memparsingnya, dan mengeluarkan output berupa file CSV. Command yang dipakai:
```
AppCompatCacheParser.exe --csv <path simpan output> -f <path hive SYSTEM> -c <control set yang mau diparsing>
```
Output CSV-nya bisa dilihat pakai **EZViewer**, tool lain dari EZtools.

### AmCache

AmCache adalah artefak yang berhubungan dengan ShimCache — fungsinya mirip, tapi juga menyimpan data tambahan terkait eksekusi program: path eksekusi, waktu instalasi/eksekusi/penghapusan, dan **hash SHA1** dari program yang dieksekusi. Lokasinya:
```
C:\Windows\appcompat\Programs\Amcache.hve
```
Info program yang terakhir dieksekusi ada di:
```
Amcache.hve\Root\File\{Volume GUID}\
```

### BAM/DAM

**Background Activity Moderator (BAM)** melacak aktivitas aplikasi yang berjalan di background. **Desktop Activity Moderator (DAM)** adalah komponen serupa yang mengoptimalkan konsumsi daya perangkat. Keduanya bagian dari sistem Modern Standby di Windows. Lokasinya:
```
SYSTEM\CurrentControlSet\Services\bam\UserSettings\{SID}
SYSTEM\CurrentControlSet\Services\dam\UserSettings\{SID}
```
Lokasi ini menyimpan info program yang terakhir dijalankan, **path lengkapnya**, dan waktu eksekusi terakhir.

### Tabel Perbandingan Cepat

| Artefak | Fungsi Utama | Kelebihan Unik |
|---|---|---|
| **UserAssist** | Mencatat aplikasi GUI yang dijalankan per-user, jumlah & waktu eksekusi | Tidak mencatat eksekusi lewat command line |
| **ShimCache / AppCompatCache** | Melacak kompatibilitas aplikasi, sekaligus jadi jejak semua aplikasi yang dijalankan | Menyimpan nama file, ukuran, dan waktu modifikasi terakhir |
| **AmCache** | Database terpisah (`Amcache.hve`) untuk program terinstall/dieksekusi | Satu-satunya yang menyimpan **hash SHA1** — berguna untuk dicocokkan ke database malware/threat intel |
| **BAM/DAM** | Manajemen aplikasi background & power throttling | Menyimpan **full path lengkap** dari program yang dijalankan |

### Soal & Jawaban

**Q: Berapa kali File Explorer dijalankan?**
**A:** `26`

**Q: Nama lain dari ShimCache?**
**A:** `AppCompatCache`

**Q: Artefak mana yang juga menyimpan hash SHA1 dari program yang dieksekusi?**
**A:** `AmCache`

**Q: Artefak mana yang menyimpan full path dari program yang dieksekusi?**
**A:** `BAM/DAM`

---

## Task 9 — External Devices/USB Device Forensics

### Identifikasi Device

Lokasi berikut melacak USB key yang pernah dicolok ke sistem, menyimpan vendor ID, product ID, dan versi device (kombinasi ini dipakai untuk identifikasi device secara unik), sekaligus waktu device tersebut dicolok:
```
SYSTEM\CurrentControlSet\Enum\USBSTOR
SYSTEM\CurrentControlSet\Enum\USB
```

### Waktu First/Last Connection

Untuk detail timestamp yang lebih spesifik, key berikut melacak waktu pertama kali device terhubung, terakhir terhubung, dan terakhir dilepas:
```
SYSTEM\CurrentControlSet\Enum\USBSTOR\Ven_Prod_Version\USBSerial#\Properties\{83da6326-97a6-4088-9453-a19231573b29}\####
```
Nilai `####` menentukan jenis timestamp yang ditampilkan:

| Value | Informasi |
|---|---|
| `0064` | Waktu koneksi pertama (First Connection time) |
| `0066` | Waktu koneksi terakhir (Last Connection time) |
| `0067` | Waktu pelepasan terakhir (Last removal time) |

Walau bisa dicek manual, Registry Explorer sudah otomatis mem-parsing data ini begitu key USBSTOR dipilih.

### USB Device Volume Name

Nama device dari drive yang terhubung bisa dicek di:
```
SOFTWARE\Microsoft\Windows Portable Devices\Devices
```
GUID yang muncul di key ini bisa dibandingkan dengan Disk ID yang ada di key identifikasi device (USBSTOR/USB) untuk mengkorelasikan nama device dengan device yang unik secara teknis.

Menggabungkan semua info ini (identifikasi device, waktu koneksi, dan nama device), kita bisa menyusun gambaran cukup lengkap soal USB device apa saja yang pernah terhubung ke mesin yang diinvestigasi.

### Soal & Jawaban

**Q: Berapa serial number device dari manufacturer 'Kingston'?**
**A:** `1C6F654E59A3B0C179D366AE&0`

**Q: Apa nama device tersebut?**
**A:** `Kingston Data Traveler 2.0 USB Device`

**Q: Apa friendly name device dari manufacturer 'Kingston'?**
**A:** `USB`

---

## Task 10 — Hands-on Challenge

### Setup

Room ini punya VM (login: `THM-4n6` / `123`) dengan dua folder di Desktop:
- **triage** — hasil koleksi triage lewat KAPE, strukturnya sama seperti direktori aslinya.
- **EZtools** — kumpulan tools Eric Zimmerman, termasuk Registry Explorer, EZViewer, dan AppCompatCacheParser.exe.

Autopsy sebenarnya juga tersedia di Desktop dan bisa dipakai, tapi room ini merekomendasikan pemakaian EZtools sesuai yang diajarkan.

### Skenario

Salah satu Desktop di lab riset Organization X dicurigai diakses oleh pihak tak berwenang. Biasanya cuma ada satu akun user per Desktop, tapi di sistem ini ditemukan **banyak** akun user. Ada juga dugaan sistem ini terhubung ke suatu network drive, dan sebuah USB device pernah terhubung. Data triage dari sistem ini sudah dikumpulkan dan ditaruh di VM yang di-attach.

### Menangani "Dirty Hive"

Saat me-load hive di Registry Explorer, akan muncul peringatan bahwa hive-nya "dirty". Ini berhubungan langsung dengan konsep transaction log yang sudah dibahas sebelumnya — solusinya, arahkan Registry Explorer ke file `.LOG1` dan `.LOG2` dengan nama file yang sama dengan hive-nya. Registry Explorer akan otomatis menggabungkan transaction log tersebut ke hive dan membuat hive yang "bersih". Setelah lokasi penyimpanan hive bersih ini ditentukan, hive itulah yang dipakai untuk analisis, tanpa perlu lagi me-load hive "dirty"-nya berulang kali.

### Analisis SAM — Info Akun User

SAM (Security Account Manager) menyimpan info krusial soal akun lokal: username, RID, waktu login terakhir, hint password, dan hash password (bukan plaintext).

**Q: Berapa banyak user-created account di sistem?**
**A:** `3`

**Q: Username akun mana yang belum pernah login?**
**A:** `thm-user2`

**Q: Apa password hint untuk user THM-4n6?**
**A:** `count`

### Analisis NTUSER.DAT — Jejak Eksekusi File Teks

NTUSER.DAT adalah hive per-user, lokasinya di `C:\Users\<username>\NTUSER.DAT`. File ini hidden by default dan cukup sensitif — mengubahnya langsung berisiko merusak profil user, jadi selalu diperlakukan read-only dalam forensik. Untuk mengaksesnya, opsi "show hidden files" di File Explorer harus diaktifkan dulu.

Path untuk cek recent file:
```
NTUSER.DAT\Software\Microsoft\Windows\CurrentVersion\Explorer\RecentDocs
```

**Q: Kapan file 'Changelog.txt' diakses?**
**A:** `2021-11-24 18:18:48`

### Proses Elimination Mencari Instalasi Python

Bagian menarik dari investigasi ini adalah proses trial-and-error yang realistis:
1. Cek **UserAssist** → dead end.
2. Cek **ShimCache**, **AmCache**, **BAM/DAM** → semuanya dead end juga.
3. Solusinya: search keyword **"Apps"** di search bar Registry Explorer → ketemu key **RecentApps**, yang menyimpan jejak instalasi Python 3.8.2.

Pelajarannya: kalau artefak "standar" tidak menemukan apa yang dicari, jangan langsung menyerah — coba cari berdasarkan keyword yang relevan dengan konteks (nama aplikasi, tipe file) untuk menemukan key registry lain yang mungkin jarang dicek tapi tetap menyimpan jejaknya.

**Q: Apa full path instalasi python 3.8.2 dijalankan?**
**A:** `Z:\setups\python-3.8.2.exe`
*(Catatan: drive `Z:\` biasanya menandakan network drive/mapped drive, bukan disk lokal — sejalan dengan dugaan awal skenario soal koneksi ke network drive.)*

### Melacak USB Device

Cari GUID device dulu di `SOFTWARE\Microsoft\Windows Portable Devices\Devices` (berdasarkan friendly name "USB"), lalu cocokkan GUID itu ke `SYSTEM\CurrentControlSet\Enum\USBSTOR` untuk mendapat detail waktu first/last connected.

**Q: Kapan USB device dengan friendly name 'USB' terakhir terhubung?**
**A:** `2021-11-24 18:40:06`

---

## Task 11 — Conclusion

Room ini menutup topik forensik registry Windows dengan menekankan bahwa hampir seluruh aktivitas signifikan seorang user — instalasi program, file yang dibuka, device yang dicolok, jaringan yang terhubung — meninggalkan jejak di registry, meski user tersebut tidak pernah sadar jejak itu ada. Kombinasi dari SYSTEM, SOFTWARE, SAM, dan hive per-user (NTUSER.DAT/USRCLASS.DAT) memberikan gambaran yang sangat komprehensif soal apa yang terjadi di suatu mesin Windows — modal penting sebelum lanjut ke topik forensik Windows yang lebih lanjut.

---

## Ringkasan: Peta Cepat "Cari Artefak Apa, Di Mana"

```
Info sistem (build, timezone, computer name)       → SYSTEM hive
Network interface & riwayat jaringan               → SYSTEM hive / SOFTWARE hive (NetworkList)
Autorun program & service                          → NTUSER.DAT + SOFTWARE hive + SYSTEM\...\Services
Akun user & password hash/hint                     → SAM hive
Software terinstall & USB device mapping           → SOFTWARE hive
Program dieksekusi + hash SHA1                     → AmCache (Amcache.hve)
Program dieksekusi + full path                     → BAM/DAM (di SYSTEM)
Program GUI dijalankan (per user)                  → UserAssist (di NTUSER.DAT)
File/folder terakhir dibuka user (Explorer)        → NTUSER.DAT (RecentDocs)
File Office terakhir dibuka                        → NTUSER.DAT (Office\VERSION)
Layout folder & jejak navigasi folder              → NTUSER.DAT / USRCLASS.DAT (ShellBags)
Lokasi terakhir dialog Open/Save                    → NTUSER.DAT (ComDlg32 MRU)
Path & pencarian yang diketik di Explorer           → NTUSER.DAT (TypedPaths / WordWheelQuery)
Aplikasi non-GUI/instalasi tak umum                 → NTUSER.DAT (RecentApps) - coba search keyword
Perubahan terbaru yang belum ke-flush ke hive       → Transaction Log (.LOG/.LOG1/.LOG2)
Key yang dicurigai dihapus/diubah baru-baru ini     → RegBack (backup 10 harian)
```

## Ringkasan: Tool Acquisition vs Tool Analysis

| Tahap | Tool | Fungsi |
|---|---|---|
| Akuisisi | KAPE | Ekstrak artefak (termasuk registry) dari sistem live secara cepat & terstruktur |
| Akuisisi | Autopsy | Ekstrak file dari live system/disk image (fitur "Extract File(s)") |
| Akuisisi | FTK Imager | Ekstrak file via mount image, atau "Obtain Protected Files" khusus live system (tidak menarik Amcache.hve) |
| Analisis | Registry Viewer (AccessData) | Baca 1 hive, UI mirip regedit, tidak baca transaction log |
| Analisis | Registry Explorer (EZtools) | Baca banyak hive sekaligus + auto-merge transaction log + fitur Bookmarks |
| Analisis | RegRipper | Otomatis generate laporan text dari key-key penting, tidak baca transaction log |
| Analisis | AppCompatCache Parser (EZtools) | Khusus parsing ShimCache jadi CSV yang bisa dibaca EZViewer |
| Analisis | ShellBag Explorer (EZtools) | Khusus parsing data ShellBags jadi format mudah dibaca |

---

## Referensi
- Room asli: https://tryhackme.com/room/windowsforensics1
- Artikel acuan: *TryHackMe: Windows Forensics 1 — Detailed Write-Up* oleh Cindy (Shunxian) Ou
- Eric Zimmerman Tools (EZtools): dipakai luas di industri DFIR untuk parsing artefak Windows
- KAPE (Kroll Artifact Parser and Extractor)
- Autopsy, FTK Imager (Exterro)
- RegRipper

## Lisensi
Catatan pribadi untuk keperluan belajar/dokumentasi CTF. Silakan gunakan/modifikasi bebas.
