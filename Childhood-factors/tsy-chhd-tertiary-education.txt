
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



****************************************************************************************************************************************
****************************************************************************************************************************************

Developer: Sarah Tumen (A&I, NZ Treasury)
Date created: 14Jan2015

This code selected tertiary education indicators at the age of reference person. 
Indicators include tertiary participation, education qualifications at age of the reference person.

****************************************************************************************************************************************
****************************************************************************************************************************************

****************************************************************************************************************************************
****************************************************************************************************************************************;

* Set macro variables;
* Run 00_code.sas code to get macros and libraries;


****************************************************************************************************************************************
****************************************************************************************************************************************
TERTIARY ENROLMENT

*****************************************************************************************************;

*****************************************************************************************************;

**********************************************************************************************************************
**********************************************************************************************************************

TERTIARY QUALIFICATIONS attained
individuals can attain several qualifications in a year, the aim is to create indicators
of qualifications gained and highest qualification gained can be derived
Our defined levels
		Doctorate degree and Masters degree
		Postraguate diploma and Certificates +Bachelors with honours
		Bachelor's degree plus Graduate Diploma and Certificates
		Other Diploma and Certicicates
USING QACC

	01	Higher Doctorate
	10	PhD
	11	Masters
	12	Bachelors with Honours
	13	Post-graduate Diplomas
	14	Post-graduate Certificates
	20	Bachelors
	21	Graduate Diploma/ Certificate
	25	Certificate of Proficiency (credit to a Degree)
	30	Professional Association Diploma
	31	National Diploma/ National Certificate Levels 5-7
	32	New Zealand Diploma
	33	Diploma/ Certificate issued by TEO
	34	Advanced Trade Certificate
	35	New Zealand Certificate/ Technicians Certificate
	36	National Certificate Level 4 and other Level 4 Certificates
	37	Certificate of Proficiency (credit to a Diploma)
	40	Professional Association Certificate
	41	National Certificate Levels 1-3
	43	Trade Certificate Level 4
	46	Certificate issued by TEO
	60	Licence
	90	Certificate of Personal Interest
	96	STAR
	97	Programmes of study taught under contract
	98	Programmes of study made up of selected unit standards
	99	Community Education Programmes

proc format 
    value $lv8id
          "40"-"41","46", "60", "96", "98"      ="1"
          "36"-"37","43"                        ="2"
          "30"-"35"                       ="3"
          "20","25"                       ="4"
          "21","12"-"14"                  ="6"
          "11"                            ="7"
          "01","10"                       ="8"
          "90", "97", "99"                ="9"
          Other                           ="E"

    value $lv8d
          "1"     =      "Level 1-3 certificates"
          "2"     =      "Level 4 certificates"
          "3"     =      "Diplomas"
          "4"     =      "Bachelors"
          "6"     =      "Level 7-8 graduate honours certs/dips"
          "7"     =      "Masters"
          "8"     =      "Doctorates"
          "9"     =      "Non formal"
         
run;

************************************************************************************************************************************
************************************************************************************************************************************

PARTICIAPTION in TERTIARY Education 

************************************************************************************************************************************
************************************************************************************************************************************;
proc format;
	value $lv8id
		"40","41","46", "60", "96", "98"      ="1"
		"36"-"37","43"                        ="2"
		"30"-"35"                       ="3"
		"20","21","25"                       ="4"
		"12"-"14"                       ="6"
		"11"                            ="7"
		"01","10"                       ="8"
		"90", "97", "99"                ="9"
		Other                           ="E";
run;

proc format;
	value $subsector
		"1","3"="Universities"
		"2"="Polytechnics"
		"4"="Wananga"
		"5","6"="Private Training Establishments";
run;


* FORMATING, CLEANING AND SENSORING;
* creates clean enrolment file ENROL_CLEAN for popualtion with interest, contains DOB of ref person;
%creating_ter_enrol_table;



* creating aggregated summary of enrolment days and indicators of enrolment
* students can be enrolled in multiple programmes so overlap can happen
* we are calculating days enrolled and indicators of enrolment;
%overlap(ter_enrol_clean);
%aggregate_by_year(ter_enrol_clean_OR,enrol_2,&first_anal_yr,&last_anal_yr);

* ENROL_2 to be used for duartion summary;
	
* ENROL_CLEAN to be used for EFTS summary;
	
	* calcualting enrolments at age;
