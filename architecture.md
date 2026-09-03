# CI/CD Pipeline Architecture: Build & Push Docker Image

Dokumen ini mendokumentasikan arsitektur dan alur kerja (*pipeline*) otomatisasi CI/CD yang dikonfigurasi menggunakan **GitHub Actions** pada repositori ini.

---

## 1. Diagram Alur Proses (Pipeline Flowchart)

Alur kerja berikut berjalan secara otomatis di runner **Ubuntu** (`ubuntu-latest`) saat ada *event* tertentu:

```mermaid
flowchart TD
    subgraph Trigger ["1. Trigger Event"]
        A1["Push ke branch 'main'"]
        A2["Manual Trigger ('workflow_dispatch')"]
    end

    subgraph GitHubRunner ["2. GitHub Actions Runner (ubuntu-latest)"]
        direction TB
        
        B["Step 1: Checkout Repository<br/>actions/checkout@v6"]
        C["Step 2: Validasi & Sanitasi Variabel<br/>Cek Secret & Konversi ke Lowercase"]
        D["Step 3: Docker Hub Login<br/>docker/login-action@v4"]
        E["Step 4: Build Docker Image<br/>Tag: SHA & 'latest'"]
        F["Step 5: Ephemeral Smoke Test<br/>Jalankan container: 'nginx -v'"]
        G["Step 6: Security Vulnerability Scan<br/>aquasecurity/trivy-action@v0.36.0"]
        H["Step 7: Push Image ke Docker Hub<br/>Upload tag SHA dan 'latest'"]
        
        B --> C
        C --> D
        D --> E
        E --> F
        F --> G
        G --> H
    end

    subgraph External ["3. External Services"]
        DH[("Docker Hub Registry<br/>hub.docker.com")]
    end

    A1 --> B
    A2 --> B
    H -->|Push Image Layers| DH
```

---

## 2. Sequence Diagram Interaksi Komponen

Diagram berikut menggambarkan interaksi antara Developer, GitHub, Runner, dan Docker Hub Registry:

```mermaid
sequenceDiagram
    autonumber
    actor Dev as Developer
    participant GH as GitHub Repository
    participant Runner as GitHub Actions Runner
    participant DH as Docker Hub Registry

    Dev->>GH: Push commit ke branch 'main' / Run Workflow
    GH->>Runner: Trigger job 'build-and-deploy'
    
    activate Runner
    Runner->>GH: Step 1: Checkout kode sumber
    Runner->>Runner: Step 2: Validasi kredensial & format lowercase
    
    Runner->>DH: Step 3: Autentikasi akun Docker Hub
    DH-->>Runner: Login Sukses
    
    Runner->>Runner: Step 4: Build image Docker dari Dockerfile
    Runner->>Runner: Step 5: Smoke test (nginx -v)
    Runner->>Runner: Step 6: Scan kerentanan keamanan (Trivy)
    
    Runner->>DH: Step 7: Push image :commit-sha
    Runner->>DH: Step 7: Push image :latest
    deactivate Runner
    
    DH-->>Dev: Image siap di-pull dari Docker Hub
```

---

## 3. Rincian Setiap Tahapan (*Steps*)

| No | Tahapan (*Step*) | Action / Perintah | Deskripsi & Tujuan |
|---|---|---|---|
| **1** | **Source Checkout** | `actions/checkout@v6` | Mengambil (*fetch*) seluruh kode sumber repositori ke dalam direktori kerja runner. |
| **2** | **Validasi & Format Variabel** | Bash Shell Script | Memvalidasi keberadaan kredensial (`DOCKERHUB_USERNAME` & `DOCKERHUB_TOKEN`). Mengonversi nama repositori dan username ke format huruf kecil (*lowercase*) sesuai standar penamaan Docker. |
| **3** | **Docker Hub Login** | `docker/login-action@v4` | Melakukan otentikasi aman ke Docker Hub menggunakan kredensial rahasia yang tersimpan di GitHub Secrets. |
| **4** | **Build Docker Image** | `docker build` | Membangun image Docker berdasarkan `Dockerfile` dan langsung menyematkan 2 tag: `:commit-sha` dan `:latest`. |
| **5** | **Smoke Test Container** | `docker run --rm ... nginx -v` | Menjalankan container secara *ephemeral* untuk memastikan runtime Nginx berjalan normal sebelum image dipublikasikan. |
| **6** | **Trivy Vulnerability Scan** | `aquasecurity/trivy-action@v0.36.0` | Memindai paket OS dan *library* di dalam container untuk mencari celah keamanan (*CVE*). Berjalan secara informatif (`exit-code: '0'`, `continue-on-error: true`). |
| **7** | **Push to Docker Hub** | `docker push` | Mengunggah layer image Docker beserta tag commit SHA dan tag `latest` ke Docker Hub. |

---

## 4. Konfigurasi Kredensial & Lingkungan

Alur kerja ini memerlukan konfigurasi rahasia pada menu **Settings > Secrets and variables > Actions** di repositori GitHub:

* **`DOCKERHUB_USERNAME`**: Username akun Docker Hub tempat image disimpan.
* **`DOCKERHUB_TOKEN`**: Personal Access Token (PAT) Docker Hub dengan izin *Read & Write*.

> [!NOTE]
> Alur kerja ini mendukung pembacaan kredensial baik dari **GitHub Secrets** maupun **GitHub Variables** secara fleksibel menggunakan sintaks fallback: `${{ secrets.DOCKERHUB_USERNAME || vars.DOCKERHUB_USERNAME }}`.

---

## 5. Strategi Penandaan (*Image Tagging Strategy*)

Setiap kali alur kerja dijalankan, dua jenis tag dibuat dan diunggah ke registry:

1. **`:<commit-sha>`** (Contoh: `push-image:8b706fd...`):
   * Bersifat *immutable* (tidak dapat ditimpa).
   * Memudahkan pelacakan kode sumber (*auditability & traceability*) dari commit tertentu ke container yang sedang berjalan di *production*.
2. **`:latest`**:
   * Selalu menunjuk pada versi build paling baru dari branch `main`.
   * Memudahkan pengguna untuk menarik (*pull*) versi paling mutakhir secara instan tanpa perlu mengetahui hash commit.
