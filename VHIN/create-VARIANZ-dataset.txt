*Created by Sheree Gibb and Jinfeng Zhao in June 2016;
*This code takes the VARIANZ population file and adds demographic, geographic and health variables;

libname dlab "\\wprdfs08\Datalab-MA\MAA2016-11 VIEW-IDI";
libname dlab2 "\\wprdfs08\Datalab-MA\MAA2016-11 VIEW-IDI\UOA_DATA";
libname moh ODBC dsn=idi_clean_20160224_srvprd schema=moh_clean;
libname cen ODBC dsn=idi_clean_20160224_srvprd schema=cen_clean;
libname data ODBC dsn=idi_clean_20160224_srvprd schema=data;
libname sand ODBC dsn=idi_sandpit_srvprd schema="DL-MAA2016-11";
libname sandir ODBC dsn=idi_sandpit_srvprd schema=clean_read_IR_restrict;
libname metadata ODBC dsn=IDI_Metadata_srvprd schema=clean_read_CLASSIFICATIONS;
libname dia ODBC dsn=idi_clean_20160224_srvprd schema=dia_clean;


********************************************************
********************************************************
*              DEMOGRAPHIC VARIABLES                   *
********************************************************
********************************************************
	

		*Resident status for each year of followup;

			
		%macro overseasdays(year);

		proc sql;
			create table overseas_spells_&year. as
			select unique snz_uid, pos_applied_date, pos_ceased_date, pos_day_span_nbr
			from data.person_overseas_spell
			where pos_applied_date < "31DEC&year.:00:00:00.000"dt and pos_ceased_date >= "01JAN&year.:00:00:00.000"dt and snz_uid in (select unique snz_uid from dlab.varianz_pop)
			order by snz_uid, pos_applied_date;
		quit;


		***calculate time spent overseas in each year for each spell;
		data overseas_spells_&year.;
		   	set overseas_spells_&year.;

		   	if pos_ceased_date >= "31DEC&year.:00:00:00.000"dt and pos_applied_date < "01JAN&year.:00:00:00.000"dt 
		      	then time_overseas_&year. = 365;

		   	else if pos_ceased_date >= "31DEC&year.:00:00:00.000"dt
		      then time_overseas_&year. = ("31DEC&year.:00:00:00.000"dt - pos_applied_date) / 86400;

		   else if pos_ceased_date < "31DEC&year.:00:00:00.000"dt and pos_applied_date >= "01JAN&year.:00:00:00.000"dt 
		      then time_overseas_&year. = pos_day_span_nbr;

		   else if pos_ceased_date <= "31DEC&year.:00:00:00.000"dt and pos_applied_date < "01JAN&year.:00:00:00.000"dt 
		      then time_overseas_&year. = (pos_ceased_date - "01JAN&year.:00:00:00.000"dt) / 86400;
		run;


		***sum days spent overseas per year for each person ***;
		proc sql;
		   create table overseas_spells_&year._sum as select snz_uid, ROUND(SUM(time_overseas_&year.), 1) as days_overseas_&year.
		   from overseas_spells_&year.
		   group by snz_uid;
		quit;


		***replace missing with 0 days overseas***
		*create flags for residence in each year;
		data resident_&year.;
		  	set overseas_spells_&year._sum;
		if days_overseas_&year. =. then days_overseas_&year. = 0;
		if days_overseas_&year.>183 then resident_&year.=0;
		else resident_&year.=1;	
		keep snz_uid resident_&year.;
		run;

		%mend;

		%overseasdays(2013);
		%overseasdays(2014);
		%overseasdays(2015);

		*Merge files from all years;
		proc sql;
		create table resident_final as
		select *
		from resident_2013 as a full join resident_2014 as b on a.snz_uid=b.snz_uid
		full join resident_2015 as c on b.snz_uid=c.snz_uid
		order by resident_2013 descending, resident_2014 descending, resident_2015 descending;
		quit;

		*Transfer resident flags to VARIANZ population file;
		proc sql;
		create table varianz_resident as
		select a.*, b.resident_2013, b.resident_2014, b.resident_2015
		from dlab.varianz_pop as a left join resident_final as b on a.snz_uid=b.snz_uid;
		quit;



	*AGE and SEX are already on the population file;
	
	
	****   ETHNICITY   ****;

	*Get ethnicity from census if available, otherwise from health, otherwise from personal detail table;


	*Extract ethnicity info from census;
	proc sql;
		create table ethnicity_cen as
		select unique a.*, input(b.cen_ind_maori_eth_ind_code, 2.0) as cen_maori, input(b.cen_ind_pac_islnd_eth_ind_code, 2.0) as cen_pacific,
		input(b.cen_ind_asian_eth_ind_code, 2.0) as cen_asian, input(b.cen_ind_melaa_eth_ind_code, 2.0) as cen_melaa, 
	    input(b.cen_ind_otr_eth_ind_code, 2.0) as cen_othereth, input(b.cen_ind_eur_eth_ind_code, 2.0) as cen_euro, 
		b.cen_ind_eth_rand6_grp1_code as cen_ethgrp1, b.cen_ind_eth_rand6_grp2_code as cen_ethgrp2, 
		b.cen_ind_eth_rand6_grp3_code as cen_ethgrp3, b.cen_ind_eth_rand6_grp4_code as cen_ethgrp4, 
		b.cen_ind_eth_rand6_grp5_code as cen_ethgrp5, b.cen_ind_eth_rand6_grp6_code as cen_ethgrp6 
		from varianz_resident as a left join cen.census_individual as b on a.snz_uid=b.snz_uid;
	quit;

	*Extract ethnicity from health;
	proc sql;
		create table ethnicity_moh as
		select a.*, b.moh_pop_ethnicity_1_code as moh_eth1, b.moh_pop_ethnicity_2_code as moh_eth2, b.moh_pop_ethnicity_3_code as moh_eth3, 
		b.moh_pop_ethnic_grp1_snz_ind as moh_euro, b.moh_pop_ethnic_grp2_snz_ind as moh_maori, b.moh_pop_ethnic_grp3_snz_ind as moh_pacific, 
		b.moh_pop_ethnic_grp4_snz_ind as moh_asian, b.moh_pop_ethnic_grp5_snz_ind as moh_melaa, b.moh_pop_ethnic_grp6_snz_ind as moh_other
		from ethnicity_cen as a left join moh.pop_cohort_demographics as b on a.snz_uid=b.snz_uid;
	quit;

	*Extract ethnicity from personal detail;
	proc sql;
		create table ethnicity_pd as
		select a.*, b.snz_ethnicity_grp1_nbr as pd_euro, b.snz_ethnicity_grp2_nbr as pd_maori, b.snz_ethnicity_grp3_nbr as pd_pacific,
		b.snz_ethnicity_grp4_nbr as pd_asian, b.snz_ethnicity_grp5_nbr as pd_melaa, b.snz_ethnicity_grp6_nbr as pd_other
		from ethnicity_moh as a left join data.personal_detail as b on a.snz_uid=b.snz_uid;
	quit;


	**Recode ethnicity variables, create flags for Indian and Chinese, create prioritised ethnicity**;
	data eth;
		set ethnicity_pd;

	*Recode census ethnicities;
		if (cen_maori=2) then cen_maori = 1;
		else if cen_maori>2 then cen_maori=.;

		if (cen_pacific=2) then cen_pacific = 1;
		else if cen_pacific>2 then cen_pacific=.;

		if (cen_asian=2) then cen_asian = 1;
		else if cen_asian>2 then cen_asian=.;

		if (cen_melaa=2) then cen_melaa = 1;
		else if cen_melaa>2 then cen_melaa=.;

		if (cen_euro=2) then cen_euro = 1;
		else if cen_euro>2 then cen_euro=.;

		if (cen_othereth=2) then cen_othereth = 1;
		else if cen_othereth>2 then cen_othereth=.;

		*Create flags for Indian and Chinese ethncities from census;
		if cen_ethgrp1="" then cen_chinese=.;
		else if (cen_ethgrp1 >= "42000" and cen_ethgrp1 < "43000") then cen_chinese = 1;
		else if (cen_ethgrp2 >= "42000" and cen_ethgrp2 < "43000") then cen_chinese = 1;
		else if (cen_ethgrp3 >= "42000" and cen_ethgrp3 < "43000") then cen_chinese = 1;
		else if (cen_ethgrp4 >= "42000" and cen_ethgrp4 < "43000") then cen_chinese = 1;
		else if (cen_ethgrp5 >= "42000" and cen_ethgrp5 < "43000") then cen_chinese = 1;
		else if (cen_ethgrp6 >= "42000" and cen_ethgrp6 < "43000") then cen_chinese = 1;
		else cen_chinese = 0;

		if cen_ethgrp1="" then cen_indian=.;
		else if (cen_ethgrp1 >= "43000" and cen_ethgrp1 < "44000") then cen_indian = 1;
		else if (cen_ethgrp2 >= "43000" and cen_ethgrp2 < "44000") then cen_indian = 1;
		else if (cen_ethgrp3 >= "43000" and cen_ethgrp3 < "44000") then cen_indian = 1;
		else if (cen_ethgrp4 >= "43000" and cen_ethgrp4 < "44000") then cen_indian = 1;
		else if (cen_ethgrp5 >= "43000" and cen_ethgrp5 < "44000") then cen_indian = 1;
		else if (cen_ethgrp6 >= "43000" and cen_ethgrp6 < "44000") then cen_indian = 1;
		else cen_indian = 0;
		
		*Create flags for Indian and Chinese ethnicity from MoH;
		if (moh_eth1='' and moh_eth2='' and moh_eth3='') then moh_chinese=.;
		else if (moh_eth1='42' or moh_eth2='42' or moh_eth3='42') then moh_chinese=1;
		else moh_chinese=0;

		if (moh_eth1='' and moh_eth2='' and moh_eth3='') then moh_indian=.;
		else if (moh_eth1='43' or moh_eth2='43' or moh_eth3='43') then moh_indian=1;
		else moh_indian=0;

		*Create final ethnicities by using census if available, if not use MoH, if not use personal detail (Indian and Chinese not available from personal detail);
		if cen_maori ne . then do;
			maori_eth=cen_maori;
			pacific_eth=cen_pacific;
			asian_eth=cen_asian;
			euro_eth=cen_euro;
			melaa_eth=cen_melaa;
			other_eth=cen_othereth;
			indian_eth=cen_indian;
			chinese_eth=cen_chinese;
		end;

		else if cen_maori=. and moh_maori ne . then do;
			maori_eth=moh_maori;
			pacific_eth=moh_pacific;
			asian_eth=moh_asian;
			euro_eth=moh_euro;
			melaa_eth=moh_melaa;
			other_eth=moh_other;
			indian_eth=moh_indian;
			chinese_eth=moh_chinese;
		end;

		else if cen_maori=. and moh_maori=. then do;
			maori_eth=pd_maori;
			pacific_eth=pd_pacific;
			asian_eth=pd_asian;
			euro_eth=pd_euro;
			melaa_eth=pd_melaa;
			other_eth=pd_other;
			indian_eth=.;
			chinese_eth=.;
		end;

		*Create prioritised ethnicity from final ethnicities;

			format pri_eth $12.;
			if maori_eth=. then pri_eth="";
			else if maori_eth = 1 then pri_eth = "Maori";
			else if pacific_eth = 1 then pri_eth = "Pacific";	
			else if (asian_eth = 1 and chinese_eth = 1) then pri_eth = "Chinese";
			else if (asian_eth = 1 and indian_eth = 1) then pri_eth = "Indian";
			else if asian_eth = 1 then pri_eth = "Other Asian";
			else pri_eth = "Other";
			**Assign pacific indian to "Indian" for pri_eth **;
			if (maori_eth = 0 and pacific_eth=  1 and asian_eth=  1 and indian_eth= 1) then pri_eth = "Indian";

		drop cen_ethgrp1 cen_ethgrp2 cen_ethgrp3 cen_ethgrp4 cen_ethgrp5 cen_ethgrp6 moh_eth1 moh_eth2 moh_eth3
				cen_asian cen_euro cen_maori cen_melaa cen_othereth cen_pacific cen_indian cen_chinese
				moh_asian moh_euro moh_maori moh_melaa moh_other moh_pacific moh_indian moh_chinese
				pd_asian pd_euro pd_maori pd_melaa pd_other pd_pacific;

	run;




