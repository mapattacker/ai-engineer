# Spark

## Launching Spark

 * The **SparkConf** object sets the configuration for the Spark Application.
 * The **SparkContext** is the entry point of Spark functionality. It allows your Spark Application to access Spark Cluster with the help of Resource Manager.
 * The **SparkSession** is the entry point into the Structured API.

```python
from pyspark import SparkContext, SparkConf
from pyspark.sql import *
from pyspark.sql.functions import *
from pyspark.sql.types import *

# set master can increase the cores
cnfg = SparkConf().setAppName("CustomerApplication").setMaster("local[2]")
sc = SparkContext(conf=cnfg)
spark = SparkSession(sc)
```


## Read Files

```python
# from database
cnfg = SparkConf().setAppName("CustomerApplication").setMaster("local[2]")
cnfg.set("spark.jars","D:\\mysql-connector-java-5.1.49.jar")
sc = SparkContext(conf=cnfg)
spark = SparkSession(sc)
df = (spark.read.format("jdbc")
        .options(url="jdbc:mysql://localhost/videoshop",
        driver="com.mysql.jdbc.Driver",
        dbtable="SomeTableName",
        user="venkat", password="P@ssw0rd1").load())

# read csv
df = (spark.read.option("header", "true").option("inferSchema", "true").csv(filepath))
# read json
df = (spark.read.option("header", "true").option("inferSchema", "true").json(filepath))

```

We can also define a schema for a particular data.

```python
custschema = StructType([
                StructField("Customerid", IntegerType(), True),
                StructField("CustName", StringType(), True),
                StructField("MemCat", StringType(), True),
                StructField("Age", IntegerType(), True),
                StructField("Gender", StringType(), True),
                StructField("AmtSpent", DoubleType(), True),
                StructField("Address", StringType(), True),
                StructField("City", StringType(), True),
                StructField("CountryID", StringType(), True),
                StructField("Title", StringType(), True),
                StructField("PhoneNo", StringType(), True)
            ])

df.printSchema()
df = (spark.read
        .schema(schema=custschema)
        .csv(inputFilePath))
df.show()
```

## Write Files

Note that we write files to a folder. The file name is system generated, e.g. "part-00000-2ee2d8b6-169a-4f48-a503-8b6a2fdedab0-c000.json".

```python
df.write.json("D:\\CountryOUT")
```

## Basic Summary

```python
# bool for col truncation
df.show(20, False) 
# schema
df.printSchema()
# statistics
df.describe("MemberCategory", "Gender", "AmountSpent", "Age").show()
```

## Spark SQL

The Spark SQL API works in structure similar to SQL. This is used when the data format is well structured, and presentable in a tabular format. Spark SQL is a high level API compared to RDD, therefore is easy to use. 

```python
# simple query
df2 = df.orderBy("Age").where("Age>20").select(df["CustomerID"], df["CustomerName"], df["Age"])
df2.show(200, False)

# aggregate (single value)
tot = df.agg(sum("AmountSpent")).first()[0]
tot = df.agg(avg("Age")).first()[0]
std = df.agg(stddev_pop("AmountSpent")).first()[0]
skw = df.agg(skewness("AmountSpent")).first()[0]

# group by
df.groupBy("MemberCategory").sum("AmountSpent").show(200,False)
```

We can also do joins from multiple dataframes.

```python
customerFilePath = r"D:\workspace\Customer.csv"
countryFilePath = r"D:\workspace\Country.csv"

dfCustomer = ( spark.read
                .option("header", "true")
                .option("inferSchema", "true")
                .csv(customerFilePath) )

dfCountry = ( spark.read
                .option("header", "true")
                .option("inferSchema", "true")
                .csv(countryFilePath) )

joinDF = dfCustomer.join(dfCountry, "CountryCode")

( joinDF.select("CustomerID", "CustomerName", "CountryCode",
"CountryName", "Currency", "TimeZone")
.show(300, False) ) )
```

## RDD

Resilient Distributed Dataset