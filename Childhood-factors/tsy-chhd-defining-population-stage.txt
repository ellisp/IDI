
******************************************************************************************************************************************************************
******************************************************************************************************************************************************************

DICLAIMER:
This code has been created for research purposed by Analytics and Insights Team, The Treasury. 
The business rules and decisions made in this code are those of author(s) not Statistics New Zealand and New Zealand Treasury. 
This code can be modified and customised by users to meet the needs of specific research projects and in all cases, 
Analytics and Insights Team, NZ Treasury must be acknowledged as a source. While all care and diligence has been used in developing this code, 
Statistics New Zealand and The Treasury gives no warranty it is error free and will not be liable for any loss or damage suffered by the use directly or indirectly

******************************************************************************************************************************************************************
******************************************************************************************************************************************************************;


******************************************************************************************************************************************************
DEFINING BIRTH COHORTS

Developer: Sarah Tumen (A&I, NZ Treasury)
Created date: 11 Sep 2014

******************************************************************************************************************************************************
Obtaining those records of unique citizen in IDI
Notes: This includes unique citizen that have records with IRD, MOE, MSD, Justice)
This includes records of seasonal workers, overseas residents, international students 
at later stage they will be exlcuded from population).
*******************************************************************************************************************************************************;

* Run 00_code.sas code to get macros and libraries;
*Obtaining all records from admin datasets using SNZ concordance table;
proc SQL;
	Connect to sqlservr (server=WPRDSQL36\iLeed database=IDI_clean_&version);
	create table CONC as select * from connection to  sqlservr
		(select 
			snz_uid, 
			snz_ird_uid, 
			snz_dol_uid, 
			snz_moe_uid, 
			snz_msd_uid,
			snz_dia_uid,
			snz_moh_uid, 
			snz_jus_uid from security.concordance);
quit;

proc SQL;
	Connect to sqlservr (server=WPRDSQL36\iLeed database=IDI_clean_&version);
	create table PERS as select * from connection to  sqlservr
		(select 
			snz_uid, 
			snz_sex_code,
			snz_birth_year_nbr,
			snz_birth_month_nbr,
			snz_ethnicity_grp1_nbr,
			snz_ethnicity_grp2_nbr,
			snz_ethnicity_grp3_nbr,
			snz_ethnicity_grp4_nbr,
			snz_ethnicity_grp5_nbr,
			snz_ethnicity_grp6_nbr,
			snz_deceased_year_nbr,
			snz_deceased_month_nbr,
			snz_person_ind,
			snz_spine_ind
		from data.personal_detail);
quit;

proc sort data=conc;
	by snz_uid;
run;

Proc sort data=pers;
	by snz_uid;
run;

* This definition of the cohorts includes international students, overseas residents, deceased as at today and other subgroups of population, these subgroups need to be identified
* We will create indicators that will help to define population of interest;

* We decided to run this refresh for cohort 1990;
data Birth_1988_&last_anal_yr.;
	merge conc (in=a) pers (in=b);
	by snz_uid;

	if a and b;

	do i=1988 to &last_anal_yr.;
		if (snz_birth_year_nbr=i and snz_birth_month_nbr>=7) or (snz_birth_year_nbr=i+1 and snz_birth_month_nbr<7) then
			cohort=i;

		* Born from July 1990 to June 1991;
	end;

	if snz_person_ind=1;

	* Limiting to actual people;
	if snz_spine_ind=1;

	* This is to limit to people only who linked to the spine;
	if snz_birth_year_nbr>=1988 and snz_birth_year_nbr<=&last_anal_yr.;
	format DOB DOD date9.;
	DOB=MDY(snz_birth_month_nbr,15,snz_birth_year_nbr);
	DOD=MDY(snz_deceased_month_nbr,15,snz_deceased_year_nbr);
run;

proc freq data=Birth_1988_&last_anal_yr.;
	tables cohort snz_birth_year_nbr;
run;

data project.Population1988_&last_anal_yr.;
	set Birth_1988_&last_anal_yr.;
	drop i;
run;

/*%contents(project.Population1988_2013);*/

******************************************************************************************************************************************
******************************************************************************************************************************************;

* let's check whether Sylvia's population is fully captured;
proc datasets lib=work kill nolist memtype=data;
quit;