********************************************************
********************************************************
*               GEOGRAPHIC VARIABLES                   *
********************************************************
********************************************************;

	*Get latest address from address notification table;

	*Using SQL mgmt server (SAS is too slow) I have already created a subset of address notification table 
	with "MSDP" type addresses removed, and updates after 31 Dec 2012 removed;
	*Subset is called "address_subset" in the sandpit;

	*Link the address subset with the IDIERP;
	proc sql;
		create table addr2012 as
		select distinct a.*, b.ant_address_source_code as addr_source, b.ant_meshblock_code as MB13, input(b.ant_notification_date,anydtdte10.) as addr_date_updated format ddmmyy10.
		from eth as a left join sand.address_subset as b on a.snz_uid=b.snz_uid
		order by snz_uid, addr_date_updated descending;
	quit;

	**select the most recently updated addresses**;
	data final_address;
	   set addr2012;
	   by snz_uid;
	   if first.snz_uid;

	   *create a numeric version of mb13 for merging with concordance files;
	   mb13_num=input(mb13, 10.0);
	run;

	/**Check how many addresses come from each source, and how many are missing;*/
	/*proc freq data=final_address;*/
	/*tables addr_source;*/
	/*run;*/

	*Merge with concordances to get DHB and NZDEP;
	*There are about 70000 people with meshblocks that don't match NZDEP or DHB- are they old meshblocks? Or newer ones?;
	proc sql;
	create table addr_nzdep as 
	select a.*, b.depindex2013 as nzdep2013, b.depscore2013
	from final_address as a left join metadata.depindex2013 as b on a.mb13=b.meshblock2013;
	quit;
	
	proc sql;
	create table addr_dhb as
	select a.*, b.dhb_code, b.dhb_label, b.regc2013_code as region, b.regc2013_label as region_name, b.ta2013_code as TA13, b.ta2013_label as ta_name,
    put(b.mb2006_code,z7.) as mb2006_code   
	from addr_nzdep as a left join dlab2.areas_file_2013 as b on a.mb13_num=b.mb2013_code;
	quit;


