# VCLODs: Variable Configuration Locking Operation Destination Scripts
## What are VCLODs?
An open-source directory based ksh framework to productionize programs from component scripts

## What are the benefits?
* Build from simple scripts
* Encoding Timing and Configuration in script absolute path, allows complex behavior
* Automatic human readable process reporting
* Connections are easy (see the connections/ directory)
  * InfluxDB, MongoDB, Redis, Postgres would be easy to add
  * MySQL/MariaDB, MSSQL already implemented
* Use your own language (shebang) - Javascript/Node, Ruby, Go
* Scripts/programs can focus on their purpose without dealing with their infrastructure
* Fast dev, Fast execution, Dev mobility, Extensibility - Especially for Data tasks
  * ~500 C program lines convert to 14 VCLODs lines
  * At a glance debugging

## How does it work?
* ~400 ksh lines that provide...
* Modularizing simple scripts/programs to automate ~95% of the boilerplate needed to productionize
  * Configuration
  * Locking 
  * Logging
  * Alerting
  * Timing (cron over directories)
    * Precedence (alpha order)
    * Parallelization
  * Database Connections
  * Piped Operations: Batching, Alerting, Advanced Logging ... 

![discriptive diagram of how VCLODs works](/VCLODs.png)  

# VCLODs Detailed How it Works
## Configuration
There is a global config file to make life easier (`/etc/vclods`), then each directory has its own configs to fine tune what all the scripts in a given directory will do. [Here is a list of all the Configuration Variables](/docs/Configuration.md).

## Locking
Each script file automatically locks out redundant execution (or allows up to `VCLOD_BATCH_JOBS` number of instances to run).

## Operation
Based on the file extension list, different operations can be assigned. If the extension is `.sh` then it is sourced as a ksh script. If the extension is `.sql` then it is passed as sql to the primary sql connection. Extensions are recursively applied, so `.dst.sql` will run sql on the primary database connection, then pipe the output into the secondary database connection, effectively making it a metasql script. [Here is a list of all the .extension Operations](/docs/Operation.md).

## Destination
Log output (anything in stdout at pipe's end) can go to 5 locations: 
Control | Where | Description
--------|-------|------------
Always | log files | in $LOG_BASE_DIR and $VCLOD_ERR_DIR
Always | syslog | This can be pulled in by systems like graylog and datadog
Conditional | email | stderr goes to email ($OPERATIONS_EMAIL) for alerting
Optional | stdout | if you are manually running the script in a terminal
Optional | post process script | As defined in $LOG_POST_PROCESS. The provided post process script (vclod_pp_log2sql) logs to SQL for relational querying

## What is the Strategy?
* Accomidating lazy Database Programmers ;)
* Prioriting work done over copy paste boilerplate

## What are the Objections?
* Unnecessary!! I don't need this brain pain!!
  * Copy and paste is always an option. As well as re-debugging
  * Since you have to run shell anyways, make it do the common, critical tasks so that you dont have to re-implement then every time you add a new language to your tech stack
  
* ksh < python3
  * ok, so use python inside VCLODs. Productionization is free
  * Shell scripting is the universal languge

## Pseudocode Examples: Note `.` is shorthand for `|` so VCLODScript names are self descriptive
* For more examples, look in this repo's test directory. Output is compared to `test/expected`
* script.sh: run a script in directory context (VCLODs handles Timing, Configuration, Locking, Logging, ...)
* script.sql.sh: script.sh except it spits out SQL to avoid connection call
* script.sh.sql: a query generates a shell script (usually curl)
* script.sh.tee-file.sql: script.sh.sql except shell commands go to file for analysis
* script.err.diff-file.\*: run something, compare output with file, and if diff treat as error + send email
* script.dst.sql: run a query on primary connection that generates a query for secondary connection (data migration)
* script.sql.tee-file.batch.sql: run a query, batch the output, stash batched statements into a file for auditing, run generated batch statements

## Example Crontab
Specify when you want which directories to run and then everything in them run

    44 4 1 1 *   /usr/local/bin/vclod /vclod/yearly/
    15 2 1 * *   /usr/local/bin/vclod /vclod/monthly/
    7  1 * * Sun /usr/local/bin/vclod /vclod/weekly/
    7  1 * * Fri /usr/local/bin/vclod /vclod/fridays/
    7  1 * * Wed /usr/local/bin/vclod /vclod/wednesdays/
    3  6 * * *   DEBUG_SHOULD_TIME_IT=1 VCLOD_JOBS=5 /usr/local/bin/vclod /vclod/nightly/
    30  4 * * *   DEBUG_SHOULD_TIME_IT=1 VCLOD_JOBS=2 /usr/local/bin/vclod /vclod/0430_nightly/
    6 0,8-23/2 * * * /usr/local/bin/vclod /vclod/bihourly/
    22 * * * *   /usr/local/bin/vclod /vclod/hourly/
    *  * * * *   /usr/local/bin/vclod /vclod/minutely/