data _enrol_2;
	set enrol_2;

	* using overlap removed file;
	array ter_enr_da_at_age_(*)	ter_enr_da_at_age_&firstage-ter_enr_da_at_age_&lastage;
	array ter_enr_id_at_age_(*)	ter_enr_id_at_age_&firstage-ter_enr_id_at_age_&lastage;

	array f_ter_enr_da_at_age_(*)	f_ter_enr_da_at_age_&firstage-f_ter_enr_da_at_age_&lastage;
	array f_ter_enr_id_at_age_(*)	f_ter_enr_id_at_age_&firstage-f_ter_enr_id_at_age_&lastage;

	array nf_ter_enr_da_at_age_(*)	nf_ter_enr_da_at_age_&firstage-nf_ter_enr_da_at_age_&lastage;
	array nf_ter_enr_id_at_age_(*)	nf_ter_enr_id_at_age_&firstage-nf_ter_enr_id_at_age_&lastage;

	do ind=&firstage to &lastage;
		i=ind-(&firstage-1);
		ter_enr_da_at_age_(i)=0;
		ter_enr_id_at_age_(i)=0;

		f_ter_enr_da_at_age_(i)=0;
		f_ter_enr_id_at_age_(i)=0;

		nf_ter_enr_da_at_age_(i)=0;
		nf_ter_enr_id_at_age_(i)=0;

		start_window=intnx('YEAR',DOB,i-1,'S');
		end_window=intnx('YEAR',DOB,i,'S');

		if not((startdate > end_window) or (enddate < start_window)) then
			do;
			ter_enr_id_at_age_(i)=1;
			f_ter_enr_id_at_age_(i)=1;
			nf_ter_enr_id_at_age_(i)=1;

				if (startdate <= start_window) and  (enddate > end_window) then
					days=(end_window-start_window)+1;
				else if (startdate <= start_window) and  (enddate <= end_window) then
					days=(enddate-start_window)+1;
				else if (startdate > start_window) and  (enddate <= end_window) then
					days=(enddate-startdate)+1;
				else if (startdate > start_window) and  (enddate > end_window) then
					days=(end_window-startdate)+1;
				ter_enr_da_at_age_[i]=days*ter_enr_id_at_age_(i);
				f_ter_enr_da_at_age_[i]=days*f_ter_enr_id_at_age_(i);
				nf_ter_enr_da_at_age_[i]=days*nf_ter_enr_id_at_age_(i);

			end;

		drop i ind;
	end;
run;

proc summary data=_enrol_2 nway;
	class snz_uid;
	var 
ter_enr_da_at_age_&firstage-ter_enr_da_at_age_&lastage ter_enr_id_at_age_&firstage-ter_enr_id_at_age_&lastage
f_ter_enr_da_at_age_&firstage-f_ter_enr_da_at_age_&lastage f_ter_enr_id_at_age_&firstage-f_ter_enr_id_at_age_&lastage
nf_ter_enr_da_at_age_&firstage-nf_ter_enr_da_at_age_&lastage nf_ter_enr_id_at_age_&firstage-nf_ter_enr_id_at_age_&lastage;
	output out= TEMP (drop=_TYPE_ _FREQ_) sum=;
run;

