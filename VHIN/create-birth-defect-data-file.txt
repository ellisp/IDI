
************************************************************************************************************************************************
****          -This code was written in July 2016 by Sheree Gibb;                                                                           ****
****          -Code was written for the Virtual Health Information Network catalyst project with                                            ****
****           Andrea 't Mannetje and Jeroen Douwes from Masey University;                                                                  ****
****          -The code uses an existing dataset of births from a prior case-control study at Massey;                                       ****
****          -It creates a dataset with additional information about those births, and about the mothers and fathers of the babies         ****
****          -Note these analyses can ONLY be run with the 20160418 refresh. If you want to run with another refresh, you will             ****
****           need to go back and link the snz_moh_uid to the snz_uids in the new refresh (the snz_uids change between refreshes,          ****
****           the snz_moh_uids don't) and recreate BD1a and BD2a in the sandpit;                                                           ****
************************************************************************************************************************************************;

libname sand ODBC dsn=idi_sandpit_srvprd schema="DL-MAA2016-10";
libname secur ODBC dsn=IDI_Clean_20160418_srvprd schema=security;
libname birth ODBC dsn=IDI_Clean_20160418_srvprd schema=dia_clean;
libname Dlab '\\wprdfs08\datalab-ma\MAA2016-10 Environmental and occupational risk factors for chronic conditions';
libname moh ODBC dsn=IDI_Clean_20160418_srvprd schema=moh_clean;
libname metadata ODBC dsn=idi_metadata_srvprd schema=clean_read_classifications;
libname data ODBC dsn=IDI_Clean_20160418_srvprd schema=data;
libname cen ODBC dsn=IDI_Clean_20160418_srvprd schema=cen_clean;


************************************************************
********  Get information from DIA birth record;    ********
************************************************************;

	*I didn't extract sex for parents because I checked and all 'parent1' are female and all 'parent2' male for this dataset;
	proc sql;
		create table birth_info as 
		select a.*, b.parent1_snz_uid, b.parent2_snz_uid, b.dia_bir_birth_year_nbr, 
		b.dia_bir_birth_month_nbr, CAT(dia_bir_birth_year_nbr, '-', dia_bir_birth_month_nbr, '-', '15') format=$10. as dob, 
		b.snz_dia_uid, b.dia_bir_multiple_birth_code as dia_multiple_birth, b.dia_bir_birth_weight_nbr as dia_birth_weight, 
		b.dia_bir_birth_gestation_nbr as dia_gestation, b.dia_bir_parents_rel_code as dia_parent_rel, 
		b.dia_bir_parent1_birth_month_nbr, b.dia_bir_parent1_birth_year_nbr, b.dia_bir_parent2_birth_month_nbr, b.dia_bir_parent2_birth_year_nbr, 
		b.dia_bir_sex_snz_code as dia_sex, b.dia_bir_parent1_ethnic_grp1_snz_ as dia_m_nzeuro, b.dia_bir_parent1_ethnic_grp2_snz_ as dia_m_maori, 
		b.dia_bir_parent1_ethnic_grp3_snz_ as dia_m_pacific, b.dia_bir_parent1_ethnic_grp4_snz_ as dia_m_asian,
		b.dia_bir_parent1_ethnic_grp5_snz_ as dia_m_melaa, b.dia_bir_parent1_ethnic_grp6_snz_ as dia_m_othereth, 
		b.dia_bir_parent2_ethnic_grp1_snz_ as dia_f_nzeuro, b.dia_bir_parent2_ethnic_grp2_snz_ as dia_f_maori, b.dia_bir_parent2_ethnic_grp3_snz_ as dia_f_pacific, 
		b.dia_bir_parent2_ethnic_grp4_snz_ as dia_f_asian, b.dia_bir_parent2_ethnic_grp5_snz_ as dia_f_melaa, b.dia_bir_parent2_ethnic_grp6_snz_ as dia_f_othereth
		from sand.BD1a as a left join birth.births as b on a.snz_uid=b.snz_uid;
	quit;

	data birth_info2;
		set birth_info;

		rename _case=case;

		*Convert date of birth to date format;
		format dia_dob ddmmyy10.;
		dia_dob=input(dob, anydtdte10.);
		drop dob;

		*Create flags for having a birth record, and presence of mother and father on birth record;
		if snz_dia_uid=. then birth_rec_flag=0;
		else birth_rec_flag=1;
		if birth_rec_flag=0 then parent1_flag=.;
		else if parent1_snz_uid=. then parent1_flag=0;
		else parent1_flag=1;
		if birth_rec_flag=0 then parent2_flag=.;
		else if parent2_snz_uid=. then parent2_flag=0;
		else parent2_flag=1;

		*Flag for multiple birth;
		if birth_rec_flag=0 then multi_birth_flag=.;
		else if birth_rec_flag=1 and dia_multiple_birth>. then multi_birth_flag=1;
		else multi_birth_flag=0;

		*Recoding blank gestations and birthweights;
		if dia_gestation=0 then dia_gestation=.;
		if dia_birth_weight=0 then dia_birth_weight=.;

		*Calculating parents' ages at time of baby's birth;
			*Convert parent month and year of birth to estimated birth date;
			dia_mother_date=cat(dia_bir_parent1_birth_year_nbr, '-', dia_bir_parent1_birth_month_nbr, '-15');
			format dia_mother_dob ddmmyy10.;
			dia_mother_dob=input(dia_mother_date, anydtdte10.);
			dia_father_date=cat(dia_bir_parent2_birth_year_nbr, '-', dia_bir_parent2_birth_month_nbr, '-15');
			format dia_father_dob ddmmyy10.;
			dia_father_dob=input(dia_father_date, anydtdte10.);
			drop dia_mother_date dia_father_date dia_bir_parent1_birth_year_nbr dia_bir_parent1_birth_month_nbr dia_bir_parent2_birth_month_nbr dia_bir_parent2_birth_year_nbr;
			*Calculate parent ages;
			dia_mother_age=(dia_dob-dia_mother_dob)/365;
			dia_father_age=(dia_dob-dia_father_dob)/365;

		*Estimated date of conception;
		con_date=dia_dob-(dia_gestation*7);
		format con_date ddmmyy10.;

		*Calculate parents' ages at conception;
		dia_mother_age_con=(con_date-dia_mother_dob)/365;
		dia_father_age_con=(con_date-dia_father_dob)/365;

		*Calculating end date for first trimester;
		format first_tri_end ddmmyy10.;
		first_tri_end=con_date+93;
		*Calculating end date for second trimester;
		format sec_tri_end ddmmyy10.;
		sec_tri_end=con_date+186;
	run;





******************************************************************************
********  Get address information from address notification table;    ********
******************************************************************************;

	*Get mother's and father's latest address at time of baby's birth;
	
	***********
	* MOTHERS *
	*********** 

	*Link birth defects dataset with address table to get addresses for mothers at the time of baby's birth;
	proc sql;
		create table mother_address as
		select unique a.*, b.ant_address_source_code as addr_source_m, b.ant_meshblock_code as MB13_m, b.ant_notification_date as addr_date_updated_m
		from birth_info2 as a left join data.address_notification_full as b 
		on a.parent1_snz_uid=b.snz_uid 
		order by snz_uid, ant_notification_date descending;
	quit;

	*Remove addresses that are after baby's birth;
	data mother_address2;
		set mother_address;
		address_date=input(addr_date_updated_m, yymmdd10.);
		format address_date ddmmyy10.;
		dia_dob2=datepart(dia_dob);
		format dia_dob2 ddmmyy10.;
		if address_date>dia_dob2 then delete;
		drop address_date dia_dob2;
	run;

	*and select the most recently updated address for mother before baby's birth;
	data final_address_m;
	   set mother_address;
	   by snz_uid;
	   if first.snz_uid;
	run;

	*Merge with concordances to get DHB and NZDEP;
	proc sql;
		create table addr_nzdep_m as 
		select a.*, b.depindex2013 as nzdep2013_m, b.depscore2013 as depscore2013_m
		from final_address_m as a left join metadata.depindex2013 as b on a.mb13_m=b.meshblock2013;
	quit;


	***********
	* FATHERS *
	***********;

	*Link birth defects dataset with address table to get addresses for fathers at the time of baby's birth;
	proc sql;
		create table father_address as
		select unique a.*, b.ant_address_source_code as addr_source_f, b.ant_meshblock_code as MB13_f, b.ant_notification_date as addr_date_updated_f
		from addr_nzdep_m as a left join data.address_notification_full as b 
		on a.parent2_snz_uid=b.snz_uid 
		order by snz_uid, ant_notification_date descending;
	quit;

	*Remove addresses that are after baby's birth;
	data father_address2;
		set father_address;
		address_date=input(addr_date_updated_f, yymmdd10.);
		format address_date ddmmyy10.;
		dia_dob2=datepart(dia_dob);
		format dia_dob2 ddmmyy10.;
		if address_date>dia_dob2 then delete;
		drop address_date dia_dob2;
	run;

	*and select the most recently updated address for mother before baby's birth;
	data final_address_f;
	   set father_address;
	   by snz_uid;
	   if first.snz_uid;
	run;

	*Merge with concordances to get DHB and NZDEP;
	proc sql;
		create table bd_final_address as 
		select a.*, b.depindex2013 as nzdep2013_f, b.depscore2013 as depscore2013_f
		from final_address_f as a left join metadata.depindex2013 as b on a.mb13_f=b.meshblock2013;
	quit;






**********************************************
**********************************************
***       ADD PHARMACEUTICAL DATA         ***;
**********************************************
**********************************************;

*At this stage I have extracted pharmaceutical data for mothers only;

	*Get all prescriptions for mothers for 2005-2011 (1 year before earliest birth in cohort, until after last birth);
	proc sql;
		create table dlab.pharms_subset_new as 
		select a.*, input(b.moh_pha_dispensed_date,anydtdte10.) as moh_pha_dispensed_date format ddmmyy10. as moh_pha_dispensed_date, b.moh_pha_dim_form_pack_code, b.moh_pha_dose_nbr, b.moh_pha_frequency_nbr, b.moh_pha_daily_dose_nbr,
		b.moh_pha_days_supply_nbr, b.moh_pha_quan_presc_nbr, b.moh_pha_quan_disp_nbr, b.moh_pha_disp_presc_nbr
		from bd_final_address as a left join moh.pharmaceutical as b 
		on a.parent1_snz_uid=b.snz_uid and moh_pha_dispensed_date>'2005-12-31' and moh_pha_dispensed_date<'2011-01-01';
	quit;


	*Keep only those prescriptions that were dated between 3 months before conception date and birth of child;
	data mother_presc_short;
		set dlab.pharms_subset_new;
		format presc_date ddmmyy10.;
		presc_date=moh_pha_dispensed_date;
		format birth_date ddmmyy10.;
		birth_date=dia_dob;
		time_pre_con=con_date-presc_date;
		if time_pre_con gt 92 then delete;
		if presc_date>birth_date then delete;
		drop moh_pha_dispensed_date dia_dob time_pre_con;
	run;


	*Change format of dim_form_pack_code in the pharmaceutical lookup table so it can be merged with pharmaceutical data;
	data chemical_ids;
		set metadata.moh_dim_form_pack_subsidy_code;
		dim_form_pack_code=strip(put(dim_form_pack_subsidy_key, 8.));
		drop dim_form_pack_subsidy_key;
	run;

	*Get chemical IDs, formulation IDs, and treatment groups for prescriptions;
	proc sql;
		create table mother_presc_chems as
		select a.*, b.tg_name2, b.tg_name1, b.tg_name3, b.chemical_id, b.chemical_name, b.formulation_id, b.formulation_name
		from mother_presc_short as a left join chemical_ids as b on a.moh_pha_dim_form_pack_code=b.dim_form_pack_code;
	quit;

	*Create flags for major drug groups during four time periods (prior to conception, first, second, and third trimesters);
	data with_drug_flags;
		set mother_presc_chems;

		*To start with, create flags for all broad treatment groups and some specific treatment groups;

		%macro drugflags(tglevel,longname,shortname);
			if birth_rec_flag=1 then do;
				if tg_name&tglevel.=&longname then do;
					if presc_date<con_date then tg&tglevel._&shortname._pc=1;
					else tg&tglevel._&shortname._pc=0;
					if presc_date>=con_date and presc_date<first_tri_end then tg&tglevel._&shortname._t1=1;
					else tg&tglevel._&shortname._t1=0;
					if presc_date>=first_tri_end and presc_date<sec_tri_end then tg&tglevel._&shortname._t2=1;
					else tg&tglevel._&shortname._t2=0;
					if presc_date>=sec_tri_end then tg&tglevel._&shortname._t3=1;
					else tg&tglevel._&shortname._t3=0;
				end;
				else if tg_name&tglevel ne &longname then do;
					tg&tglevel._&shortname._pc=0;
					tg&tglevel._&shortname._t1=0;
					tg&tglevel._&shortname._t2=0;
					tg&tglevel._&shortname._t3=0; 
				end;
			end;
			else if birth_rec_flag=0 then do;
				tg&tglevel._&shortname._pc=.;
				tg&tglevel._&shortname._t1=.;
				tg&tglevel._&shortname._t2=.;
				tg&tglevel._&shortname._t3=.;
			end;
		%mend drugflags;

		%drugflags (1,'Alimentary Tract and Metabolism',ALIM);
		%drugflags (1,'Blood and Blood Forming Organs',BLOOD);
		%drugflags (1,'Dermatologicals',DERM);
		%drugflags (1,'Cardiovascular System',CARDIO);
		%drugflags (1,'Extemporaneously Compounded Preparations and Galenicals',COMPOUND);
		%drugflags (1,'Genito-Urinary System',GENURO);
		%drugflags (1,'Hormone Preparations - Systemic Excluding Contraceptive Hormones',HORMONE);
		%drugflags (1,'Infections - Agents for Systemic Use',INFECT);
		%drugflags (1,'Musculoskeletal System',MUSC);
		%drugflags (1,'Nervous System',NERV);
		%drugflags (1,'Oncology Agents and Immunosuppressants',ONCO);
		%drugflags (1,'Respiratory System and Allergies',RESP);
		%drugflags (1,'Sensory Organs',SENS);
		%drugflags (1,'Special Foods', FOOD);
		%drugflags (3,'ACE Inhibitors',ACE);
		%drugflags (3,'ACE Inhibitors with Diuretics',ACEDIU);
		%drugflags (3,'Angiotensin II Antagonists',ANGIO);
		%drugflags (3,'Antiacne Preparations',ACNE);
		%drugflags (3,'Beta Adrenoceptor Blockers',BETABLK);
		%drugflags (3,'Control of Epilepsy',EPIL);
		%drugflags (3,'Dihydropyridine Calcium Channel Blockers',CALCICHAN);
		%drugflags (3,'General',ANTIPSYC);
		%drugflags (3,'Inhaled Beta-Adrenoceptor Agonists', BAA1);
		%drugflags (3,'Inhaled Beta-Adrenoceptor Agonists with Anticholinergic Agents', BAA2);
		%drugflags (3,'Inhaled Corticosteroids with Long-Acting Beta-Adrenoceptor Agonists', BAA3);
		%drugflags (3,'Inhaled Long-Acting Beta-adrenoceptor Agonists',BAA4);
		%drugflags (3,'Macrolides',MCLIDES);
		%drugflags (3,'Oral Anticoagulants',WARFARIN);
		%drugflags (3,'Penicillins',PENCLN);
		%drugflags (3,'Thiazide and Related Diuretics',THIAZIDE);
		%drugflags (3,'Urinary Tract Infections',UTIS);
		%drugflags (3,'Vasodilators',VASODIL);
		%drugflags (3,'Cephalosporins and Cephamycins',CEPH);
		%drugflags (2,'Antimigraine Preparations',MIGRAINE);
		%drugflags (3,'Opioid Analgesics',OPIOID);
		%drugflags (3,'Non-opioid Analgesics',NONOPIOID);
		%drugflags (2,'Diabetes',DIABETES);
		%drugflags (2,'Diabetes Management',DIAB_MANAG);
		%drugflags (3,'Vitamin A',VITA);
		%drugflags (3,'Vitamin B',VITB);
		%drugflags (3,'Vitamin C',VITC);
		%drugflags (3,'Vitamin D',VITD);
		%drugflags (3,'Vitamin E',VITE);
		%drugflags (2,'Antiandrogen Oral Contraceptives',OC_AND);
		%drugflags (2,'Contraceptives - Hormonal',OC);
		%drugflags (2,'Gynaecological Anti-infectives',GYN_AI);
		%drugflags (2,'Myometrial and Vaginal Hormone Preparations',MYOM_HORM);
		%drugflags (2,'Contraceptives - Non-hormonal',CONTRACEPT);

		*Combine some of the treatment groups;
		any_presc=max(tg1_alim_pc, tg1_alim_t1, tg1_alim_t2, tg1_alim_t3, 
		tg1_blood_pc, tg1_blood_t1, tg1_blood_t2, tg1_blood_t3,
		tg1_derm_pc, tg1_derm_t1, tg1_derm_t2, tg1_derm_t3,
		tg1_cardio_pc, tg1_cardio_t1, tg1_cardio_t2, tg1_cardio_t3,
		tg1_compound_pc, tg1_compound_t1, tg1_compound_t2, tg1_compound_t3,
		tg1_genuro_pc, tg1_genuro_t1, tg1_genuro_t2, tg1_genuro_t3,
		tg1_hormone_pc, tg1_hormone_t1, tg1_hormone_t2, tg1_hormone_t3,
		tg1_infect_pc, tg1_infect_t1, tg1_infect_t2, tg1_infect_t3,
		tg1_musc_pc, tg1_musc_t1, tg1_musc_t2, tg1_musc_t3,
		tg1_nerv_pc, tg1_nerv_t1, tg1_nerv_t2, tg1_nerv_t3,
		tg1_onco_pc, tg1_onco_t1, tg1_onco_t2, tg1_onco_t3,
		tg1_resp_pc, tg1_resp_t1, tg1_resp_t2, tg1_resp_t3,
		tg1_sens_pc, tg1_sens_t1, tg1_sens_t2, tg1_sens_t3,
		tg1_food_pc, tg1_food_t1, tg1_food_t2, tg1_food_t3);

		tg3_BAA_pc=max(tg3_BAA1_pc,tg3_BAA2_pc,tg3_BAA3_pc,tg3_BAA4_pc);
		tg3_BAA_t1=max(tg3_BAA1_t1,tg3_BAA2_t1,tg3_BAA3_t1,tg3_BAA4_t1);
		tg3_BAA_t2=max(tg3_BAA1_t2,tg3_BAA2_t2,tg3_BAA3_t2,tg3_BAA4_t2);
		tg3_BAA_t3=max(tg3_BAA1_t3,tg3_BAA2_t3,tg3_BAA3_t3,tg3_BAA4_t3);

		any_terato_pc=max(tg3_ACE_pc,tg3_ACEDIU_pc,tg3_ANGIO_pc,tg3_ACNE_pc,
		tg3_BETABLK_pc,tg3_EPIL_pc,tg3_CALCICHAN_pc,tg3_ANTIPSYC_pc,tg3_BAA_pc,
		tg3_MCLIDES_pc,tg3_WARFARIN_pc,tg3_PENCLN_pc,tg3_THIAZIDE_pc,tg3_UTIS_pc,
		tg3_VASODIL_pc,tg3_VITA_pc);

		any_terato_t1=max(tg3_ACE_t1,tg3_ACEDIU_t1,tg3_ANGIO_t1,tg3_ACNE_t1,
		tg3_BETABLK_t1,tg3_EPIL_t1,tg3_CALCICHAN_t1,tg3_ANTIPSYC_t1,tg3_BAA_t1,
		tg3_MCLIDES_t1,tg3_WARFARIN_t1,tg3_PENCLN_t1,tg3_THIAZIDE_t1,tg3_UTIS_t1,
		tg3_VASODIL_t1,tg3_VITA_t1);

		any_terato_t2=max(tg3_ACE_t2,tg3_ACEDIU_t2,tg3_ANGIO_t2,tg3_ACNE_t2,
		tg3_BETABLK_t2,tg3_EPIL_t2,tg3_CALCICHAN_t2,tg3_ANTIPSYC_t2,tg3_BAA_t2,
		tg3_MCLIDES_t2,tg3_WARFARIN_t2,tg3_PENCLN_t2,tg3_THIAZIDE_t2,tg3_UTIS_t2,
		tg3_VASODIL_t2,tg3_VITA_t2);

		any_terato_t3=max(tg3_ACE_t3,tg3_ACEDIU_t3,tg3_ANGIO_t3,tg3_ACNE_t3,
		tg3_BETABLK_t3,tg3_EPIL_t3,tg3_CALCICHAN_t3,tg3_ANTIPSYC_t3,tg3_BAA_t3,
		tg3_MCLIDES_t3,tg3_WARFARIN_t3,tg3_PENCLN_t3,tg3_THIAZIDE_t3,tg3_UTIS_t3,
		tg3_VASODIL_t3,tg3_VITA_t3);

		diab_pc=max(tg2_diabetes_pc, tg2_diab_manag_pc);
		diab_t1=max(tg2_diabetes_t1, tg2_diab_manag_t1);
		diab_t2=max(tg2_diabetes_t2, tg2_diab_manag_t2);
		diab_t3=max(tg2_diabetes_t3, tg2_diab_manag_t3);

		diab_any_time=max(diab_pc, diab_t1, diab_t2, diab_t3);
		epil_any_time=max(tg3_epil_pc, tg3_epil_t1, tg3_epil_t2, tg3_epil_t3);

		contracept_hormone_pc=max(tg2_oc_and_pc, tg2_oc_pc);
		contracept_hormone_t1=max(tg2_oc_and_t1, tg2_oc_t1);
		contracept_hormone_t2=max(tg2_oc_and_t2, tg2_oc_t2);
		contracept_hormone_t3=max(tg2_oc_and_t3, tg2_oc_t3);

		*Create flags for custom drug groups in each time period;

		%macro druggroups(group,list);
			if birth_rec_flag=1 then do;
				if chemical_id in&list then do;
					if presc_date<con_date then &group._pc=1;
					else &group._pc=0;
					if presc_date>=con_date and presc_date<first_tri_end then &group._t1=1;
					else &group._t1=0;
					if presc_date>=first_tri_end and presc_date<sec_tri_end then &group._t2=1;
					else &group._t2=0;
					if presc_date>=sec_tri_end then &group._t3=1;
					else &group._t3=0;
				end;
				else if chemical_id not in&list then do;
					&group._pc=0;
					&group._t1=0;
					&group._t2=0;
					&group._t3=0; 
				end;
			end;
			else if birth_rec_flag=0 then do;
				&group._pc=.;
				&group._t1=.;
				&group._t2=.;
				&group._t3=.;
			end;
		%mend druggroups;

		%druggroups (folate_antag,(1217 1291 2421 1002 1794 1797 1861 3354 1956 
			1978 2041 1262 1375 2073 3992 2205 2293 2300 2166)); /*Folate antagonists*/
		%druggroups (benzo,(2484 1397 1730 1731 1865 2224 2295 2632 1911)); /*Benzodiazepines*/
		%druggroups (serotonin,(6006 1183 1011 4044 2800 1125 1180 1190 1193 2636 3926 3927 6009)); /*Serotonin agonists and antagonists incl SSRIs*/
		%druggroups (vasocon,(2546 1214 1458 1459 1460 1462 2066)); /*Vasoconstrictors, selected pharms as listed in van Gelder et al*/
		%druggroups (nmda,(1048 1256 1096)); /*NMDA receptor antagonists*/
		%druggroups (aspirin,(1087 1092 1093 3216)); /*Aspirin*/
		%druggroups (c3antiarr,(1057 2169)); /*Class III antiarrhythmics*/
		%druggroups (retin,(1688 2286 2363)); /*Retinoids*/
		%druggroups (iron,(1501 1676 3291 3736 3831 3862)); /*Iron supplements*/

		run;

		*Summarise all pharms flags per person;
		proc summary nway data=with_drug_flags;
			class snz_uid;
			var case tg1_alim_pc tg1_alim_t1 tg1_alim_t2 tg1_alim_t3 
			tg1_blood_pc tg1_blood_t1 tg1_blood_t2 tg1_blood_t3
			tg1_derm_pc tg1_derm_t1 tg1_derm_t2 tg1_derm_t3
			tg1_cardio_pc tg1_cardio_t1 tg1_cardio_t2 tg1_cardio_t3
			tg1_compound_pc tg1_compound_t1 tg1_compound_t2 tg1_compound_t3
			tg1_genuro_pc tg1_genuro_t1 tg1_genuro_t2 tg1_genuro_t3
			tg1_hormone_pc tg1_hormone_t1 tg1_hormone_t2 tg1_hormone_t3
			tg1_infect_pc tg1_infect_t1 tg1_infect_t2 tg1_infect_t3
			tg1_musc_pc tg1_musc_t1 tg1_musc_t2 tg1_musc_t3
			tg1_nerv_pc tg1_nerv_t1 tg1_nerv_t2 tg1_nerv_t3
			tg1_onco_pc tg1_onco_t1 tg1_onco_t2 tg1_onco_t3
			tg1_resp_pc tg1_resp_t1 tg1_resp_t2 tg1_resp_t3
			tg1_sens_pc tg1_sens_t1 tg1_sens_t2 tg1_sens_t3
			tg1_food_pc tg1_food_t1 tg1_food_t2 tg1_food_t3 
			tg3_ACE_pc tg3_ACE_t1 tg3_ACE_t2 tg3_ACE_t3 
			tg3_ACEDIU_pc tg3_ACEDIU_t1 tg3_ACEDIU_t2 tg3_ACEDIU_t3 
			tg3_ANGIO_pc tg3_ANGIO_t1 tg3_ANGIO_t2 tg3_ANGIO_t3 
			tg3_ACNE_pc tg3_ACNE_t1 tg3_ACNE_t2 tg3_ACNE_t3 
			tg3_BETABLK_pc tg3_BETABLK_t1 tg3_BETABLK_t2 tg3_BETABLK_t3 
			tg3_EPIL_pc tg3_EPIL_t1 tg3_EPIL_t2 tg3_EPIL_t3 
			tg3_CALCICHAN_pc tg3_CALCICHAN_t1 tg3_CALCICHAN_t2 tg3_CALCICHAN_t3 
			tg3_ANTIPSYC_pc tg3_ANTIPSYC_t1 tg3_ANTIPSYC_t2 tg3_ANTIPSYC_t3 
			tg3_BAA1_pc tg3_BAA1_t1 tg3_BAA1_t2 tg3_BAA1_t3 
			tg3_BAA2_pc tg3_BAA2_t1 tg3_BAA2_t2 tg3_BAA2_t3 
			tg3_BAA3_pc tg3_BAA3_t1 tg3_BAA3_t2 tg3_BAA3_t3 
			tg3_BAA4_pc tg3_BAA4_t1 tg3_BAA4_t2 tg3_BAA4_t3 
			tg3_BAA_pc tg3_BAA_t1 tg3_BAA_t2 tg3_BAA_t3 
			tg3_MCLIDES_pc tg3_MCLIDES_t1 tg3_MCLIDES_t2 tg3_MCLIDES_t3 
			tg3_WARFARIN_pc tg3_WARFARIN_t1 tg3_WARFARIN_t2 tg3_WARFARIN_t3 
			tg3_PENCLN_pc tg3_PENCLN_t1 tg3_PENCLN_t2 tg3_PENCLN_t3 
			tg3_THIAZIDE_pc tg3_THIAZIDE_t1 tg3_THIAZIDE_t2 tg3_THIAZIDE_t3 
			tg3_UTIS_pc tg3_UTIS_t1 tg3_UTIS_t2 tg3_UTIS_t3 
			tg3_VASODIL_pc tg3_VASODIL_t1 tg3_VASODIL_t2 tg3_VASODIL_t3 
			tg3_ceph_pc tg3_ceph_t1 tg3_ceph_t2 tg3_ceph_t3 
			tg3_OPIOID_pc tg3_OPIOID_t1 tg3_OPIOID_t2 tg3_OPIOID_t3 
			tg3_NONOPIOID_pc tg3_NONOPIOID_t1 tg3_NONOPIOID_t2 tg3_NONOPIOID_t3 
			tg2_MIGRAINE_pc tg2_MIGRAINE_t1 tg2_MIGRAINE_t2 tg2_MIGRAINE_t3 
			folate_antag_pc folate_antag_t1 folate_antag_t2 folate_antag_t3
			benzo_pc benzo_t1 benzo_t2 benzo_t3
			serotonin_pc serotonin_t1 serotonin_t2 serotonin_t3
			vasocon_pc vasocon_t1 vasocon_t2 vasocon_t3
			nmda_pc nmda_t1 nmda_t2 nmda_t3
			aspirin_pc aspirin_t1 aspirin_t2 aspirin_t3
			c3antiarr_pc c3antiarr_t1 c3antiarr_t2 c3antiarr_t3
			retin_pc retin_t1 retin_t2 retin_t3
			iron_pc iron_t1 iron_t2 iron_t3
			diab_pc diab_t1 diab_t2 diab_t3 tg3_VITA_pc tg3_VITA_t1 tg3_VITA_t2 tg3_VITA_t3 
			tg3_VITB_pc tg3_VITB_t1 tg3_VITB_t2 tg3_VITB_t3 tg3_VITC_pc tg3_VITC_t1 tg3_VITC_t2 tg3_VITC_t3 
			tg3_VITD_pc tg3_VITD_t1 tg3_VITD_t2 tg3_VITD_t3 tg3_VITE_pc tg3_VITE_t1 tg3_VITE_t2 tg3_VITE_t3 
			CONTRACEPT_HORMONE_pc CONTRACEPT_HORMONE_t1 CONTRACEPT_HORMONE_t2 CONTRACEPT_HORMONE_t3 
			tg2_CONTRACEPT_pc tg2_CONTRACEPT_t1 tg2_CONTRACEPT_t2 tg2_CONTRACEPT_t3 
			tg2_GYN_AI_pc tg2_GYN_AI_t1 tg2_GYN_AI_t2 tg2_GYN_AI_t3 
			tg2_MYOM_HORM_pc tg2_MYOM_HORM_t1 tg2_MYOM_HORM_t2 tg2_MYOM_HORM_t3 
			diab_any_time epil_any_time 
			any_presc any_terato_pc any_terato_t1 any_terato_t2 any_terato_t3;
			output out=pharms_flag_summ (drop=_type_ _freq_) max=;
	run;

	*Add the pharms flags back into the original dataset;
	proc sql;
		create table dlab.bd1_pharms_new as
		select *
		from bd_final_address as a left join pharms_flag_summ as b
		on a.snz_uid=b.snz_uid;
	quit;


********************************************************
********      Add data from 2013 census         ********
********************************************************;

	*For mothers: Smoking, qualifications, ethnicity;
	proc sql;
		create table bd1_cen_m as
		select a.*, b.cen_ind_smoke_regular_code as cen_m_smoke_regular_raw, b.cen_ind_smoke_ever_code as cen_m_smoke_ever_raw, b.cen_ind_std_highest_qual_code as cen_m_highest_qual,
		b.cen_ind_maori_eth_ind_code as cen_m_maori, b.cen_ind_asian_eth_ind_code as cen_m_asian, b.cen_ind_melaa_eth_ind_code as cen_m_melaa, b.cen_ind_pac_islnd_eth_ind_code as cen_m_pacific, 
		b.cen_ind_otr_eth_ind_code as cen_m_othereth, b.cen_ind_eur_eth_ind_code as cen_m_euro, b.snz_cen_uid as snz_cen_uid_m
		from dlab.bd1_pharms_new as a left join cen.census_individual as b
		on a.parent1_snz_uid=b.snz_uid;
	quit;


	*For fathers: Smoking, qualifications, ethnicity;
	proc sql;
		create table bd1_cen_f as
		select a.*, b.cen_ind_smoke_regular_code as cen_f_smoke_regular_raw, b.cen_ind_smoke_ever_code as cen_f_smoke_ever_raw, b.cen_ind_std_highest_qual_code as cen_f_highest_qual,
		b.cen_ind_maori_eth_ind_code as cen_f_maori, b.cen_ind_asian_eth_ind_code as cen_f_asian, b.cen_ind_melaa_eth_ind_code as cen_f_melaa, b.cen_ind_pac_islnd_eth_ind_code as cen_f_pacific, 
		b.cen_ind_otr_eth_ind_code as cen_f_othereth, b.cen_ind_eur_eth_ind_code as cen_f_euro, b.snz_cen_uid as snz_cen_uid_f
		from bd1_cen_m as a left join cen.census_individual as b
		on a.parent2_snz_uid=b.snz_uid;
	quit;



*****************************************
***   General tidying and recoding    ***
*****************************************;

	data bd1_tidy;
		set bd1_cen_f;

		*Crete flags for having a census record for mothers and fathers;
		if parent1_flag=0 then cen_m_flag=.;
		else if snz_cen_uid_m=. then cen_m_flag=0;
		else cen_m_flag=1;
		if parent2_flag=0 then cen_f_flag=.;
		else if snz_cen_uid_f=. then cen_f_flag=0;
		else cen_f_flag=1;

		*Recode smoking variables;
		if cen_m_smoke_regular_raw=2 then cen_m_smoke_regular=0;
		else if cen_m_smoke_regular_raw=7 then cen_m_smoke_regular=.;
		else if cen_m_smoke_regular_raw=9 then cen_m_smoke_regular=.;
		else cen_m_smoke_regular=cen_m_smoke_regular_raw;

		if cen_f_smoke_regular_raw=2 then cen_f_smoke_regular=0;
		else if cen_f_smoke_regular_raw=7 then cen_f_smoke_regular=.;
		else if cen_f_smoke_regular_raw=9 then cen_f_smoke_regular=.;
		else cen_f_smoke_regular=cen_f_smoke_regular_raw;

		if cen_m_smoke_ever_raw=2 then cen_m_smoke_ever=0;
		else if cen_m_smoke_ever_raw=7 then cen_m_smoke_ever=.;
		else if cen_m_smoke_ever_raw=9 and cen_m_smoke_regular=1 then cen_m_smoke_ever=1;
		else if cen_m_smoke_ever_raw=9 and cen_m_smoke_regular ne 1 then cen_m_smoke_ever=.;
		else cen_m_smoke_ever=cen_m_smoke_ever_raw;

		if cen_f_smoke_ever_raw=2 then cen_f_smoke_ever=0;
		else if cen_f_smoke_ever_raw=7 then cen_f_smoke_ever=.;
		else if cen_f_smoke_ever_raw=9 and cen_f_smoke_regular=1 then cen_f_smoke_ever=1;
		else if cen_f_smoke_ever_raw=9 and cen_f_smoke_regular ne 1 then cen_f_smoke_ever=.;
		else cen_f_smoke_ever=cen_f_smoke_ever_raw;

		if cen_m_smoke_regular=1 then cen_m_smoke_status='3';
		else if cen_m_smoke_ever=1 then cen_m_smoke_status='2';
		else if cen_m_smoke_ever=0 then cen_m_smoke_status='1';
		else cen_m_smoke_status='';

		if cen_f_smoke_regular=1 then cen_f_smoke_status='3';
		else if cen_f_smoke_ever=1 then cen_f_smoke_status='2';
		else if cen_f_smoke_ever=0 then cen_f_smoke_status='1';
		else cen_f_smoke_status='';

		*Recode ICD-10 CM groups so the regression analysis is easier to run (babies with CM that is not part of that group are coded to missing, 
		so analysis compares babies with specific CM vs non-CM babies);

		if case=1 and icd10_Q00Q07=0 then icd10_Q00Q07=.;
		else if case=0 then icd10_Q00Q07=0;
		if case=1 and icd10_Q20Q28=0 then icd10_Q20Q28=.;
		else if case=0 then icd10_Q20Q28=0;
		if case=1 and icd10_Q35Q37=0 then icd10_Q35Q37=.;
		else if case=0 then icd10_Q35Q37=0;
		if case=1 and icd10_Q38Q45=0 then icd10_Q38Q45=.;
		else if case=0 then icd10_Q38Q45=0;
		if case=1 and icd10_Q50Q56=0 then icd10_Q50Q56=.;
		else if case=0 then icd10_Q50Q56=0;
		if case=1 and icd10_Q60Q64=0 then icd10_Q60Q64=.;
		else if case=0 then icd10_Q60Q64=0;
		if case=1 and icd10_Q65Q79=0 then icd10_Q65Q79=.;
		else if case=0 then icd10_Q65Q79=0;

		*Rename and label ICD-10 groups;
		rename icd10_Q00Q07=nervous_bd;
		rename icd10_Q20Q28=circulatory_bd;
		rename icd10_Q35Q37=cleft_palate_bd;
		rename icd10_Q38Q45=digestive_bd;
		rename icd10_Q50Q56=genital_bd;
		rename icd10_Q60Q64=urinary_bd;
		rename icd10_Q65Q79=musculoskeletal_bd;

		label icd10_Q00Q07="nervous_bd";
		label icd10_Q20Q28="circulatory_bd";
		label icd10_Q35Q37="cleft_palate_bd";
		label icd10_Q38Q45="digestive_bd";
		label icd10_Q50Q56="genital_bd";
		label icd10_Q60Q64="urinary_bd";
		label icd10_Q65Q79="musculoskeletal_bd";

		*Create age groups (age at conception) for mother and father;
		format dia_m_age_grp_con $8.;
		if birth_rec_flag=0 then dia_m_age_grp_con='';
		else if dia_mother_age_con=. then dia_m_age_grp_con='';
		else if dia_mother_age_con<20 then dia_m_age_grp_con='under 20';
		else if dia_mother_age_con<25 then dia_m_age_grp_con='20-25';
		else if dia_mother_age_con<30 then dia_m_age_grp_con='25-29';
		else if dia_mother_age_con<35 then dia_m_age_grp_con='30-34';
		else if dia_mother_age_con<40 then dia_m_age_grp_con='35-39';
		else if dia_mother_age_con ge 40 then dia_m_age_grp_con='40 plus';

		format dia_f_age_grp_con $8.;
		if birth_rec_flag=0 then dia_f_age_grp_con='';
		else if dia_father_age_con=. then dia_f_age_grp_con='';
		else if dia_father_age_con<20 then dia_f_age_grp_con='under 20';
		else if dia_father_age_con<25 then dia_f_age_grp_con='20-25';
		else if dia_father_age_con<30 then dia_f_age_grp_con='25-29';
		else if dia_father_age_con<35 then dia_f_age_grp_con='30-34';
		else if dia_father_age_con<40 then dia_f_age_grp_con='35-39';
		else if dia_father_age_con ge 40 then dia_f_age_grp_con='40 plus';

		*Recode census highest qualification;
		format cen_m_quals $8.;
		if cen_m_highest_qual='' then cen_m_quals='';
		else if cen_m_highest_qual='00' then cen_m_quals='None';
		else if cen_m_highest_qual<'05' then cen_m_quals='School';
		else if cen_m_highest_qual<'97' then cen_m_quals='Tertiary';
		else cen_m_quals='';

		format cen_f_quals $8.;
		if cen_f_highest_qual='' then cen_f_quals='';
		else if cen_f_highest_qual='00' then cen_f_quals='None';
		else if cen_f_highest_qual<'05' then cen_f_quals='School';
		else if cen_f_highest_qual<'97' then cen_f_quals='Tertiary';
		else cen_f_quals='';

		*Convert mother's NZDEP into 5 groups;
		format nzdep_grp_m $4.;
		if nzdep2013_m=. then nzdep_grp_m='';
		else if nzdep2013_m le 2 then nzdep_grp_m='1-2';
		else if nzdep2013_m le 4 then nzdep_grp_m='3-4';
		else if nzdep2013_m le 6 then nzdep_grp_m='5-6';
		else if nzdep2013_m le 8 then nzdep_grp_m='7-8';
		else if nzdep2013_m le 10 then nzdep_grp_m='9-10';

		*Flag for whether participants were interviewed or not;
		if recruitment_result=1 then int_flag=1;
		else int_flag=0;

		*Recode census ethnicity variables;
		if cen_m_maori=2 then cen_m_maori=1;
		if cen_m_maori>2 then cen_m_maori='';
		if cen_m_pacific=2 then cen_m_pacific=1;
		if cen_m_pacific>2 then cen_m_pacific='';
		if cen_m_asian=2 then cen_m_asian=1;
		if cen_m_asian>2 then cen_m_asian='';
		if cen_m_euro=2 then cen_m_euro=1;
		if cen_m_euro>2 then cen_m_euro='';
		if cen_m_othereth=2 then cen_m_othereth=1;
		if cen_m_othereth>2 then cen_m_othereth='';
		if cen_m_melaa=2 then cen_m_melaa=1;
		if cen_m_melaa>2 then cen_m_melaa='';

		*Create prioritised ethnicity;
		format cen_m_pri_eth $8.;
		if cen_m_maori='' and cen_m_asian='' and cen_m_pacific='' and cen_m_melaa='' and cen_m_euro='' and cen_m_othereth='' then cen_m_pri_eth='';
		else if cen_m_maori='1' then cen_m_pri_eth='Maori';
		else if cen_m_pacific='1' then cen_m_pri_eth='Pacific';
		else if cen_m_asian='1' then cen_m_pri_eth='Asian';
		else cen_m_pri_eth='Other';

		drop snz_cen_uid_m snz_cen_uid_f dia_bir_birth_year_nbr dia_bir_birth_month_nbr cen_m_smoke_ever_raw cen_f_smoke_ever_raw 
		cen_m_smoke_regular_raw cen_f_smoke_regular_raw cen_m_highest_qual cen_f_highest_qual;

		*Recode blank pharm flags to zero (they are blank because no prescriptions were recorded. Exception is those not linked to birth record, 
		those should be left blank;

		%macro recode(tglevel,shortname);
			if birth_rec_flag=1 then do;
				if tg&tglevel._&shortname._pc=. then tg&tglevel._&shortname._pc=0;
				if tg&tglevel._&shortname._t1=. then tg&tglevel._&shortname._t1=0;
				if tg&tglevel._&shortname._t2=. then tg&tglevel._&shortname._t2=0;
				if tg&tglevel._&shortname._t3=. then tg&tglevel._&shortname._t3=0;
			end;
		%mend recode;

		%recode (1,ALIM);
		%recode (1,BLOOD);
		%recode (1,DERM);
		%recode (1,CARDIO);
		%recode (1,COMPOUND);
		%recode (1,GENURO);
		%recode (1,HORMONE);
		%recode (1,INFECT);
		%recode (1,MUSC);
		%recode (1,NERV);
		%recode (1,ONCO);
		%recode (1,RESP);
		%recode (1,SENS);
		%recode (1,FOOD);
		%recode (3,ACE);
		%recode (3,ACEDIU);
		%recode (3,ANGIO);
		%recode (3,ACNE);
		%recode (3,BETABLK);
		%recode (3,EPIL);
		%recode (3,CALCICHAN);
		%recode (3,ANTIPSYC);
		%recode (3,BAA1);
		%recode (3,BAA2);
		%recode (3,BAA3);
		%recode (3,BAA4);
		%recode (3,MCLIDES);
		%recode (3,WARFARIN);
		%recode (3,PENCLN);
		%recode (3,THIAZIDE);
		%recode (3,UTIS);
		%recode (3,VASODIL);
		%recode (3,VITA);
		%recode (3,CEPH);
		%recode (2,MIGRAINE);
		%recode (3,OPIOID);
		%recode (3,NONOPIOID);

		%macro recode2(name);
			if birth_rec_flag=1 then do;
				if &name._pc=. then &name._pc=0;
				if &name._t1=. then &name._t1=0;
				if &name._t2=. then &name._t2=0;
				if &name._t3=. then &name._t3=0;
				if any_presc=. then any_presc=0;
			end;
		%mend recode2;

		%recode2(folate_antag);
		%recode2(benzo);
		%recode2(serotonin);
		%recode2(vasocon);
		%recode2(nmda);
		%recode2(aspirin);
		%recode2(c3antiarr);
		%recode2(retin);
		%recode2(iron);
		%recode2(any_terato);

	run;


*Add flags for chronic conditions from MoH chronic condition table;
*Save final file to datalab folder;
proc sql;
create table dlab.bd1_final as
select a.*, b.moh_chr_condition_text as chr_condition, b.moh_chr_fir_incidnt_date as chr_condition_start, b.moh_chr_last_incidnt_date as chr_condition_last
from bd1_tidy as a left join moh.chronic_condition as b
on a.parent1_snz_uid=b.snz_uid;
quit;

*Write final file to sandpit;
proc sql;
create table sand.bd1_final as
select * from dlab.bd1_final
order by parent1_snz_uid;
quit;

*End of code;