**************	*Check that there are not many-to-one matches here;
	proc sql;
	create table addr_nzdep06 as 
	select a.*, b.depindex2006 as nzdep2006, b.depscore2006 
	from addr_dhb as a left join metadata.depindex2006 as b on a.mb2006_code=b.meshblock2006
	order by nzdep2013 descending;
	quit;



********************************************************************
********************************************************************
*               SMOKING, INCOME, OCCUPATION AND QUALS              *
********************************************************************
********************************************************************;


	*Smoking, income, occupation and quals from census;
	proc sql;
	create table varianz_cen as 
	select a.*, b.cen_ind_std_highest_qual_code as cen_highest_qual, b.cen_ind_occupation2006_code as cen_occ_anzsco2006, b.cen_ind_smoke_regular_code as cen_smoke_regular, b.cen_ind_smoke_ever_code as cen_smoke_ever,
	b.cen_ind_ttl_inc_code
	from addr_nzdep06 as a left join cen.census_individual as b on a.snz_uid=b.snz_uid;
	quit;

	*Income from Inland Revenue;
	proc sql;
	create table ir_income as 
	select a.snz_uid, b.inc_cal_yr_tot_yr_amt
	from varianz_cen as a left join sandir.ems_cal_yr_summary as b 
	on a.snz_uid=b.snz_uid and b.ir_inc_year_nbr = 2012;
	quit;

	*Sum across different income types to get total annual income for each person;
	proc summary nway data=ir_income;
	class snz_uid;
	var inc_cal_yr_tot_yr_amt;
	output out=ir_income_sum (drop=_type_ _freq_) sum=;
	run;

	*Merge total annual incomes back into varianz file;
	proc sql;
	create table varianz_income as
	select *
	from varianz_cen as a left join ir_income_sum as b on a.snz_uid=b.snz_uid;
	quit;


	*Recode census bands and group IR income into census bands;
	*Recode occupation residual categories to missing;
	data varianz_income_final;
	set varianz_income;
	
	format cen_income $18.;	
	if cen_ind_ttl_inc_code = "11" then cen_income = "Loss";
	else if cen_ind_ttl_inc_code = "12" then cen_income = "Zero income";
	else if cen_ind_ttl_inc_code = "13" then cen_income = "$1-$5,000";
	else if cen_ind_ttl_inc_code = "14" then cen_income = "$5,001-$10,000";
	else if cen_ind_ttl_inc_code = "15" then cen_income = "$10,001-$15,000";
	else if cen_ind_ttl_inc_code = "16" then cen_income = "$15,001-$20,000";
	else if cen_ind_ttl_inc_code = "17" then cen_income = "$20,001-$25,000";
	else if cen_ind_ttl_inc_code = "18" then cen_income = "$25,001-$30,000";
	else if cen_ind_ttl_inc_code = "19" then cen_income = "$30,001-$35,000";
	else if cen_ind_ttl_inc_code = "20" then cen_income = "$35,001-$40,000";
	else if cen_ind_ttl_inc_code = "21" then cen_income = "$40,001-$50,000";
	else if cen_ind_ttl_inc_code = "22" then cen_income = "$50,001-$60,000";
	else if cen_ind_ttl_inc_code = "23" then cen_income = "$60,001-$70,000";
	else if cen_ind_ttl_inc_code = "24" then cen_income = "$70,001-$100,000";
	else if cen_ind_ttl_inc_code = "25" then cen_income = "$100,001-$150,000";
	else if cen_ind_ttl_inc_code = "26" then cen_income = "$150,001 or More";
	else if cen_ind_ttl_inc_code = "99" then cen_income = "Not stated";

   	format income $18.;
	if cen_income ne '' then income=cen_income;
	else if cen_income='' and inc_cal_yr_tot_yr_amt ne . then do;
		if inc_cal_yr_tot_yr_amt<0 then income = "Loss";
		else if inc_cal_yr_tot_yr_amt=0 then income = "Zero income";
		else if (inc_cal_yr_tot_yr_amt >=1 and inc_cal_yr_tot_yr_amt <= 5000) then income = "$1-$5,000";
		else if (inc_cal_yr_tot_yr_amt >=5001 and inc_cal_yr_tot_yr_amt <= 10000) then income = "$5,001-$10,000";
		else if (inc_cal_yr_tot_yr_amt >=10001 and inc_cal_yr_tot_yr_amt <= 15000) then income = "$10,001-$15,000";
		else if (inc_cal_yr_tot_yr_amt >=15001 and inc_cal_yr_tot_yr_amt <= 20000) then income = "$15,001-$20,000";
		else if (inc_cal_yr_tot_yr_amt >=20001 and inc_cal_yr_tot_yr_amt <= 25000) then income = "$20,001-$25,000";
		else if (inc_cal_yr_tot_yr_amt >=25001 and inc_cal_yr_tot_yr_amt <= 30000) then income = "$25,001-$30,000";
		else if (inc_cal_yr_tot_yr_amt >=30001 and inc_cal_yr_tot_yr_amt <= 35000) then income = "$30,001-$35,000";
		else if (inc_cal_yr_tot_yr_amt >=35001 and inc_cal_yr_tot_yr_amt <= 40000) then income = "$35,001-$40,000";
		else if (inc_cal_yr_tot_yr_amt >=40001 and inc_cal_yr_tot_yr_amt <= 50000) then income = "$40,001-$50,000";
		else if (inc_cal_yr_tot_yr_amt >=50001 and inc_cal_yr_tot_yr_amt <= 60000) then income = "$50,001-$60,000";
		else if (inc_cal_yr_tot_yr_amt >=60001 and inc_cal_yr_tot_yr_amt <= 70000) then income = "$60,001-$70,000";
		else if (inc_cal_yr_tot_yr_amt >=70001 and inc_cal_yr_tot_yr_amt <= 100000) then income = "$70,001-$100,000";
		else if (inc_cal_yr_tot_yr_amt >=100001 and inc_cal_yr_tot_yr_amt <= 150000) then income = "$100,001-$150,000";
		else if (inc_cal_yr_tot_yr_amt >=150001) then income = "$150,001 or More";
	end;
	else if cen_income='' and inc_cal_yr_tot_yr_amt=. then income='';
	
	*Add a flag for income source;
	if cen_income ne '' then income_source='CENSUS';
   	else if (cen_income='' and inc_cal_yr_tot_yr_amt ne .) then income_source='IRD';
   	else income_source='';

	*Code missing 2006 meshblocks correctly;
	if mb2006_code<'0000000' then mb2006_code='';

	*Recode census occupation residual categories (99----) to missing;
	if cen_occ_anzsco2006 in('997000' '999000' '999999') then cen_occ_anzsco2006='';

	*Recode census highest quals residual categories (9-) to missing;
	if cen_highest_qual in('94' '95' '96' '97' '98' '99') then cen_highest_qual='';

	*Recode census smoking residual categories (7 and 9) to missing, and recode '2' to '0';
	if cen_smoke_ever in('7' '9') then cen_smoke_ever='';
	if cen_smoke_ever='2' then cen_smoke_ever='0';
	if cen_smoke_regular in('7' '9') then cen_smoke_regular='';
	if cen_smoke_regular='2' then cen_smoke_regular='0';

	drop cen_ind_ttl_inc_code inc_cal_yr_tot_yr_amt cen_income mb13_num;
	run;