data project._IND_TER_ENROL_at_age_&date;
	merge &population(keep=snz_uid DOB) TEMP;
	by snz_uid;
	array ter_enr_id_at_age_(*)	ter_enr_id_at_age_&firstage-ter_enr_id_at_age_&lastage;
	array ter_enr_da_at_age_(*)	ter_enr_da_at_age_&firstage-ter_enr_da_at_age_&lastage;

	array f_ter_enr_id_at_age_(*)	f_ter_enr_id_at_age_&firstage-f_ter_enr_id_at_age_&lastage;
	array f_ter_enr_da_at_age_(*)	f_ter_enr_da_at_age_&firstage-f_ter_enr_da_at_age_&lastage;

	array nf_ter_enr_id_at_age_(*)	nf_ter_enr_id_at_age_&firstage-nf_ter_enr_id_at_age_&lastage;
	array nf_ter_enr_da_at_age_(*)	nf_ter_enr_da_at_age_&firstage-nf_ter_enr_da_at_age_&lastage;

	do ind=&firstage to &lastage;
		i=ind-(&firstage-1);

		if ter_enr_id_at_age_(i)=. then
			ter_enr_id_at_age_(i)=0;

		if ter_enr_id_at_age_(i)>1 then
			ter_enr_id_at_age_(i)=1;

		if ter_enr_da_at_age_(i)=. then
			ter_enr_da_at_age_(i)=0;


		if f_ter_enr_id_at_age_(i)=. then
			f_ter_enr_id_at_age_(i)=0;

		if f_ter_enr_id_at_age_(i)>1 then
			f_ter_enr_id_at_age_(i)=1;

		if f_ter_enr_da_at_age_(i)=. then
			f_ter_enr_da_at_age_(i)=0;


		if nf_ter_enr_id_at_age_(i)=. then
			nf_ter_enr_id_at_age_(i)=0;

		if nf_ter_enr_id_at_age_(i)>1 then
			nf_ter_enr_id_at_age_(i)=1;

		if nf_ter_enr_da_at_age_(i)=. then
			nf_ter_enr_da_at_age_(i)=0;



		* Now sensoring for not fully observed data;
		start_window=intnx('YEAR',DOB,i-1,'S');
		end_window=intnx('YEAR',DOB,i,'S');

		if ((end_window>"&sensor"d) or (start_window>"&sensor"d)) then
			ter_enr_id_at_age_(i)=.;

		if ((end_window>"&sensor"d) or (start_window>"&sensor"d)) then
			ter_enr_da_at_age_(i)=.;


		if ((end_window>"&sensor"d) or (start_window>"&sensor"d)) then
			f_ter_enr_id_at_age_(i)=.;

		if ((end_window>"&sensor"d) or (start_window>"&sensor"d)) then
			f_ter_enr_da_at_age_(i)=.;

			
		if ((end_window>"&sensor"d) or (start_window>"&sensor"d)) then
			nf_ter_enr_id_at_age_(i)=.;

		if ((end_window>"&sensor"d) or (start_window>"&sensor"d)) then
			nf_ter_enr_da_at_age_(i)=.;
	end;

	drop i ind start_window end_window;
run;

* To calculate EFTS consumed lets use file where overlap is not removed;
data _enrol;
	set ter_enrol_clean;

	* using overlap removed file;
	array ter_efts_cons_at_age_(*)	ter_efts_cons_at_age_&firstage-ter_efts_cons_at_age_&lastage;
	array f_ter_efts_cons_at_age_(*)	f_ter_efts_cons_at_age_&firstage-f_ter_efts_cons_at_age_&lastage;
	array nf_ter_efts_cons_at_age_(*)	nf_ter_efts_cons_at_age_&firstage-nf_ter_efts_cons_at_age_&lastage;

	do ind=&firstage to &lastage;
		i=ind-(&firstage-1);
		ter_efts_cons_at_age_(i)=0;
		f_ter_efts_cons_at_age_(i)=0;
		nf_ter_efts_cons_at_age_(i)=0;

		start_window=intnx('YEAR',DOB,i-1,'S');
		end_window=intnx('YEAR',DOB,i,'S');

		if not((startdate > end_window) or (enddate < start_window)) then
			do;
				if (startdate <= start_window) and  (enddate > end_window) then
					days=(end_window-start_window)+1;
				else if (startdate <= start_window) and  (enddate <= end_window) then
					days=(enddate-start_window)+1;
				else if (startdate > start_window) and  (enddate <= end_window) then
					days=(enddate-startdate)+1;
				else if (startdate > start_window) and  (enddate > end_window) then
					days=(end_window-startdate)+1;
				ter_efts_cons_at_age_[i]=(EFTS_consumed/dur)*days;
				if formal=1 then f_ter_efts_cons_at_age_[i]=(EFTS_consumed/dur)*days;
				if formal=0 then nf_ter_efts_cons_at_age_[i]=(EFTS_consumed/dur)*days;

			end;

		drop i ind;
	end;
run;

proc summary data=_enrol nway;
	class snz_uid;
	var ter_efts_cons_at_age_&firstage-ter_efts_cons_at_age_&lastage
	f_ter_efts_cons_at_age_&firstage-f_ter_efts_cons_at_age_&lastage
	nf_ter_efts_cons_at_age_&firstage-nf_ter_efts_cons_at_age_&lastage;
	output out=TEMP (drop=_TYPE_ _FREQ_) sum=;
run;

