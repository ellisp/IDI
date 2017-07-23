
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


*************************************************************************************************************************************
*************************************************************************************************************************************
Developer: Sarah Tumen and Robert Templeton ( A&I, NZ Treasury)
Date created: 7 Dec 2015


This code creates MSD days spent in a year on main benefit as an adult.


*************************************************************************************************************************************
*************************************************************************************************************************************;

* Run 00_code.sas code to get macros and libraries;

*************************************************************************************************************************************
*************************************************************************************************************************************;
%let population=project.Population1988_2013;

*************************************************************************************************************************************;
*************************************************************************************************************************************;
* Will create MSD pre and post reform high level classification;
%creating_MSD_SPEL;

/*
proc freq data=msd_spel;  tables BEN*ben_new/list missing;run;
proc freq data=msd_spel; where prereform='675'; tables BEN*ben_new/list missing;run;
proc freq data=msd_spel; where prereform='370'; tables BEN*ben_new/list missing;run;
*/

***********************************************************************************************************************************
***********************************************************************************************************************************;

* BDD spell dataset;
data msd_spel;
	set msd_spel;
	spell=msd_spel_spell_nbr;
	keep snz_uid spell servf spellfrom spellto ben ben_new;
run;

proc sort data=msd_spel out=mainbenefits(rename=(spellfrom=startdate spellto=enddate));
	by snz_uid spell spellfrom spellto;
run;

* BDD partner spell table;
data icd_bdd_ptnr;
	set msd.msd_partner;
	format ptnrfrom ptnrto date9.;
	spell=msd_ptnr_spell_nbr;
	ptnrfrom=input(compress(msd_ptnr_ptnr_from_date,"-"), yymmdd10.);
	ptnrto=input(compress(msd_ptnr_ptnr_to_date,"-"), yymmdd10.);

	* Sensoring;
	if ptnrfrom>"&sensor"d then
		delete;

	if ptnrto=. then
		ptnrto="&sensor"d;

	if ptnrto>"&sensor"d then
		ptnrto="&sensor"d;
	keep snz_uid partner_snz_uid spell ptnrfrom ptnrto;
run;

* EXTRACTING MAIN BENEFIT AS PRIMARY;
proc sql;
	create table prim_mainben_prim_data as
		select
			s.snz_uid, s.spellfrom as startdate, s.spellto as enddate, s.ben, s.ben_new, s.spell,
			t.DOB
		from
			msd_spel  s inner join &population t
			on t.snz_uid= s.snz_uid;
run;

* MAIN BENEFITS AS PARTNER (relationship);
proc sql;
	create table prim_mainben_part_data as
		select
			s.partner_snz_uid, s.ptnrfrom as startdate, s.ptnrto as enddate,s.spell,
			s.snz_uid as main_snz_uid,
			t.DOB
		from  icd_bdd_ptnr  s inner join &population t
			on t.snz_uid = s.partner_snz_uid
		order by s.snz_uid, s.spell;

	**ADD benefit type to the partner's dataset;

	**Note that snz_uid+spell does not uniquely identify benefit spells therefore the start
	and enddate of each spell is also used below to correctly match partner spells to those of the main beneficiary;

	**This is done in two steps - (1) spells with fully matching start and end dates
	(2) partner spells that fall within the matching main benefit spell but are not as long;
proc sort data=mainbenefits out=main nodupkey;
	by snz_uid spell startdate enddate;
run;

proc sort data=prim_mainben_part_data out=partner(rename=(main_snz_uid=snz_uid)) nodupkey;
	by main_snz_uid spell startdate enddate;
run;

data fullymatched  unmatched(drop=ben ben_new servf);
	merge partner (in = a)
		main (in = b);
	by snz_uid spell startdate enddate;

	if a and b then
		output fullymatched;
	else if a and not b then
		output unmatched;
run;

proc sql;
	create table partlymatched as
		select a.partner_snz_uid, a.snz_uid, a.spell, a.dob, a.startdate, a.enddate,
			b.ben, b.ben_new, b.servf
		from unmatched a left join main b
			on a.snz_uid=b.snz_uid and a.spell=b.spell and a.startdate>=b.startdate and a.enddate<=b.enddate;
quit;

run;

data prim_mainben_part_data_2;
	set fullymatched partlymatched;
run;

proc freq data=prim_mainben_part_data_2;
	tables ben_new ben;
run;

* CONSOLIDATING BENEFIT SPELLS AS PRIMARY AND PARTNER;
data prim_bennzs_data_1 del;
	set prim_mainben_prim_data (in=a)
		prim_mainben_part_data_2 (in=b);

	if b then
		snz_uid=partner_snz_uid;

	* Deleting benefit spells Before DOB of refrence person;
	if startdate<DOB then
		output del;
	else output prim_bennzs_data_1;
run;

* sorting before running ROGER'S OVERLAP CODE spells as spells should not overlap;
proc sort data = prim_bennzs_data_1;
	by snz_uid startdate enddate;
run;

%overlap(prim_bennzs_data_1,examine=F);

*** Aggregate by year;
%aggregate_by_year(prim_bennzs_data_1_OR,prim_bennzs_data_2,&first_anal_yr,&last_anal_yr,gross_daily_amt=);

*** Create wide file;
%create_BDD_wide(infile=prim_bennzs_data_2, outfile=project.IND_BDD_adult_&date.);

*** Create long file;
%create_BDD_long(merge1=project.IND_BDD_adult_&date., merge2=cohort_1, outfile=project._IND_BDD_adult_&date.);

