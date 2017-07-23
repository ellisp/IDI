


****************************************************************************************************************************
****************************************************************************************************************************

CORRECTIONS DATA CORRECTIONS DATA CORRECTIONS DATA CORRECTIONS DATA CORRECTIONS DATA CORRECTIONS DATA
		PRISON	Prison sentenced	PRISON
		REMAND	Remanded in custody		PRISON
		ESO	Extended supervision order	COMMUNITY
		HD_REL	Released to HD	HOME
		PAROLE	Paroled	COMMUNITY
		ROC	Released with conditions	COMMUNITY
		HD_SENT	Home detention sentenced	HOME
		PDC	Post detention conditions	COMMUNITY
		INT_SUPER	Intensive supervision	COMMUNITY
		COM_DET	Community detention	COMMUNITY
		SUPER	Supervision	COMMUNITY
		CW	Community work		COMMUNITY
		PERIODIC	Periodic detention		COMMUNITY
		COM_PROG	Community programme		COMMUNITY
		COM_SERV	Community service		COMMUNITY
		OTH_COM	Other community		COMMUNITY

****************************************************************************************************************************;
****************************************************************************************************************************;

* FORMATING AND SENSORING;
* CORRECTIONS;

%macro Creating_clean_corr;
proc sql;
Connect to sqlservr (server=WPRDSQL36\iLeed database=IDI_clean_&Version);

	create table COR as
		SELECT distinct 
		 snz_uid,
			input(cor_mmp_period_start_date,yymmdd10.) as startdate,
			input(cor_mmp_period_end_date, yymmdd10.) as enddate,
			cor_mmp_mmc_code,  
           /* Creating wider correction sentence groupings */
	    	(case when cor_mmp_mmc_code in ('PRISON','REMAND' ) then 'COR_Custody'
			     when cor_mmp_mmc_code in ('HD_SENT','HD_SENT', 'HD_rel' ) then 'COR_HD'
                 when cor_mmp_mmc_code in ('ESO','PAROLE','ROC','PDC' ) then 'COR_Post_Re'
				 when cor_mmp_mmc_code in ('COM_DET','CW','COM_PROG','COM_SERV' ,'OTH_COMM','INT_SUPER','SUPER','PERIODIC') then 'COR_Com'
                 else 'COR_OTHER' end) as sentence 
		FROM COR.ov_major_mgmt_periods 
		where snz_uid in (SELECT DISTINCT snz_uid FROM &population) 
		/* exclude birthdate and aged out records */
		AND cor_mmp_mmc_code IN ('PRISON','REMAND','HD_SENT','HD_REL','ESO','PAROLE','ROC','PDC','PERIODIC',
			'COM_DET','CW','COM_PROG','COM_SERV','OTH_COMM','INT_SUPER','SUPER')

		ORDER BY snz_uid,startdate;
quit;

proc sql;
create table COR_1 as select
a.* ,
b.DOB
from COR a left join &population b
on a.snz_uid=b.snz_uid;

data COR_1 Del; set COR_1;
format startdate enddate date9.;

if startdate>"&sensor"d then delete;
if enddate>"&sensor"d then enddate="&sensor"d;

if startdate <intnx('YEAR',DOB,7,'S') then output del; else output  COR_1;
run;

* Remove overlaps:

* Prerequisite: the dataset need to contain startdate and enddate in a date format;
* Dummy year- any odd record, for corrections this is the aged out record and = 9999;
* If start date is less than end date of previous spell, means that record is overlapping and hence need sorting;
* output dataset has _OR suffix (stands for overlaps removed)
* requires inputs OVERLAP(filein);

%OVERLAP (COR_1); 

data CORR_clean; set COR_1_OR; run;

%mend;

********************************************************************************************************;