********************************************************
********************************************************
*              DATE OF LAST HEALTH CONTACT             *
********************************************************
********************************************************;


		*Get date of last health contact from NNPAC, NMDS, or PHO;
		*Get the last consultation date from any of those sources prior to 31 Dec 2012;

		*PHO;
		proc sql;
		create table lc_pho as
		select a.snz_uid, b.moh_pho_last_consul_date as date, 'PHO' as flag length 6
		from dlab.varianz_pop as a left join moh.pho_enrolment as b 
		on a.snz_uid=b.snz_uid and moh_pho_last_consul_date <= '2012-12-31'
		order by snz_uid, moh_pho_last_consul_date descending;
		quit;

		*Delete everything except the most recent visit;
		data lc_pho2;
		set lc_pho;
		if snz_uid ne lag(snz_uid) then first=1;
		else first=0;
		if first ne 1 then delete;
		drop first;
		run;

		*NMDS;
		proc sql;
		create table lc_nmds as 
		select a.snz_uid, b.moh_evt_even_date as date, 'NMDS' as flag length 6
		from dlab.varianz_pop as a left join moh.pub_fund_hosp_discharges_event as b 
		on a.snz_uid=b.snz_uid and b.moh_evt_even_date<='2012-12-31'
		order by snz_uid, moh_evt_even_date descending;
		quit;

		*Delete everything except the most recent visit;
		data lc_nmds2;
		set lc_nmds;
		if snz_uid ne lag(snz_uid) then first=1;
		else first=0;
		if first ne 1 then delete;
		drop first;
		run;

		*NNPAC;
		proc sql;
		create table lc_nnpac as
		select a.snz_uid, b.moh_nnp_service_date as date, 'NNPAC' as flag length 6
		from dlab.varianz_pop as a left join moh.nnpac as b 
		on a.snz_uid=b.snz_uid and moh_nnp_service_date <= '2012-12-31'
		order by snz_uid, moh_nnp_service_date descending;
		quit;

		*Delete everything except the most recent visit;
		data lc_nnpac2;
		set lc_nnpac;
		if snz_uid ne lag(snz_uid) then first=1;
		else first=0;
		if first ne 1 then delete;
		drop first;
		run;

		*Combine dates from PHO, NMDS and NNPAC;
		*Can replace this with SQL so it doesn't need to be sorted?;
		data all_dates;
		set lc_pho2 lc_nmds2 lc_nnpac2;
		run;

		proc sort data=all_dates;
		by snz_uid descending date;
		run;

		*Delete everything except the most recent visit;
		data final_dates;
		set all_dates;
		if snz_uid ne lag(snz_uid) then first=1;
		else first=0;
		if first ne 1 then delete;
		drop first;
		if date='' then flag='';
		run;

		*Merge latest dates back into VARIANZ dataset;
		proc sql;
		create table dlab.varianz_date_lc as
		select a.*, input(b.date,anydtdte10.) as last_contact_date format ddmmyy10., b.flag as last_contact_flag 
		from varianz_income_final as a left join final_dates as b on a.snz_uid=b.snz_uid;
		quit;





