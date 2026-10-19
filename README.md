# Ujian Tengah Semester (UTS) — Graf Sains Data
**Program Studi:** S1 Sains Data, Fakultas Matematika dan Ilmu Pengetahuan Alam (FMIPA)  
**Universitas Negeri Surabaya (UNESA)**  
**Materi:** Python Driver untuk Neo4j dan Sistem Rekomendasi  
**Dataset:** RetailRocket E-Commerce Recommender Dataset  

---

## 📌 Identitas Pengerjaan
- **Periode Waktu Terpilih (Filter Timestamp):**
  - **P1 (Training Fold):** 08 – 21 Juli 2015 (14 hari)
  - **P2 (Testing Fold):** 22 – 28 Juli 2015 (7 hari)
- **Database Graph:** Neo4j Community / Enterprise v2026.09 (Bolt: `bolt://localhost:7687`)
- **Bahasa & Driver:** Python 3.11, `neo4j` Python Driver, `pandas`, `scipy`, `scikit-learn`, `networkx`, `matplotlib`
- **Jupyter Notebook Utama:** [`UTS_GSD.ipynb`](UTS_GSD.ipynb) *(Seluruh 21 code cell telah dieksekusi lengkap beserta visualisasi dan outputnya)*

---

## 📂 Struktur Berkas Repository

```text
├── UTS_GSD.ipynb                    # Notebook pengerjaan lengkap (semua output & grafik ter-render)
├── README.md                        # Laporan dokumentasi UTS
├── events_p1_training.csv           # Dataset interaksi P1 (Training: 316.292 baris)
├── events_p2_testing.csv            # Dataset interaksi P2 (Testing: 177.033 baris)
├── item_properties_p1.csv           # Snapshot properti item periode P1 (1.157.232 baris)
├── item_properties_p2.csv           # Snapshot properti item periode P2 (519.460 baris)
├── user_projection_graph.png        # Visualisasi graf proyeksi User-User (NetworkX)
├── item_projection_graph.png        # Visualisasi graf proyeksi Item-Item (NetworkX)
├── graph_before_after_similarity.png# Visualisasi graf SEBELUM vs SESUDAH perhitungan similarity
├── summary_results.png              # Ringkasan tabel metrik evaluasi rekomendasi
├── list_yg _tidak_boleh.md          # Daftar periode waktu mahasiswa lain (referensi keunikan)
└── 2601_UTS-GSD.pdf                 # Naskah soal resmi UTS Graf Sains Data
```

---

## 📝 Ringkasan Jawaban Tugas UTS

### 1. Tugas Pendahuluan (Checkpoint)
* **Lingkungan Komputasi:** Seluruh proses dijalankan langsung pada **Lingkungan Python Lokal (Local Environment)** di Windows dengan pustaka lokal (`Python 3.11`, `neo4j 4.2.1`, `pandas`, `numpy`, `scipy`, `scikit-learn`, `networkx`, `matplotlib`) serta server lokal **Neo4j Desktop** (`bolt://localhost:7687`), tanpa memerlukan virtual environment terpisah (`venv`).
1. **Nama Database:** Database default yang digunakan adalah **`neo4j`**, serta database sistem **`system`** untuk administrasi instans/metadata.
2. **Akses Database Lain:**
   - Melalui CYPHER: Klausul `USE <database_name>`, contoh: `USE system SHOW DATABASES;`
   - Melalui Python Driver: Parameter database pada session: `driver.session(database="<nama_db>")`
3. **Jenis Graf Pertemanan:** **Directed Graph (Graf Berarah / DiGraph)** unipartite dengan label simpul `:Person` dan relasi tunggal berarah `[:KNOWS]`.
4. **CYPHER Total Reset:**  
   ```cypher
   MATCH (n) DETACH DELETE n;
   ```

---

### 2. Soal 2.1 — Wrangling dan EDA (Skor Maks: 45)

#### 1 & 2. Filter Periode Waktu & File CSV
- Rentang waktu:
  - **P1:** `1436313600000` s.d. `1437523199000` (08 Juli 2015 00:00:00 UTC – 21 Juli 2015 23:59:59 UTC)
  - **P2:** `1437523200000` s.d. `1438127999000` (22 Juli 2015 00:00:00 UTC – 28 Juli 2015 23:59:59 UTC)
- Berkas tersimpan: `events_p1_training.csv` (11,1 MB), `events_p2_testing.csv` (6,2 MB), `item_properties_p1.csv` (55,6 MB), `item_properties_p2.csv` (26,1 MB).

#### 3. Subgraf Neo4j & Query Komposisi
- Subgraf interaktif dibentuk dari pengunjung aktif yang berinteraksi di P1 dan memiliki riwayat pengujian di P2 (5.116 visitor, 14.062 item, 31.726 relasi).
- **Komposisi Simpul:**
  - `Item`: **14.062** simpul
  - `Visitor`: **5.116** simpul
  - *Total Simpul:* **19.178**
- **Komposisi Relasi:**
  - `VIEWED`: **29.514** relasi
  - `ADDED_TO_CART`: **1.538** relasi
  - `BOUGHT`: **674** relasi
  - *Total Relasi:* **31.726**