## Example Directory Structure
    /vclod/nightly/
    /vclod/nightly/config
    /vclod/nightly/script1.sh
    /vclod/nightly/server_database/
    /vclod/nightly/server_database/config: configures `VCLOD_HOST`/`VCLOD_DB`
    /vclod/nightly/server_database/script2.sql
    /vclod/nightly/server_database/script3.dst.sql
    ...

# Commands
specialized stream commands for a few of the dot extensions:

## .batch Commands:

Command | default if start exists | description
--------|-------------------------|------------
#batch | 1000 | Number of lines to put in each batch
#start | | Start of statement (like INSERT INTO .... VALUES )
#sep | ',' | how to separate lines
#end | ';' | how to end lines
#del_start | | if desired, the start of a delete statement to be used in archiving data
#del_sep | ',' | delete statement separator
#del_end | ');' | delete statement end
#RESET | | reset to start state so you can start a new batch in the same pipe

## .etl Commands:
Commands applied to a field in the Temp table create statement. These Commands (and the CREATE TEMPORARY TABLE statement they are attached to) must be in and extension option file. Stdin into the .etl extension must be a the VALUES part of the computed INSERT statement (the fields in the order of the CREATE TEMPORARY TABLE statement that exclusively `#ingest`, `#unique`, or `#map` commands attached to them). Fields may have multiple commands attached to them. `.etl` should always be followed by `.batch` unless you just want to test (ie, `do.sql.batch.etl-file.sh`)
Command | Requires preceding field ^1 | no_update Option? ^2| Positional Args ^3 | Description
--------|-----------------------------|---------------------|--------------------|------------
#ingest |Y|N|N| force this field to be ingested in the initial INSERT INTO tmp table
#ignore |Y|N|N| Do not ingest this field into the initial temp table.
#key |Y|N|Std| The auto_incrementing Primary key that will be used to sync deep FK chains. Will not be ingested, but rather derived after syncing with the destination table
#unique |Y|N|Std| Unique fields candidate keys on the table. If there is no UNIQUE index, you can spoof the behavior with #unique_no_update. Does not need to be unique in the temp table (useful for deep FK chains)
#map |Y|Y|Std| A regular field on the given table. 
#generate |N|Y|Std + SQL statement| Generate a virtual field that is not in the temp table. The SQL statements that follow the field name are used instead of a column name in the temp table when doing the ETL into the destination table. Used in place of either a VIRTUAL column on the temp table or an #include script to do the generation
#generate_unique |N|N|Std + SQL statement| A logical combination of #generate and #unique
#include |YN|N|local SQL filename| include a sql script file (with no .extension) to handle any additional reformatting or processing that is required. 
#sync |N|Y|Destination Table + More ^4 | Command to sync the temp table with the destination table. Order between #sync and #include commands indicates execution order
#mode|N|N|Destination Table + odku_ai or ui_split|odku_ai is the default mode: it will force primary keys not to bloat; ui_split tries its best to not bloat Autoinc keys, but doesn't force an ALTER TABLE. Use it with lower cardinality tables.

1. If Y, add the command after the temp table field definition. If N, then put it on its own line as a stand alone command. If YN (ie for `#include`), if the command is on a temp table field line, it acts as both `#include` AND `#ignore`.
1. on_update means the field will not be updated when it changes (ie, will not be in the ON DUPLICATE KEY UPDATE list). It is annotated by updating the command name, for instance `#sync` becomes `#sync_no_update`.
1. N means there are no Positional Arguments after the command. Std means the standard ones are required (destination table name followed by the destiniation field name -- spaces not allowed).
1. #sync extra options are either `SET_NOT_PRESENT` followed by whatever you would put in an UPDATE SET clause for any rows that are not in the temp table followed by an optional WHERE clause to indicate additional cohort constraints, or `DELETE_NOT_PRESENT` followed by an optional WHERE clause that acts the same way. (In the SET version, the WHERE is a requied delimiter; in the DELETE version, the WHERE keyword may not be present.)

## .vfs Commands:
Only 1 Positional Argument is used. everything else is a comment
Command | Argument | Description
--------|----------|------------
#fifo | filename | creates a fifo (a virtual file) in the INPUT_DIR (ie, where the file we are processing lives, or pwd if that is stdin) and pipes everything after this line into that file (until it reaches another command)
#run | vitual vclod filename | runs everything from the start of stream to the first .vfs command through vclod_operation. The vitural filename here can then use the fifo files as extesion arguments

# Testing

First setup the `./test/secure_config` file to have the right mysql permissions.
`./run_test.sh` - confirms that the proper log files are generated with the right contents; checks syslog; check post_process log2sql; prints all output to the terminal
`./run_test.sh | cat` - does the same thing, but with no output except on test error