********************************************************
********************************************************
*               PHARMACEUTICAL DISPENSING              *
********************************************************
********************************************************;


		*Extract all prescriptions from 1 July 2012 (6 mths prior to reference date, for baseline dispensing) 
		to end of 2014 (most recent dates available) for people in VARIANZ population;
		proc sql;
		create table varianz_pharms as
		select a.*, b.moh_pha_dispensed_date, input(b.moh_pha_dim_form_pack_code, 8.) as moh_pha_dim_form_pack_code
		from dlab.varianz_pop as a left join moh.pharmaceutical as b
		on a.snz_uid=b.snz_uid and b.moh_pha_dispensed_date>='2012-07-01';
		quit;

		*Apply pharm group codes to all prescriptions;
		proc sql;
		create table varianz_pharm_flags as
		select a.*, b.Aspirin_low_dose, b.Aspirin_any_dose, b.Clopidogrel, b.Dipyridamole, b.Prasugrel, b.Ticagrelor, b.Ticlopidine, b.Warfarin, b.Other_anticoagulant, b.ACEI,
		b.ARB, b.Thiazide, b.Beta_blocker, b.Calcium_channel_blocker, b.Alphablocker, b.Spironolactone, b.Other_AHT, b.Statin, b.Other_lipid_lowering, b.Loop_diuretic,
		b.Metolazone, b.Nitrate, b.Other_antianginal, b.Nonaspirin_NSAIDS, b.Corticosteroids, b.Proton_pump_inhibitors, b.H2_blockers, b.H_pylori_eradication,
		b.Diabetes, b.SSRI
		from varianz_pharms as a left join dlab2.dim_form_pack_subsidy_10Jun16 as b
		on a.moh_pha_dim_form_pack_code=b.dim_form_pack_subsidy_key ;
		quit;


		*Create flags for dispensing at least once in each six-month period from 1 July 2012 to 31 Dec 2014;

		%macro pharms6mth(startdate, enddate, period);

		*Summarise flags per person for 6-month period;
		proc summary nway data=varianz_pharm_flags;
		class snz_uid;
		var Aspirin_low_dose Aspirin_any_dose Clopidogrel Dipyridamole Prasugrel Ticagrelor Ticlopidine Warfarin Other_anticoagulant ACEI
		ARB Thiazide Beta_blocker Calcium_channel_blocker Alphablocker Spironolactone Other_AHT Statin Other_lipid_lowering Loop_diuretic
		Metolazone Nitrate Other_antianginal Nonaspirin_NSAIDS Corticosteroids Proton_pump_inhibitors H2_blockers H_pylori_eradication
		Diabetes SSRI;
		where moh_pha_dispensed_date>="&startdate." and moh_pha_dispensed_date<="&enddate.";
		output out=pharm_flag_&period (drop=_type_ _freq_) max=;
		run;

		data pharm_flag_&period._recode;
		set pharm_flag_&period.;
		*Rename all vars with time period identifier;
		rename aspirin_low_dose=&period._aspirin_low_dose;
		rename aspirin_any_dose=&period._aspirin_any_dose;
		rename clopidogrel=&period._clopidogrel;
		rename dipyridamole=&period._dipyridamole;
		rename prasugrel=&period._prasugrel;
		rename ticagrelor=&period._ticagrelor;
		rename ticlopidine=&period._ticlopidine ;
		rename warfarin=&period._warfarin;
		rename other_anticoagulant=&period._other_anticoag;
		rename ACEI=&period._ACEI;
		rename ARB=&period._ARB;
		rename thiazide=&period._thiazide;
		rename beta_blocker=&period._beta_blocker;
		rename calcium_channel_blocker=&period._calcium_channel;
		rename alphablocker=&period._alphablocker;
		rename spironolactone=&period._spironolactone;
		rename other_AHT=&period._other_AHT;
		rename statin=&period._statin;
		rename other_lipid_lowering=&period._other_lipid_lower;
		rename loop_diuretic=&period._loop_diuretic;
		rename metolazone=&period._metolazone;
		rename nitrate=&period._nitrate;
		rename other_antianginal=&period._other_antianginal;
		rename nonaspirin_NSAIDS=&period._nonaspirin_NSAIDS;
		rename corticosteroids=&period._corticosteroids;
		rename proton_pump_inhibitors=&period._proton_pump;
		rename H2_blockers=&period._H2_blockers;
		rename H_pylori_eradication=&period._H_pylori_erad;
		rename Diabetes=&period._diabetes;
		rename SSRI=&period._SSRI;
		run;

		%mend pharms6mth;
		
		%pharms6mth(2012-07-01,2012-12-31,base);
		%pharms6mth(2013-01-01,2013-06-30,post_2013a);
		%pharms6mth(2013-07-01,2013-12-31,post_2013b);
		%pharms6mth(2014-01-01,2014-06-30,post_2014a);
		%pharms6mth(2014-07-01,2014-12-31,post_2014b);



		*Put all 6 month periods together;
		data all_post_pharms;
		merge pharm_flag_base_recode pharm_flag_post_2013a_recode pharm_flag_post_2013b_recode pharm_flag_post_2014a_recode pharm_flag_post_2014b_recode;
		by snz_uid;

		*Recode all missing pharms to '0';
		%macro recode2(period);
		*Recode missing to zero for all pharms flags;

		if &period._aspirin_low_dose=. then &period._aspirin_low_dose=0;
		if &period._aspirin_any_dose=. then &period._aspirin_any_dose=0;
		if &period._clopidogrel=. then &period._clopidogrel=0;
		if &period._dipyridamole=. then &period._dipyridamole=0;
		if &period._prasugrel=. then &period._prasugrel=0;
		if &period._ticagrelor=. then &period._ticagrelor=0;
		if &period._ticlopidine=. then &period._ticlopidine =0;
		if &period._warfarin=. then &period._warfarin=0;
		if &period._other_anticoag=. then &period._other_anticoag=0;
		if &period._ACEI=. then &period._ACEI=0;
		if &period._ARB=. then &period._ARB=0;
		if &period._thiazide=. then &period._thiazide=0;
		if &period._beta_blocker=. then &period._beta_blocker=0;
		if &period._calcium_channel=. then &period._calcium_channel=0;
		if &period._alphablocker=. then &period._alphablocker=0;
		if &period._spironolactone=. then &period._spironolactone=0;
		if &period._other_AHT=. then &period._other_AHT=0;
		if &period._statin=. then &period._statin=0;
		if &period._other_lipid_lower=. then &period._other_lipid_lower=0;
		if &period._loop_diuretic=. then &period._loop_diuretic=0;
		if &period._metolazone=. then &period._metolazone=0;
		if &period._nitrate=. then &period._nitrate=0;
		if &period._other_antianginal=. then &period._other_antianginal=0;
		if &period._nonaspirin_NSAIDS=. then &period._nonaspirin_NSAIDS=0;
		if &period._corticosteroids=. then &period._corticosteroids=0;
		if &period._proton_pump=. then &period._proton_pump=0;
		if &period._H2_blockers=. then &period._H2_blockers=0;
		if &period._H_pylori_erad=. then &period._H_pylori_erad=0;
		if &period._Diabetes=. then &period._diabetes=0;
		if &period._SSRI=. then &period._SSRI=0;

		%mend recode2;
		
		%recode2(base);
		%recode2(post_2013a);
		%recode2(post_2013b);
		%recode2(post_2014a);
		%recode2(post_2014b);

		run;


		*Merge flags back into main VARIANZ dataset;
		proc sql;
		create table dlab.varianz_pharms as
		select * 
		from dlab.varianz_date_lc as a left join all_post_pharms as b on a.snz_uid=b.snz_uid;
		quit;





