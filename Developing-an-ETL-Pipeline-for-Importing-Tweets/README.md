### ETL Pipeline


Getting tweets via Twitter Developer Account API

Python code for getting tweets is uploaded separately.

Creating a batch file for scheduling import of Tweets

cd /d D:\PROJECT
C:\ProgramData\Anaconda3\python.exe D:\PROJECT\p.py
pscp -pw "password" D:\PROJECT\*.json root@***.***.*.***:/root/tweet_received && move D:\PROJECT\*.json D:\PROJECT\tweet_processed
pause

Note: ***.***.*.*** denotes the computer’s IP address. 


Creating Hive Table 
SET hive.support.sql11.reserved.keywords=false; CREATE EXTERNAL TABLE IF NOT EXISTS (created_at string, id string, id_str string, text string, source string, truncated string, user struct< id:string, id_str:string, name:string, screen_name:string, location:string, url:string,description:string, translator_type:string, protected:string, verified:string, followers_count:string, friends_count:string, listed_count:string, favourites_count:string, created_at:string, utc_offset:string,time_zone:string>)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe' STORED AS TEXTFILE;
Load Data into Hive Table
SET hive.support.sql11.reserved.keywords=false; LOAD DATA INPATH '/<path-to-tweets-folder>/’ OVERWRITE INTO TABLE <your-table-name>;
Load Hive Table to Pig and Process it
REGISTER hdfs:////.jar; a = LOAD '' USING org.apache.hive.hcatalog.pig.HCatLoader(); f = FOREACH a GENERATE ToDate(created_at, 'EEE MMM dd HH:mm:ss Z yyyy') as (date_time:DateTime ),id as iden,text as t; y = FOREACH f GENERATE GetYear(date_time)as (year:chararray), GetMonth(date_time)as(month:chararray), GetDay(date_time)as(day:chararray), GetHour(date_time)as(hour:chararray), GetMinute(date_time)as(minute:chararray), iden as id, t as text; STORE y INTO '' USING org.apache.hive.hcatalog.pig.HCatStorer();

Remove new line characters in hive 
SET hive.support.sql11.reserved.keywords=false; 
create view as SELECT hour,id,regexp_replace(text,'\n','') as text FROM <processed-table-name>;

Use Hive Explode Function to Separate Words into Multiple Rows

SET hive.support.sql11.reserved.keywords=false;
 CREATE VIEW <view-name> AS SELECT hour,id,t FROM <view-name> lateral view explode(split(lower(text),'\\W+')) text as t;

Get the Data into Spark to Get Insights

hdfs dfs -copyToLocal /usr/json-serde-1.1.9.9.jar ~
Give execution permission to serde
chmod 777 json-serde-1.1.9.9.jar
spark-shell --jars json-serde-1.1.9.9.jar
val q = sqlContext.sql("select hour, count(*) as words_per_hour from final_out group by hour")
q.saveAsTable("end_result")
 


Connecting to Tableau via the ODBC Server



Automation of the Entire Process

hdfs dfs -put ~/tweet_received/* /usr/tweet_received 
hive -f hive1.sql ;create hive table, load data into it,create proctweets for pig output
pig -useHCatalog -f pig.pig
hive -f hive2.sql ;remove newline characters in hive, explode function separates each function into a different row
spark-shell --jars hive-hcatalog-core.jar,json-serde-1.1.9.9.jar -i sp.scala 
