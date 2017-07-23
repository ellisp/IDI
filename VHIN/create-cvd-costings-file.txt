
************************************************************************************************************************************************
****          -This code was written in May 2016 by Sheree Gibb with edits from June Atkinson;                                              ****
****          -Code was written for the Virtual Health Information Network catalyst project with                                            ****
****           Tony Blakely, June Atkinson and Giorgi Khvinzinadze from University of Otago Wellington.                                     ****
****          -The code is the first in a series of programs used to estimate the costs of cardiovascular disease in NZ. The other          ****
****           programs were written by June Atkinson.                                                                                      ****
****          -This program extracts all the relevant data from IDI and organises it in preparation for June's programs.                    ****
****		   Several output files are created for use in June's programs.                                                                 ****
************************************************************************************************************************************************;




%let basepath=\\wprdfs08\Datalab-MA\MAA2015-53 BODE3 and HIRP-led VHIN Research in the IDI;

libname dlab "&basepath\CVD catalyst project";
libname june "&basepath\CVD catalyst project\First run results";
libname moh ODBC dsn=idi_clean_archive_srvprd schema=moh_clean;
libname data ODBC dsn=idi_clean_archive_srvprd schema=data;
libname sandmoh ODBC dsn=idi_sandpit_srvprd schema="clean_read_MOH_Health_Tracker";
libname sand ODBC dsn=idi_sandpit_srvprd schema="DL-MAA2015-53";
libname metadata ODBC dsn=idi_metadata_srvprd schema=clean_read_classifications;

%include "&basepath\CVD catalyst project\SAS formats\SASforHTCancer.sas";
%include "&basepath\CVD catalyst project\SAS formats\SASFormatsforUOWBODE.sas";
%include "&basepath\CVD catalyst project\SAS formats\HealthDataFormats.sas";
%include "&basepath\CVD catalyst project\SAS formats\ifinyr format for datetime.sas";


