# 📦 Implementasi Sistem - TIBD Kelompok 4

# 📦 Implementasi Sistem Big Data dengan Apache Oozie dan Hive

Dokumentasi langkah-langkah implementasi sistem Big Data untuk analisis ekspedisi terbanyak menggunakan Apache Oozie dan Hive di lingkungan Cloudera.

## A. Apache Oozie

### 1. Menjalankan Cloudera VM dan Memastikan Oozie Aktif

```bash
oozie admin -oozie http://localhost:11000/oozie -status
```

### 2. Membuat Struktur Workflow

```bash
mkdir -p ~/oozie/ekspedisi_workflow
cd ~/oozie/ekspedisi_workflow
```

### 3. Membuat File workflow.xml

```bash
gedit workflow.xml
```

Isi file:

```xml
<workflow-app name="ecommerce-wf" xmlns="uri:oozie:workflow:0.5" xmlns:hive="uri:oozie:hive-action:0.2">
 <start to="hive-node"/>
 <action name="hive-node">
   <hive:hive>
     <hive:job-tracker>${jobTracker}</hive:job-tracker>
     <hive:name-node>${nameNode}</hive:name-node>
     <hive:script>/user/cloudera/oozie/ekspedisi_workflow/ekspedisi_analysis.q</hive:script>
   </hive:hive>
   <ok to="end"/>
   <error to="fail"/>
 </action>
 <kill name="fail">
   <message>Workflow gagal pada langkah: ${wf:LastErrorNode()}</message>
 </kill>
 <end name="end"/>
</workflow-app>
```

### 4. Membuat File Hive Script (ekspedisi_analysis.q)

```bash
gedit ekspedisi_analysis.q
```

Isi file:

```sql
-- Membuat database ecommerce
CREATE DATABASE IF NOT EXISTS ecommerce;
USE ecommerce;

-- Membuat tabel dataset ecommerce
CREATE EXTERNAL TABLE IF NOT EXISTS ecommerce_orders (
 order_id INT,
 product_id INT,
 order_date STRING,
 courier_delivery STRING,
 city STRING,
 district STRING,
 type_of_delivery STRING,
 estimated_delivery_time_days INT,
 product_rating INT,
 ontime STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE
LOCATION '/user/cloudera/oozie/ekspedisi_workflow';

-- Query ekspedisi terbanyak
SELECT city, courier_delivery AS ekspedisi_terbanyak, COUNT(*) AS jumlah
FROM ecommerce_orders
GROUP BY city, courier_delivery
ORDER BY jumlah DESC;
```

### 5. Membuat File job.properties

```bash
gedit job.properties
```

Isi:

```properties
nameNode=hdfs://localhost:8020
jobTracker=localhost:803
queueName=default
oozie.wf.application.path=${nameNode}/user/cloudera/oozie/ekspedisi_workflow
```

### 6. Upload File ke HDFS

```bash
hdfs dfs -mkdir -p /user/cloudera/oozie/ekspedisi_workflow

hdfs dfs -put ~/oozie/ekspedisi_workflow/dataset_ecommerce.csv /user/cloudera/oozie/ekspedisi_workflow/
hdfs dfs -put ~/oozie/ekspedisi_workflow/ekspedisi_analysis.q /user/cloudera/oozie/ekspedisi_workflow/
hdfs dfs -put ~/oozie/ekspedisi_workflow/workflow.xml /user/cloudera/oozie/ekspedisi_workflow/
hdfs dfs -put ~/oozie/ekspedisi_workflow/job.properties /user/cloudera/oozie/ekspedisi_workflow/
```

### 7. Cek dan Hapus File di HDFS (Jika Perlu)

```bash
hdfs dfs -ls /user/cloudera/oozie/ekspedisi_workflow
hdfs dfs -rm /user/cloudera/oozie/ekspedisi_workflow/[nama_file]
hdfs dfs -rm -r /user/cloudera/oozie/ekspedisi_workflow
```

### 8. Menjalankan Workflow

```bash
cd ~/oozie/ekspedisi_workflow
oozie job -oozie http://localhost:11000/oozie -config job.properties -run
```

---

## B. Analisis Hive untuk Ekspedisi Terbanyak

### 1. Masuk ke Hive CLI

```bash
hive
```

### 2. Menggunakan Database dan Membuat Tabel (jika belum ada)

```sql
USE ecommerce;

CREATE TABLE IF NOT EXISTS ekspedisi (
 order_id INT,
 product_id INT,
 order_date STRING,
 courier_delivery STRING,
 city STRING,
 district STRING,
 type_of_delivery STRING,
 estimated_delivery_time_days INT,
 product_rating INT,
 ontime STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE;
```

### 3. Load Data ke Tabel Hive

```sql
LOAD DATA INPATH '/user/cloudera/oozie/ekspedisi_workflow/dataset_ecommerce.csv' INTO TABLE ekspedisi;
```

### 4. Cek Isi Tabel

```sql
SELECT * FROM ekspedisi LIMIT 10;
```