*************************************************
*************************************************
*****            CVD conditions             *****
*************************************************
*************************************************;

		*Extract dispensings of antianginals in the 5 years prior to reference date;

		proc sql;
		create table angina_pharms as
		select a.*, input(b.moh_pha_dispensed_date,anydtdte10.) as moh_pha_dispensed_date format ddmmyy10., b.moh_pha_dim_form_pack_code, 1 as angina_pharm_flag
		from dlab.varianz_pop as a left join moh.pharmaceutical as b
		on a.snz_uid=b.snz_uid and b.moh_pha_dispensed_date>='2008-01-01' and moh_pha_dim_form_pack_code in('55914' '55915' '55916' '55917' '55918' '55919' 
		'55920' '55922' '55923' '55924' '55925' '55927' '55928' '58523' '58524' '59906' '59907' '60864' '60865' '60866' '60867' '60868' '62201' '62202' '62203' '62204' '62399' '62400'
		'62401' '63723' '63724' '63725' '64022' '64023' '64930' '64931' '64932' '65054' '65055' '65056' '65400' '65401' '65402' '65403' '65404' '65405' '66281' '66282' '66283' '66284'
		'66285' '67219' '67220' '67221' '67222' '67223' '67224' '67225' '67546' '67547' '67548' '67549' '67550' '67551' '67860' '67861' '67862' '67863' '69364' '69365' '69366' '69367'
		'69718' '69719' '70005' '70006' '70007' '70008' '70009' '70010' '70216' '70217' '70218' '70219' '70220' '70221' '70222' '70223' '70224' '70225' '70226' '70227' '70228' '70229'
		'70230' '70335' '70336' '70337' '70740' '70741' '70742' '70743' '70744' '70745' '70746' '72738' '72739' '72758' '72759' '72760' '73367' '73368' '73604' '73813' '75331' '75332'
		'75431' '77846' '78080' '78153' '78391' '78725' '78776' '78828' '80596' '80616' '80977' '80978' '81699' '55879' '55921' '55929' '55930' '61774' '61775' '65587' '65588' '65589'
		'74111' '79338' '79339' '79408' '79409');
		quit;

		
		*Sum to get total number of dispensings per person in the 5 year period prior to reference date, and date of first dispensing;
		proc summary nway data=angina_pharms (where=(moh_pha_dim_form_pack_code ne '' and moh_pha_dispensed_date<='31DEC2012'd));
		class snz_uid;
		var angina_pharm_flag moh_pha_dispensed_date;
		output out=angina_pharms_count (drop=_type_ to_drop) min=to_drop first_date;
		run;
	
		*Flag people with 3 or more dispensings. Date of 'onset' is the date of the first dispensing in the 5 year period;
		data angina_flag;
		set angina_pharms_count;
		if _freq_ ge 3 then hist_angina_pharms=1;
		else hist_angina_pharms=0;
		format date_first_angina_pharms ddmmyy10.;
		if _freq_ ge 3 then date_first_angina_pharms=first_date;
		if _freq_<3 then delete; 		
		keep snz_uid hist_angina_pharms date_first_angina_pharms;
		run;



*Select all CVD diagnoses from diagnosis table;
proc sql;
create table cvd_diags as 
select distinct moh_dia_event_id_nbr, moh_dia_clinical_code
from moh.pub_fund_hosp_discharges_diag as b 
where moh_dia_clinical_code in(select distinct clinical_code from dlab2.latest_cvd_subgrp_longfm);
quit;

*Get event information for those diagnoses;
proc sql;
create table cvd_events as 
select distinct a.*, b.snz_uid, input(b.moh_evt_evst_date,anydtdte10.) as moh_evt_evst_date format ddmmyy10., b.moh_evt_even_date
from cvd_diags as a left join moh.pub_fund_hosp_discharges_event as b 
on a.moh_dia_event_id_nbr=b.moh_evt_event_id_nbr;
quit;

*Restrict to CVD events for varianz cohort;
proc sql;
create table cvd_events_varianz as 
select distinct a.snz_uid, b.moh_evt_evst_date, b.moh_evt_even_date, b.moh_dia_event_id_nbr, b.moh_dia_clinical_code
from sand.varianz_pop as a left join cvd_events as b 
on a.snz_uid=b.snz_uid;
quit;

*Get subgroup flags for those events;
proc sql;
create table cvd_subgroups_varianz as 
select *
from cvd_events_varianz as a left join dlab2.latest_cvd_subgrp_longfm as b 
on a.moh_dia_clinical_code=b.clinical_code;
quit;


*Create flags and dates of first hospitalisation prior to reference date for people with CVD events;

*Macro for creating flag for each CVD subgroup;

%macro subgrp(subgroup,shortname);

*Sort by subgroup and date, oldest first;
proc sort data=cvd_subgroups_varianz;
by snz_uid &subgroup. moh_evt_evst_date;
run;

data &shortname._hist;
set cvd_subgroups_varianz (where=(&subgroup.=1 and moh_evt_evst_date<='31DEC2012'd));
last_uid=lag(snz_uid);
format date_first_&shortname. ddmmyy10.;
if snz_uid ne last_uid then date_first_&shortname.=moh_evt_evst_date;
hist_&shortname.=1;
if snz_uid=last_uid then delete;
keep snz_uid hist_&shortname. date_first_&shortname.;
run;

%mend subgrp;

%subgrp(subgroup_acs,ACS);
%subgrp(subgroup_CHD,CHD);
%subgrp(subgroup_MI,MI);
%subgrp(subgroup_PVD,PVD);
%subgrp(subgroup_TIA,TIA);
%subgrp(subgroup_ischaemic_stroke,isch_str);
%subgrp(angina,angina);
%subgrp(heart_failure,HF);
%subgrp(diabetes,diabetes);
%subgrp(atrial_fibrillation,AF);
%subgrp(artherosclerotic_cvd,arth_CVD);
%subgrp(history_haemorrhagic_stroke,haem_str);

run;


*Flag for Diabetes from VDR;
proc sql;
create table diab_all as
select snz_uid, input(moh_chr_fir_incidnt_date,anydtdte10.) as moh_chr_fir_incidnt_date format ddmmyy10.
from moh.chronic_condition
where moh_chr_condition_text='DIA' and moh_chr_fir_incidnt_date<='2012-12-31'
order by snz_uid, moh_chr_fir_incidnt_date;
quit;

