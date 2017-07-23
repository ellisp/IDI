* This code was modified from the original posted by Nathaniel Matheson-Dunning on the IDI code sharing wiki;
* It has been modified for a 31 December 2012 reference date and to work without access to the IRD source tables;
* Modified by Sheree Gibb in July 2016;


%let refresh = 20160224; ** Specify IDI refresh to use for extractions;

%let year = 2012; ** Specify year of interest;

%let prevyear = %eval(&year. - 1);

%let startyrsql = '2012-01-01';
%let endyrsql = '2012-12-31';

libname central ODBC dsn=idi_clean_&refresh._srvprd schema=data;
libname moe ODBC dsn=idi_clean_&refresh._srvprd schema=moe_clean;
libname msd ODBC dsn=idi_clean_&refresh._srvprd schema=msd_clean;
libname ird ODBC dsn=idi_sandpit_srvprd schema=clean_read_ir_restrict;
libname moh ODBC dsn=idi_clean_&refresh._srvprd schema=moh_clean;
libname acc ODBC dsn=idi_clean_&refresh._srvprd schema=acc_clean;
libname dia ODBC dsn=idi_clean_&refresh._srvprd schema=dia_clean;
libname dlab "\\wprdfs08\Datalab-MA\MAA2016-11 VIEW-IDI";
libname sand ODBC dsn=idi_sandpit_srvprd schema="DL-MAA2016-11";

** Create a list of all people in the spine;
data spinepop;
   set central.personal_detail (where=(snz_spine_ind=1) keep=snz_uid snz_spine_ind);
run;

**********************************************************************************************************
*** Produce a list of all individuals with activity in relevant datasets in last 12 months / 5 years   ***
**********************************************************************************************************;

*********************************************************************************
***   Identify individuals with activity in education datasets in last year   ***
*********************************************************************************;


** Tertiary enrolments;
proc sql;
connect to odbc(dsn="idi_clean_20160224_srvprd");
   create table tertiary as 
   select * from connection to ODBC
   (select distinct snz_uid, 1 as flag_ed
   from moe_clean.enrolment
   where (moe_enr_year_nbr = &year.)
   order by snz_uid);
   disconnect from ODBC;
quit;

** Industry training;
proc sql;
connect to odbc(dsn="idi_clean_20160224_srvprd");
   create table industry_tr as
   select * from connection to ODBC
   (select distinct snz_uid, 1 as flag_ed
   from moe_clean.tec_it_learner
   where (moe_itl_year_nbr = &year.)
   order by snz_uid);
   disconnect from ODBC;
quit;

** School enrolments;
proc sql;
   connect to odbc(dsn="idi_clean_20160224_srvprd");
   create table school as 
   select * from connection to ODBC
   (select distinct snz_uid, 1 as flag_ed, moe_esi_start_date
   from moe_clean.student_enrol 
   where ((moe_esi_start_date<=&endyrsql.) and (moe_esi_end_date>=&startyrsql. or moe_esi_end_date is NULL))
   order by snz_uid);
   disconnect from ODBC;
quit;





***************************************************************************
***   Identify individuals with activity in tax datasets in last year   ***
***************************************************************************;

**This section has been modified from the Census Transformation code as we only have access to the tax summary table in the IRI restrict area;
*All individuals with taxable income in the summary table for current or previous calendar year;
proc sql;
connect to odbc(dsn="idi_sandpit_srvprd");
   create table tax_sum as
   select * from connection to ODBC
   (select snz_uid, ir_inc_year_nbr, 
	inc_cal_yr_mth_01_amt+inc_cal_yr_mth_02_amt+inc_cal_yr_mth_03_amt+inc_cal_yr_mth_04_amt+inc_cal_yr_mth_05_amt+inc_cal_yr_mth_06_amt as tot_first_six,
	inc_cal_yr_mth_07_amt+inc_cal_yr_mth_08_amt+inc_cal_yr_mth_09_amt+inc_cal_yr_mth_10_amt+inc_cal_yr_mth_11_amt+inc_cal_yr_mth_12_amt as tot_last_six
   from clean_read_ir_restrict.ems_cal_yr_summary
   where ir_inc_year_nbr=&year or ir_inc_year_nbr=&prevyear
   order by snz_uid);
disconnect from ODBC;
quit;

*Select only those individuals with income in the last 6 months of last calendar year, or first six months of this calendar year;
proc sql;
create table tax as 
select distinct snz_uid, 1 as flag_ir
from tax_sum
where (ir_inc_year_nbr=&year and tot_first_six>0) or (ir_inc_year_nbr=&prevyear and tot_last_six>0);
quit;


**********************************************************************************
***   Identify individuals with activity in health datasets in the last year   ***
**********************************************************************************;