data project._IND_TER_EFTS_at_age_&date;
	merge &population (keep=snz_uid DOB) TEMP;
	by snz_uid;
	array ter_efts_cons_at_age_(*)	ter_efts_cons_at_age_&firstage-ter_efts_cons_at_age_&lastage;
	array f_ter_efts_cons_at_age_(*)	f_ter_efts_cons_at_age_&firstage-f_ter_efts_cons_at_age_&lastage;
	array nf_ter_efts_cons_at_age_(*)	nf_ter_efts_cons_at_age_&firstage-nf_ter_efts_cons_at_age_&lastage;

	do ind=&firstage to &lastage;
		i=ind-(&firstage-1);

		if ter_efts_cons_at_age_(i)=. then
			ter_efts_cons_at_age_(i)=0;
		if f_ter_efts_cons_at_age_(i)=. then
			f_ter_efts_cons_at_age_(i)=0;
		if nf_ter_efts_cons_at_age_(i)=. then
			nf_ter_efts_cons_at_age_(i)=0;


		* Now sensoring for not fully observed data;
		start_window=intnx('YEAR',DOB,i-1,'S');
		end_window=intnx('YEAR',DOB,i,'S');

		if ((end_window>"&sensor"d) or (start_window>"&sensor"d)) then
			ter_efts_cons_at_age_(i)=.;
		if ((end_window>"&sensor"d) or (start_window>"&sensor"d)) then
			f_ter_efts_cons_at_age_(i)=.;
		if ((end_window>"&sensor"d) or (start_window>"&sensor"d)) then
			nf_ter_efts_cons_at_age_(i)=.;
	end;

	drop i ind start_window end_window;
run;

*********************************************************************************************************************
Highest Tertiary completed qualification at age
*********************************************************************************************************************;

* Calculating Highest Tertiary completion;
proc format;
	value Ter_qual
		0='No Ter Qual'
		1='Ter Other qual'
		2='Ter L1-3 Cert'
		3='Ter L4 Cert'
		4='Ter Diploma'
		5='Bachelor degrees'
		6='Postgrad Dip Cert'
		7='Masters PHD';
run;

* calculating highest ter qual attained in a year;
data Compl;
	set project.IND_TER_COMPL_&date;
	high_ter_comp_qual=0;

	if att_TER_oth=1 then
		high_ter_comp_qual=1;

	* Other Tertiary qualifications;
	if att_TER_L1_3Cert=1 then
		high_ter_comp_qual=2;

	* Level 1-3 Certificates;
	if att_TER_L4Cert=1 then
		high_ter_comp_qual=3;

	* Level 4 Certificates;
	if att_TER_Dipl=1 then
		high_ter_comp_qual=4;

	* Tertiary diplomas;
	if att_TER_Bach=1 then
		high_ter_comp_qual=5;

	* Bachelor degrees;
	if att_TER_Postgrad=1 then
		high_ter_comp_qual=6;

	* Postgraduate degrees;
	if att_TER_MastPHD=1 then
		high_ter_comp_qual=7;

	* Masters & PhD degrees;
	format startdate enddate date9.;
	startdate=MDY(12,30,year);
	enddate=startdate+1;

	/*keep snz_uid year high_ter_comp_qual startdate enddate;*/
run;

* Brining in DOB;
proc sql;
	create table Compl_at_age
		as select 
			a.*,
			b.DOB
		from COMPL a left join &population b
			on a.snz_uid=b.snz_uid;

data cohort_1;
	set &population(keep=snz_uid DOB);
run;

data Compl_at_age1;
	set Compl_at_age cohort_1;
	array high_ter_qual_at_age_(*)	high_ter_qual_at_age_&firstage-high_ter_qual_at_age_&lastage;

	do ind=&firstage to &lastage;
		i=ind-(&firstage-1);
		high_ter_qual_at_age_(i)=0;
		start_window=intnx('YEAR',DOB,i-1,'S');
		end_window=intnx('YEAR',DOB,i,'S');

		if not((startdate > end_window) or (enddate < start_window)) then
			do;
				high_ter_qual_at_age_(i)=high_ter_comp_qual;
			end;

		drop i ind start_window end_window;
	end;
run;

proc summary data=Compl_at_age1 nway;
	class snz_uid DOB;
	var high_ter_qual_at_age_&firstage-high_ter_qual_at_age_&lastage;
	output out=TEMP(drop=_TYPE_ _FREQ_) sum=;
run;