data diabetes;
set diab_all;
last_uid=lag(snz_uid);
format date_first_diabetes_vdr ddmmyy10.;
if snz_uid ne last_uid then date_first_diabetes_vdr=moh_chr_fir_incidnt_date;
hist_diabetes_vdr=1;
if snz_uid=last_uid then delete;
keep snz_uid hist_diabetes_vdr date_first_diabetes_vdr;
run;


*Add CVD flags and dates back into main dataset;
data dlab.varianz_cvd_hist;
merge dlab.varianz_pharms (in=a) arth_cvd_hist angina_hist haem_str_hist diabetes_hist diabetes angina_flag
acs_hist chd_hist mi_hist pvd_hist tia_hist isch_str_hist hf_hist af_hist;
by snz_uid;
if a;

*Create 'any cvd' variable;
*As per Suneela's email, CVD is defined as at least one incident of:
artherosclerotic CVD at any prior hospitalisation
or
angina at any prior hospitalisation
or
dispensing of 3x angina pharms in the last 5 years
or
haemorrhagic stroke at any prior hospitalisation;
hist_any_cvd=max(hist_arth_cvd,hist_angina,hist_angina_pharms,hist_haem_str);
format date_first_any_cvd ddmmyy10.;
date_first_any_cvd=min(date_first_angina,date_first_angina_pharms,date_first_arth_cvd,date_first_haem_str);

*Recode blanks to 0;
if hist_any_cvd=. then hist_any_cvd=0;
if hist_arth_cvd=. then hist_arth_cvd=0;
if hist_angina=. then hist_angina=0;
if hist_angina_pharms=. then hist_angina_pharms=0;
if hist_diabetes=. then hist_diabetes=0;
if hist_diabetes_vdr=. then hist_diabetes_vdr=0;
if hist_haem_str=. then hist_haem_str=0;
if hist_acs=. then hist_acs=0;
if hist_chd=. then hist_chd=0;
if hist_mi=. then hist_mi=0;
if hist_pvd=. then hist_pvd=0;
if hist_tia=. then hist_tia=0;
if hist_isch_str=. then hist_isch_str=0;
if hist_hf=. then hist_hf=0;
if hist_af=. then hist_af=0;

*Create var for people with antianginal pharms history ONLY;
if hist_angina_pharms=1 and max(hist_arth_cvd, hist_angina, hist_diabetes, hist_diabetes_vdr, hist_haem_str, hist_acs, hist_chd, hist_mi, hist_pvd, 
hist_tia, hist_isch_str, hist_hf, hist_af)=0 then hist_angina_pharms_only=1;
else hist_angina_pharms_only=0;

run;





*Create flags and dates of first hospitalisation after reference date for people with CVD events;

	*Extract dispensings of antianginals after the reference date;

		proc sql;
		create table angina_pharms_post as
		select a.*, input(b.moh_pha_dispensed_date,anydtdte10.) as moh_pha_dispensed_date format ddmmyy10., b.moh_pha_dim_form_pack_code, 1 as angina_pharm_flag
		from dlab.varianz_pop as a left join moh.pharmaceutical as b
		on a.snz_uid=b.snz_uid and b.moh_pha_dispensed_date>='2013-01-01' and moh_pha_dim_form_pack_code in('55914' '55915' '55916' '55917' '55918' '55919' 
		'55920' '55922' '55923' '55924' '55925' '55927' '55928' '58523' '58524' '59906' '59907' '60864' '60865' '60866' '60867' '60868' '62201' '62202' '62203' '62204' '62399' '62400'
		'62401' '63723' '63724' '63725' '64022' '64023' '64930' '64931' '64932' '65054' '65055' '65056' '65400' '65401' '65402' '65403' '65404' '65405' '66281' '66282' '66283' '66284'
		'66285' '67219' '67220' '67221' '67222' '67223' '67224' '67225' '67546' '67547' '67548' '67549' '67550' '67551' '67860' '67861' '67862' '67863' '69364' '69365' '69366' '69367'
		'69718' '69719' '70005' '70006' '70007' '70008' '70009' '70010' '70216' '70217' '70218' '70219' '70220' '70221' '70222' '70223' '70224' '70225' '70226' '70227' '70228' '70229'
		'70230' '70335' '70336' '70337' '70740' '70741' '70742' '70743' '70744' '70745' '70746' '72738' '72739' '72758' '72759' '72760' '73367' '73368' '73604' '73813' '75331' '75332'
		'75431' '77846' '78080' '78153' '78391' '78725' '78776' '78828' '80596' '80616' '80977' '80978' '81699' '55879' '55921' '55929' '55930' '61774' '61775' '65587' '65588' '65589'
		'74111' '79338' '79339' '79408' '79409');
		quit;

		
		*Sum to get total number of dispensings per person following the reference date, and date of first dispensing;
		proc summary nway data=angina_pharms_post (where=(moh_pha_dim_form_pack_code ne ''));
		class snz_uid;
		var angina_pharm_flag moh_pha_dispensed_date;
		output out=angina_pharms_count_post (drop=_type_ to_drop) min=to_drop first_date;
		run;
	
		*Flag people with 3 or more dispensings. Date of 'onset' is the date of the first dispensing in the 5 year period;
		data angina_flag_post;
		set angina_pharms_count_post;
		if _freq_ ge 3 then post_angina_pharms=1;
		else post_angina_pharms=0;
		format date_first_post_angina_pharms ddmmyy10.;
		if _freq_ ge 3 then date_first_post_angina_pharms=first_date;
		if _freq_<3 then delete; 		
		keep snz_uid post_angina_pharms date_first_post_angina_pharms;
		run;



*Create flags and dates of first hospitalisation after the reference date for people with CVD events;

*Macro for creating flag for each CVD subgroup;

%macro subgrp(subgroup,shortname);

*Sort by subgroup and date, oldest first;
proc sort data=cvd_subgroups_varianz;
by snz_uid &subgroup. moh_evt_evst_date;
run;

data &shortname._post;
set cvd_subgroups_varianz (where=(&subgroup.=1 and moh_evt_evst_date>='01JAN2013'd));
last_uid=lag(snz_uid);
format date_first_post_&shortname. ddmmyy10.;
if snz_uid ne last_uid then date_first_post_&shortname.=moh_evt_evst_date;
post_&shortname.=1;
if snz_uid=last_uid then delete;
keep snz_uid post_&shortname. date_first_post_&shortname.;
run;

