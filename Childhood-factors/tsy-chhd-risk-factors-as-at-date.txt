* code to create child risk factors - as per TSY social investment analysis;
* set up to run in refresh dated 20160224;
* set up to run for 0 to 14 population at end of 2014;
%let latest_ver=20160224;
%let by_year=2015;
%let first_year=1988;
%let last_year=2014;
%let refdate=31dec2014;

libname indic '\\wprdfs08\TreasuryData\MAA2013-16 Citizen pathways through human services\Social Investment_2016\1_Indicator_at_age_datasets\Dataset_rerun_24022016';

* left join the required "indicator at year" datasets onto the current population;
proc sql;
	create table pop_0_14_at_year_indicators as select
		a.* ,	 b.* ,     c.* ,     d.* ,     e.* ,      f.* ,     g.* ,     h.*
	from indic.currentpopn2014 (where=(ageat&refdate.<15)) a 
		left join   indic._ind_bdd_child_at_age_&date.(keep = snz_uid supp:) b  on a.snz_uid=b.snz_uid
		left join   indic._ind_bdd_child_&date.(keep=snz_uid ch_total_da_onben:) c on a.snz_uid=c.snz_uid
		left join   indic._IND_CG_CORR_&date.(keep=snz_uid cg_cust: cg_comm:) d on a.snz_uid=d.snz_uid 
		left join   indic._IND_CYF_&date.(keep=snz_uid child_any_fdgs: child_cyf_place: child_not: ) e on a.snz_uid=e.snz_uid 
		left join   indic._IND_CYF_CARE_&date.(keep=snz_uid child_ce: child_yj:) f on a.snz_uid=f.snz_uid 
		left join   indic._IND_SIBLING_CYF_&date.(keep=snz_uid othchd_any: othchd_not: othchd_cyf:) g on a.snz_uid=g.snz_uid 
		left join   indic.MAT_EDUC_COMB_YR_&date.(keep=snz_uid maternal_edu:) h on a.snz_uid=h.snz_uid;
run;


data pop_0_14_risk_indicators;
	set pop_0_14_at_year_indicators;

	keep snz_uid 
		ageat&refdate. 
		cyf_risk_&by_year.
		risk_factors_&by_year.
		risk_factors_2plus_&by_year.
		corr_risk_&by_year.
		wi_risk_&by_year.
		maternal_no_edu_&by_year. 	
		WI_onben_ge75_&by_year.
		prop_of_life_onben_&by_year.  
		ch_total_da_onben_&by_year. 
		days_of_life
		child_any_fdgs_&by_year.
		child_CYF_place_&by_year.
		child_CE_da_&by_year.
		child_not_&by_year.
		othchd_any_fdgs_abuse_&by_year.
		othchd_not_&by_year.
		othchd_CYF_place_&by_year.
		maternal_no_edu_&by_year. 
		maternal_edu_&by_year.
		CORR_risk_&by_year. 
		cg_cust_&by_year. 
		cg_comm_&by_year.;
	
	ch_total_da_onben_&by_year.=sum(of ch_total_da_onben_1988-ch_total_da_onben_2014);
	child_not_&by_year.=sum(of child_not_1988-child_not_2014);
	child_any_fdgs_&by_year.=sum(of child_any_fdgs_1988-child_any_fdgs_2014);
	child_CYF_place_&by_year.=sum(of child_CYF_place_1988-child_CYF_place_2014);
	child_CE_da_&by_year.=sum(of child_CE_da_1988-child_CE_da_2014);
	child_YJ_CE_da_&by_year.=sum(of child_YJ_CE_da_1988-child_YJ_CE_da_2014);
	othchd_any_fdgs_abuse_&by_year.=sum(of othchd_any_fdgs_abuse_year_1988-othchd_any_fdgs_abuse_year_2014);
	othchd_not_&by_year.=sum(of othchd_not_year_1988-othchd_not_year_2014);
	othchd_CYF_place_&by_year.=sum(of othchd_CYF_place_year_1988-othchd_CYF_place_year_2014);
	maternal_edu_&by_year.=max(of maternal_edu_1988-maternal_edu_2014);
	cg_cust_&by_year.=sum(of cg_cust_year_1988-cg_cust_year_2014);
	cg_comm_&by_year.=sum(of cg_comm_year_1988-cg_comm_year_2014);

