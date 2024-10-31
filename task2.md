# **Task 2: Complex Query Writing**

## 1. High-Risk Patients

```sql
select 
    patient.patient_id,
    patient.first_name,
    patient.last_name,
    patient.gender,
    patient.dob,
    appointment.scheduled_date,
    appointment.speciality,
    appointment.hospital
from appointment
inner join prediction on prediction.appointment_id = appointment.appointment_id
inner join patient on patient.patient_id = appointment.patient_id
where appointment.priority = 'high'
and prediction.probability_dna > 0.9;
```



## 2. Clinic Performance

```sql

select 
    clinic.clinic_id,
    clinic.clinic,
    avg(prediction.probability_dna) as avg_appt_probability_dna
from appointment
inner join prediction on prediction.appointment_id = appointment.appointment_id
inner join clinic on clinic.clinic_id = appointment.clinic_id
group by clinic.clinic_id, clinic.clinic
order by avg_appt_probability_dna desc
limit 3;

```



## 3. Patient Engagement Rate

```sql
with all_appointments as
(
    select 
        appointment.patient_id,
        count(*) as appt_count
    from appointment
    group by appointment.patient_id
),

low_engagement_appointments as
(
    select 
        appointment.patient_id,
        count(prediction.id) as appt_count --count will be zero for patients who have never had a probability_dna > 0.8
    from appointment
    left join prediction
                        /*
                         Left joining with prediction so that all data from appointments is preserved, which will force
                         count(prediction.id) to return a zero value where there is no data for the associated appointment.patient_id

                         This is to ensure the patients who have never had a probability_dna > 0.8
                         are reflected with a count of zero in count(prediction.id).
                         With an inner join such patients will be filtered out and only those with
                         probability_dna > 0.8 will be returned.
                         */
        on prediction.appointment_id = appointment.appointment_id
        and prediction.probability_dna > 0.8
    group by appointment.patient_id
)

select
    patient.patient_id,
    patient.first_name,
    patient.last_name,
    patient.gender,
    patient.dob,
    (low_engagement_appointments.appt_count / all_appointments.appt_count) * 100 as engagement_rate
from patient
inner join all_appointments on all_appointments.patient_id = patient.patient_id
inner join low_engagement_appointments on low_engagement_appointments.patient_id = patient.patient_id
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
        appointment.patient_id,
        count(*) as missed_count
    from appointment
    inner join prediction on prediction.appointment_id = appointment.appointment_id
    where prediction.probability_dna > 0.85
    group by appointment.patient_id
    having count(*) > 3
) as missed_appts
inner join patient on patient.patient_id = missed_appts.patient_id;
```
