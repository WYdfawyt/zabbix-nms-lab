<h1 align="center">Zabbix NMS Lab</h1>
<p align="center"><b>Mini NOC untuk memantau server Linux dan Windows dengan Docker, ICMP, dan alert Telegram</b></p>

<p align="center">
  <img alt="Zabbix" src="https://img.shields.io/badge/Zabbix-7.0_LTS-D40000?logo=zabbix&logoColor=white">
  <img alt="Docker" src="https://img.shields.io/badge/Docker_Compose-2496ED?logo=docker&logoColor=white">
  <img alt="Ubuntu" src="https://img.shields.io/badge/Ubuntu_Server-26.04_LTS-E95420?logo=ubuntu&logoColor=white">
  <img alt="Windows Server" src="https://img.shields.io/badge/Windows_Server-2025-0078D4?logo=windows&logoColor=white">
  <img alt="VirtualBox" src="https://img.shields.io/badge/VirtualBox-183A61?logo=virtualbox&logoColor=white">
  <img alt="Telegram" src="https://img.shields.io/badge/Alert-Telegram_Bot-26A5E4?logo=telegram&logoColor=white">
</p>

> Dari VM kosong sampai notifikasi masuk ke HP: membangun, menguji, dan mendokumentasikan sistem monitoring ala NOC. **5 VM, 2 sistem operasi, 8 skenario insiden**, semuanya berjalan di satu laptop.

> **Summary (EN):** A NOC-style monitoring lab. Zabbix 7.0 LTS runs in Docker on one VM and monitors four more VMs (three Ubuntu Server, one Windows Server 2025) using agent checks, ICMP ping, and custom service triggers. Alerts (problem and recovery) are delivered through a Telegram bot. The repo includes a network map, a NOC dashboard, eight documented incident tests with timestamps, and a troubleshooting log.

<p align="center">
  <img src="docs/screenshots/19-dashboard-noc-normal.jpeg" alt="Dashboard NOC" width="49%">
  <img src="docs/screenshots/17-network-map-normal.jpeg" alt="Network map topologi lab" width="49%">
</p>

## Sekilas

| Angka | Keterangan |
|---|---|
| **5 VM** | 1 server monitoring, 3 Ubuntu Server (Nginx, MariaDB, klien), 1 Windows Server 2025 |
| **8 skenario insiden** | Dicatat waktu muncul, waktu pulih, dan durasinya |
| **30 detik** | Interval pengecekan layanan, sehingga layanan mati terdeteksi dalam sekitar 30 detik |
| **3 metode pengecekan** | Zabbix Agent, ICMP ping, dan pengecekan port atau layanan |
| **1 laptop** | Seluruh lab berjalan di Dell Latitude 7390 (Intel Core i7-8650U, RAM 16 GB) |

## Daftar isi