*** Create at age variables;
%create_BDD_at_age(infile=prim_bennzs_data_1_OR, outfile=project._IND_BDD_adult_at_age_&date);

/*proc means data=project._IND_BDD_adult_at_age_&date;*/
/*run;*/

*************************************************************************************************************************************
*************************************************************************************************************************************

Checking
*************************************************************************************************************************************
************************************************************************************************************************************;

/*
data check; set project._ind_bdd_adult_at_age__23062015
;array total_da_onben_at_age_ [*] total_da_onben_at_age_&firstage-total_da_onben_at_age_&lastage;
array da_dpb_at_age_ [*] da_DPB_at_age_&firstage-da_DPB_at_age_&lastage;
array da_ub_at_age_ [*] da_UB_at_age_&firstage-da_UB_at_age_&lastage;
array da_sb_at_age_ [*] da_SB_at_age_&firstage-da_SB_at_age_&lastage;
array da_ib_at_age_ [*] da_IB_at_age_&firstage-da_IB_at_age_&lastage;
array da_iyb_at_age_ [*] da_IYB_at_age_&firstage-da_IYB_at_age_&lastage;
array da_othben_at_age_ [*] da_OTHBEN_at_age_&firstage-da_OTHBEN_at_age_&lastage;
array da_ucb_at_age_ [*] da_UCB_at_age_&firstage-da_UCB_at_age_&lastage;

array da_YP_at_age_ [*] da_YP_at_age_&firstage-da_YP_at_age_&lastage;
array da_YPP_at_age_ [*] da_YPP_at_age_&firstage-da_YPP_at_age_&lastage;
array da_SPSR_at_age_ [*] da_SPSR_at_age_&firstage-da_SPSR_at_age_&lastage;
array da_JSWR_at_age_ [*] da_JSWR_at_age_&firstage-da_JSWR_at_age_&lastage;
array da_JSWR_TR_at_age_ [*] da_JSWR_TR_at_age_&firstage-da_JSWR_TR_at_age_&lastage;
array da_JSHCD_at_age_ [*] da_JSHCD_at_age_&firstage-da_JSHCD_at_age_&lastage;
array da_SLP_C_at_age_ [*] da_SLP_C_at_age_&firstage-da_SLP_C_at_age_&lastage;
array da_SLP_HCD_at_age_ [*] da_SLP_HCD_at_age_&firstage-da_SLP_HCD_at_age_&lastage;
array da_OTH_at_age_ [*] da_OTH_at_age_&firstage-da_OTH_at_age_&lastage;

do i=&firstage to &lastage; age=i-(&firstage-1);
if total_da_onben_at_age_[age]>=1 then total_da_onben_at_age_[age]=1;
if da_dpb_at_age_[age]>=1 then da_dpb_at_age_[age]=1;
if da_ub_at_age_[age]>=1 then da_UB_at_age_[age]=1;
if da_SB_at_age_[age]>=1 then da_SB_at_age_[age]=1;
if da_IB_at_age_[age]>=1 then da_IB_at_age_[age]=1;
if da_IYB_at_age_[age]>=1 then da_IYB_at_age_[age]=1;
if da_OTHBEN_at_age_[age]>=1 then da_OTHBEN_at_age_[age]=1;
if da_UCB_at_age_[age]>=1 then da_UCB_at_age_[age]=1;

if da_YP_at_age_[age]>=1 then da_YP_at_age_[age]=1;
if da_YPP_at_age_[age]>=1 then da_YPP_at_age_[age]=1;
if da_SPSR_at_age_[age]>=1 then da_SPSR_at_age_[age]=1;
if da_JSWR_at_age_[age]>=1 then da_JSWR_at_age_[age]=1;
if da_JSWR_TR_at_age_[age]>=1 then da_JSWR_TR_at_age_[age]=1;
if da_JSHCD_at_age_[age]>=1 then da_JSHCD_at_age_[age]=1;
if da_SLP_C_at_age_[age]>=1 then da_SLP_C_at_age_[age]=1;
if da_SLP_HCD_at_age_[age]>=1 then da_SLP_HCD_at_age_[age]=1;
if da_OTH_at_age_[age]>=1 then da_OTH_at_age_[age]=1;
	end;
run;

proc means data=check;
var total_da_onben_at_age_0-total_da_onben_at_age_24
da_DPB_at_age_0-	da_DPB_at_age_24
da_IB_at_age_0-	da_IB_at_age_24
da_ub_at_age_0-	da_ub_at_age_24
da_IYB_at_age_0-	da_IYB_at_age_24
da_SB_at_age_0-	da_SB_at_age_24
da_UCB_at_age_0-	da_UCB_at_age_24
da_OTHBEN_at_age_0-	da_OTHBEN_at_age_24

da_YP_at_age_0-	da_YP_at_age_24
da_YPP_at_age_0	-da_YPP_at_age_24
da_SPSR_at_age_0-	da_SPSR_at_age_24
da_SLP_C_at_age_0-	da_SLP_C_at_age_24
da_SLP_HCD_at_age_0-	da_SLP_HCD_at_age_24
da_JSHCD_at_age_0	-da_JSHCD_at_age_24
da_JSWR_at_age_0-	da_JSWR_at_age_24
da_JSWR_TR_at_age_0-	da_JSWR_TR_at_age_24
da_OTH_at_age_0-	da_OTH_at_age_24;
output out=x;
run;
*/
proc datasets lib=work kill nolist memtype=data;
quit;