# Parquet as long term storage data format

Parquet was launched and developed by Cloudera and Twitter to serve as a column-based storage format in 2013. It's 
optimized for work with multi-column datasets. You can visit their [official site](https://parquet.apache.org/)

**Parquet files are binary files that contain metadata about their content**. It means without reading/parsing the 
content of the file(s), we can just rely on the metadata to determine column names, compression/encodings, data types 
and even some basic statistics. **The column metadata for a Parquet file is stored at the end of the file, which 
allows for fast, one-pass writing.**

**Parquet is optimized for the Write Once Read Many (WORM) paradigm. It’s slow to write, but incredibly fast to read, 
especially when you’re only accessing a subset of the total columns. Parquet is a good choice for read-heavy workloads. 
For use cases requiring operating on entire rows of data, a format like CSV or AVRO should be used.**

For more details about the parquet data format, please visit [Parquet format explained](https://github.com/pengfei99/ParquetPyArrow/blob/main/docs/Parquet_format_Introduction.md)

## 1. Why we choose parquet?
Normally, when we evaluate a data format, we use the following basic properties. 

- Human Readable
- Compressable
- Splittable
- Complex data structure
- Schema evolution
- Columnar(for better compression and operation performance)
- Framework support

|Property |CSV |Json|Parquet|Avro|ORC|
|---------|----|----|-------|----|---|
|Human Readable|YES|YES|NO|NO|NO|
|Compressable|YES|YES|YES|YES|YES|
|Splittable|YES*|YES*|YES|YES|YES|
|Complex data structure|NO|YES|YES|YES|YES|
|Schema evolution|NO|NO|YES|YES|YES|
|Columnar|NO|NO|YES|NO|YES|
|Framework support|YES++|YES++|YES+|YES+|YES|

Note:

1. CSV is splittable when it is a raw, uncompressed file or using a splittable compression format such as BZIP2 or LZO (note: LZO needs to be indexed to be splittable!)
2. JSON has the same conditions about splittability when compressed as CSV with one extra difference. When “wholeFile” option is set to true in Spark(re: SPARK-18352), JSON is NOT splittable.

More chocking Number: (todo: add job latency time) 

``` shell
mc ls --summarize s3/pengfei/diffusion/data_format/ny_taxis/parquet/compress/2019_gzip | grep "Total Size"
Total Size: 3.8 GiB
mc ls --summarize s3/pengfei/diffusion/data_format/ny_taxis/csv/2009 | grep "Total Size"
Total Size: 22 GiB
```

### 1.2. Operation latency evaluation for all above data formats 

We have benchmarked all above data formats for the common data operations latency such as:
- read/write
- get basic stats (min, max, avg, count)
- Random data lookup
- Filtering/GroupBy(column-wise)
- Distinct(row-wise)

You can check the results in below figure:

![Common data operations latency](https://raw.githubusercontent.com/pengfei99/ParquetPyArrow/main/img/format_op_latency_stats.PNG)

For more details about the benchmark, please check [data format overview](https://github.com/pengfei99/data_format_and_optimization/blob/main/notebooks/data_format_overview.ipynb)

After the above analysis, we can say that Orc and Parquet are the best data formats for OLAP applications. They both support various compression algorithms which reduce significantly disk usage. They are both very efficient on columnar-oriented data analysis operations. 

**Parquet has better support on nested data types than Orc**. Orc loses compression ratio and analysis performance when data contains complex nested data types.

**Orc supports data update and ACID (atomicity, consistency, isolation, durability). Parquet does not**, so if you want to update a Parquet file, you need to create a new one based on the old one.

**Parquet has better interoperability** compare to Orc. Because almost all data analysis tools and framework supports parquet. Orc is only supported by Spark, Hive, Impala, MapReduce.

### 1.3 Long-term storage

Parquet is designed for long-term storage and archival purposes, meaning if you write a file today, you can expect that any system that says they can “read Parquet” will be able to read the file in 5 years or 10 years.

### 1.4 Dis-advantage of parquet

- If you need to constantly update(write) your data, do not use columnar-based data formats such as ORC, Parquet. 
- If you need to constantly update your data schema, do not use Parquet. Use Avro.

### 1.5 Choose your format based on your project
There is no magic format that can solve all your problems. But you can find a format that is the best for your project. Define your project requirements first, then quantify these requirements into metrics. At last, evaluate each data foramt based on the metric. This allows you to find the best data format for your project.  

## 2. Compatibility issues

As parquet is a standard, and is implemented by various frameworks. Some frameworks do not respect or implemented 100% of the parquet standard. This means one parquet file that are generated by one framework may not be readable by another framework. We did not find an official doc which address this issue completely. As a result, the below list may not be complete. So far, we have found four possible compatibility issues:

- Supported compression algorithme 
- Timestamp unity variation (e.g. nanoseconds, microseconds, seconds )
- Unsupported data type
- Metadata specification variation

### 2.1 Supported compression algorithme
Parquet allows us to use various compression algorithme to compress each column. 

Parquet officially supports the following compression algorithms:

- UNCOMPRESSED = 0;
- SNAPPY = 1;
- GZIP = 2;
- LZO = 3;
- BROTLI = 4; (Added in 2.4)
- LZ4 = 5;    (Added in 2.4)
- ZSTD = 6;   (Added in 2.4)

But not all implementation implement all the compression algorithms. For example, pyarrow implements all except **LZO**. Spark by default only includes the **GZIP, and SNAPPY**. For the rest of the algorithme, we need to include the compression codec by ourselves. 

Below shows a benchmark on parquet write time and file size with various framework and compression type. The origin data
is in csv format(pengfei/diffusion/data_format/Fire_Department_Calls_for_Service.csv), and it takes about 1.9GB. You can
notice even without compression, parquet format requires only 1.2 GB to store the same data.

![Parquet_compression_stats](https://raw.githubusercontent.com/pengfei99/ParquetPyArrow/main/img/parquet_compression_stats.png)

There are other implementation differences. For example, pyarrow allows us to compress each column with a different compression algo. Spark only allows us to specify one compression algo for the entire parquet file when writing a parquet file. Spark can read the mixed compression parquet file without problems as long as the compression algo is supported.

In this notebook [Spark Arrow compression benchmark](https://github.com/pengfei99/ParquetPyArrow/blob/main/notebook/compatibility/SparkArrowCompression.ipynb), we first benchmark the compression latency of pyarrow and spark.
Then we test the compatibility of the output parquet file between pyarrow and spark. 

In this notebook [R Arrow compression benchmark](https://github.com/pengfei99/ParquetPyArrow/blob/main/R/ArrowCompression.Rmd), we first benchmark the compression latency of Rarrow. Then we test the compatibility of the
output parquet file between Rarrow, Pyarrow, and spark.


### 2.2 Timestamp implementation variation

#### 2.2.1 Unity variation
Each framework has their own implementation of timestamp, and they may not be compatible. For example, some framework only support timestamps stored in millisecond ('ms') or microsecond ('us') resolution. 

Since pandas uses nanoseconds to represent timestamps, this can occasionally be a nuisance. 
If we use pyarrow to write pandas timestamp in parquet format version 1.0, the nanoseconds must be cast to microseconds (‘us’) manually by using the option **coerce_timestamps**. Otherwise, an error will be raised. After adding the option, it still raise an warning, because we lose time precision.
To suppress this warning, we need to add **allow_truncated_timestamps=True**. Below is a full example

``` python
pq.write_table(table, where, coerce_timestamps='ms', allow_truncated_timestamps=True)
```
If we write in parquet format version 2.0, the nanoseconds are supported, no need to do the conversion.

This notebook [Spark Pyarrow timestamp](https://github.com/pengfei99/ParquetPyArrow/blob/main/notebook/compatibility/ArrowSparkTimeStamp.ipynb)
test the compatibility of timestamp unity between Pyarrow(pandas), Rarrow and spark.

This notebook [Rarrow timestamp](https://github.com/pengfei99/ParquetPyArrow/blob/main/R/ArrowTimeStamp.Rmd) use R to 
read parquet file with timestamp which is generated by spark and Pyarrow.

#### 2.2.2 Timestamp column type

Older Parquet implementations use **INT96** as column type to store the timestamps, but this is now deprecated. Now, the **long** column type is recommended. The **INT96** implementations include some older versions of Apache Impala and Apache Spark. To write timestamps in this format, set the use_deprecated_int96_timestamps option to True in write_table.

``` python
pq.write_table(table, where, use_deprecated_int96_timestamps=True)

```


### 2.3 Unsupported data type

You can get the supported type list from [parquet official website](https://parquet.apache.org/documentation/latest/)
The types are:

- BOOLEAN: 1 bit boolean
- INT32: 32 bit signed ints
- INT64: 64 bit signed ints
- INT96: 96 bit signed ints
- FLOAT: IEEE 32-bit floating point values
- DOUBLE: IEEE 64-bit floating point values
- BYTE_ARRAY: arbitrarily long byte arrays.

#### 2.3.1 Spark does not support unsigned int
Spark(version 2.*) does not support unsigned int as parquet column type. But pyarrow does. So if you write parquet with
unsigned int with pyarrow, spark can't read it. You will get 
```text
org.apache.spark.sql.AnalysisException: Parquet type not supported: INT32 (UINT_16);
```
For more detail, please check 
- [Spark can't read a parquet file created with pyarrow](https://github.com/apache/arrow/issues/1470)
- [parquet-compatibility-with-dask-pandas-and-pyspark](https://stackoverflow.com/questions/59948321/parquet-compatibility-with-dask-pandas-and-pyspark)

This issue has been fixed in Spark 3

### 2.4 Metadata specification variation

#### 2.4.1 Version indicateur
PyArrow uses footer metadata to determine the format version of parquet file, while parquet-mr lib 
(which is used by spark) determines version on the page level by page header type. Moreover, in ParquetFileWriter 
parquet-mr hardcoded version in footer to '1'. 

This should be corrected in spark3. 

For more details, please check [Parquet files v2.0 created by spark can't be read by pyarrow](https://issues.apache.org/jira/browse/ARROW-6057)

**Pyarrow allows us to write parquet with version ({"1.0", "2.4", "2.6"}, default "1.0")** 
**Spark (<3.2) does not allow us to write parquet with a version argument**
### 2.5 Recommendation for parquet file compatibility

#### 2.5.1 Compression 
In most frameworks, the default compression algorithm of the parquet reader and writer is **SNAPPY**. The second most 
supported compression algorithm is **GZIP**. In [Spark/Arrow parquet compression benchmark](https://github.com/pengfei99/ParquetPyArrow/blob/main/notebook/compatibility/SparkArrowCompression.ipynb), 
we have noticed: 
- **SNAPPY provides better compression time, but worse compression ratio**
  
- **GZIP provides better compression ratio (30%~50% better than SNAPPY), but worse compression time(200%~300% longer than snappy)**

- **ZSTD provides better compression ratio and time, but not well-supported**

Our recommendation is requirement specific :

1. **For hot data, use SNAPPY as default compression. For cold data, use GZIP**. They are both well-supported. All parquet readers and writers are compatible with
   these two algorithms.

1. If your organization has enough budge for humain resource, use ZSTD. The data engineer will update all 
   your frameworks that need to read or write parquet file to support ZSTD.
   
##### 2.5.1.1 Spark write parquet file to s3 with custom compression algo

```python
# df is the target dataframe
# path is the output s3 path
# partition_number defines how many partitions that your parquet file will have.
# compression_algo default value is SNAPPY 
def check_spark_parquet_write_time(df,path,partition_number,compression_algo="SNAPPY"):
    df.coalesce(partition_number).write \
    .option("parquet.compression",compression_algo) \
    .parquet(path) 

```
##### 2.5.1.2 PyArrow write parquet file to s3 with custom compression algo

The full code can be found in [Spark/Arrow parquet compression benchmark](https://github.com/pengfei99/ParquetPyArrow/blob/main/notebook/compatibility/SparkArrowCompression.ipynb)
```python
# This function write an arrow table to s3 as parquet files, you can specify a compression type
# compression (str or dict) – Specify the compression codec, either on a general basis or per-column. 
# Valid values: {‘NONE’, ‘SNAPPY’, ‘GZIP’, ‘BROTLI’, ‘LZ4’, ‘ZSTD’}.
# default is snappy.

def write_parquet_as_partitioned_dataset(table, endpoint, bucket_name, path, partition_cols=None, compression="SNAPPY"):
    url = f"https://{endpoint}"
    fs = s3fs.S3FileSystem(client_kwargs={'endpoint_url': url})
    file_uri = f"{bucket_name}/{path}"
    pq.write_to_dataset(table, root_path=file_uri, partition_cols=partition_cols, filesystem=fs, compression=compression)
```

##### 2.5.1.2 RArrow write parquet file to s3 with custom compression algo   

```R
# If you do not have the following env var set. You need to replace all the Sys.getenv by real values.
minio <- S3FileSystem$create(
   access_key = Sys.getenv("AWS_ACCESS_KEY_ID"),
   secret_key = Sys.getenv("AWS_SECRET_ACCESS_KEY"),
   session_token = Sys.getenv("AWS_SESSION_TOKEN"),
   scheme = "https",
   endpoint_override = Sys.getenv("AWS_S3_ENDPOINT")
   )
  
# df is the target dataframe
# output_path is the s3 path
# default compression is snappy.
write_parquet(df, sink = minio$path(output_path), compression = "snappy")
```

#### 2.5.2 Timestamp

The best solution is to use **timestamp with timezone specification in String type**. The string type can avoid auto conversion
of each framework. In [Spark Pyarrow timestamp conversion notebook](https://github.com/pengfei99/ParquetPyArrow/blob/main/notebook/compatibility/ArrowSparkTimeStamp.ipynb)
and [Rarrow timestamp conversion notebook](https://github.com/pengfei99/ParquetPyArrow/blob/main/R/ArrowTimeStamp.Rmd), we have seen
how wrong it could go with numeric timestamp type auto conversion.

As a result, we recommend the **ISO 8601 Date and Time Format**. ISO 8601 represents date and time by starting with the 
**year, followed by the month, the day, the hour, the minutes, seconds and milliseconds**.

YYYY-MM-DDThh:mm:ss.sTZD (eg 1997-07-16T19:20:30.45+01:00)
where:
- YYYY = four-digit year 
  
- MM = two-digit month (01=January, etc.)
  
- DD = two-digit day of month (01 through 31)
  
- hh = two digits of hour (00 through 23) (am/pm NOT allowed)
  
- mm = two digits of minute (00 through 59)
  
- ss = two digits of second (00 through 59)
  
- s = one or more digits representing a decimal fraction of a second 
  
- TZD  = time zone designator. Possible value is 
  - Z: no offset to UTC
  - +hh:mm: plus hh:mm to UTC
  - -hh:mm: minus hh:mm to UTC

We found that some country may use two different timezone for different period of time in a year. For example, France use
CET(UTC+01:00) during winter, and CEST(UTC+02:00) during summer. For people who are not familiar with timezone, this can be confusing.
With the UTC offset, it's clearer to express time shift of different timezones.

One **drawback** of the ISO 8601 Date and Time Format, for the time specification, it can only express millisecond precision.

##### 2.5.2.1 Convert string to timestamp in spark
In spark, we need to override the system default timezone by UTC, if it's different from UTC. Otherwise, the conversion will
be based on a wrong timezone.

```python
spark.conf.set("spark.sql.session.timeZone", "UTC")

to_timestamp("2046-01-01 00:15:00+01:00","yyy-MM-dd HH:mm:ssXXX")
```
##### 2.5.2.2 Convert string to timestamp in Python/pandas
Pandas allows you to specify the timezone during the conversion. 
```python
# Pandas provides the to_datetime() function which can convert string with time zone to a timezone aware timestamp.
t1=pd.to_datetime('2046-01-01 00:15:00+01:00', utc=True)
t2=pd.to_datetime('2046-01-01 00:15:00-01:00', utc=True)
print(f"t1 type is : {type(t1)}, t1 value is: {t1}")
print(f"t2 type is : {type(t2)}, t2 value is: {t2}")
```
##### 2.5.2.3 Convert string to timestamp in R

The complete R [doc](https://www.rdocumentation.org/packages/base/versions/3.6.2/topics/as.POSIX*) of POSIXct.

You can notice that the digits behinds the seconds are ignored, even though the doc says with %OS, POSIXct can keep
up to 6 digits after second. 
```R
str_timestamp = "2009-01-27 20:02:22.666+02:00"
timestamp <- as.POSIXct(gsub(":","",str_timestamp),  format = "%Y-%m-%d %H%M%OS%z")
typeof(timestamp)
cat(timestamp)

# It returns
# [1] "double"
# 1233079342
```

##### 2.5.2.4 Other data type for timestamp.
For one reason or another, if you have to use long or other numeric timestamp type. You need to read very carefully 
how the framework implement the timestamp to avoid error led by unity variation.


#### 2.5.3 Data types

Do not use unsupported data type in your data frame that you want to generate to parquet file. Check section 2.3 to get
all supported data types. For example, all int in your dataframe should be signed. **Never use unsigned int**. 

#### 2.5.4 Data format version and metadata

Even though spark, Pyarrow and Rarrow can read parquet format version 2.0. We still recommend that you use version 1.0 when you
write parquet file. For more details about the parquet format version, please visit this [page](https://github.com/apache/parquet-format/blob/master/CHANGES.md)

### 2.6 SAS and parquet
In SAS, you need to have one of the following interfaces to read parquet files:
- SAS/Access to Hadoop 
  
- SAS/Access to ODBC 
  
- SAS/Access to Impala

## 3. Parquet Optimization

### 3.1 Partitions

#### The maximum partition number limit

##### R
The R arrow package has a maximum partition limit (i.e. 1024). For the current version, you can not modify this value. As a result, you can not write parquet with more than 1024 partitions by using R arrow. For more details, please check https://issues.apache.org/jira/browse/ARROW-12321.
```r
# note the max_partitions=12808 is not implemented yet

d %>%
  group_by(Year, `Reporter ISO`) %>%
  write_dataset("parquet", hive_style = F, max_partitions = 12808)
```

For R arrow partition code example, please check section 3.6 of [RArrow_basics](https://github.com/pengfei99/ParquetPyArrow/blob/main/R/ArrowS3.Rmd).

##### Python
PyArrow also has a maximum partition limit (i.e. 1024). But since PyArrow 4.1, you can change the maximum partition limit. A code example can be found(not tested yet)
https://stackoverflow.com/questions/68708477/repartition-large-parquet-dataset-by-ranges-of-values
##### Spark
Spark does not have maximum partition limit. For code example, please check section 2.2 of [SparkParquet_basics](https://github.com/pengfei99/ParquetPyArrow/blob/main/notebook/basic/SparkParquetBasics.ipynb) 