%mend subgrp;

%subgrp(subgroup_acs,ACS);
%subgrp(subgroup_CHD,CHD);
%subgrp(subgroup_MI,MI);
%subgrp(subgroup_PVD,PVD);
%subgrp(subgroup_TIA,TIA);
%subgrp(subgroup_ischaemic_stroke,isch_str);
%subgrp(angina,angina);
%subgrp(heart_failure,HF);
%subgrp(diabetes,diabetes);
%subgrp(atrial_fibrillation,AF);
%subgrp(artherosclerotic_cvd,arth_CVD);
%subgrp(outcomes_haemorrhagic_stroke,haem_str);
run;


*Flag for Diabetes from VDR;
proc sql;
create table diab_all_post as
select snz_uid, input(moh_chr_fir_incidnt_date,anydtdte10.) as moh_chr_fir_incidnt_date format ddmmyy10.
from moh.chronic_condition
where moh_chr_condition_text='DIA' and moh_chr_fir_incidnt_date>='2013-01-01'
order by snz_uid, moh_chr_fir_incidnt_date;
quit;

data diabetes_vdr_post;
set diab_all_post;
last_uid=lag(snz_uid);
format date_first_post_diabetes_vdr ddmmyy10.;
if snz_uid ne last_uid then date_first_post_diabetes_vdr=moh_chr_fir_incidnt_date;
post_diabetes_vdr=1;
if snz_uid=last_uid then delete;
keep snz_uid post_diabetes_vdr date_first_post_diabetes_vdr;
run;


*Add CVD flags and dates back into main dataset;
data varianz_cvd_all;
merge dlab.varianz_cvd_hist (in=a) arth_cvd_post angina_post haem_str_post diabetes_post diabetes_vdr_post angina_flag_post
acs_post chd_post mi_post pvd_post tia_post isch_str_post hf_post af_post;
by snz_uid;
if a;

*Create 'any cvd' variable;
*As per Suneela's email, CVD is defined as at least one incident of:
artherosclerotic CVD at any prior hospitalisation
or
angina at any prior hospitalisation
or
dispensing of 3x angina pharms in the last 5 years
or
haemorrhagic stroke at any prior hospitalisation;
post_any_cvd=max(post_arth_cvd, post_angina, post_angina_pharms, post_haem_str);
format date_first_post_any_cvd ddmmyy10.;
date_first_post_any_cvd=min(date_first_post_angina, date_first_post_angina_pharms, date_first_post_arth_cvd, date_first_post_haem_str);

*Recode blanks to 0;
if post_any_cvd=. then post_any_cvd=0;
if post_arth_cvd=. then post_arth_cvd=0;
if post_angina=. then post_angina=0;
if post_angina_pharms=. then post_angina_pharms=0;
if post_diabetes=. then post_diabetes=0;
if post_diabetes_vdr=. then post_diabetes_vdr=0;
if post_haem_str=. then post_haem_str=0;
if post_acs=. then post_acs=0;
if post_chd=. then post_chd=0;
if post_mi=. then post_mi=0;
if post_pvd=. then post_pvd=0;
if post_tia=. then post_tia=0;
if post_isch_str=. then post_isch_str=0;
if post_hf=. then post_hf=0;
if post_af=. then post_af=0;

*Recode missing resident flags to 1 (if there were no overseas spells they will be missing);
if resident_2013=. then resident_2013=1;
if resident_2014=. then resident_2014=1;
if resident_2015=. then resident_2015=1;

**Calculate date of loss to followup;
* At present this is only loss to outmigration as we don't have updated death data. When we do will need to change to include date of death;
format date_lost_followup ddmmyy10.;
if resident_2013=0 then date_lost_followup='31DEC2013'd;
else if resident_2014=0 then date_lost_followup='31DEC2014'd;
else if resident_2015=0 then date_lost_followup='31DEC2015'd;
else date_lost_followup=.;
run;

*****************************************************
********* Dates of death- temporary measure *********
*****************************************************;

*Get dates of death from DIA data- no cause of death at this stage, and less reliable than MoH data, but it will do until we have more recent MoH data;
proc sql;
create table deaths as
select a.snz_uid, b.dia_dth_death_year_nbr, b.dia_dth_death_month_nbr
from dlab.varianz_pop as a left join dia.deaths as b on a.snz_uid=b.snz_uid
order by dia_dth_death_year_nbr descending;
quit;

*Reformat dates, impute day of death as last day of month (I didn't worry about leap years for now);
data death_dates;
set deaths;
format dia_dod_v1 $10.;
if dia_dth_death_month_nbr=. then dia_dod_v1='';
else if dia_dth_death_month_nbr=2 then dia_dod_v1=cat('28','-',dia_dth_death_month_nbr,'-',dia_dth_death_year_nbr);
else if dia_dth_death_month_nbr in(4 6 9 11) then dia_dod_v1=cat('30','-',dia_dth_death_month_nbr,'-',dia_dth_death_year_nbr);
else dia_dod_v1=cat('31','-',dia_dth_death_month_nbr,'-',dia_dth_death_year_nbr);

format dia_dod ddmmyy10.;
dia_dod=input(dia_dod_v1,anydtdte10.);

drop dia_dod_v1 dia_dth_death_year_nbr dia_dth_death_month_nbr;
run;

*There are duplicates in the deaths dataset- take the one with the earliest date;
proc summary nway data=death_dates;
class snz_uid;
var dia_dod;
output out=death_dates_no_dups (drop=_type_ _freq_) min=;
run;

*Merge dates back into main varianz file;
proc sql;
create table varianz_idi_final as
select *
from varianz_cvd_all as a left join death_dates_no_dups as b on a.snz_uid=b.snz_uid;
quit;

*Adjust loss to followup dates to include death;
data dlab.varianz_idi_final;
set varianz_idi_final;

format new_date_lost_followup ddmmyy10.;

if dia_dod=. then new_date_lost_followup=date_lost_followup;
else if date_lost_followup=. then new_date_lost_followup=dia_dod;
else new_date_lost_followup=min(dia_dod, date_lost_followup);

if dia_dod ne . then do;
	if dia_dod le '31DEC2013'd then do;
		resident_2014=0;
		resident_2015=0;
	end;
	else if dia_dod le '31DEC2014'd then resident_2015=0;
end;

drop date_lost_followup;
rename new_date_lost_followup=date_lost_followup;

run;



