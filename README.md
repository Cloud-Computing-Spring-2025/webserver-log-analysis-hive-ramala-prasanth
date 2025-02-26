# Web Server Log Analysis with Hadoop and Hive

## Project Overview
This project analyzes web server logs using Hadoop and Hive. It processes large log files, partitions data for efficient querying, and extracts insights such as request counts, error rates, and user behavior patterns. The project involves:
- Storing and processing logs in HDFS.
- Using Hive for structured querying and partitioning.
- Extracting insights from logs such as top visited URLs, frequent errors, and user agent distributions.

## Approach and Implementation
### **Mapper and Reducer Logic**
- **Mapper:** Extracts relevant fields (IP, timestamp, URL, status, user-agent) from raw logs and emits key-value pairs for aggregation.
- **Reducer:** Aggregates data to compute metrics like total requests, error counts, and top user agents.
- **Partitioning Strategy:** The logs are partitioned by HTTP status codes to improve query performance.

## Execution Steps
### **Docker Setup & Copying Files**
```sh
docker cp web_server_logs.csv resourcemanager:/opt/hadoop-3.2.1/share/hadoop/mapreduce/
docker cp web_server_logs.csv namenode:/tmp/
```

### **Loading Data into HDFS**
```sh
docker exec -it namenode /bin/bash
hdfs dfs -mkdir -p /user/hive/warehouse/web_logs
hdfs dfs -put /tmp/web_server_logs.csv /user/hive/warehouse/web_logs/
```

### **Creating Hive Tables**
```sql
CREATE TABLE web_server_logs_partitioned (
    ip STRING,
    timestamp_ STRING,
    url STRING,
    user_agent STRING
) PARTITIONED BY (status INT)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE;
```

### **Configuring Dynamic Partitioning**
```sql
SET hive.exec.dynamic.partition = true;
SET hive.exec.dynamic.partition.mode = nonstrict;
```

### **Inserting Data into Partitioned Table**
```sql
INSERT INTO TABLE web_server_logs_partitioned PARTITION (status)
SELECT web_server_logs.ip, web_server_logs.timestamp_, web_server_logs.url, web_server_logs.user_agent, web_server_logs.status
FROM web_server_logs;
```

### **Verifying Table Structure**
```sql
DESCRIBE web_server_logs;
```

### **Exporting Analysis Results**
```sql
INSERT OVERWRITE DIRECTORY '/user/hive/output/web_logs_analysis'
SELECT * FROM web_server_logs;
```

### **Creating External Table**
```sql
CREATE DATABASE web_logs;
USE web_logs;

CREATE EXTERNAL TABLE web_server_logs (
    ip STRING,
    timestamp_ STRING,
    url STRING,
    status INT,
    user_agent STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE
LOCATION '/user/hive/warehouse/web_logs/';
```

### **Loading Data into External Table**
```sql
LOAD DATA INPATH '/user/hive/warehouse/web_logs/web_server_logs.csv' INTO TABLE web_server_logs;
```

### **Running Queries on Log Data**
#### **1. Total Requests Count**
```sql
SELECT COUNT(*) AS total_requests FROM web_server_logs;
```
#### **2. Requests by Status Code**
```sql
SELECT status, COUNT(*) AS count FROM web_server_logs GROUP BY status;
```
#### **3. Top 3 Most Visited URLs**
```sql
SELECT url, COUNT(*) AS visits FROM web_server_logs GROUP BY url ORDER BY visits DESC LIMIT 3;
```
#### **4. Most Frequent User Agents**
```sql
SELECT user_agent, COUNT(*) AS count FROM web_server_logs GROUP BY user_agent ORDER BY count DESC;
```
#### **5. Failed Requests Per IP (More than 3 Errors)**
```sql
SELECT ip, COUNT(*) AS failed_requests
FROM web_server_logs
WHERE status IN (404, 500)
GROUP BY ip
HAVING COUNT(*) > 3;
```
#### **6. Request Frequency Over Time**
```sql
SELECT SUBSTR(timestamp_, 1, 16) AS time_slot, COUNT(*) AS request_count
FROM web_server_logs
GROUP BY SUBSTR(timestamp_, 1, 16)
ORDER BY time_slot;
```

### **Exporting Results to HDFS Output Directory**
```sql
INSERT OVERWRITE DIRECTORY '/user/hadoop/web_logs/output'
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
SELECT * FROM web_server_logs;
```

### **Verifying Output in HDFS**
```sh
hdfs dfs -ls /user/hadoop/web_logs/output
exit
```

### **Copying Output from HDFS to Host**
```sh
docker cp namenode:/opt/output output/output
```

## Challenges Faced & Solutions
### **1. Data Loading Issues**
- **Problem:** Errors while loading CSV files into Hive due to incorrect delimiters.
- **Solution:** Used `FIELDS TERMINATED BY ','` in table definitions to correctly parse CSV.

### **2. Partitioning Not Working Initially**
- **Problem:** Queries failed when inserting data into partitioned tables.
- **Solution:** Enabled dynamic partitioning with:
  ```sql
  SET hive.exec.dynamic.partition = true;
  SET hive.exec.dynamic.partition.mode = nonstrict;
  ```

### **3. Copying Data Between HDFS and Local Machine**
- **Problem:** Docker `cp` failed because data was inside HDFS, not `/opt/output`.
- **Solution:** Used:
  ```sh
  hdfs dfs -copyToLocal /user/hadoop/web_logs/output /opt/output
  ```
  before running `docker cp`.

## Sample Input and Output
### **Sample Log Entry (Input CSV Format)**
```
192.168.1.1,2024-02-25 12:34:56,/home,200,Mozilla/5.0
192.168.1.2,2024-02-25 12:35:10,/login,500,Chrome/99.0
192.168.1.3,2024-02-25 12:35:30,/about,404,Safari/14.1
```

### **Sample Output (Top Visited URLs Query)**
```
/home     |  1500
/login    |  900
/about    |  750
```

## Conclusion
This project demonstrates efficient log processing using Hadoop and Hive. It enables scalable querying and analysis of web logs, helping derive insights such as error trends, frequent visitors, and peak request times. By partitioning data and leveraging Hive’s querying capabilities, we optimize performance and improve data retrieval efficiency.

---
This `README.md` serves as documentation for the project and ensures that the repository is self-explanatory.

