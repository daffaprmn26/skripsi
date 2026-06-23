# Lab Otomasi Migrasi CentOS ke RHEL

Repository ini digunakan untuk membangun lab migrasi CentOS Linux 7.9 menuju Red Hat Enterprise Linux menggunakan Ansible Automation Platform dan Convert2RHEL.

## Urutan Pelaksanaan

### 1. Persiapan AAP

Pastikan komponen berikut tersedia:

* Project: `Provisioning`
* Inventory: `AWS Provisioning`
* Host provisioning: `localhost-aws`
* Credential AWS: `AWS`
* Credential SSH CentOS: `CentOS SSH`
* Execution Environment: `Product Demos EE`

Variabel host `localhost-aws`:

```yaml
ansible_connection: local
ansible_python_interpreter: "{{ ansible_playbook_python }}"
```

---

### 2. Uji Akses AWS

Playbook:

```text
playbooks/00_test_aws_access.yml
```

Tujuan:

* Memastikan credential AWS dapat digunakan.
* Menampilkan AWS Account, ARN, dan User ID.

Job Template:

```text
00 - Test AWS Access
```

---

### 3. Provisioning Jaringan AWS

Playbook:

```text
playbooks/01_provision_network.yml
```

Resource yang dibuat:

* VPC
* Public subnet
* Internet Gateway
* Route Table
* Default route ke internet

Job Template:

```text
01 - Provision AWS Network
```

---

### 4. Provisioning Akses AWS

Playbook:

```text
playbooks/02_provision_access.yml
```

Resource yang dibuat:

* AWS EC2 Key Pair
* Security Group
* Akses SSH dari AAP
* Akses SSH dari IP laptop

Job Template:

```text
02 - Provision AWS Access
```

Public key harus tersedia pada:

```text
playbooks/files/ansible_aws_lab.pub
```

Private key hanya disimpan di laptop dan AAP Credential. Jangan disimpan di Git.

---

### 5. Provisioning CentOS

Playbook:

```text
playbooks/03_provision_centos.yml
```

Spesifikasi lab:

* OS: CentOS Linux 7.9.2009
* Architecture: `x86_64`
* Instance type: `t3.medium`
* Root volume: `30 GiB`
* SSH user: `ec2-user`

Job Template:

```text
03 - Provision CentOS Source
```

AMI yang digunakan:

```text
ami-0a170f4e0a24ed220
```

AMI ID bersifat khusus region dan environment. Periksa kembali apabila environment AWS berubah.

---

### 6. Bootstrap Awal CentOS

Masuk ke server:

```bash
ssh -i ~/.ssh/ansible_aws_lab ec2-user@PUBLIC_IP
```

Verifikasi OS:

```bash
cat /etc/centos-release
cat /etc/os-release
rpm -q centos-release
rpm -q centos-stream-release
uname -m
```

Hasil yang diharapkan:

```text
CentOS Linux release 7.9.2009 (Core)
x86_64
```

Backup repository:

```bash
sudo cp -a /etc/yum.repos.d/CentOS-Base.repo \
  /etc/yum.repos.d/CentOS-Base.repo.backup
```

Arahkan repository ke CentOS Vault:

```bash
sudo tee /etc/yum.repos.d/CentOS-Base.repo > /dev/null <<'EOF'
[base]
name=CentOS-7.9.2009 - Base
baseurl=https://vault.centos.org/7.9.2009/os/$basearch/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

[updates]
name=CentOS-7.9.2009 - Updates
baseurl=https://vault.centos.org/7.9.2009/updates/$basearch/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

[extras]
name=CentOS-7.9.2009 - Extras
baseurl=https://vault.centos.org/7.9.2009/extras/$basearch/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
EOF
```

Bersihkan cache dan tes repository:

```bash
sudo yum clean all
sudo rm -rf /var/cache/yum
sudo yum makecache
sudo yum repolist
```

Instal Python 3:

```bash
sudo yum install -y python3
python3 --version
```

---

### 7. Uji Koneksi AAP ke CentOS

Playbook:

```text
playbooks/04_test_centos_access.yml
```

Tujuan:

* Menemukan EC2 berdasarkan tag.
* Menambahkan host ke temporary inventory.
* Menguji koneksi SSH.
* Menguji sudo.
* Menjalankan gathering facts.

Job Template:

```text
04 - Test CentOS Access
```

Credential:

```text
AWS
CentOS SSH
```

Target hasil:

```text
ping: pong
Distribution: CentOS
Version: 7.9
Architecture: x86_64
failed=0
unreachable=0
```

---

### 8. Precheck Migrasi

Playbook:

```text
playbooks/05_precheck_migration.yml
```

Pemeriksaan utama:

* Sistem operasi CentOS Linux.
* Versi CentOS Linux 7.9.2009.
* Bukan CentOS Stream.
* Arsitektur `x86_64`.
* FIPS tidak aktif.
* Repository Vault aktif.
* Repository dapat digunakan.
* Tidak ada package update tersisa.
* Package `sos` terpasang.

Parameter operasional tambahan:

* Memory minimum penelitian.
* Ruang kosong filesystem `/`.
* Ruang kosong filesystem `/var`.

Parameter operasional hanya menghasilkan warning dan tidak menggantikan pemeriksaan resmi.

Job Template:

```text
05 - Migration Precheck
```

Credential:

```text
AWS
CentOS SSH
```

---

## Skenario Pengujian

### Skenario Gagal

Jalankan precheck sebelum melakukan remediasi.

Contoh hasil:

```text
Masih terdapat package CentOS yang belum diperbarui
Package sos belum terpasang
```

Status:

```text
Official stage-one passed: False
failed=1
```

Simpan hasil ini sebagai sampel kondisi tidak layak.

### Remediasi

Jalankan pada server CentOS:

```bash
sudo yum update -y
sudo yum install -y sos
sudo reboot
```

Setelah server aktif:

```bash
rpm -q sos
sudo yum -q check-update
echo $?
```

Target return code:

```text
0
```

### Skenario Berhasil

Jalankan ulang:

```text
05 - Migration Precheck
```

Target hasil:

```text
Official stage-one passed: True
Official failures: []
failed=0
```

Status tersebut berarti server lolos pemeriksaan awal dan dapat dilanjutkan ke persiapan Convert2RHEL serta `convert2rhel analyze`.

---

## Alur Singkat

```text
00 Test AWS Access
        ↓
01 Provision AWS Network
        ↓
02 Provision AWS Access
        ↓
03 Provision CentOS Source
        ↓
Bootstrap Vault dan Python
        ↓
04 Test CentOS Access
        ↓
05 Migration Precheck
        ↓
Remediasi jika gagal
        ↓
Convert2RHEL Analyze
        ↓
Konversi CentOS ke RHEL
        ↓
Verifikasi hasil migrasi
```

## Catatan

* Semua playbook dibuat reusable dan idempotent.
* Hindari hard-coded VPC ID, Subnet ID, Security Group ID, dan Public IP.
* Resource AWS ditemukan menggunakan tag.
* Private key dan credential tidak boleh dimasukkan ke Git.
* AMI ID dapat berubah ketika region atau environment AWS berubah.
* Konfigurasi repository Vault dan instalasi Python saat ini masih dilakukan manual dan nantinya dapat dipindahkan ke playbook bootstrap otomatis.

