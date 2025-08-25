# 📚 Panduan GitHub Classroom untuk Mahasiswa
## Mata Kuliah: Cloud Computing - Tugas UTS

---

## 🎯 **LANGKAH 1: Accept Assignment**

### 1. Klik Link Assignment
Buka link berikut di browser: 
```
https://classroom.github.com/a/h0Ljg1hv
```

### 2. Login GitHub
- Login dengan akun GitHub Anda
- Jika belum punya akun GitHub, buat terlebih dahulu di [github.com](https://github.com)

### 3. Accept Assignment
- Klik tombol **"Accept this assignment"**
- Tunggu hingga repository terbuat
- Repository akan bernama: `cc-tugas-uts-[username-anda]`

---

## 🎯 **LANGKAH 2: Clone Repository ke Komputer**

### 1. Install GitHub Desktop
- Download GitHub Desktop dari [desktop.github.com](https://desktop.github.com)
- Install dan login dengan akun GitHub Anda

### ⚠️ **WAJIB MENGGUNAKAN GITHUB DESKTOP**

**Alasan mengapa HARUS pakai GitHub Desktop:**

✅ **AMAN untuk pembelajaran:**
- Tidak ada akses ke command berbahaya (force push, reset --hard)
- Error messages yang jelas dan mudah dipahami
- Visual interface mencegah kesalahan destructive

❌ **RISIKO jika pakai Command Line (Terminal/CMD):**
- **Data loss permanent:** Satu command salah bisa hapus semua history project
- **Force push accident:** Command `git push --force` bisa merusak repo mahasiswa lain
- **Complex error messages:** Pesan error cryptic yang membingungkan
- **Panic solutions:** Mahasiswa google solusi berbahaya saat panic
- **Assessment terganggu:** Dosen tidak bisa track progress jika history rusak

### 🚫 **LARANGAN KERAS:**
```
DILARANG menggunakan:
- Git via Terminal/Command Prompt
- Git Bash
- VS Code integrated terminal untuk Git
- Command line Git apapun

Jika ketahuan pakai command line → PENGURANGAN NILAI
```

### 2. Clone Repository
- Buka GitHub Desktop
- Klik **"Clone a repository from the Internet"**
- Pilih repository: `cc-tugas-uts-[username-anda]`
- Pilih lokasi folder di komputer (contoh: `D:\Kuliah\CloudComputing\`)
- Klik **"Clone"**

---

## 🎯 **LANGKAH 3: Workflow Pengerjaan Tugas**

### **📁 Struktur Folder Tugas**
Setiap minggu, Anda akan mengerjakan tugas yang saling terhubung:
```
CC Tugas UTS/
├── Week-1/    (Environment Setup & GitHub Integration)
├── Week-2/    (Laravel Installation & Tailwind CSS Setup)
├── Week-3/    (Database Design & Migration System)
├── Week-4/    (CRUD Operations - Create & Read)
├── Week-5/    (CRUD Operations - Update & Delete)
├── Week-6/    (API Development & Authentication)
├── Week-7/    (User Authentication & Authorization)
└── Week-8/    (Production Deployment & Optimization - UTS)
```

### **🌿 Git Branch Strategy**
Setiap tugas mingguan dikerjakan di **branch terpisah** yang saling terhubung:

```
main → week-1 → week-2 → week-3 → week-4 → week-5 → week-6 → week-7 → week-8
```

---

## 🎯 **LANGKAH 4: Mengerjakan Week-1**

### 1. Buat Branch Week-1
- Buka GitHub Desktop
- Pastikan berada di branch **"main"**
- Klik **"New branch"**
- Nama branch: `week-1`
- Based on: **"main"**
- Klik **"Create branch"**

### 2. Kerjakan Tugas Week-1
- Klik **"Open in Visual Studio Code"** (atau editor favorit)
- Buat folder `Week-1/`
- Kerjakan tugas: **Environment Setup & GitHub Integration**
  - Setup development environment
  - Configure GitHub integration
  - Install necessary tools
- Simpan semua file dan dokumentasi

### 3. Commit Changes
- Kembali ke GitHub Desktop
- Anda akan melihat perubahan file di panel kiri
- Isi **Summary**: `Week-1: Environment Setup & GitHub Integration`
- Isi **Description**: Detail environment yang di-setup dan tools yang diinstall
- Klik **"Commit to week-1"**

### 4. Push ke GitHub
- Klik **"Push origin"** (upload ke GitHub)

### 5. Create Pull Request
- Klik **"Create Pull Request"** 
- Title: `Week-1: Environment Setup & GitHub Integration`
- Description: Jelaskan environment setup dan tools yang sudah diinstall
- Klik **"Create pull request"**

### 6. Merge ke Main (Setelah Review)
- Setelah dosen review dan approve
- Merge pull request ke main branch
- **PENTING:** Main branch sekarang berisi hasil Week-1

---

## 🎯 **LANGKAH 5: Mengerjakan Week-2**

### 1. Update Main Branch
- Di GitHub Desktop, switch ke branch **"main"**
- Klik **"Fetch origin"** untuk update

### 2. Buat Branch Week-2
- Klik **"New branch"**
- Nama: `week-2`
- Based on: **"main"** (yang sudah berisi hasil Week-1)
- Klik **"Create branch"**

### 3. Kerjakan Tugas Week-2
- Branch week-2 sudah berisi hasil Week-1 (Environment Setup)
- Tambahkan folder `Week-2/`
- Kerjakan: **Laravel Installation & Tailwind CSS Setup**
  - Install Laravel framework
  - Setup Tailwind CSS
  - Configure basic styling
- **Tugas Week-2 = Hasil Week-1 + Laravel & Tailwind setup**

### 4. Commit dan Push
- Summary: `Week-2: Laravel Installation & Tailwind CSS Setup`
- Commit → Push → Create Pull Request
- Merge ke main setelah review

---

## 🎯 **LANGKAH 6: Pattern untuk Week Selanjutnya**

### **Ulangi Pattern yang Sama:**
```
1. Switch ke main branch
2. Fetch origin (update)
3. Create new branch dari main
4. Kerjakan tugas (build on previous work)
5. Commit → Push → Pull Request
6. Merge ke main setelah review
```

### **Branch Naming Convention:**
- `week-1` (Environment Setup & GitHub Integration)
- `week-2` (Laravel Installation & Tailwind CSS Setup)
- `week-3` (Database Design & Migration System)
- `week-4` (CRUD Operations - Create & Read)
- `week-5` (CRUD Operations - Update & Delete)
- `week-6` (API Development & Authentication)
- `week-7` (User Authentication & Authorization)
- `week-8` (Production Deployment & Optimization - UTS)

---

## 🎯 **LANGKAH 7: Tips Penting**

### ✅ **DO (Lakukan):**
- Selalu buat branch baru dari **main** yang sudah ter-update
- Commit dengan pesan yang jelas dan deskriptif
- Push secara berkala (jangan tunggu selesai semua)
- Buat Pull Request untuk setiap tugas mingguan
- Test aplikasi sebelum commit

### ❌ **DON'T (Jangan):**
- Jangan langsung edit di main branch
- **Jangan gunakan Git command line/terminal** (risk data loss tinggi)
- Jangan force push (akan merusak history)
- Jangan copy-paste dari teman (plagiarism detection aktif)
- Jangan menunda push sampai deadline
- **Jangan install Git extensions atau tools lain** selain GitHub Desktop

---

## 🆘 **Troubleshooting**

### **Problem 1: "Cannot push - permission denied"**
**Solusi:**
- Pastikan GitHub Desktop sudah login
- Check username/email di Git settings
- Re-authenticate GitHub account

### **Problem 2: "Merge conflict"**
**Solusi:**
- Koordinasi dengan dosen
- Jangan resolve sendiri tanpa bimbingan

### **Problem 3: "Lost my work"**
**Solusi:**
- Check di GitHub Desktop → History tab
- File belum hilang, mungkin di branch lain
- Contact dosen untuk bantuan

### **Problem 4: "Ingin pakai command line Git"**
**Solusi:**
- **DILARANG!** Hanya boleh GitHub Desktop
- Command line = risk data loss & pengurangan nilai
- Semua kebutuhan sudah tersedia di GitHub Desktop

---

## 📞 **Bantuan & Kontak**

### **Tim Pengajar:**

#### **👨‍🏫 Dosen Pengampu:**
**Aidil Saputra Kirsan, M.Tr.Kom**
- Untuk konsultasi materi dan penilaian
- Review Pull Request dan feedback assignment
- Jam konsultasi: [Isi sesuai jadwal]

#### **👥 Asisten Dosen:**
**Jein & Taufik Ilham**
- Bantuan teknis GitHub dan troubleshooting
- Support daily workflow dan Git issues
- Panduan hands-on GitHub Desktop

### **📱 Prosedur Bantuan:**

#### **Untuk Masalah Teknis (GitHub/Git):**
1. Screenshot error message yang lengkap
2. Jelaskan langkah yang sudah dilakukan
3. Hubungi **Asisten Dosen** (Jein/Taufik) terlebih dahulu
4. Jika masalah kompleks, akan di-escalate ke Dosen

#### **Untuk Konsultasi Materi/Penilaian:**
1. Siapkan progress work yang sudah dikerjakan
2. Hubungi **Aidil Saputra Kirsan, M.Tr.Kom** langsung
3. Bawa laptop untuk demo jika diperlukan

### **🆘 Emergency Contact:**
- **Jangan panic** - semua masalah Git bisa diperbaiki!
- Worst case: Contact dosen untuk **repository reset**
- **Backup lokal** selalu ada di komputer Anda

### **📚 Resources Tambahan:**
- [GitHub Desktop Documentation](https://docs.github.com/en/desktop)
- [Git Cheat Sheet](https://education.github.com/git-cheat-sheet-education.pdf)
- [Laravel Documentation](https://laravel.com/docs)
- [Tailwind CSS Documentation](https://tailwindcss.com/docs)

---

## 🎓 **Penilaian**

### **Yang Dinilai:**
- **Commit Quality:** Pesan commit yang jelas
- **Code Quality:** Kualitas code dan struktur project
- **Progressive Development:** Setiap week builds on previous
- **Documentation:** README dan comments yang baik
- **Git Usage:** Proper branching dan PR workflow

### **Timeline:**
- **Week-1:** Environment Setup & GitHub Integration - [Tanggal deadline]
- **Week-2:** Laravel Installation & Tailwind CSS Setup - [Tanggal deadline]
- **Week-3:** Database Design & Migration System - [Tanggal deadline]
- **Week-4:** CRUD Operations - Create & Read - [Tanggal deadline]
- **Week-5:** CRUD Operations - Update & Delete - [Tanggal deadline]
- **Week-6:** API Development & Authentication - [Tanggal deadline]
- **Week-7:** User Authentication & Authorization - [Tanggal deadline]
- **Week-8:** Production Deployment & Optimization (UTS) - [Tanggal deadline]

---

**🚀 Good luck dengan tugas UTS Cloud Computing! Remember: Commit early, commit often!**