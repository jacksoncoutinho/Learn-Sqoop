# Learn ~ Sqoop

## Apache Sqoop v1.4.6

- Sqoop is a tool designed to **transfer data between Hadoop and relational databases or mainframes**. You can use Sqoop to import data from a relational database management system (RDBMS) such as MySQL or Oracle or a mainframe into the Hadoop Distributed File System (HDFS), transform the data in Hadoop MapReduce, and then export the data back into an RDBMS.
- It supports incremental loads of a single table or a free form SQL query as well as saved jobs which can be run multiple times to import updates made to a database since the last import. Imports can also be used to populate tables in Hive or HBase. Exports can be used to put data from Hadoop into a relational database. Sqoop got the name from sql+hadoop. Sqoop became a top-level Apache project in March 2012.
- Sqoop uses MapReduce to import and export the data, which provides parallel operation as well as fault tolerance.

## Import Tool ##
- Imports an individual table from RDBMS to HDFS. Each row from a table is represented as a separate record in HDFS. Record can be stored as text file (one record per line), or in a binary representation as Avro or Sequence file
	
## Apache Sqoop - Imports ##

To connect to mysql
  > mysql --host=system.name --user=xxxx --password=xxxx
	> use training;

```	
CREATE TABLE stocks(
  id INT NOT NULL PRIMARY KEY AUTO_INCREMENT,
  symbol VARCHAR(64) NOT NULL,
  name VARCHAR(64) NOT NULL,
  trade_date DATE,
  close_price DECIMAL(10,2),
  volume INT,
  updated_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP);
```	

```
INSERT INTO stocks (symbol, name, trade_date, close_price, volume) VALUES ('AAL', 'American Airlines', '2015-11-11', 44.39, 4416900);
INSERT INTO stocks (symbol, name, trade_date, close_price, volume) VALUES ('AAPL', 'Apple', '2015-11-11', 116.1, 45217900);
INSERT INTO stocks (symbol, name, trade_date, close_price, volume) VALUES ('AMGN', 'Amgen', '2015-11-11', 156.0, 1923900);
INSERT INTO stocks (symbol, name, trade_date, close_price, volume) VALUES ('GARS', 'Garrison', '2015-11-11', 13.57, 24800);
INSERT INTO stocks (symbol, name, trade_date, close_price, volume) VALUES ('SBUX', 'Starbucks', '2015-11-11', 61.87, 4437300);
INSERT INTO stocks (symbol, name, trade_date, close_price, volume) VALUES ('SGI', 'Silicon Graphics', '2015-11-11', 5.320, 134300);
INSERT INTO stocks (symbol, name, trade_date, close_price, volume) VALUES ('TSLA', 'Tesla', '2015-11-11', 219.1, 3347800);
INSERT INTO stocks (symbol, name, trade_date, close_price, volume) VALUES ('TXN', 'Texas Instruments', '2015-11-11', 57.60, 6293100);
INSERT INTO stocks (symbol, name, trade_date, close_price, volume) VALUES ('MAT', 'Mattel', '2015-11-11', 24.04, 2920100);
INSERT INTO stocks (symbol, name, trade_date, close_price, volume) VALUES ('INTC', 'Intel', '2015-11-11', 32.86, 19864700);
```
### Default number of MAPPERS : **4** ###

> sqoop import --connect jdbc:mysql://ip-172-31-45-216.ec2.internal/training --table stocks

### Default sqoop checks for primary key and then divides the partitions its takes MIN() and MAX() and then divides ###

