# **Task 3: Query Optimization**

>NOTE: Please see materialized view `'mv_appointment_prediction'` in task#1 before reviewing queries in this task

In contrast to the queries provided for task#2, the queries below:

- Use the materialized view `'mv_appointment_prediction'` which has the `appointment` and `prediction` tables pre-joined.

- Additionally, `'mv_appointment_prediction'` has indices on `patient_id`, `clinic_id`, `priority` and `probability_dna` fields which help speed up filtering and joining.

- In addition to using `'mv_appointment_prediction'`, the 'Patient Engagement Rate' query is not using CTEs as the code is a lot clearer and readable. I had isolated the logic into CTEs in task#2 to and making it "modular" and easier to read.


## 1. High-Risk Patients

```sql
select 
    patient.patient_id,
    patient.first_name,
    patient.last_name,
    patient.gender,
    patient.dob,
    ap.scheduled_date,
    ap.speciality,
    ap.hospital
from mv_appointment_prediction as ap
inner join patient on patient.patient_id = ap.patient_id
where ap.priority = 'high'
and ap.probability_dna > 0.9;

```


## 2. Clinic Performance

```sql
select 
    clinic.clinic_id,
    clinic.clinic,
    avg(ap.probability_dna) as avg_appt_probability_dna
from mv_appointment_prediction as ap
inner join clinic on clinic.clinic_id = ap.clinic_id
group by clinic.clinic_id, clinic.clinic
order by avg_appt_probability_dna desc
limit 3;

```


## 3. Patient Engagement Rate

```sql

select
    patient.patient_id,
    patient.first_name,
    patient.last_name,
    patient.gender,
    patient.dob,
    (COALESCE(low_engagement_appointments.appt_count, 0) / all_appointments.appt_count) * 100 as engagement_rate
from patient
inner join
(
    select
        patient_id,
        count(*) as appt_count
    from mv_appointment_prediction
    group by patient_id
) as all_appointments on all_appointments.patient_id = patient.patient_id
left join
(
    select
        patient_id,
        count(*) as appt_count
    from mv_appointment_prediction
    where probability_dna > 0.8
    group by patient_id
) as low_engagement_appointments on low_engagement_appointments.patient_id = patient.patient_id
order by engagement_rate desc;

```


## 4. Missed Appointments

```sql
select
    patient.patient_id,
    patient.first_name,
    patient.last_name,
    patient.gender,
    patient.dob,
    missed_appts.missed_count
from
(
    select 
        patient_id,
        count(*) as missed_count
    from mv_appointment_prediction
    where probability_dna > 0.85
    group by patient_id
    having count(*) > 3   
) as missed_appts
inner join patient on patient.patient_id = missed_appts.patient_id;

```
