# ADMS PySpark Jobs

ADMS PySpark jobs are used in migrating and combining data from ADMS tables.

It is composed of three types of jobs:
* job 1 (raw table manager)
* job 2 (DMS emulator)
* job 3 (reporting table manager)

## Job 1 - Raw Table Manager

A raw table manager is responsible for creating and populating a raw table in an Aurora database.

The word raw is used to denote that this table manager does not manipulate the contents of an Oracle source table but just migrates it in an Aurora target table.

To keep track of the history of each row in the table (i.e., from the time it is inserted, updated, and deleted), the target table is designed to be an SCD type 2 table.

A raw table manager does not directly query the rows of an Oracle source table.  It uses as an input a Kafka topic that is populated by an AWS DMS task.  The AWS DMS task monitors the logs of a source table for every operation performed in each row of the source table and creates and sends the corresponding messages to the Kafka topic.  Only messages with an operation equal to load, insert, update, and delete are processed by a raw table manager.

A Kafka message created by a DMS tasked is a JSON string that is composed two main fields:
* data - a dictionary with name/value of its fields equal to the name/value of the columns of a source table
* metadata - a dictionary containing the following fields:
  * operation - operation performed (load, insert, update, delete)
    * load is treated similarly as insert
    * there other operations such as create or drop but these are ignored by the raw table manager
    * saved in the Aurora target table as the operation column
  * timestamp - when message was created by DMS task
    * saved in the Aurora target table as the dms_timestamp column
  * commit-timestamp - timestamp when operation (insert/update/delete) happened
    * saved in the Aurora target table as the commit_timestamp column
  * transaction-id - transaction ID where operation belongs
    * saved in the Aurora target table as the transaction_id column
    * if operation is load, field transaction-id is not available, column transaction_id is just assugned a value of 1980-01-01T00:00:00.000000Z
  * transaction-record-id - transaction record ID that uniquely identifies the operation in a transaction
    * saved in the Aurora target table as the transaction_record_id column
    * if operation is load, field transaction-record-id is not available, column transaction_record_id is just assigned a value of -1

Aside from the metadata described above, the following properties of a Kafka message are also saved in the Aurora target table:
* Kafka timestamp - when message was inserted in the Kafka topic
    * saved in the Aurora target table as the kafka_timestamp column
* Kafka offset
    * saved in the Aurora target table as the kafka_offset column
* Kafka partition
    * saved in the Aurora target table as the kafka_partition column


For the data field, as mentioned, it contains a dictionary that represents the column names/values of a source table.  These are used as column names/values in the Aurora target table.

The migration of data from Kafka message to the Aurora target table may seem to be straightforward at first.  However, there are limitations in the way an AWS DMS task creates a Kafka message:
* LOB columns may be truncated 
* Only the primary columns and columns that got updated are included in an update message


The values of large object (LOB) columns are truncated if it goes beyond the maximum allowable size.  A raw table manager assumes that a value is truncated if its field value in the Kafka message is equal to the maximum allowable size.  In such cases, the complete value must be retrieved directly by a raw table manager from the source table.  Currently, this logic is implemented but not yet tested and not yet used due to lack of direct access of a raw table manager to the source table.

As mentioned, another limitation in the way Kafka messages with update operations are handled.   Only columns that are primary keys or got columns that got updated are included in the data field of the Kafka messages.  This means that for columns that did not change, they are not present in the Kafka message.  Converting the Kafka message to a row that will be inserted in the Aurora target table will have missing column values.  Missing column values are not necessarily NULL values.  Missing values mean that a raw table manager does not know what value to use in those missing columns.  Since this problem occurs only in update messages, it means that the previous versions of the row already exist in the Aurora target table.  A raw table manager retrieves the latest version and use this as a basis to fill-in the missing values.

TODO: Discuss output Kafka topic

## Job 2 - DMS Emulator

A DMS emulator is responsible for joining (i.e., inner or left join) several Aurora raw tables created and populated by raw table managers, creating a Kafka message for each row generated from joining the tables, and sending each Kafka message to a Kafka topic

Job 2 is called a DMS emulator since it mimics the output of the AWS DMS task.  All the Kafka messages generated by a DMS emulator follows the format used by AWS DMS, specifically Kafka messages for insert operations.

Since the raw tables created and populated by the raw table managers are SCD type 2, performing an inner or left join to these tables are not straight forward.  Using the typical join in SQL will result to unnecessary columns since a row (with a unique primary key) in an Oracle source table, may appear more than once (i.e., several versions of the row) in its corresponding Aurora row table since the history of this row (its original value and all of its updates) are saved in the SCD type 2 table.

To address this, a DMS emulator compares the commit_timestamp and transaction_record_id of the Aurora raw tables to be joined in order to determine the correct versions of the rows to join.


## Job 3 - Reporting Table Manager

A reporting table manager is responsible for creating and populating a reporting table in an Aurora database.

The word reporting is used to denote that this table manager creates and populates an Aurora target table that contains rows that are based on the joining of several Aurora raw tables that were created and populated by raw table managers.

Similar to a raw table manager, a reporting table manager uses a Kafka topic as an input.  In the case of a raw table manager, it uses a Kafka topic that is populated by and AWS DMS task, while for a reporting table manager, it uses a Kafka topic that is populated by a DMS emulator.

Since a DMS emulators mimics an AWS DMS task and populates a Kafka topic with messages with a similar format used by an AWS DMS task, the approach used by a reporting table manager is very similar to a raw table manager.
