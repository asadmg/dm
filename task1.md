# **Task 1: Database Schema Optimization**

## 1. Schema modifications

The text below discusses options around how database performance could potentially be improved by employing a mix of three approaches:

- Denormalizing
- Partitioning
- Indexing

While these are some of the potential performance enhancing options, without having physical access to the database server and analysing its tables and queries, it's difficult to say how successful these will be.
There is always the risk that measures taken to improve the performance of one query could adversely affect another and a balance needs to be struck.


### Denormalizing
It appears that the `prediction` table has a one-to-one relationship with `appointment` table.
Instead of having to join these two tables on `'appointment_id'`, the fields from the `prediction` table could be added to the `appointment` table.
Doing so should result in faster a querying time as the database engine will have one less join on large tables.

However if my understanding is incorrect and these two tables have a one-to-many relationship, a `materialized view` that combines these tables could be considered as an option. This also enables less disruption by retaining both the existing tables and leaving the processes populating them unchanged.

Unlike regular views, materialized views store the output of a query in a table like structure. You run SELECT queries and create indices on them as if they were a normal table. Unlike regular views however, materialized views need to be refreshed periodically to ensure they reflect the data added/modified/removed in the underlying tables.


```sql
----------------------------------------------------------
/*
materialised view persists data from appointment and
prediction tables in one place to allow quicker access
without having to perform a join for every query.
*/
----------------------------------------------------------

create materialized view mv_appointment_prediction
as
    select 
        appointment.appointment_id,
		appointment.hospital,
		appointment.speciality,
		appointment.scheduled_date,
		appointment.patient_id,
        appointment.priority,
		appointment.clinic_id,
		prediction.id as prediction_id,
		prediction.prediction_batch,
		prediction.probability_dna
    from appointment
    inner join prediction on prediction.appointment_id = appointment.appointment_id;


----------------------------------------------------------
/*
improvements to query filtering performance can be made
by creating indices on frequently filtered and joined
columns in the materialised view
*/
----------------------------------------------------------

create index idx_priority
on mv_appointment_prediction (priority);

create index idx_probability_dna
on mv_appointment_prediction (probability_dna);

create index idx_patient_id
on mv_appointment_prediction (patient_id);

create index idx_clinic_id
on mv_appointment_prediction (clinic_id);


----------------------------------------------------------
--command for refreshing the materialised view
----------------------------------------------------------
refresh materialized view mv_appointment_prediction;

```

>NOTE: The materialized view would need a mechanism to be refreshed periodically, e.g. as a cron job.

>Indices can be created on top of materialized view for potentially better performance.

>NOTE: A postgres materialized view can't be partitioned.



### Partitioning
Partition the `appointment` table by `'scheduled_date'` into monthly partitions.

>NOTE: Partitioning the `appointment` table by `'scheduled_date'` into monthly partitions will only benefit the performance of queries that filter on `'scheduled_date'`, as the database engine will only have to read the relevant partitions where those records exist. For other queries the database engine could have to do full table scans, however that could be improved by having indices on suitable columns.

```sql
--- SQL DDL (Data Definition Language) for schema modifications and indexing:

----------------------------------------------------------
--create new appointment table with partitions
----------------------------------------------------------

--backup existing appointment table
alter table appointment rename to appointment_bak;

--create new appointment table with the same structure as appointment
create table appointment (like appointment_bak)
partition by range (scheduled_date);

--create primary and partition key
alter table appointment
add constraint pk_partitioned_appointments primary key (appointment_id, scheduled_date);
	-- https://www.postgresql.org/docs/current/ddl-partitioning.html#ddl-partitioning-declarative-limitations


--create appointment table partitions
create table appointment_202401 partition of appointment
for values from ('2024-01-01') to ('2024-02-01');

create table appointment_202402 partition of appointment
for values from ('2024-02-01') to ('2024-03-01');

create table appointment_202403 partition of appointment
for values from ('2024-03-01') to ('2024-04-01');

...

create table appointment_202412 partition of appointment
for values from ('2024-12-01') to ('2025-01-01');


----------------------------------------------------------
--populate the new appointment table with data
----------------------------------------------------------
insert into appointment 
select *
from appointment_bak;

```