*********************************************************
***     Extract all relevant health data from IDI     ***
*********************************************************;

	*Rename all relevant dates as 'visit date' os it is easier for merging later;

	*NOTE FROM SHEREE: COULD DELETE UNNECESSARY VARIABLES IN SOME OF THESE TO REDUCE SIZE;

	*First, extract datasets that will be used to identify CVD events, we need those data back to 2001;

		*NMDS events;
		proc sql;
			connect to odbc(dsn="idi_clean_archive_srvprd");
			create table work.nmds_event_raw as 
			select * from connection to odbc
			(select CAST(moh_evt_even_date as DATETIME) as visit_datedt, CAST(moh_evt_evst_date as DATETIME) as evstdatedt, moh_evt_event_id_nbr as event_id, *
			from moh_clean.pub_fund_hosp_discharges_event 
			WHERE moh_evt_even_date>='01JUN1996'
			order by event_id);
			disconnect from odbc;
		quit;

		*NMDS diagnosis;
		*Can't use user-defined formats in the passthrough, so need to extract all diagnoses and then restrict to the ones that match the CVD codes;
		proc sql;
			connect to odbc(dsn="idi_clean_archive_srvprd");
			create table work.nmds_diag_all as 
			select * from connection to odbc
			(select moh_dia_event_id_nbr as event_id, CAST(moh_dia_diag_sequence_code as INT) as moh_dia_diag_sequence_code, moh_dia_clinical_sys_code, moh_dia_submitted_system_code,
			moh_dia_diagnosis_type_code, moh_dia_clinical_code
			from moh_clean.pub_fund_hosp_discharges_diag 
			where moh_dia_diagnosis_type_code='A' 
			order by event_id);
			disconnect from odbc;
		quit;

		*Restrict to diagnoses that match a CVD code. We aren't interested in the others;
		data nmds_diag_raw;
			set nmds_diag_all;
			cvd_code_any=input(substr(moh_dia_clinical_code,1,5),$ianycvd.);
			cvd_code_4=input(substr(moh_dia_clinical_code,1,4),$i4cvd.);
			if strip(cvd_code_4) eq '???' then cvd_code_4=input(substr(moh_dia_clinical_code,1,5),$iothcvd.);
			if cvd_code_any='???' and cvd_code_4='???' then delete;
		run;

		*Pharmaceutical;
		proc sql;
			connect to odbc(dsn="idi_clean_archive_srvprd");
			create table work.pharms_raw as 
			select * from connection to odbc
			(select snz_uid, CAST(moh_pha_dispensed_date as DATETIME) as visit_datedt, moh_pha_domicile_code, moh_pha_dim_form_pack_code, moh_pha_dose_nbr, 
			moh_pha_frequency_nbr, moh_pha_daily_dose_nbr, moh_pha_patent_category_code, moh_pha_funding_dhb_code, moh_pha_nss_flag_code, 
			moh_pha_patient_contrib_exc_gst_amt as moh_pha_patient_contrib_exc_gst, moh_pha_remimburs_cost_exc_gst_amt as moh_pha_remimburs_cost_exc_gst, 
			moh_pha_csc_holder_code, moh_pha_huhc_holder_code, moh_pha_pha_subsidy_card_ind
			from moh_clean.pharmaceutical 
			order by snz_uid, visit_datedt);
			disconnect from odbc;
		quit;


	*Next, extract datasets that will be used for costings only. We only need those from June 2006 onwards;
		*GMS claims;
		proc sql;
			connect to odbc(dsn="idi_clean_archive_srvprd");
			create table work.gms_raw as 
			select * from connection to odbc
			(select snz_uid, CAST(moh_gms_visit_date as DATETIME) as visit_datedt, moh_gms_amount_paid_amt
			from moh_clean.gms_claims 
			WHERE moh_gms_visit_date >= '01JUN2006'
			order by snz_uid, visit_datedt);
			disconnect from odbc;
		quit;

		*Mortality;
		proc sql;
			connect to odbc(dsn="idi_clean_archive_srvprd");
			create table work.mort_raw as 
			select * from connection to odbc
			(select cast(b.moh_clean_death_date as DATETIME) as doddt, a.moh_mor_death_year_nbr, a.moh_mor_birth_year_nbr, a.snz_uid, b.snz_uid as snz_uid_full, a.moh_mor_icd_d_code as cause_of_death, a.moh_mor_ethnic_grp2_snz_ind
			from moh_clean.mortality as a left join data.full_death_date as b on (a.snz_uid=b.snz_uid and b.moh_clean_death_date>='2006-01-01')
			order by snz_uid, doddt);
			disconnect from odbc;
		quit;

		*NNPAC;
		proc sql;
			connect to odbc(dsn="idi_clean_archive_srvprd");
			create table work.nnpac_raw as 
			select * from connection to odbc
			(select snz_uid, CAST(moh_nnp_service_date as DATETIME) as visit_datedt, moh_nnp_attendence_code, moh_nnp_domicile_code, moh_nnp_purchase_unit_code, moh_nnp_volume_amt,
			moh_nnp_service_type_code, moh_nnp_purchaser_code, moh_nnp_unit_of_measure_key
			from moh_clean.nnpac 
			WHERE moh_nnp_service_date>='01JUN2006'
			order by snz_uid, visit_datedt, moh_nnp_service_type_code);
			disconnect from odbc;
		quit;

		*PHO;
		proc sql;
			connect to odbc(dsn="idi_clean_archive_srvprd");
			create table work.pho_raw as 
			select * from connection to odbc
			(select snz_uid, moh_pho_practice_type_code, moh_pho_domicile_code, moh_pho_year_and_quarter_text,
			moh_pho_eth_priority_grp_code, moh_pho_sex_snz_code as sex,
			moh_pho_ethnicity_1_code, moh_pho_ethnicity_2_code, moh_pho_ethnicity_3_code
			from moh_clean.pho_enrolment 
			WHERE moh_pho_last_consul_date>='01JUN2006'
			order by snz_uid, moh_pho_year_and_quarter_text);
			disconnect from odbc;
		quit;

		*Lab claims;
		proc sql;
			connect to odbc(dsn="idi_clean_archive_srvprd");
			create table work.lab_raw as 
			select * from connection to odbc
			(select snz_uid, CAST(moh_lab_visit_date as DATETIME) as visit_datedt, moh_lab_amount_paid_amt
			from moh_clean.lab_claims
			WHERE moh_lab_visit_date>='01JUN2006'
			order by snz_uid, visit_datedt, moh_lab_test_type_code, moh_lab_test_code);
			disconnect from odbc;
		quit;

		*Change all datetimes to dates;
		*I couldn't manage to extract these in correct date format using the passthrough;

		%macro datechange(dataset, dtvar);
			data &dataset;
			set &dataset;
			format &dtvar ddmmyy10.;
			&dtvar=datepart(&dtvar.dt);
			drop &dtvar.dt;
			run;
		%mend datechange;

		%datechange(lab_raw, visit_date);
		%datechange(nmds_event_raw, visit_date);
		%datechange(nmds_event_raw, evstdate);
		%datechange(gms_raw, visit_date);
		%datechange(mort_raw, dod);
		%datechange(pharms_raw, visit_date);
		%datechange(nnpac_raw, visit_date);




