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