1. [Latar belakang dan tujuan](#latar-belakang-dan-tujuan)
2. [Arsitektur](#arsitektur)
3. [Alur alert](#alur-alert)
4. [Cakupan pemantauan](#cakupan-pemantauan)
5. [Uji insiden](#uji-insiden)
6. [Galeri bukti](#galeri-bukti)
7. [Kendala dan solusi](#kendala-dan-solusi)
8. [Keterampilan yang dilatih](#keterampilan-yang-dilatih)
9. [Keamanan](#keamanan)
10. [Cara mereplikasi](#cara-mereplikasi)
11. [Rencana pengembangan](#rencana-pengembangan)
12. [Penulis](#penulis)

## Latar belakang dan tujuan

Pekerjaan NOC berputar pada satu siklus: **mendeteksi gangguan, memberi tahu orang yang tepat, menganalisis, memulihkan, lalu mendokumentasikan**. Project ini menjalankan seluruh siklus itu dalam skala kecil:

- Memantau ketersediaan dan kinerja server Linux dan Windows di satu jaringan lab.
- Membuat aturan deteksi sendiri (item dan trigger), bukan hanya memakai template bawaan.
- Mengirim notifikasi otomatis (Problem dan Resolved) ke Telegram dengan filter severity.
- Membuktikannya lewat simulasi insiden yang waktunya tercatat.
- Mendokumentasikan kendala nyata dan cara mengatasinya.

## Arsitektur

```mermaid
flowchart TB
  ADM["Admin (browser laptop)<br/>http://192.168.56.10:8080"]
  TG["Telegram Bot API"]
  subgraph HO["Jaringan Host-only 192.168.56.0/24 (VirtualBox)"]
    NMS["NMS-ZABBIX .10<br/>Zabbix Server, Web, PostgreSQL<br/>(Docker Compose)"]
    WEB["WEB-01 .11<br/>Ubuntu + Nginx<br/>Zabbix Agent"]
    DB["DB-01 .12<br/>Ubuntu + MariaDB<br/>Zabbix Agent"]
    WIN["WIN-01 .13<br/>Windows Server 2025<br/>Zabbix Agent"]
    CL["CLIENT-01 .14<br/>Ubuntu<br/>Zabbix Agent"]
  end
  ADM --> NMS
  NMS -->|"ICMP + Agent 10050"| WEB
  NMS -->|"ICMP + Agent 10050"| DB
  NMS -->|"ICMP + Agent 10050"| WIN
  NMS -->|"ICMP + Agent 10050"| CL
  NMS -->|"HTTPS (lewat NAT)"| TG
```

| VM | Sistem operasi | Peran | IP (Host-only) | RAM | CPU | Disk |
|---|---|---|---|---|---|---|
| NMS-ZABBIX | Ubuntu Server 26.04 LTS | Zabbix Server + Web + PostgreSQL (Docker) | 192.168.56.10 | 3 GB | 2 | 25 GB |
| WEB-01 | Ubuntu Server 26.04 LTS | Web server (Nginx) | 192.168.56.11 | 1 GB | 1 | 25 GB |
| DB-01 | Ubuntu Server 26.04 LTS | Database (MariaDB) | 192.168.56.12 | 1,5 GB | 1 | 25 GB |
| WIN-01 | Windows Server 2025 (Evaluation) | Server Windows | 192.168.56.13 | 3 GB | 2 | 40 GB |
| CLIENT-01 | Ubuntu Server 26.04 LTS | Klien uji | 192.168.56.14 | 1 GB | 1 | 25 GB |

**Desain jaringan:** setiap VM memiliki dua adapter. **NAT** untuk akses internet (unduh paket, Telegram) dan **Host-only** untuk jaringan lab serta akses web Zabbix dari laptop. Ping dilakukan dari **NMS ke tiap VM**, bukan antar sesama VM.

**Teknologi:** Oracle VirtualBox, Ubuntu Server 26.04 LTS, Windows Server 2025, Docker dan Docker Compose, Zabbix 7.0 LTS (7.0.32), PostgreSQL 16, Zabbix Agent (Linux dan Windows), Nginx 1.28, MariaDB 11.8, Telegram Bot API.

## Alur alert

```mermaid
flowchart LR
  A["Item<br/>(dicek tiap 30 detik)"] --> B["Trigger<br/>(aturan masalah)"]
  B --> C["Problem<br/>(punya severity)"]
  C --> D{"Action:<br/>severity Average atau lebih?"}
  D -->|"ya"| E["Media type Telegram"]
  E --> F["Notifikasi Problem<br/>di HP"]
  C -->|"kondisi pulih"| G["Notifikasi Resolved<br/>di HP"]
```

Action memiliki syarat *severity Average atau lebih* supaya peringatan kecil tidak membanjiri notifikasi, dan memiliki *Recovery operation* sehingga pesan **Resolved** ikut terkirim. Zona waktu container Zabbix server disamakan dengan VM (WIB) agar jam pada pesan Telegram sama dengan jam di web Zabbix.

## Cakupan pemantauan

| Aspek | Metode | Trigger | Severity |
|---|---|---|---|
| Ketersediaan host | ICMP Ping dari NMS (template *ICMP Ping*) | Unavailable by ICMP ping (3 ping gagal berturut-turut) | High |
| Latency dan packet loss | ICMP Ping | Trigger bawaan template | Bawaan template |
| Agent | Zabbix Agent (pasif, port 10050) | Zabbix agent is not available (3 menit) | Average |
| Kondisi OS Linux | Template *Linux by Zabbix agent* | Trigger bawaan template | Bawaan template |
| Kondisi OS Windows | Template *Windows by Zabbix agent* | Trigger bawaan template | Bawaan template |
| Layanan Nginx (web-01) | Item `net.tcp.service[http,,80]` tiap 30 detik | **Nginx down** (nilai terakhir = 0) | High |
| Layanan MariaDB (db-01) | Item `net.tcp.port[127.0.0.1,3306]` tiap 30 detik | **MariaDB down** (nilai terakhir = 0) | High |
| CPU web-01 | Item `system.cpu.util` | **High CPU on web-01**: `avg(/web-01/system.cpu.util,#2)>80` | High |
| Layanan Windows (win-01) | Item `service_state[Spooler]` tiap 30 detik | **Print Spooler down on win-01**: `last(...)<>0` | High |

## Uji insiden

Waktu dalam WIB. **Durasi** adalah lama masalah terlihat oleh Zabbix, dihitung dari alert muncul sampai pulih. **Waktu deteksi** bergantung pada interval pengecekan: sekitar 30 detik untuk layanan, dan sekitar 2 sampai 3 menit untuk ICMP karena trigger menunggu 3 kegagalan berturut-turut agar satu paket yang hilang tidak memicu alarm palsu.

| # | Skenario | Host | Alert muncul | Pulih | Durasi | Severity | Notifikasi |
|---|---|---|---|---|---|---|---|
| 1 | `systemctl stop nginx` | web-01 | 29 Okt 18:02:57 | 18:04:27 | 1m 30s | High | Web Zabbix |
| 2 | `systemctl stop mariadb` | db-01 | 29 Okt 18:18:58 | 18:20:28 | 1m 30s | High | Web Zabbix |
| 3 | `systemctl stop nginx` | web-01 | 29 Okt 18:49:27 | 18:50:57 | 1m 30s | High | Telegram |
| 4 | `systemctl stop mariadb` | db-01 | 29 Okt 18:52:28 | 18:54:58 | 2m 30s | High | Telegram |
| 5 | `systemctl stop nginx` | web-01 | 30 Okt 12:40:57 | 12:41:57 | 1m 0s | High | Telegram |
| 6 | VM dimatikan (`poweroff`) | client-01 | 30 Okt 13:00:33 (ICMP) dan 13:01:44 (agent) | 13:07:33 | 7m | High dan Average | Telegram |
| 7 | Beban CPU penuh pada web-01 | web-01 | 30 Okt 13:29:17 | 13:36:17 | 7m | High | Telegram |
| 8 | `Stop-Service Spooler` (Windows) | win-01 | 31 Okt 15:46:39 | 15:48:39 | 2m 0s | High | Telegram |

**Cara membaca:** siklus penanganan diukur lewat dua istilah NOC, yaitu **MTTD** (rata-rata waktu mendeteksi masalah) dan **MTTR** (rata-rata waktu memulihkan). Pada lab ini MTTD layanan sekitar 30 detik, sedangkan MTTR bergantung pada kecepatan pemulihan manual.

## Galeri bukti

### Network map dan dashboard NOC

| Normal | Saat insiden |
|---|---|
| <img src="docs/screenshots/17-network-map-normal.jpeg" alt="Network map saat normal"> | <img src="docs/screenshots/18-network-map-insiden.jpeg" alt="Network map saat ada insiden"> |
| <img src="docs/screenshots/19-dashboard-noc-normal.jpeg" alt="Dashboard NOC saat normal"> | <img src="docs/screenshots/20-dashboard-noc-insiden.jpeg" alt="Dashboard NOC saat ada insiden"> |

<details>
<summary><b>Infrastruktur: VM, jaringan, dan Docker</b> (klik untuk membuka)</summary>

![Daftar VM di VirtualBox](docs/screenshots/01-virtualbox-vm-list.jpeg)
![Pengaturan adapter jaringan](docs/screenshots/02-network-adapters.jpeg)
![IP static pada NMS](docs/screenshots/03-ip-nms.jpeg)
![Container Zabbix berjalan](docs/screenshots/04-docker-compose-ps.jpeg)
![Semua host hijau di Zabbix](docs/screenshots/05-hosts-semua-hijau.jpeg)

</details>

<details>
<summary><b>Konfigurasi monitoring: item, trigger, dan action</b> (klik untuk membuka)</summary>

![Latest data ICMP](docs/screenshots/06-latest-data-icmp.jpeg)
![Item buatan sendiri](docs/screenshots/07-form-item.jpeg)
![Trigger buatan sendiri](docs/screenshots/08-form-trigger.jpeg)
![Konfigurasi action Telegram](docs/screenshots/11-form-action.jpeg)

</details>

<details>
<summary><b>Alert Telegram</b> (klik untuk membuka)</summary>

![Pesan Problem dan Resolved di Telegram](docs/screenshots/09-telegram-problem-resolved.jpeg)
![Action log berstatus Sent](docs/screenshots/10-action-log-sent.jpeg)
![Pesan Telegram host down](docs/screenshots/14-telegram-icmp-down.jpeg)
![Pesan Telegram CPU tinggi](docs/screenshots/16-telegram-high-cpu.jpeg)

</details>

<details>
<summary><b>Riwayat insiden dan grafik</b> (klik untuk membuka)</summary>

![Riwayat problem berstatus RESOLVED](docs/screenshots/12-problems-history-resolved.jpeg)
![Grafik ICMP loss client-01: 0% lalu 100% lalu kembali 0%](docs/screenshots/13-grafik-icmp-loss.jpeg)
![Grafik CPU web-01 saat uji beban](docs/screenshots/15-grafik-cpu.jpeg)

</details>

<details>
<summary><b>Windows Server: pemantauan multi-OS</b> (klik untuk membuka)</summary>

![Latest data win-01](docs/screenshots/21-windows-latest-data.jpeg)
![Pesan Telegram layanan Windows](docs/screenshots/22-telegram-windows-service.jpeg)
![Riwayat problem Print Spooler](docs/screenshots/23-windows-problems-history.jpeg)

</details>

## Kendala dan solusi

Bagian ini sengaja ditulis lengkap karena kemampuan mengurai masalah sama pentingnya dengan hasil akhirnya.

| Kendala | Penyebab | Solusi |
|---|---|---|
| Instalasi Ubuntu Server 26.04 gagal di langkah *installing kernel* (apt exit code 100) | Unduhan paket lewat jaringan NAT gagal | Instal dengan kabel jaringan VM dilepas (kernel diambil dari ISO), jaringan diaktifkan setelah instalasi |
| VM tanpa alamat IPv4 setelah instalasi offline | Tidak ada konfigurasi netplan | Atur IP dengan `netplan set` |
| `netplan apply` menolak file karena indentasi YAML | Kesalahan ketik pada file YAML | Pakai `netplan set` sehingga tidak perlu menulis YAML manual |
| Dua VM berbagi IP setelah *clone* | Clone mewarisi konfigurasi jaringan | Hapus alamat lama, set alamat baru, buat ulang `machine-id` dan kunci SSH host |
| Host "Zabbix server" tidak tersedia | Server berjalan di container, sehingga `127.0.0.1` adalah container itu sendiri | Arahkan interface ke IP VM dan izinkan subnet Docker pada parameter `Server` agent |
| Peringatan SSH *REMOTE HOST IDENTIFICATION HAS CHANGED* | Alamat IP dipakai VM lain | `ssh-keygen -R <ip>` |
| Tes media type Telegram gagal (`event_source`) | Macro tidak terisi di dialog Test | Isi nilai uji manual, lalu uji dengan alert nyata |
| Jam pesan Telegram (UTC) berbeda dari web Zabbix (WIB) | Container server memakai UTC | Ubah zona waktu VM dan mount `/etc/localtime` ke container server |
| `docker compose up` gagal setelah menambah file override | Nama service di file override tidak sama dengan file utama | Cek nama dengan `docker compose config --services` |
| Windows Server tidak menjawab ping | Firewall bawaan memblokir ICMP dan port agent | Buat aturan inbound untuk ICMPv4 Echo dan TCP 10050 lewat PowerShell |
| Peta dan dashboard penuh pemberitahuan *"installed packages has been changed"* | Pemberitahuan rutin template (bukan insiden) setiap paket berubah | Tutup problem tersebut dan saring tampilan berdasarkan severity |

## Keterampilan yang dilatih

| Bidang | Yang dikerjakan |
|---|---|
| Administrasi Linux | Ubuntu Server, netplan, systemd, SSH, clone dan penyesuaian identitas VM |
| Jaringan | Pengalamatan static, NAT dan Host-only, ICMP, pengecekan port dan layanan |
| Container | Docker Compose, volume, jaringan antar container, zona waktu container |
| Monitoring | Zabbix: host, template, item, trigger, map, dashboard, grafik |
| Otomasi alert | Telegram Bot API, filter severity, notifikasi Problem dan Resolved |
| Windows Server | Instalasi, IP static, Remote Desktop, firewall, Zabbix Agent, pemantauan layanan |
| Operasional NOC | Simulasi insiden, pencatatan waktu (MTTD dan MTTR), dokumentasi dan analisis masalah |

## Keamanan

- Token bot Telegram hanya tersimpan di Zabbix, tidak ada di repositori.
- File `.env` masuk `.gitignore`. Gunakan `.env.example` sebagai contoh.
- Jaringan lab bersifat Host-only (tidak terpapar ke luar laptop).
- Agent hanya menerima permintaan dari alamat Zabbix server (parameter `Server`).
- Password bawaan akun `Admin` Zabbix wajib diganti setelah login pertama.

## Cara mereplikasi

1. Buat VM Ubuntu Server dengan adapter NAT dan Host-only, atur IP static sesuai tabel arsitektur.
2. Di VM NMS: pasang Docker (`docker.io`, `docker-compose-v2`), lalu:

   ```bash
   cd compose
   cp ../.env.example .env   # isi DB_PASS
   docker compose up -d
   ```

3. Buka `http://192.168.56.10:8080`, login `Admin` dengan password bawaan Zabbix, lalu segera ganti.
4. Linux: pasang `zabbix-agent` di tiap VM, atur `Server=` dan `Hostname=` pada `/etc/zabbix/zabbix_agentd.conf`.
5. Windows: pasang Zabbix Agent (MSI) dengan `SERVER=192.168.56.10 HOSTNAME=win-01`, dan buka firewall untuk ICMP dan TCP 10050.
6. Tambahkan host, tempelkan template *Linux by Zabbix agent* atau *Windows by Zabbix agent* serta *ICMP Ping*, lalu buat item dan trigger sesuai tabel cakupan pemantauan.
7. Telegram: buat bot lewat BotFather, isi token di media type Telegram, daftarkan chat ID pada media user, lalu buat action dengan syarat severity Average atau lebih.

## Rencana pengembangan

- *Trigger dependency* agar satu gangguan tidak menghasilkan dua notifikasi.
- Pemantauan perangkat jaringan (router atau switch) lewat **SNMP**.
- *Web scenario* untuk memantau halaman web, bukan hanya port.
- Laporan ketersediaan (**SLA report**) per host.
- Otomasi pemasangan agent (misalnya Ansible) dan ekspor konfigurasi Zabbix sebagai cadangan.

## Penulis

**Cahaya Jaya Prabasa**

- LinkedIn: https://www.linkedin.com/in/cahaya-jaya-prabasa
- Email: cahayaprabasa@gmail.com
- GitHub: https://github.com/WYdfawyt