* note: the above variables are summed over all the years 1988 onwards even though many individuals are born well  after 1988;
* they are missing prior to anyones birth date o this does not cause any problema nd just makes the coding simpler;

	****************************************;
	* create WI risk factor;
	****************************************;
	* indicator of being supported by benefit at birth;
	supp_ben_at_birth = max(supp_JSHCD_atbirth,
		supp_JSWR_TR_atbirth,
		supp_JSWR_atbirth,
		supp_OTH_atbirth,
		supp_SLP_C_atbirth,
		supp_SLP_HCD_atbirth,
		supp_SPSR_atbirth,
		supp_YPP_atbirth,
		supp_YP_atbirth,
		supp_dpb_atbirth,
		supp_ib_atbirth,
		supp_othben_atbirth,
		supp_sb_atbirth,
		supp_ub_atbirth,
		supp_ucb_atbirth
		);
	days_of_life = mdy(1,1,2015) - dob;
	prop_of_life_onben_&by_year.= ch_total_da_onben_&by_year./days_of_life;
	WI_onben_ge75_&by_year.= (prop_of_life_onben_&by_year. ge 0.75);

	if ageat&refdate.=0 then
		do;
			if supp_ben_at_birth or WI_onben_ge75_&by_year. then
				WI_risk_&by_year. = 1;
			else WI_risk_&by_year. = 0;
		end;
	else
		do;
			if WI_onben_ge75_&by_year. then
				WI_risk_&by_year. = 1;
			else WI_risk_&by_year. = 0;
		end;

	****************************************;
	* create CYF risk factor;
	****************************************;

	* indicator for the children themselves. But if the child is under 3 then
	  include indicator of their siblings;

	* adding sibling info for 0 and 1 year olds;
	* converting the abuse variables into indicators;
	if ageat&refdate. = 0 then
		do;
			CYF_risk_&by_year. = ((sum(child_any_fdgs_&by_year.>0
				,child_CYF_place_&by_year.>0
				,child_CE_da_&by_year.>0
				,child_not_&by_year.>0
				,othchd_any_fdgs_abuse_&by_year.>0
				,othchd_not_&by_year.>0
				,othchd_CYF_place_&by_year.>0
				))>0);
		end;
	else if ageat&refdate. <=2 then
		do;
			CYF_risk_&by_year. = (sum(child_any_fdgs_&by_year.>0
				,child_CYF_place_&by_year.>0
				,child_CE_da_&by_year.>0
				,othchd_any_fdgs_abuse_&by_year.>0
				,othchd_CYF_place_&by_year.>0
				)>0);
		end;
	else
		do;
			CYF_risk_&by_year. = (sum(child_any_fdgs_&by_year.>0
				,child_CYF_place_&by_year.>0
				,child_CE_da_&by_year.>0
				)>0);
		end;

	****************************************;
	* create mother education risk factor;
	****************************************;
	maternal_no_edu_&by_year. = (maternal_edu_&by_year. in (0, 0.5));

	****************************************;
	* create CG CORR_ections risk factor;
	****************************************;
	CORR_risk_&by_year. = (sum(cg_cust_&by_year., cg_comm_&by_year.)>0);

	****************************************;
	* Count risk factors;
	****************************************;
	risk_factors_&by_year.= sum(WI_risk_&by_year.
		,CYF_risk_&by_year.
		,CORR_risk_&by_year.
		,maternal_no_edu_&by_year.
		,0);
	risk_factors_2plus_&by_year.=(risk_factors_&by_year. ge 2);
run;


* tabulate to check;
proc tabulate data=pop_0_14_risk_indicators missing;
	class ageat&refdate. risk_factors_&by_year. / missing;
	var risk_factors_2plus_&by_year.
		CYF_risk_&by_year.
		corr_risk_&by_year.
		wi_risk_&by_year.
		maternal_no_edu_&by_year.;
	tables ageat&refdate. all,n risk_factors_&by_year.*n 
		(risk_factors_2plus_&by_year.
		CYF_risk_&by_year.
		corr_risk_&by_year.
		wi_risk_&by_year.
		maternal_no_edu_&by_year.)*(mean sum);
run;