** GMS claims;
proc sql;
connect to odbc(dsn="idi_clean_20160224_srvprd");
   create table gms_activity as
   select * from connection to ODBC
   (select distinct snz_uid
   from moh_clean.gms_claims
   where moh_gms_visit_date >= &startyrsql and moh_gms_visit_date <= &endyrsql
   order by snz_uid);
   disconnect from odbc;
quit;

** Laboratory tests;
proc sql;
connect to odbc(dsn="idi_clean_20160224_srvprd");
   create table lab_claims as
   select * from connection to ODBC
   (select distinct snz_uid
   from moh_clean.lab_claims
   where moh_lab_visit_date >= &startyrsql and moh_lab_visit_date <= &endyrsql
   order by snz_uid);
disconnect from odbc;
quit;

** Non-admissions events;
proc sql;
connect to odbc(dsn="idi_clean_20160224_srvprd");
   create table nnpac as
   select * from connection to ODBC
   (select distinct snz_uid
   from moh_clean.nnpac
   where moh_nnp_service_date >= &startyrsql and moh_nnp_service_date <= &endyrsql and moh_nnp_attendence_code = 'ATT'
   order by snz_uid);
disconnect from odbc;
quit;

** Prescriptions dispensed;
proc sql;
connect to odbc(dsn="idi_clean_20160224_srvprd");
   create table pharma as
   select * from connection to ODBC
   (select distinct snz_uid
   from moh_clean.pharmaceutical
   where moh_pha_dispensed_date >= &startyrsql and moh_pha_dispensed_date <= &endyrsql
   order by snz_uid);
disconnect from odbc;
quit;

** Consultation with PHO-registered GP;
proc sql;
connect to odbc(dsn="idi_clean_20160224_srvprd");
   create table pho as
   select * from connection to ODBC
   (select distinct snz_uid
   from moh_clean.pho_enrolment
   where moh_pho_last_consul_date >= &startyrsql and moh_pho_last_consul_date <= &endyrsql
   order by snz_uid);
disconnect from odbc;
quit;

** Discharged from publically funded hospitals;
proc sql;
connect to odbc(dsn="idi_clean_20160224_srvprd");
   create table hospital as
   select * from connection to ODBC
   (select distinct snz_uid
   from moh_clean.pub_fund_hosp_discharges_event
   where moh_evt_even_date >= &startyrsql and moh_evt_even_date <= &endyrsql
   order by snz_uid);
disconnect from odbc;
quit;

** Combine all health activity datasets to get list of all people with health activity in last year;
data health_activity_dups;
   set gms_activity lab_claims nnpac pharma pho hospital;
run;
proc sort; by snz_uid;

data health_activity;
   set health_activity_dups;
   by snz_uid;
   if first.snz_uid;

   flag_health=1;
run;

**********************************************************************
***   Identify individuals with activity in ACC in the last year   ***
**********************************************************************;

** ACC claims (date of file within the last year, not date of accident);
proc sql;
connect to odbc(dsn="idi_clean_20160224_srvprd");
   create table acc as
   select * from connection to ODBC
   (select distinct snz_uid, 1 as flag_acc
   from acc_clean.claims
   where acc_cla_lodgement_date >= &startyrsql and acc_cla_lodgement_date <= &endyrsql
   order by snz_uid);
disconnect from odbc;
quit;

********************************************
***   Get births from the last 5 years   ***
********************************************;
proc sql;
connect to odbc(dsn="idi_clean_20160224_srvprd");
   create table births as
   select * from connection to ODBC
   (select distinct snz_uid, 1 as flag_birth
   from dia_clean.births
   where (dia_bir_birth_year_nbr > %eval(&year. - 5) and dia_bir_birth_year_nbr <= &year.) 
   order by snz_uid);
disconnect from odbc;
quit;

******************************************************************************
***   Combine all activity files with spine to create a total population   ***
******************************************************************************;

** Combine all activity files;
data total_activity_pop;
   merge tax industry_tr tertiary school health_activity acc births;
   by snz_uid;
   if flag_ed = . then flag_ed = 0;
   if flag_ir = . then flag_ir = 0;
   if flag_health = . then flag_health = 0;
   if flag_acc = . then flag_acc = 0;
   if flag_birth = . then flag_birth = 0;
   activity_flag = 1;
run;

proc sort data=spinepop;
   by snz_uid;
run;

** Combine with spine;
data totalpop;
   merge total_activity_pop (in=a) spinepop (in=b);
   by snz_uid;

   ** Only include invididuals who had activity and are in spine;
   if a and b;
run;

********************************************************************
***          Get age and sex from personal details file          ***
***        Remove deceased individuals from the population       ***
********************************************************************;