****************************************************
***  Get population and demographic information  ***
****************************************************;


	*Create table with health tracker flags and sex, dob for everyone in health tracker;
	proc sql;
		connect to odbc(dsn="idi_clean_archive_srvprd");
		create table work.uids as 
		select * from connection to odbc
		(select distinct snz_uid, snz_moh_uid 
		from security.concordance 
		where snz_moh_uid IS NOT NULL
		order by snz_uid);
		disconnect from odbc;
	quit;


	proc sql;
		create table with_snzuid as
		select b.snz_uid, input(a.pop2006_2007,3.) as pop200607, input(a.pop2007_2008,3.) as pop200708, input(a.pop2008_2009,3.) as pop200809, 
		input(a.pop2009_2010,3.) as pop200910, input(a.pop2010_2011,3.) as pop201011, input(a.pop2011_2012,3.) as pop201112, input(a.pop2012_2013,3.) as pop201213, 
		input(a.pop2013_2014,3.) as pop201314, input(a.pop2014_2015,3.) as pop201415
		from sand.Health_Tracker_pop_201603 as a left join work.uids as b on a.snz_moh_uid=b.snz_moh_uid
		order by b.snz_uid;
	quit;

	*Select the final value from the sets of duplicates;
	*Rules we have decided on are:
		- 1 (resident) vs 0 (no health records) = 1
		- 2 (non-resident) vs 0 (no health records) = 2
		- 1 (resident) vs 2 (non-resident) = 1
		- 0 vs 1 vs 2 = 1;
	*As we are going to recode '2' to '0' anyway, we can just start by recoding '2' to '0' and then take the max value;

	data with_snzuid_recode;
		set with_snzuid;

		*Recode '2' to '0';
		if pop200607=2 then pop200607=0; if pop200708=2 then pop200708=0; if pop200809=2 then pop200809=0; if pop200910=2 then pop200910=0;      
		if pop201011=2 then pop201011=0; if pop201112=2 then pop201112=0; if pop201213=2 then pop201213=0; if pop201314=2 then pop201314=0;
		if pop201415=2 then pop201415=0;

		*There are a few records with no snz_uid (snz_moh_uid in health tracker does not match to a record in IDI);
		*Delete them;
		if snz_uid=. then delete;
		*Delete people who are not residents in any year;
		if sum(of pop200607--pop201415) eq 0 then delete;
	run;

	*Take the maximum;
	proc summary nway data=with_snzuid_recode;
		class snz_uid;
		var pop200607 pop200708 pop200809 pop200910 pop201011 pop201112 pop201213 pop201314 pop201415;
		output out=ht_final (drop=_type_ _freq_) max=;
	quit;

	*Get sex and ethnicity from personal detail table;
	proc sql;
		create table demog_v1 as 
		select b.snz_sex_code, b.snz_ethnicity_grp2_nbr as snz_eth_maori, a.* 
		from ht_final as a left join data.personal_detail as b on a.snz_uid=b.snz_uid
		order by snz_uid;
	quit;

	*Get full date of birth from full birth date table;
	proc sql;
		create table demog as 
		select b.moh_clean_birth_date as dob, a.* 
		from demog_v1 as a left join data.full_birth_date as b on a.snz_uid=b.snz_uid
		order by snz_uid;
	quit;

	*Couple of hundred are missing sex in PD table, and don't have it recorded in NHI either. Drop them;
	data demog_final;
		set demog;
		if snz_sex_code='' then delete;
	run;

	*Get ethnicity, dod, cause of death from mortality for those who have died;
	proc sql;
		create table with_mort_eth as 
		select a.*, b.moh_mor_ethnic_grp2_snz_ind as mort_eth_maori, b.dod, b.cause_of_death
		from demog as a left join mort_raw as b
		on a.snz_uid=b.snz_uid;
	quit;

	*Get NHI ethnicity for everyone;
	proc sql;
		create table with_nhi_eth as 
		select a.*, b.moh_pop_ethnic_grp2_snz_ind as nhi_eth_maori
		from with_mort_eth as a left join moh.pop_cohort_demographics as b
		on a.snz_uid=b.snz_uid;
	quit;


	*Create final ethnicity: use mortality if available, then NHI, then personal detail;
	data june.corepop;
		set with_nhi_eth;
		if mort_eth_maori ne '' then UseEthMaori=mort_eth_maori;
		else if nhi_eth_maori ne '' then UseEthMaori=nhi_eth_maori;
		else if snz_eth_maori ne '' then UseEthMaori=snz_eth_maori;
		else UseEthMaori='';
		drop mort_eth_maori snz_eth_maori nhi_eth_maori;
		rename snz_sex_code=UseSex;
		format dob2 ddmmyy10.;
		dob2=input(dob,anydtdte10.);
		drop dob;
		rename dob2=UseDOB;
		rename dod=UseDOD;
		if dob2='' or snz_sex_code='' or useethmaori='' then delete;
	run;