data project._IND_TER_COMPL_at_age_&date;
	set TEMP;
	array high_ter_qual_at_age_(*)	high_ter_qual_at_age_&firstage-high_ter_qual_at_age_&lastage;

	do ind=&firstage to &lastage;
		i=ind-(&firstage-1);

		* Now sensoring for not fully observed data;
		start_window=intnx('YEAR',DOB,i-1,'S');
		end_window=intnx('YEAR',DOB,i,'S');

		if ((end_window>"&sensor"d) or (start_window>"&sensor"d)) then
			high_ter_qual_at_age_(i)=.;
		drop i ind start_window end_window;
	end;
run;

*********************************************************************************************************************
Highest Industry training qualification completed at age
*********************************************************************************************************************;

* calculating highest ter qual attained in a year;
data IT_Compl;
	set project.IND_ITL_QUAL_&date;
	format startdate enddate date9.;
	startdate=MDY(12,30,year);
	enddate=startdate+1;
	keep snz_uid year high_IT_qual startdate enddate;
run;

* Brining in DOB;
proc sql;
	create table IT_Compl_at_age
		as select 
			a.*,
			b.DOB
		from IT_Compl a left join &population b
			on a.snz_uid=b.snz_uid;

data cohort_1;
	set &population(keep=snz_uid DOB);
run;

data IT_Compl_at_age1;
	set IT_Compl_at_age cohort_1;
	array high_IT_qual_at_age_(*)	high_IT_qual_at_age_&firstage-high_IT_qual_at_age_&lastage;

	do ind=&firstage to &lastage;
		i=ind-(&firstage-1);
		high_IT_qual_at_age_(i)=0;
		start_window=intnx('YEAR',DOB,i-1,'S');
		end_window=intnx('YEAR',DOB,i,'S');

		if not((startdate > end_window) or (enddate < start_window)) then
			do;
				high_IT_qual_at_age_(i)=high_IT_qual;
			end;

		drop i ind start_window end_window;
	end;
run;

proc summary data=IT_Compl_at_age1 nway;
	class snz_uid DOB;
	var high_IT_qual_at_age_&firstage-high_IT_qual_at_age_&lastage;
	output out=TEMP(drop=_TYPE_ _FREQ_) sum=;
run;

data project._IND_IT_COMPL_at_age_&date;
	set TEMP;
	array high_IT_qual_at_age_(*)	high_IT_qual_at_age_&firstage-high_IT_qual_at_age_&lastage;

	do ind=&firstage to &lastage;
		i=ind-(&firstage-1);

		* Now sensoring for not fully observed data;
		start_window=intnx('YEAR',DOB,i-1,'S');
		end_window=intnx('YEAR',DOB,i,'S');

		if ((end_window>"&sensor"d) or (start_window>"&sensor"d)) then
			high_IT_qual_at_age_(i)=.;
		drop i ind start_window end_window;
	end;
run;


***************************************************************************************************************
***************************************************************************************************************
Last tertiary qualification enrolled
***************************************************************************************************************
***************************************************************************************************************;

* calcualting enrolments at age;
data enrol_;
	set ter_enrol_clean;
	array ter_enr_id_at_age_(*)	ter_enr_id_at_age_&firstage-ter_enr_id_at_age_&lastage;
	array f_ter_enr_id_at_age_(*)	f_ter_enr_id_at_age_&firstage-f_ter_enr_id_at_age_&lastage;
	array nf_ter_enr_id_at_age_(*)	nf_ter_enr_id_at_age_&firstage-nf_ter_enr_id_at_age_&lastage;
	array ter_enr_ld_at_age_(*)	ter_enr_ld_at_age_&firstage-ter_enr_ld_at_age_&lastage;
	array f_ter_enr_ld_at_age_(*)	f_ter_enr_ld_at_age_&firstage-f_ter_enr_ld_at_age_&lastage;
	array nf_ter_enr_ld_at_age_(*)	nf_ter_enr_ld_at_age_&firstage-nf_ter_enr_ld_at_age_&lastage;
	format ter_enr_ld_at_age_: f_ter_enr_ld_at_age_:  nf_ter_enr_ld_at_age_: date9.;
	array ter_enr_lev_at_age_(*)	ter_enr_lev_at_age_&firstage-ter_enr_lev_at_age_&lastage;
	array f_ter_enr_lev_at_age_(*)	f_ter_enr_lev_at_age_&firstage-f_ter_enr_lev_at_age_&lastage;
	array nf_ter_enr_lev_at_age_(*)	nf_ter_enr_lev_at_age_&firstage-nf_ter_enr_lev_at_age_&lastage;

	do ind=&firstage to &lastage;
		i=ind-(&firstage-1);
		start_window=intnx('YEAR',DOB,i-1,'S');
		end_window=intnx('YEAR',DOB,i,'S');

		if not((startdate > end_window) or (enddate < start_window)) then
			do;
				ter_enr_id_at_age_(i)=1;

				if formal=1 then
					f_ter_enr_id_at_age_(i)=1;

				if formal=0 then
					nf_ter_enr_id_at_age_(i)=1;

				if (startdate <= start_window) and  (enddate > end_window) then
					last_day=end_window;
				else if (startdate <= start_window) and  (enddate <= end_window) then
					last_day=enddate;
				else if (startdate > start_window) and  (enddate <= end_window) then
					last_day=enddate;
				else if (startdate > start_window) and  (enddate > end_window) then
					last_day=end_window;
				ter_enr_lev_at_age_[i]=level*ter_enr_id_at_age_(i);

				if formal=1 then
					f_ter_enr_lev_at_age_[i]=level*f_ter_enr_id_at_age_(i);

				if formal=0 then
					nf_ter_enr_lev_at_age_[i]=level*nf_ter_enr_id_at_age_(i);
				ter_enr_ld_at_age_[i]=last_day*ter_enr_id_at_age_(i);

				if formal=1 then
					f_ter_enr_ld_at_age_[i]=last_day*f_ter_enr_id_at_age_(i);

				if formal=0 then
					nf_ter_enr_ld_at_age_[i]=last_day*nf_ter_enr_id_at_age_(i);
			end;

		drop i ind;
	end;