### 5. Query Ekspedisi Terbanyak per Kota

```sql
INSERT OVERWRITE DIRECTORY '/user/cloudera/output_ekspedisi_terbanyak'
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
SELECT city, courier_delivery AS ekspedisi_terbanyak, jumlah, persen_penggunaan
FROM (
 SELECT e.city, e.courier_delivery, COUNT(*) AS jumlah,
        ROUND((COUNT(*) / t.total) * 100, 2) AS persen_penggunaan,
        ROW_NUMBER() OVER (PARTITION BY e.city ORDER BY COUNT(*) DESC) AS rank
 FROM ekspedisi e
 JOIN (
   SELECT city, COUNT(*) AS total FROM ekspedisi GROUP BY city
 ) t ON e.city = t.city
 GROUP BY e.city, e.courier_delivery, t.total
) ranked
WHERE rank = 1
ORDER BY jumlah DESC;
```

### 6. Menampilkan Output

```bash
hdfs dfs -ls /user/cloudera/output_ekspedisi_terbanyak
hdfs dfs -cat /user/cloudera/output_ekspedisi_terbanyak/000000_0
```

### 7. Menghapus Output Lama (jika perlu)

```bash
hdfs dfs -rm /user/cloudera/output_ekspedisi_terbanyak/000000_0
```

### 8. Menambahkan Header dan Konversi ke CSV

```bash
echo -e "city,courier_delivery,ekspedisi_terbanyak,jumlah,persen_penggunaan" > /tmp/header.csv

hdfs dfs -cat /user/cloudera/output_ekspedisi_terbanyak/* > /tmp/output_without_header.csv

cat /tmp/header.csv /tmp/output_without_header.csv > /tmp/output_with_header.csv

mv /tmp/output_with_header.csv /home/cloudera/output_with_header.csv

cat /home/cloudera/output_with_header.csv
```

## C. MapReduce WordCount Menggunakan Spark

### 1. Ekstraksi Kolom Ekspedisi dari CSV

```bash
cut -d',' -f4 ecommerce.csv > courier_only.txt
```

### 2. Upload ke HDFS

```bash
hdfs dfs -mkdir -p /user/cloudera/input_courier
hdfs dfs -put -f courier_only.txt /user/cloudera/input_courier/
```

### 3. Menjalankan Spark-shell

```bash
spark-shell
```

### 4. Eksekusi WordCount (Tanpa Tuning)

```scala
val t0 = System.nanoTime()
val data = sc.textFile("hdfs:///user/cloudera/input_courier/courier_only.txt")
val noHeader = data.filter(line => line != "courier_delivery")
val cached = noHeader.map(_.trim).cache()
cached.count()
val counts = cached.map(kurir => (kurir, 1)).reduceByKey(_ + _)
counts.collect().foreach(println)
val t1 = System.nanoTime()
println("Durasi (non-tuned): " + (t1 - t0)/1e9 + " detik")
```

### 5. Caching Data (Opsional)

```scala
val cachedData = data.cache()
cachedData.count()
```

### 6. Keluar dari Spark-shell

```bash
:quit
```

### 7. Jalankan Spark-shell dengan Tuning

```bash
spark-shell --executor-memory 2G --driver-memory 1G --conf spark.default.parallelism=4
```

### 8. Eksekusi WordCount (Dengan Tuning)

```scala
val t0 = System.nanoTime()
val data = sc.textFile("hdfs:///user/cloudera/input_courier/courier_only.txt")
val noHeader = data.filter(line => line != "courier_delivery")
val cached = noHeader.map(_.trim).cache()
cached.count()
val counts = cached.map(kurir => (kurir, 1)).reduceByKey(_ + _)
counts.collect().foreach(println)
val t1 = System.nanoTime()
println("Durasi (tuned): " + (t1 - t0)/1e9 + " detik")
```

## D. Visualisasi di Google Colab

### 🔧 Setup dan Import Dataset

```python
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns

from google.colab import drive
drive.mount('/content/drive')

path = '/content/drive/MyDrive/Kaggle/'
df_peringkat = pd.read_csv(path + 'TIBD_peringkat_ekspedisi.csv')
df_rating = pd.read_csv(path + 'TIBD_rating_vs_ontime.csv')
df_terbanyak = pd.read_csv(path + 'TIBD_hasil_ekspedisi_terbanyak.csv')

df_peringkat['jumlah_delayed'] = df_peringkat['total_pengiriman'] - df_peringkat['jumlah_ontime']
```

### 📊 Visualisasi 1: Barplot Persentase On-Time

```python
plt.figure(figsize=(12, 7))
ax = sns.barplot(data=df_peringkat, x='courier_delivery', y='persen_ontime',
                 palette='husl')

for p in ax.patches:
    ax.annotate(f"{p.get_height():.2f}%", (p.get_x() + p.get_width()/2., p.get_height()),
                ha='center', va='bottom', fontsize=11, xytext=(0, 5), textcoords='offset points')

plt.title('Performance Pengiriman Ekspedisi')
plt.xlabel('Ekspedisi')
plt.ylabel('Persentase On-Time (%)')
plt.ylim(88, 92)
plt.tight_layout()
plt.show()
```