***************************************************************************************************
***  Calculate total cost per day per person for labs, GMS, pharmaceutical, and NNPAC combined  ***
***************************************************************************************************
	
	* NNPAC costs ;

	*Import spreadsheet with PUCs for each year;
	PROC IMPORT 
		OUT= WORK.PUCPricetouse 
	    DATAFILE= "&basepath\CVD catalyst project\0102 - 1617 Price HistorytoUse.csv" 
	    DBMS=CSV REPLACE;
	    GETNAMES=YES;
	    DATAROW=2;
	    guessingrows=max; 
	RUN;

	data pucpricetouse2001_2016;
		set pucpricetouse(keep=pu price2001 price2002 price2003 price2004 price2005 
		  price2006 price2007 price2008 price2009 price2010
		  price2011 price2012 price2013 price2014 price2015 price2016);
		if substr(pu,1,2) eq 'MH' then delete;
	run;

	/*Make a informat of puccode and finyear to produce price*/
	data pucfinyr(keep=puc finyr pucprice puc_finyr);
		set pucpricetouse2001_2016(rename=(pu=PUC));
		array PUCpricearray {*} price2006 price2007 price2008 price2009 price2010
		  price2011 price2012 price2013 price2014 price2015 price2016;
		array finarr {11} _temporary_ (200607 200708 200809 200910 201011 201112
		  201213 201314 201415 201516 201617);
		length FinYr 8 puc_finyr $15;
		meanprice=mean(price2006, price2007, price2008, price2009, price2010,
		  price2011, price2012, price2013, price2014, price2015, price2016);
		meanall=mean(price2001, price2002, price2003, price2004, price2005,
		  price2006, price2007, price2008, price2009, price2010,
		  price2011, price2012, price2013, price2014, price2015, price2016); 
		do i=1 to dim(PUCpricearray);
		 FinYr=finarr{i};
		 puc_finyr=put(puc,8.)||'_'||put(finyr,6.);
		 PUCPrice=PUCpricearray{i};
		 if pucprice eq 0 and meanprice ne 0 then pucprince=-10;  /*If price for that year is zero but
		                       there is a price for other years then estimate value from inflation or deflation*/
		 else if pucprice eq . and meanprice ne . then pucprice=-20;  /*This will mean that there are prices for
		                       others in the time range. Estiamte value*/
		 else if pucprice eq . and meanall ne . then pucprice=-30;  /*This will mean that there are prices for
		                       others in earlier years. Estimate value*/
		 else if pucprice eq . then pucprice=-99;  /*Have no price*/
		 output;
		end;
	run;

	data dlab.pucfinyrformat(keep=fmtname start end label type min max length default fuzz sexcl eexcl hlo
	  DateFMade puc finyr pucprice puc_finyr);
		length fmtname $8 start end $15 label $16 min max default length 3 fuzz 8
		 type sexcl eexcl $1 hlo $2;
		set pucfinyr;
		retain DateFMade;
		format datefmade datetime.;
		if _n_ eq 1 then DateFMade=datetime();
		type='I';fuzz=0;sexcl='N';hlo='  ';eexcl='N';
		min=1;max=15;default=15;length=15;
		start=puc_finyr;end=start;
		fmtname='ipucfy';
		label=put(pucprice,16.5);
		output;
	run;

	options fmtsearch=(fmtlib work library);

	proc sort data=dlab.pucfinyrformat nodupkey;
		by type fmtname start sexcl eexcl;
	run;

	proc format cntlin=dlab.pucfinyrformat;
	run;

	*Apply format to NNPAC file to get costs;
	data nap_pucs(keep=finyr puc volume moh_nnp_unit_of_measure_key purchaser_code visit_date
	  snz_uid possdelflag pucprice costexcl);
		set nnpac_raw (rename=(moh_nnp_purchase_unit_code=purchase_unit moh_nnp_volume_amt=volume moh_nnp_purchaser_code=purchaser_code));
		length PUC $8 possdelflag 3;
		PUC=substr(left(purchase_unit),1,8);
		FinYr=input(visit_date,ifinyr.);
		possdelflag=.;
		if substr(puc,1,2) eq 'MH' then possdelflag=1;
		else if substr(right(puc),8,1) eq 'A' then possdelflag=2;
		else if volume eq 0 then possdelflag=3; /*Check - there may be some that need counting once per year or month*/
		else if purchaser_code in ('06','08','10','17','19','98','A0','A1','A2','A3','A4','A5','A6','A7')
		  then possdelflag=10; /*Need to check if we want to exclude all of these or not*/
		else possdelflag=0;
		pucprice=input(put(puc,8.)||'_'||put(finyr,6.),?? ipucfy.);
		if pucprice eq . then pucprice=-99;
		if pucprice lt 0 then costexcl=.; else costexcl=pucprice*volume;
		format purchaser_code $fpurch.;
	run;

	data nap_costs;
		set nap_pucs(rename=(costexcl=CostexclActYrNNPAC));
		label CostExclActYrNNPAC="Cost ExGST Actual Year (and not cpi adjusted) dollars (from PUC)";
	run;

	data nap_CostsUse(keep=finyr visit_date snz_uid costexclactyrNNPAC);
		set nap_Costs;
		where finyr ne . and finyr ge 200607 and finyr le 201314;
	run;

	/*End of dealing with NNPAC costs, back to rest of program*/

	*Sum GMS costs per person per day;
	proc summary data=gms_raw chartype nway;
		class snz_uid visit_date;
		vars moh_gms_amount_paid_amt;
		output out=gms_daily_sum (drop=_type_) sum=GMSTotal_amount_paid;
	run;

	*Sum lab costs per person per day;
	proc summary data=lab_raw chartype nway;
		class snz_uid visit_date;
		vars moh_lab_amount_paid_amt;  
		output out=lab_daily_sum (drop=_type_) sum=LABTotal_amount_paid ;
	run;

	*Sum pharmaceutical costs per person per day;
	proc summary data=pharms_raw chartype nway;
		class snz_uid visit_date;
		vars moh_pha_patient_contrib_exc_gst moh_pha_remimburs_cost_exc_gst;
		output out=pharms_daily_totals (drop=_type_) sum(moh_pha_patient_contrib_exc_gst moh_pha_remimburs_cost_exc_gst)=PHARMTotal_Patient PHARMTotal_reimburse;
	run;

	*Create final file with total cost from NNPAC, labs, pharmaceutical and GMS per person per day;
	data daily_totals;
		merge pharms_daily_totals lab_daily_sum gms_daily_sum nap_CostsUse (drop=finyr);
		by snz_uid visit_date;
		CostExclEndYr=SUM(PHARMTotal_Patient, PHARMTotal_reimburse, GMSTotal_amount_paid, LABTotal_amount_paid, costexclactyrNNPAC);
		FinYr=input(visit_date,ifinyr.);
		drop PHARMTotal_Patient PHARMTotal_reimburse GMSTotal_amount_paid LABTotal_amount_paid costexclactyrnnpac _freq_;
		CostExclRefYr=CostExclEndYr*1000/input(finyr,icpi11b.);
	run;

	*Add demographics to the daily cost file;
	proc sql;
		create table june.combined4sourcecosts as
		select a.*, b.usedod, b.usedob, b.useethmaori, b.usesex 
		from daily_totals as a left join june.corepop as b
		on a.snz_uid=b.snz_uid;
		delete from june.combined4sourcecosts where (usedob is null or usesex is null or useethmaori is null);
	quit;




	*************************************
	****     Estimate PHO costs       ***
	*************************************;

	/*	Estimate PHO costs in 2011/12 dollars*/
	/*	Modification of June's basic code for old PHO extracts. IDI doesn't have all the variables we need so */
	/*	  we are just using domicile code, practice type, age, sex and ethnicity. */
	/*	No age on PHO dataset so we need to transfer age from NHI file before starting*/
	/*	Note from June - ideally the next extract will have all the other variables so you can calculate the costs correctly*/

	*Add ages to PHO file;
	proc sql;
		create table pho_with_age as 
		select a.*, b.usedob, b.usesex, b.useethmaori, b.usedod
		from pho_raw as a left join june.corepop as b on a.snz_uid=b.snz_uid;
		delete from pho_with_age where (usedob is null or usesex is null or useethmaori is null);
	quit;

	*Estimate PHO costs;
	*Multipliers come from June's previous work;
	data june.pho_costsuse (keep= snz_uid finyr visit_date costexclpermth_refyr costexclpermth_actyr usedob usesex useethmaori usedod);
		set pho_with_age;
		format visit_date ddmmyy10.;
		visit_date=input(compress(moh_pho_year_and_quarter_text),yyq6.);
		FinYr=input(visit_date,ifinyr.);
		if strip(moh_pho_practice_type_code) eq 'ACCESS' then pract=1; else pract=0;  /*1=Access, 0=Interim*/
		if strip(sex)='2' then fem=1; else fem=0; /*1=female, 0=Not Female*/
		if substr(moh_pho_ethnicity_1_code,1,1) in ('2','3') or substr(moh_pho_ethnicity_2_code,1,1) in ('2','3') or substr(moh_pho_ethnicity_3_code,1,1) in ('2','3')
		 then MaoriPac=1; else MaoriPac=0;
		if usedob ne . then age_quart=yrdif(usedob,visit_date,'AGE'); /*Age at start of quarter*/
		else age_quart=.;
		*Assuming not HUHC and only calculating First Contact not other costs.
		Costs are capitation rates from 1 July 2011, rates excl GST and are annualised so need dividing by 12 to get monthly;
		 CostExclpermth_RefYr=.;CostexclperMth_ActYr=.;
		if age_quart ge 0  and age_quart lt 5  then CostExclpermth_RefYr=( pract*(379.4228*fem+399.4788*(1-fem)) +
		 (1-pract)*(370.2764*fem+394.0248*(1-fem)) + maoripac*(72.9380*fem+76.7928*(1-fem)) )/12;
		else if age_quart ge 5  and age_quart lt 15 then CostExclpermth_RefYr=( pract*(120.1000*fem+112.4156*(1-fem)) +
		 (1-pract)*(95.3308*fem+90.3508*(1-fem)) + maoripac*(23.0868*fem+21.6104*(1-fem)) )/12;
		else if age_quart ge 15 and age_quart lt 25 then CostExclpermth_RefYr=( 110.8208*fem+60.9928*(1-fem) + maoripac*(21.3032*fem+11.7248*(1-fem)) )/12;
		else if age_quart ge 25 and age_quart lt 45 then CostExclpermth_RefYr=( 97.3828*fem+62.9500*(1-fem) + maoripac*(18.7200*fem+12.1012*(1-fem)) )/12;
		else if age_quart ge 45 and age_quart lt 65 then CostExclpermth_RefYr=( 133.3836*fem+99.6236*(1-fem) + maoripac*(25.6404*fem+19.1512*(1-fem)) )/12;
		else if age_quart ge 65                     then CostExclpermth_RefYr=( 229.8604*fem+198.2284*(1-fem) + maoripac*(44.1868*fem+38.1068*(1-fem)) )/12;
		CostExclperMth_ActYr=CostExclpermth_RefYr*input(finyr,icpi11b.)/1000;
		label CostexclperMth_RefYr="Cost ExGST Per Month Ref Year (2011/12) dollars (estimate)"; /*Note Reference year*/
		label CostexclperMth_ActYr="Cost ExGST Per Month Actual Year dollars (estimate converted back from RefYr)";  /*Note Actual year*/
		label visit_date="Date of start of Year Quarter";
	run;



	***************************
	****     NMDS costs     ***
	***************************;

	data NMDS_costs (keep= snz_uid visit_date totallos misscostwgt CostExclEndYr CostPerDayActYr Totallos CostExclRefYr CostExclEndYr CostPerDayRefYr CostPerDayEndYr
	 CostExclEndYrCasemx CostPerDayEndYrCasemx evstdate finyr finyrstart finyrend);
		set nmds_event_raw (rename=(moh_evt_cost_weight_amt=cost_weight));
		where visit_date ge '01Jun2006'd;

		FinYr=input(visit_date,ifinyr.);

		length TotalLOS 4;
		TotalLOS=moh_evt_los_nbr;
		if cost_weight eq . then MissCostWgt=1; else MissCostWgt=0;
		FinYrEnd=input(visit_date,ifinyr.);
		FinYrStart=input(evstdate,ifinyr.);
		cpiadjust=1000/input(FinYrEnd,icpi11b.);

		CostExclEndYr=cost_weight*input(finyrend,imedsurpu.)*cpiadjust;  /*Use finyrend dollar values and CPI adjust*/
		CostExclActYr=cost_weight*input(finyr,imedsurpu.);  /*Use actual finyr dollar values for the analyses*/
		CostExclRefYr=cost_weight*input(201112,imedsurpu.);  /*If use 2011/12 finyr dollar values for all years*/

		if TotalLOS in (.,0,1) then CostPerDayRefYr=CostExclRefYr; else CostPerDayRefYr=CostExclRefYr/totallos;
		if TotalLOS in (.,0,1) then CostPerDayEndYr=CostExclEndYr; else CostPerDayEndYr=CostExclEndYr/totallos;
		if TotalLOS in (.,0,1) then CostPerDayActYr=CostExclActYr; else CostPerDayActYr=CostExclActYr/totallos;

		CostExclEndYrCasemx=CostExclEndYr; CostPerDayEndYrCasemx=CostPerDayEndYr;
		if upcase(pur_unit) eq 'EXCLU' then do; CostExclEndYrCasemx=0; CostPerDayEndYrCasemx=0; end;

		label CostExclRefYr="Cost ExGST Ref Year (2011/12) dollars (From CostWeights)"
		 CostExclEndYr="Cost ExGST Each Year dollars (From CostWeights) CPI adjusted"
		 CostPerDayRefYr="Cost ExGST Per Day Ref Year (2011/12) dollars"
		 CostPerDayEndYr="Cost ExGST Per Day FinYrEnd Year dollars CPI adjusted" /*Use these as alternative costs*/
		 CostExclEndYrCasemx="Cost ExGST Each Year dollars (From CostWeights) CPI adj. (Casemix Costs Only)"
		 CostPerDayEndYrCasemx="Cost ExGST Per Day FinYrEnd Year dollars CPI adj (Casemix Costs Only)"  /*Use these as main costs*/
		 FinYrEnd="Financial Year of End of event"
		 FinYrStart="Financial Year of Start of event";
	run;

	*Add demographics to NMDS costs file;
	proc sql;
		create table june.nmds_CostsUse as
		select a.snz_uid, a.visit_date, a.costexclrefyr, a.costexclendyr, a.costperdayrefyr, a.costperdayactyr, a.costexclendyrcasemx,
		a.costperdayendyrcasemx, a.totallos, a.evstdate, a.costperdayendyr, a.finyrend, a.finyrstart, b.usedod, b.usedob, b.useethmaori, b.usesex 
		from nmds_costs as a left join june.corepop as b
		on a.snz_uid=b.snz_uid
		where finyrend ne . and finyrend ge 200607 and finyrstart le 201314;
		delete from june.nmds_CostsUse where (usedob is null or usesex is null or useethmaori is null);
	quit;


	*There are some people with two entries (with different costs) for the same start and end dates.
	*Leave them as two separate entries and they will be costed separately;