```
hadoop fs -ls stocks
hadoop fs -cat stocks/part-m-00001
	
sqoop import --connect jdbc:mysql://ip-172-31-45-216.ec2.internal/training --table stocks -m 2 --target-dir /user/hirw/sqoop/stocks_nmaps

hadoop fs -ls sqoop/stocks_nmaps
hadoop fs -cat sqoop/stocks_nmaps/part-m-00001
	
sqoop import --connect jdbc:mysql://ip-172-31-45-216.ec2.internal/training --table stocks -m 1 --target-dir /user/hirw/sqoop/stocks_terminated --fields-terminated-by '\t' --enclosed-by '"'
	
hadoop fs -ls sqoop/stocks_terminated
hadoop fs -cat sqoop/stocks_terminated/part-m-00000
	
sqoop import --connect jdbc:mysql://ip-172-31-45-216.ec2.internal/training --table stocks --columns "symbol,name,trade_date,volume" --where "id > 5" -m 1 --target-dir /user/hirw/sqoop/stocks_selective
	
hadoop fs -ls sqoop/stocks_selective
hadoop fs -cat sqoop/stocks_selective/part-m-00000
```	
### IMPORT WITH $CONDITIONS ###
	
Data boundary using a different column
```
sqoop import --connect jdbc:mysql://ip-172-31-45-216.ec2.internal/training --table stocks --split-by volume --target-dir /user/hirw/sqoop/stocks_conds
	
hadoop fs -ls sqoop/stocks_conds
```
### Custom query import with **$CONDITIONS** ###
```
sqoop import --connect jdbc:mysql://ip-172-31-45-216.ec2.internal/training --query 'SELECT a.id, a.name, a.trade_date, a.volume, b.dividend_amount from stocks a INNER JOIN dividends b ON a.symbol = b.symbol WHERE a.id > 2 and $CONDITIONS'  --split-by a.volume --target-dir /user/hirw/sqoop/stocks_join_conds
	
hadoop fs -ls sqoop/stocks_join_conds
hadoop fs -cat sqoop/stocks_join_conds/part-m-00000
hadoop fs -cat sqoop/stocks_join_conds/part-m-00001
hadoop fs -cat sqoop/stocks_join_conds/part-m-00002
```
### COMPRESSION ###
```	
sqoop import --connect jdbc:mysql://ip-172-31-45-216.ec2.internal/training --table stocks --compress -m 2 --target-dir /user/hirw/sqoop/stocks_comp
	
hadoop fs -ls sqoop/stocks_comp
hadoop fs -cat sqoop/stocks_comp/part-m-00000
```
	
### SEQUENCE FILE ###
```	
sqoop import --connect jdbc:mysql://ip-172-31-45-216.ec2.internal/training --table stocks --as-sequencefile -m 2 --target-dir /user/hirw/sqoop/stocks_seq
	
hadoop fs -ls sqoop/stocks_seq
hadoop fs -cat sqoop/stocks_seq/part-m-00000
```	
## AVRO ##
```	
sqoop import --connect jdbc:mysql://ip-172-31-45-216.ec2.internal/training --table stocks --as-avrodatafile -m 2 --target-dir /user/hirw/sqoop/stocks_avro

hadoop fs -ls sqoop/stocks_avro
hadoop fs -cat sqoop/stocks_avro/part-m-00000.avro
```	

### Apache Sqoop - Jobs ###
	
## Classic Import with Selection ###
```	
sqoop import --connect jdbc:mysql://ip-172-31-45-216.ec2.internal/training --table stocks --columns "symbol,name,trade_date,volume" --where "id > 10" -m 1 --target-dir /user/hirw/sqoop/stocks_selective
```	
## Incremental Import - Append ##
```
sqoop job --create incrementalImportJob -- import --connect jdbc:mysql://ip-172-31-45-216.ec2.internal/training --table stocks --target-dir /user/hirw/sqoop/stocks_append --incremental append --check-column id
```	
### Incremental has 2 modes ###
- Append - when importing a table where new rows are continually being added with increasing row id values. You specify the column containing the row’s id with --check-column. Sqoop imports rows where the check column has a value greater than the one specified with --last-value.
- Lastmodified - when rows of the source table may be updated, and each such update will set the value of a last-modified column to the current timestamp. Rows where the check column holds a timestamp more recent than the timestamp specified with --last-value are imported.
	
> sqoop job --list
	