** Match with personal details file to pick up age, sex
** using hash object programming to reduce processing time and memory load;
data finalpop2;
   length snz_uid flag_ir flag_ed flag_health flag_acc flag_birth 8;

   if _N_ = 1 then
      do;
         declare hash h(dataset:'totalpop');
         h.defineKey('snz_uid');
         h.defineData('flag_ir', 'flag_ed', 'flag_health', 'flag_acc', 'flag_birth');
         h.defineDone();
      end;

   set central.personal_detail;

   if h.find() = 0 then
      output;
run;

** Calculate age at 30 June and remove deceased individuals;
data finalpop_age2;
   set finalpop2;
   age = &year. - snz_birth_year_nbr;

   if age lt 0 then  delete;
   if snz_deceased_year_nbr < &year. and snz_deceased_year_nbr ne . then delete;

   rename snz_sex_code = sex;
   keep snz_uid snz_sex_code age flag_ir flag_ed flag_health flag_acc flag_birth snz_birth_year_nbr snz_birth_month_nbr;
run;

** Get a list of all deaths recorded in health data;
proc sql;
connect to odbc(dsn="idi_clean_20160224_srvprd");
   create table moh_deaths as
   select * from connection to ODBC
   (select distinct snz_uid, 1 as deceased_flag
   from moh_clean.mortality
   where (moh_mor_death_year_nbr <= &year.)
   order by snz_uid);
disconnect from ODBC;
quit;

** Get a list of all deaths recorded in DIA data;
proc sql;
connect to odbc(dsn="idi_clean_20160224_srvprd");
   create table dia_deaths as
   select * from connection to ODBC
   (select distinct snz_uid, 1 as deceased_flag
   from dia_clean.deaths
   where (dia_dth_death_year_nbr <= &year.)
   order by snz_uid);
quit;

** Merge with population file and remove deceased individuals from the population;
data finalpop_age;
   merge finalpop_age2 moh_deaths dia_deaths;
   by snz_uid;
   if deceased_flag=1 then delete;
   drop deceased_flag;
run;


******************************************************************************
***   Remove individuals from the population if they are living overseas   ***
******************************************************************************;

** Calculate amount of time spent overseas in last 12 months;
proc sql;
   create table overseas_spells_1yr as
   select unique snz_uid , pos_applied_date, pos_ceased_date, pos_day_span_nbr
   from central.person_overseas_spell
   where pos_applied_date < "31DEC&year.:00:00:00.000"dt and pos_ceased_date >= "01JAN&year.:00:00:00.000"dt
   order by snz_uid, pos_applied_date;
quit;

data overseas_time_1yr;
   set overseas_spells_1yr;
   if pos_ceased_date >= "31DEC&year.:00:00:00.000"dt and pos_applied_date < "01JAN&year.:00:00:00.000"dt 
      then time_to_add = 365;

   else if pos_ceased_date >= "31DEC&year.:00:00:00.000"dt 
      then time_to_add = ("31DEC&year.:00:00:00.000"dt - pos_applied_date) / 86400;

   else if pos_ceased_date <= "31DEC&year.:00:00:00.000"dt and pos_applied_date >= "01JAN&year.:00:00:00.000"dt 
      then time_to_add = pos_day_span_nbr;

   else if pos_ceased_date <= "31DEC&year.:00:00:00.000"dt and pos_applied_date < "01JAN&year.:00:00:00.000"dt 
      then time_to_add = (pos_ceased_date - "01JAN&year.:00:00:00.000"dt) / 86400;
run;

proc sql;
   create table time_overseas_1yr as select snz_uid, ROUND(SUM(time_to_add), 1) as days_overseas_last1
   from overseas_time_1yr
   group by snz_uid;
quit;

** Combine total population with time spent overseas;
data dlab.VARIANZ_pop;
   merge finalpop_age (in=a) time_overseas_1yr (in=b);
   by snz_uid;
   if a;

   ** remove people who are overseas for more than 6 months out of the last 12;
   if days_overseas_last1 gt 183 then delete;
   drop snz_birth_year_nbr snz_birth_month_nbr days_overseas_last1;

   	*Create an age group variable;
	format agegrp $12.;
	if age=. then agegrp='';
	else if age <20 then agegrp="Under 20";
	else if (age ge 20 and age <= 24) then agegrp = "20-24";
	else if (age ge 25 and age <= 34) then agegrp = "25-34";
	else if (age ge 35 and age <= 44) then agegrp = "35-44";
	else if (age ge 45 and age <= 54) then agegrp = "45-54";
	else if (age ge 55 and age <= 64) then agegrp = "55-64";
	else if (age ge 65 and age <= 74) then agegrp = "65-74";
	else if (age ge 75 and age <= 84) then agegrp = "75-84";
	else if (age > 84) then agegrp = "85 and over";
run;

* End of code;

*Output to sandpit;
data sand.varianz_pop;
set dlab.varianz_pop;
run;