- **Kueri CYPHER:**
  ```cypher
  MATCH (n) RETURN labels(n)[0] AS tipe_simpul, count(n) AS jumlah ORDER BY jumlah DESC;
  MATCH ()-[r]->() RETURN type(r) AS jenis_relasi, count(r) AS jumlah ORDER BY jumlah DESC;
  ```

#### 4 & 5. Vector Space Modelling (VSM) User dan Item
- **Bobot Event:** `view = 1`, `addtocart = 2`, `transaction = 3`
- **Matriks User-Item:** $5.116 \times 14.062$, Sparsity: **99.9726%**
- **Proyeksi User-User:** $U = A_{\text{norm}} \cdot A_{\text{norm}}^T$ ($5.116 \times 5.116$, 21.940 pasangan similaritas $>0$)
- **Proyeksi Item-Item:** $I = A_{\text{norm\_item}}^T \cdot A_{\text{norm\_item}}$ ($14.062 \times 14.062$, 2.535.664 pasangan similaritas $>0$)

#### 6. Komparasi Proyeksi: User vs Item
> **Kesimpulan:** Proyeksi **Item-Item** jauh lebih efektif dan efisien dibandingkan User-User karena:
> 1. Menghasilkan informasi co-interaksi yang jauh lebih padat (2,53M pasangan vs 21,9K pasangan).
> 2. Katalog produk dalam e-commerce bersifat stabil, sehingga perhitungan *similarity* antar-produk dapat dilakukan secara *pre-computed/offline* dan tahan terhadap masalah *user cold-start*.

#### 7. Perhitungan Similarity & Simpan ke Neo4j
- **Item Acuan:** `itemId = 445351` (Item terpopuler di P1 dengan 84 interaksi).
- **Top 5 Similar Items (Cosine Similarity):**
  1. Item `213834` (Cosine: **0.557748**)
  2. Item `138636` (Cosine: **0.500000**)
  3. Item `350865` (Cosine: **0.375000**)
  4. Item `334520` (Cosine: **0.300000**)
  5. Item `320130` (Cosine: **0.278533**)
- Dituliskan ke graf Neo4j sebagai relasi `[:SIMILAR_TO]` dengan atribut properti `cosine`.
- **Kueri CYPHER Pembuktian:**
  ```cypher
  MATCH (anchor:Item {itemId: 445351})-[s:SIMILAR_TO]->(target:Item)
  RETURN anchor.itemId AS anchorItem, target.itemId AS similarItem, s.cosine AS cosineSimilarity
  ORDER BY s.cosine DESC LIMIT 5;
  ```

#### 8. Visualisasi Kondisi SEBELUM vs SESUDAH
- **SEBELUM:** Graf bipartite murni `:Visitor -[:VIEWED | :ADDED_TO_CART | :BOUGHT]-> :Item`
- **SESUDAH:** Graf diperkaya relasi unipartite `:Item -[:SIMILAR_TO {cosine: ...}]-> :Item`
- Disimpan dalam: `graph_before_after_similarity.png`.

---

### 3. Soal 2.2 — Analytics (Skor Maks: 55)

#### 1. Pemilihan User Acak & Rekomendasi KNN (k=5)
- **Random Seed:** `RANDOM_SEED = 298` (Deterministik & Reproducible)
- **User Terpilih:** `visitorId = 85734`
- **5 Tetangga Terdekat (KNN):**
  1. Visitor `896066` (sim: 0.4312)
  2. Visitor `546086` (sim: 0.1406)
  3. Visitor `895999` (sim: 0.1082)
  4. Visitor `1403769` (sim: 0.0962)
  5. Visitor `234291` (sim: 0.0802)
- **3 Item yang Diiklankan (Rekomendasi Teratas):**
  1. **Item `374835`** (Skor: **3.0185**)
  2. **Item `132068`** (Skor: **2.5873**)
  3. **Item `312728`** (Skor: **1.9355**)

#### 2. Evaluasi Rekomendasi: Hit@5 dan NDCG@5
Evaluasi diuji terhadap *ground truth* riil pada `events_p2_testing.csv` (periode testing 22–28 Juli 2015):
- **Hit@5:** **`1` (BERHASIL HIT ✅)**  
  *Penjelasan:* Item rekomendasi peringkat ke-3 (**`itemId = 312728`**) berhasil muncul di riwayat pengujian riil P2 pengguna.
- **DCG@5:** **`0.5000`**  
  $$\text{DCG}@5 = \frac{1}{\log_2(3 + 1)} = \frac{1}{2} = 0.5000$$
- **IDCG@5:** **`2.9485`**  
- **NDCG@5:** **`0.1696` (16.96%)**  
  $$\text{NDCG}@5 = \frac{0.5000}{2.9485} \approx 0.1696$$

---

## 🚀 Cara Menjalankan Notebook
1. Pastikan database Neo4j aktif pada port `7687` (`bolt://localhost:7687`).
2. Buka environment Python yang telah terpasang paket dependensi.
3. Buka dan jalankan seluruh sel pada notebook [UTS_GSD.ipynb](UTS_GSD.ipynb).
