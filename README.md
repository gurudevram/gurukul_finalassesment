# gurukul_finalassesment
Gurukul_training_final_assesment


!apt-get install openjdk-8-jdk-headless -qq > /dev/null

!java -version

!wget -q https://dlcdn.apache.org/spark/spark-3.1.3/spark-3.1.3-bin-hadoop3.2.tgz

!tar xf spark-3.1.3-bin-hadoop3.2.tgz

!pip install -q findspark

import os
os.environ["JAVA_HOME"] = "/usr/lib/jvm/java-8-openjdk-amd64"
os.environ["SPARK_HOME"] = "/content/spark-3.1.3-bin-hadoop3.2"

import findspark
findspark.init()


from pyspark.sql import SparkSession
spark = SparkSession.builder\
        .master("local[2]")\
        .appName("demo")\
        .config("spark.jars", "/content/drive/MyDrive/snowflake-jdbc-3.12.17.jar,/content/drive/MyDrive/spark-snowflake_2.12-2.11.0-spark_3.1.jar,/content/drive/MyDrive/scala-compiler-2.12.17.jar")\
        .getOrCreate()
        
        SparkSession.builder.enableHiveSupport().config('spark.jars.packages', 'net.snowflake:snowflake-jdbc:3.13.23,net.snowflake:spark-snowflake_2.12:2.11.0-spark_3.3').getOrCreate()
        
        
        sfOptions = {
    "sfURL" : "https://ou29747.ap-south-1.aws.snowflakecomputing.com/",
    "sfAccount" : "ou29747",
    "sfUser" : "gurudevr",
    "sfPassword" : "Welcome@1",
    "sfDatabase" : "saved_raw_data",
    "sfSchema" : "RAW_SCHEMA",
    "sfWarehouse" : "COMPUTE_WH",
    "sfRole" : "ACCOUNTADMIN",
}

SNOWFLAKE_SOURCE_NAME = "net.snowflake.spark.snowflake"
df=spark.read.format("csv").options(**sfOptions).option("header","True").option("inFerSchema","True").load("/content/drive/MyDrive/meets.csv")
df.show()


df.write.format(SNOWFLAKE_SOURCE_NAME).options(**sfOptions).\
option("dbtable" , "matches").mode("overwrite").save()

import os
os.environ["PYSPARK_SUBMIT_ARGS"] = "--master local[2] pyspark-shell"

spark



import pyspark.sql.functions as F

raw_logs = spark.read.option("header","false").option("delimiter", " ").csv("/content/drive/MyDrive/log_data_ip_request.txt")
raw_logs = raw_logs.persist()
raw_logs.show(10, False)


log_details = (raw_logs.select(
    F.monotonically_increasing_id().alias('row_id'),
    F.col("_c0").alias("ip"),
    F.split(F.col("_c3")," ").getItem(0).alias("datetime"),
    F.split(F.col("_c5"), " ").getItem(0).alias("method"),
    F.split(F.col("_c5"), " ").getItem(1).alias("request"),
    F.col("_c6").alias("status_code"),
    F.col("_c7").alias("size"),
    F.col("_c8").alias("referrer"),
    F.split(F.col("_c9")," ").getItem(1).alias("user_agent")
    ))
    
    
    log_details.show()
    
    
    from pyspark.sql.functions import *
a=log_details.withColumn('datetime', regexp_replace('datetime', '\[|\]|', ''))



a.show()


a.write.format("csv").mode("overwrite").saveAsTable("save_raw_data")


log_details.count()


spark.table("save_raw_data").show(10, False)


a.show(truncate=False)


logs = a.withColumn('datetime',to_timestamp('datetime','dd/MMM/yyyy:HH:mm:ss'))


logs.show()


referrer_drop = logs.drop("referrer")


referrer_drop.show()


clensed_data = a.withColumn('referrer', when(col('referrer') == '-', "No").otherwise("Yes"))
clensed_data.show()



clensed_data.write.format("csv").mode("overwrite").saveAsTable("save_clensed_data")
clensed_data.show()


curated_data = a.withColumn('referrer', when(col('referrer') == '-', "No").otherwise("Yes"))
curated_data.show()


curated_data.write.format("csv").mode("overwrite").saveAsTable("save_curated_data")


spark.table("save_raw_data").show(5, False)
spark.table("save_clensed_data").show(5, False)
spark.table("save_curated_data").show(5, False)


data_head= curated_data.select("method").distinct().show()




logs.show(10, False)


logs_per_device = (logs
 .withColumn("day_hour", F.date_format(F.col("datetime"), "yyyy-MM-dd_hh"))
 .groupBy(F.col("ip"), F.col("day_hour"))
 .agg(F.collect_list(F.col("method")).alias("coll"))
 .withColumn("no_get", F.size(F.filter(F.col("coll"), lambda x: x == "GET")))
 .withColumn("no_post", F.size(F.filter(F.col("coll"), lambda x: x == "POST")))
 .withColumn("no_head", F.size(F.filter(F.col("coll"), lambda x: x == "HEAD")))
 .select(F.col("ip"), F.col("day_hour"), F.col("no_get"), F.col("no_post"), F.col("no_head"))
 )
 
 
 logs_per_device.show()
 
 
 logs_across_device = (logs
 .withColumn("day_hour", F.date_format(F.col("datetime"), "yyyy-MM-dd_hh"))
 .groupBy(F.col("day_hour"))
 .agg(F.collect_set(F.col("ip")).alias("clients"), F.collect_list(F.col("method")).alias("coll"))
 .withColumn("no_of_clients", F.size(F.col("clients")))
 .withColumn("no_get", F.size(F.filter(F.col("coll"), lambda x: x == "GET")))
 .withColumn("no_post", F.size(F.filter(F.col("coll"), lambda x: x == "POST")))
 .withColumn("no_head", F.size(F.filter(F.col("coll"), lambda x: x == "HEAD")))
 .select(F.col("day_hour"), F.col("no_of_clients") , F.col("no_get"), F.col("no_post"), F.col("no_head"))
 )
 
 
 
 logs_across_device.show()
 
 
 logs_across_device.write.format("net.snowflake.spark.snowflake").options(**sfOptions) \
.option("dbtable", "logs_across_device") \
.mode("append") \
.option(header= True) \
.save()


logs_across_device.write.format("csv").mode("overwrite").saveAsTable("logs_across_device_data")


logs_per_device.write.format("csv").mode("overwrite").saveAsTable("logs_per_device_data")