>Table partitions will have to be added before data for that time range is inserted as there is
no automatic way for new partitions to get created. Althought the `pgpartman` extension could help automate that:
https://github.com/pgpartman/pg_partman?tab=readme-ov-file#postgresql-partition-manager

>Table partitioning and indexing can be used together for potentially better performance.


### Indexing
Create individual indices on the `appointment` table fields that are used for query filtering or used in join conditions
These could be:
- priority
- patient_id
- clinic_id

Index could also be placed on the `prediction` table field `'probability_dna'` as it is also used in filtering.


>Indexing can increase read speeds however that comes at the cost of write performance as each write needs to update the index as well as the table.

>NOTE: postgres is more suited to transactional workloads (OLTP) and not analytics (OLAP). Columnar databases e.g. Redshift, Snowflake, BigQuery are more suitable for analytics workloads. The `citus` extension brings columnar features to postgres, however it does have some limitations, e.g. being unable to update or delete data.
https://www.citusdata.com/blog/2021/03/06/citus-10-columnar-compression-for-postgres/


```sql
----------------------------------------------------------
--index creation
----------------------------------------------------------

create index idx_priority
on appointment (priority);

create index idx_patient_id
on appointment (patient_id);

create index idx_clinic_id
on appointment (clinic_id);

create index idx_probability_dna
on prediction (probability_dna);

create index idx_appointment_id
on prediction (appointment_id);
```


## 2. NoSQL integration

This is a difficult to analyse question as I am not aware of the structure and volume of the data, and also what usecase the existing relational database is serving. Of course unlike relational databases, NoSQL databases are schema-less and for them the data structure does not matter as much.

However assuming that the incoming features and prediction batches are in JSON format, these can be very quickly stored in a NoSQL store. Compared with relational systems, the data ingestion speeds are also faster and NoSQL databases can massively scale to deal with data volumes.
Depending on what type of processing needs to take place on that data, some querying, filtering, modifying can be done in a NoSQL database.

MongoDB has implemented a SQL compatible language, which should make it easier to query the collections compared with using a cli:
https://www.mongodb.com/docs/atlas/data-federation/query/sql/language-reference/#std-label-sql-reference

In order to provision the data from a NoSQL to a relational database (and vice-versa), a mechanism to ETL that data will be requied.

NoSQL could be used for longer term storage of the previously received features and prediction batches.
That could mean that historical data is saved in NoSQL and the relational database only keeps the features and prediction batches required for future appointments.
Doing so will help manage the size of the relational database and help with performance.

An ETL process can also provision the historical features and prediction batches from NoSQL to a data warehouse. This could help conduct analytics to derive value by learning from the past.

>A case study from Craigslist having relational and NoSQL databases working together provides some insight:
https://www.theserverside.com/feature/How-NoSQL-MySQL-and-MogoDB-worked-together-to-solve-a-big-data-problem




## 3. AWS Architecture

An architecture to handle large datasets could use some of the following services for the purposes of data storage, processing and backup:

### Data Storage
- Amazon S3 (raw and processed data)
-- S3-Tiering: move data between tiers when access patterns change, optimising costs.
- Amazon RDS (structured transactional data)
- Amazon DynamoDB (NoSQL data)

### Data Processing
- Amazon Managed Workflows for Apache Airflow (MWAA) - used for orchestration of ETL tasks
- Amazon EKS (Amazon Elastic Kubernetes Service)
- Amazon EMR (to host Apache Spark)
- AWS Glue (ETL tasks)

### Data Analytics
- Amazon Redshift (data warehouse)
- Amazon Athena (ad-hoc analysis)

### Backup and Recovery
- AWS Backup: Centralise and automate data protection across AWS services. Automate and consolidate backup tasks that were previously performed service-by-service. Supports RDS, Redshift, S3 and many more. Supports full and incremental backups.

- Amazon S3 Glacier (long-term storage)
