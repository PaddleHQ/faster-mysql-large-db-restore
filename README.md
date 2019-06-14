Restore large MySql database in one hour (which might take more than 10 hours otherwise)
=============================

How it works?

1. Generate a query to create table schema, procedures and functions of the source DB
2. Generate a query to find all indexes of all tables except foreign key constraints of the source DB
3. Generate a query to drop all the indexes found in step 2
4. Generate a query to insert all data of the source DB
5. Generate a query to create all the indexes found in step 2
6. Write all queries in the above order in one .sql file which is your new MySql backup. Zip it using LZ4 compression. 
7. Restore the DB from this file using normal restore command of MySql.

- Initially, on an Amazon Large instance, it used to take 2 hours to backup 7GB unzipped (700MB zipped) MySql DB using _mysqldump_ utility and 8 hours to restore - Total **10 hours**.

- With this technique it takes 10 mins to backup the same DB and 50 mins to restore - **savings of 9 hours!**


## REQUIREMENTS

Below are the dependencies  needed to run [`dbBackup.js`](/dbBackup.js).

* System OS: Linux
* System Packages:
   * **MySQL client**: mysql-client
   * [**Nodejs**](http://nodejs.org/): nodejs
   * **NPM**: npm
   * **lz4c**: liblz4-tool 

`NOTE` - We can install all by running:
```shell
apt-get update && apt-get install -y mysql-client nodejs npm liblz4-tool
```
* Node Modules: 
   * **MySQL**: mysql

`NOTE` - We can install the package by running:
```shell
npm install mysql
```  



## `MYSQL BACKUP USER PERMISSION`
For the backup script to run successfully the user carrying out the back up must have the following permissions (InnoDB) on the database you wish to back up 

* `SELECT`
* `SHOW VIEW` 
* `RELOAD`
* `REPLICATION CLIENT`
* `EVENT`
* `TRIGGER`

Running the command below in the `MySQL shell` connected to your desired <MYSQL_HOST> will create a user with the necessary rights.

```shell
CREATE USER '<MYSQL_USER>'@'localhost' IDENTIFIED BY '<MYSQL_PASSWORD>';
GRANT SELECT, SHOW VIEW, RELOAD, REPLICATION CLIENT, EVENT, TRIGGER ON *.* TO '<MYSQL_USER>'@'localhost';
```


## `HOW TO RUN`

In order to run the `dbBackup.js` script you must must ensure you have satisfied all the [requirements](#REQUIREMENTS)
```shell
node dbBackup.js <TABLES_TO_IGNORE> <TAKE_PROCEDURES> <MYSQL_HOST> <MYSQL_PORT> <MYSQL_USER> <MYSQL_PASSWORD> <MYSQL_DATBASE_NAME> '<LOCATION_TO_SAVE_FILE>' <NAME_OF_BACK_UP_FILE>
```

`<TABLES_TO_IGNORE>`    - A comma-separated list of table names you wish to ignore in backup if you don't want to ignore **any** tables, use two single quotes (`''`).

`<TAKE_PROCEDURES>`     - Either `1` OR `0`. Selecting `1` tells the script it should back up the [triggers](https://dev.mysql.com/doc/refman/8.0/en/trigger-syntax.html) and [routines (functions and procedure)](https://dev.mysql.com/doc/refman/5.7/en/stored-routines.html). Otherwise, these will be ignored.

`<MYSQL_HOST>`          -  The URL of the MySQL server 

`<MYSQL_PORT>`          -  The port of the MySQL server 

`<MYSQL_USER>`          -   The user to use for the backup process 

`<MYSQL_PASSWORD>`      -   The password for the user

`<LOCATION_TO_SAVE_FILE>`    - If left blank `''` it will be written to the root dir, i.e. `/`. When passing a path, do not give the trailing forward slash

`<NAME_OF_BACK_UP_FILE>`   - This name of the `lz4c` output file 


An example of the command with options filled credentials:
```shell
node dbBackup.js '' 1 127.0.0.1 3306 root secretrootpassword employees '/home/paddle/backup' db_backup
```

## `NICE TO KNOWS`
While running through the backup/ restore process we encountered several "gotcha" moments, they included:
* `MySQL: Access denied` - despite adding the DEFINER replacement command we were still facing Access denied errors when trying to restore our back up after a bit of digging we found that the error was caused because our SQL included information regarding `GTID`. [This](https://help.poralix.com/articles/mysql-access-denied-you-need-the-super-privilege-for-this-operation) article was very helpful in helping us debug the issue. If you do not want to redo your back up, you can run up the following command: 
```shell
lz4 -cd <NAME_OF_BACK_UP_FILE>.lz4| sed -E 's/SET @MYSQLDUMP_TEMP.*//g' | sed -E 's/SET @@SESSION.SQL_LOG_BIN.*//g' | sed -E 's/SET @@GLOBAL.GTID_PURGED=.*//g' | mysql -h <MYSQL_HOST> -P 3306 -u <MYSQL_USER> -p<MYSQL_PASSWORD> <DESIRED_MYSQL_DB_NAME>
```