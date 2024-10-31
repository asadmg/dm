# **Task 4: Backup and disaster recovery plan**

## Background information

Before going into a plan, here are a few key concepts around RDS backups to help set the stage:

- RDS uses the concept of `snapshots` when it comes to backups. `Snapshots` are essentially full backups, although they are created incrementally by comparing the database state with the previous backup.

- RDS provides the functionality for taking automated instance backups, the default frequency for which is daily with a retention period of 7 days.

- The backups can be copied to a different AWS region for greater protection in case of the host region going offline.

- RDS also offers "point-in-time" restores, which allow restoring the DB instance to a specific time during the automated backup retention period. The latest restorable point-in-time for a DB instance is typically within 5 minutes of the current time, resuling in a Recovery Point Objective (RPO) of 5 minutes or under. Database transaction logs are used to provide this functionality.

- Database transaction logs can also be copied to a different AWS region for greater protection and to allow point-in-time restores if the host region goes offline.

- "AWS Backup" is a comprehensive backup service that can be used to manage the backup of individual AWS resources, including RDS.


## Backup and disaster recovery plan

### Introduction

As a word of caution, any recovery requirements need to be tested and verified as being achievable. If with the existing setup the required plan is not achievable, then measures will need to be taken to make it possible. That could mean splitting data across more than one database servers, archiving historical data to speed-up backups and restores, or a more powerful server.

Once the plan has been created, it then requires periodic testing to make sure it continues to perfom within expected parameters.

### Assumptions for recovery objectives

For the purposes of this exercise, let's assume that Recovery Time Objective (RTO) is 1 hour and Recovery Point Objective (RPO) is also 1 hour.

### Resource setup to enable planned recovery

Use AWS Backup service to set up the following for the DB instance:

- enable automated backup snapshots every 12 hours
- enable point-in-time restores
- set automated backup snapshots' retention period to 35 days
- select option for copying the snapshots to a different region

>Go into the "automated backups" section on RDS and ensure that cross region replication is enabled for the DB instance. This is to ensure DB transaction logs will be available alongside the snapshots for point-in-time restores in the other region.

>The plan above is simplistic and does not take into account legal requirements around data being exported out of a region.

### Plan execution

If there is an incident, we should be able to very quickly restore into an instance from to a point-in-time within the snapshot retention period, potentially to an Recovery Point Objective (RPO) of 5 mins.

Depending on the size of data, a Recovery Time Objective (RTO) of one hour is achievable, although as mentioned above in the introduction this would need testing.

If the main AWS host region is compromised, the restore can be done in the backup region, and traffic will need to be routed accordingly until the main region is back online.

### Caveats

- There is the chance that not all snapshots (less likely) or transactions (more likely), make it to the backup region in time due to network latencies or other factors.
- There remains the chance that some transactions closer to the time that the incident took place on the host region get lost (depending of course on the nature of the incident, how busy our RDS DB instance is, and consequently how much transactional data needed to be sent across to the other region just before failure).
- It would need testing but I would expect adhering to a Recovery Point Objective (RPO) of 1 hour on the backup region
- Taking more frequent automated backups could help mitigate this, i.e. a snapshot every hour instead or every 12, although that would need testing to ensure it doesn't put undue pressure on the DB instance