### 📈 Visualisasi 2: Bubble Chart Rating vs Status

```python
plt.figure(figsize=(10, 6))
sns.scatterplot(data=df_rating, x='ontime_status', y='rata_rating_produk',
                size='total_transaksi', hue='ontime_status', palette='husl', sizes=(500, 2000))

for i, row in df_rating.iterrows():
    plt.text(row['ontime_status'], row['rata_rating_produk'], f"{row['total_transaksi']//1000}k",
             ha='center', va='center', fontsize=10)

plt.title('Hubungan Status Pengiriman dan Rating Produk')
plt.tight_layout()
plt.show()
```

### 🍩 Visualisasi 3: Donut Chart Status Pengiriman

```python
plt.figure(figsize=(8, 8))
colors = ['#e74c3c', '#2ecc71']
plt.pie(df_rating['total_transaksi'], labels=df_rating['ontime_status'],
        autopct='%1.1f%%', colors=colors, startangle=90,
        wedgeprops=dict(width=0.4))

plt.gca().add_artist(plt.Circle((0, 0), 0.3, color='white'))
plt.title('Distribusi Status Pengiriman')
plt.tight_layout()
plt.show()
```

### 🌡️ Visualisasi 4: Heatmap Ekspedisi per Kota

```python
pivot_data = df_terbanyak.pivot(index="city", columns="courier_delivery", values="persen_penggunaan")
plt.figure(figsize=(14, 8))
sns.heatmap(pivot_data, annot=True, cmap="YlGnBu", fmt=".1f", linewidths=.5)
plt.title('Heatmap Popularitas Ekspedisi per Kota')
plt.tight_layout()
plt.show()
```

### 📦 Visualisasi 5: Treemap Total Penggunaan Ekspedisi

```python
!pip install squarify
import squarify

df_agg = df_terbanyak.groupby('courier_delivery')['jumlah_penggunaan'].sum().reset_index()
plt.figure(figsize=(14, 8))
squarify.plot(sizes=df_agg['jumlah_penggunaan'], label=df_agg['courier_delivery'], alpha=0.8)
plt.title('Total Penggunaan Ekspedisi di Semua Kota')
plt.axis('off')
plt.show()
```

### 📡 Visualisasi 6: Radar Chart Perbandingan Ekspedisi

```python
from math import pi

df_peringkat['total_scaled'] = (df_peringkat['total_pengiriman'] / df_peringkat['total_pengiriman'].max()) * 100
categories = ['Persen On-time', 'Total Pengiriman (scaled)', 'Rata Estimasi Hari']
N = len(categories)
angles = [n / float(N) * 2 * pi for n in range(N)] + [0]

plt.figure(figsize=(8, 8))
ax = plt.subplot(111, polar=True)
ax.set_theta_offset(pi / 2)
ax.set_theta_direction(-1)
plt.xticks(angles[:-1], categories)

for i, row in df_peringkat.iterrows():
    values = [row['persen_ontime'], row['total_scaled'], row['rata_estimasi_hari'] * 20]
    values += values[:1]
    ax.plot(angles, values, linewidth=2, linestyle='solid', label=row['courier_delivery'])
    ax.fill(angles, values, alpha=0.1)

plt.title('Perbandingan Komprehensif Ekspedisi')
plt.legend(loc='upper right', bbox_to_anchor=(1.3, 1.1))
plt.tight_layout()
plt.show()
```

### 🧩 Visualisasi 7: Small Multiples Penggunaan Ekspedisi per Kota

```python
g = sns.FacetGrid(df_terbanyak, col='courier_delivery', col_wrap=3, height=4)
g.map(sns.barplot, 'city', 'persen_penggunaan', order=df_terbanyak['city'].unique())
g.set_titles("{col_name}")
g.set_xticklabels(rotation=90)
plt.tight_layout()
plt.show()
```

### 🏙️ Visualisasi 8: Top 5 Kota dengan Penggunaan Tertinggi

```python
top_cities = df_terbanyak.groupby('city')['jumlah_penggunaan'].sum().nlargest(5).index
df_top = df_terbanyak[df_terbanyak['city'].isin(top_cities)]

plt.figure(figsize=(12, 6))
sns.barplot(x='jumlah_penggunaan', y='city', hue='courier_delivery', data=df_top)
plt.title('Top 5 Kota dengan Penggunaan Ekspedisi Tertinggi')
plt.tight_layout()
plt.show()
```

### 🌍 Visualisasi 9: Distribusi Ekspedisi per Kota (Semua)

```python
plt.figure(figsize=(14, 8))
sns.barplot(x='jumlah_penggunaan', y='city', hue='courier_delivery', data=df_terbanyak)
plt.title('Distribusi Penggunaan Jasa Ekspedisi di Setiap Kota')
plt.tight_layout()
plt.show()
```