> sqoop job --show incrementalImportJob
	
>	sqoop job --exec incrementalImportJob
	
hadoop fs -ls sqoop/stocks_append
hadoop fs -cat sqoop/stocks_append/part-m-00000

```	
INSERT INTO stocks (symbol, name, trade_date, close_price, volume) VALUES ('AAL', 'American Airlines', '2015-11-12', 42.4, 4404500);
INSERT INTO stocks (symbol, name, trade_date, close_price, volume) VALUES ('AAPL', 'Apple', '2015-11-12', 115.23, 40217300);
INSERT INTO stocks (symbol, name, trade_date, close_price, volume) VALUES ('AMGN', 'Amgen', '2015-11-12', 157.0, 1964900);
```	

### sqoop job --exec incrementalImportJob ###
	
> hadoop fs -ls sqoop/stocks_append
> hadoop fs -cat sqoop/stocks_append/part-m-00004
		
### Incremental Import - lastmodified ###
```
sqoop job --create incrementalImportModifiedJob -- import --connect jdbc:mysql://ip-172-31-45-216.ec2.internal/training --table stocks --target-dir /user/hirw/sqoop/stocks_modified --incremental lastmodified --check-column updated_time -m 1 --append
sqoop job --list
sqoop job --show incrementalImportModifiedJob
sqoop job --exec incrementalImportModifiedJob

hadoop fs -ls sqoop/stocks_modified
hadoop fs -cat sqoop/stocks_modified/part-m-00000
	
UPDATE stocks SET volume = volume+100, updated_time=now() WHERE id in (10,11,12);
	
INSERT INTO stocks (symbol, name, trade_date, close_price, volume) VALUES ('GARS', 'Garrison', '2015-11-12', 12.4, 23500);
INSERT INTO stocks (symbol, name, trade_date, close_price, volume) VALUES ('SBUX', 'Starbucks', '2015-11-12', 62.90, 4545300);
INSERT INTO stocks (symbol, name, trade_date, close_price, volume) VALUES ('SGI', 'Silicon Graphics', '2015-11-12', 4.12, 123200);
```

sqoop job --show incrementalImportModifiedJob
	
sqoop job --exec incrementalImportModifiedJob
	
hadoop fs -ls sqoop/stocks_modified
hadoop fs -cat sqoop/stocks_modified/part-m-00001
	
### MERGE ###
	
sqoop codegen --connect jdbc:mysql://ip-172-31-45-216.ec2.internal/training --table stocks --outdir /home/hirw/sqoop/sqoop-codegen-stocks
	
--Copy the jar file
	
sqoop merge --new-data /user/hirw/sqoop/stocks_modified/part-m-00001 --onto /user/hirw/sqoop/stocks_modified/part-m-00000 --target-dir /user/hirw/sqoop/stocks_modified/merged --jar-file stocks.jar --class-name stocks --merge-key id
	
hadoop fs -ls sqoop/stocks_modified/merged
	
hadoop fs -cat sqoop/stocks_modified/merged/part-r-00000

### Apache Sqoop - Hive & Export ###

## EXPORT ##

	CREATE TABLE stocks_sqoop(
	id INT NOT NULL PRIMARY KEY AUTO_INCREMENT,
	symbol VARCHAR(64) NOT NULL,
	name VARCHAR(64) NOT NULL,
	trade_date DATE,
	close_price DECIMAL(10,2),
	volume INT,
	updated_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP);
	
	sqoop export --connect jdbc:mysql://ip-172-31-45-216.ec2.internal/training --table stocks_sqoop --export-dir /user/hirw/sqoop/stocks_modified/merged
	
## HIVE ##
	
sqoop import --connect jdbc:mysql://ip-172-31-45-216.ec2.internal/training --table stocks --target-dir /user/hirw/sqoop/hive  -m 1 --hive-import --hive-table hivetemp.stocks_sqoop
	
hive> DESCRIBE FORMATTED stocks_sqoop;