****************************************
*     Identify and flag CVD events     ;
****************************************


	*Transfer diagnoses to main NMDS event file;
	proc sql;
		create table cvd_events as
		select evstdate as diagnosis_date format ddmmyy10., *
		from nmds_event_raw as a inner join nmds_diag_raw as b on a.event_id=b.event_id;
	quit;


	*Angina medications;

	*Get a list of all dim_form_pack_subsidy_code values for Glyceryl trinitrate (chemical ID 1577), Isosorbide dinitrate (2377),
	Isosorbide mononitrate (2836), Nicorandil (1272), Perhexiline maleate (1949);

	data angina_codes;
		set metadata.moh_dim_form_pack_subsidy_code (where=(chemical_id in(1577 2377 2836 1272 1949)));
		*There are two preparations that are not for cardiovascular disease, drop them;
		if dim_form_pack_subsidy_key in(79992 79935) then delete;
		dim_form_pack_code=strip(put(dim_form_pack_subsidy_key, 8.));
	run;

	*Extract all prescriptions for those codes;
	proc sql;
		create table angina_prescriptions as
		select b.snz_uid, b.snz_moh_uid, b.moh_pha_dispensed_date, b.moh_pha_quan_presc_nbr, b.moh_pha_quan_disp_nbr, a.*
		from angina_codes as a inner join moh.pharmaceutical as b on a.dim_form_pack_code=b.moh_pha_dim_form_pack_code
		/*restrict to a sample for testing*/
		order by snz_uid, moh_pha_dispensed_date;
	quit;


	*Flag individuals with 2 or more dispensings in a 12 month period;
	*Calculate time between dispensings;
	data with_time;
		set angina_prescriptions;
		by snz_uid;
		disp_date=input(moh_pha_dispensed_date, yymmdd10.);
		format disp_date ddmmyy10.;
		last_date=lag(disp_date);
		time_since_last=disp_date-last_date;
		if first.snz_uid then time_since_last=.;
		if time_since_last le 365 and time_since_last ne . and time_since_last ne 0 then repeat=1;
		else repeat=0;
		format first_presc ddmmyy10.;
		if repeat=1 then first_presc=last_date;
	run;

	*List all pairs where the gap was less than 12 months;
	proc sql;
		create table angina_list as
		select distinct snz_uid, snz_moh_uid, first_presc
		from with_time
		where repeat=1
		order by snz_uid, first_presc;
	quit;

	*Select the first instance where the gap was less than 12 months;
	data final_angina_list;
		set angina_list;
		by snz_uid;
		if first.snz_uid then keep=1;
		if keep ne 1 then delete;
		drop keep;
		pharms_angina_flag=1;
		rename first_presc=diagnosis_date;
		*Code all angina cases identified in this way as 'AngM' in the cvd4 codes and 'ACVD' in the any CVD codes;
		cvd_code_4='AngM';
		cvd_code_any='ACVD';
	run;

	*Combine angina and CVD events;
	data cvd_all;
		set cvd_events final_angina_list;
	run;

	proc sort data=cvd_all;
		by snz_uid diagnosis_date;
	run;

	data cvd_look_back;
		set cvd_all;
		last_uid=lag(snz_uid);
		if snz_uid ne last_uid then diag=1;
		*Delete events that aren't a first diagnosis since 2001;
		if diag ne 1 then delete;
		*Delete events before July 2006;
		if diagnosis_date ge '01JUL2006'd then delete;
		drop last_uid event_id snz_moh_uid pharms_angina_flag diag;
	run;

	*Add demographics, date and cvd death flag for people with CVD diagnoses;
	proc sql;
		create table cvd_cases as
		select a.*, b.usedod, b.usedob, b.useethmaori, b.usesex, b.cause_of_death
		from cvd_look_back as a left join june.corepop as b
		on a.snz_uid=b.snz_uid;
		delete from cvd_cases where (usedob is null or usesex is null or useethmaori is null);
	quit;

	*Convert icd codes for death causes to cvd groups using June's format;
	data june.cvd_final_list;
		set cvd_cases;
		cvd_death_any=input(substr(cause_of_death,1,5),$ianycvd.);
		cvd_death_4=input(substr(cause_of_death,1,4),$i4cvd.);
		if strip(cvd_death_4) eq '???' then cvd_death_4=input(substr(cause_of_death,1,5),$iothcvd.);
		if usedod='' then do;
		 cvd_death_any='';
		 cvd_death_4='';
		end;
		keep snz_uid usesex usedob useethmaori usedod cause_of_death diagnosis_date moh_dia_clinical_code cvd_code_4 cvd_code_any 
		cvd_death_any cvd_death_4; 
	run;