run;

***************************************************************************************************************************************;
* we are going to pick up the last enrolment at age, which requires some manipulation;
%macro lastlevel;
	%do i=&firstage %to &lastage;

		data lastlevel_&i;
			* all tertiary enrolments;
			set enrol_;
			keep snz_uid ter_enr_id_at_age_&i ter_enr_ld_at_age_&i ter_enr_lev_at_age_&i;

			if ter_enr_id_at_age_&i=1;

		data f_lastlevel_&i;
			* formal enrolments;
			set enrol_;
			keep snz_uid f_ter_enr_id_at_age_&i f_ter_enr_ld_at_age_&i f_ter_enr_lev_at_age_&i;

			if f_ter_enr_id_at_age_&i=1;

		data nf_lastlevel_&i;
			* non formal enrolments;
			set enrol_;
			keep snz_uid nf_ter_enr_id_at_age_&i nf_ter_enr_ld_at_age_&i nf_ter_enr_lev_at_age_&i;

			if nf_ter_enr_id_at_age_&i=1;

		proc sort data=lastlevel_&i;* all tertiary enrolments;
			by snz_uid descending ter_enr_ld_at_age_&i;

		proc sort data=f_lastlevel_&i;* formal enrolments;
			by snz_uid descending f_ter_enr_ld_at_age_&i;

		proc sort data=nf_lastlevel_&i;
			by snz_uid descending nf_ter_enr_ld_at_age_&i;

		data lastlevel_&i;* all tertiary enrolments;
			set lastlevel_&i;
			by snz_uid descending ter_enr_ld_at_age_&i;

			if first.snz_uid then
				output;
			drop ter_enr_ld_at_age_&i;

		data f_lastlevel_&i;* formal enrolments;
			set f_lastlevel_&i;
			by snz_uid descending f_ter_enr_ld_at_age_&i;

			if first.snz_uid then
				output;
			drop f_ter_enr_ld_at_age_&i;

		data nf_lastlevel_&i;* non formal enrolments;
			set nf_lastlevel_&i;
			by snz_uid descending nf_ter_enr_ld_at_age_&i;

			if first.snz_uid then
				output;
			drop nf_ter_enr_ld_at_age_&i;
		run;

		data consolidated_&i;
			merge lastlevel_&i f_lastlevel_&i nf_lastlevel_&i;
			by snz_uid;
		run;

	%end;
%mend;

%lastlevel;

data project._ind_edu_last_ter_lev_&date.; retain snz_uid ter_enr: f_ter_enr: nf_ter_enr: ;
merge &population(keep=snz_uid DOB) LASTLEVEL_0-LASTLEVEL_24 NF_LASTLEVEL_0-NF_LASTLEVEL_24 F_LASTLEVEL_0-F_LASTLEVEL_24 ; by snz_uid;
run;

proc datasets lib=work kill nolist memtype=data;
quit;