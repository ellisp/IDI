
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


* Creating clean workable CYF tables will create 
CYF_intake_clean
CYF_abuse_clean
CYF_abuse_clean;


;
%macro Create_clean_CYF_tables;
proc sql;
create table CYF_intake as select 
	snz_uid,
	snz_composite_event_uid,
	input(compress(cyf_ine_event_from_date_wid_date,"-"),yymmdd10.) as event_start_date,
	input(compress(cyf_ine_event_to_date_wid_date,"-"),yymmdd10.) as event_end_date
from cyf.CYF_intakes_event 
order by snz_composite_event_uid;

proc sql;
create table CYF_intake1 as select 
	a.*,
	b.cyf_ind_business_area_type_code as Business_Area,
	b.cyf_ind_notifier_role_type_code as NOTIFIER_ROLE_GROUP				
from CYF_intake a left join cyf.cyf_intakes_details b
on a.snz_composite_event_uid=b.snz_composite_event_uid
order by snz_uid;

data CYF_intake_clean; 
set CYF_intake1; 
format event_start_date event_end_date date9.;
if event_start_date>"&sensor."D then delete;
if event_end_date>"&sensor."D then event_end_date="&sensor."D;
run;



* CYF abuse findings;
proc sql;
create table CYF_abuse as select 
	snz_uid,
	snz_composite_event_uid,
	cyf_abe_source_uk_var2_text as abuse_type,
	input(compress(cyf_abe_event_from_date_wid_date,"-"),yymmdd10.) as finding_date
from cyf.CYF_abuse_event 
order by snz_composite_event_uid;

* CYF notifications and YJ refferals, where  Police Family Violence notifications identfiied through notifier_role_type_code PFV;
proc sql;
create table CYF_abuse1 as select 
	a.*,
	b.cyf_abd_business_area_type_code as Business_Area
	from CYF_abuse a left join cyf.cyf_abuse_details b
on a.snz_composite_event_uid=b.snz_composite_event_uid
order by snz_uid;

data CYF_abuse_clean; set CYF_abuse1;
format finding_date date9.;
if finding_date>"&sensor."D then delete;
if abuse_type="NTF" then delete;* abuse not found;
drop snz_composite_event_uid;
run;

proc sort data=CYF_abuse_clean nodupkey; 
by snz_uid abuse_type finding_date Business_Area; run;

* CYF placements;

proc sql;
create table CYF_place as select 
	snz_uid,
	cyf_ple_event_type_wid_nbr,
	snz_composite_event_uid,
	cyf_ple_number_of_days_nbr as event_duration,
	input(compress(cyf_ple_event_from_date_wid_date,"-"),yymmdd10.) as event_start_date,
	input(compress(cyf_ple_event_to_date_wid_date,"-"),yymmdd10.) as event_end_date
from cyf.cyf_placements_event 
order by snz_composite_event_uid;

proc sql;
create table CYF_place1 as select 
	a.*,
	b.cyf_pld_business_area_type_code as Business_Area,
	b.cyf_pld_placement_type_code as placement_type				
from CYF_place a left join cyf.cyf_placements_details b
on a.snz_composite_event_uid=b.snz_composite_event_uid
order by snz_uid;

data CYF_place_clean; 
set CYF_place1;
format event_start_date event_end_date date9.;
if event_start_date>"&sensor."D then delete;
if event_end_date>"&sensor."D then event_end_date="&sensor."D;
run;
%mend;













%macro CYF_notifications(ref_id,dateofbirth,st,end,prefix,suffix);

* this is expected to be called in a datastep where a dataset of CYF/YJ events
* are being processed. Can be events relating to the individual themselves
* or it might be relating to other people (eg their siblings/caregivers) ;

* creates counts of CYF notifications, police FV notifications and YJ referrals;
* for an interval  starting at dateofbirth + st years;
*                  ending at dateofbirth + end years;

* prefix - is the first word in the name of the variables created;
*          can be anything but we use child = for reference child  
*          or othchild = sibling;  
* suffix - is the last part of the name of variables created;
*          can be anything but at_age, or by_age or at_birth are 
*          what was expected ;  


  			if first.&ref_id. then do;

			                        &prefix._not_&suffix. =0;
			                        &prefix._Pol_FV_not_&suffix. =0;
			                      	&prefix._YJ_referral_&suffix. =0;

			end;

			if Business_Area in ('CNP','UNK') and 
			intnx('YEAR',&dateofbirth., &st.,'sameday') lt 
             event_Start_Date le intnx('YEAR',&dateofbirth., &end.,'sameday') then do;
                       if NOTIFIER_ROLE_GROUP	 not in ('PFV') then do;
			                      &prefix._not_&suffix. + 1;
			                                end;
			end;

			if Business_Area in ('CNP','UNK') and 
           intnx('YEAR',&dateofbirth., &st.,'sameday') lt event_Start_Date le
                     intnx('YEAR',&dateofbirth., &end.,'sameday') then do;

			                        if NOTIFIER_ROLE_GROUP	 in ('PFV') then do;
			                                &prefix._Pol_FV_not_&suffix. + 1;
			                        end;
			end;

			if Business_Area in ('YJU') and 
         intnx('YEAR',&dateofbirth., &st.,'sameday') lt event_start_Date 
           le intnx('YEAR',&dateofbirth., &end.,'sameday') then do;
								&prefix._YJ_referral_&suffix.+1;
			end;

%mend;

%macro cyf_findings(ref_id,dateofbirth,st,end,prefix,suffix);

                if first.&ref_id. then do;

                        &prefix._fdgs_neglect_&suffix.=0;
                        &prefix._fdgs_phys_abuse_&suffix.=0;
                        &prefix._fdgs_sex_abuse_&suffix.=0;
                        &prefix._fdgs_emot_abuse_&suffix.=0;
                        &prefix._fdgs_behav_rel_&suffix.=0;
						&prefix._fdgs_sh_suic_&suffix.=0;
						&prefix._any_fdgs_abuse_&suffix.=0;
                end;

                if intnx('YEAR',&dateofbirth., &st.,'sameday') 
                   lt finding_date le intnx('YEAR',&dateofbirth., &end.,'sameday') 
          then do;

                        if abuse_type='BRD' then do;
                                &prefix._fdgs_behav_rel_&suffix. + 1;
                        end;
                        if abuse_type='EMO' then do;
                                &prefix._fdgs_emot_abuse_&suffix. + 1;
                        end;
                        if abuse_type='NEG' then do;
                                &prefix._fdgs_neglect_&suffix. + 1;
                        end;
                        if abuse_type='PHY' then do;
                                &prefix._fdgs_phys_abuse_&suffix. + 1;
                        end;
                        if abuse_type='SEX' then do;
                                &prefix._fdgs_sex_abuse_&suffix. + 1;
                        end;
						if abuse_type in ('SHS', 'SHM','SUC') then do;
                                &prefix._fdgs_sh_suic_&suffix. + 1;
                        end;
						if abuse_type in ('EMO','NEG','PHY','SEX') then do;
                                &prefix._any_fdgs_abuse_&suffix. + 1;
                        end;

                end;
%mend;
%macro cyf_placements(ref_id,dateofbirth,st,end,prefix,suffix);

			if first.&ref_id. then do;

			                        &prefix._CYF_place_&suffix.=0;
									&prefix._YJ_place_&suffix.=0;
			end;

			if intnx('YEAR',&dateofbirth., &st.,'sameday') lt 
event_Start_Date le intnx('YEAR',&dateofbirth.,&end.,'sameday') 
and event_duration ge 28 and business_area in('CNP','UNK') then do;

			                      &prefix._CYF_place_&suffix. + 1;
			                                
			end;
						if intnx('YEAR',&dateofbirth., &st.,'sameday') 
lt event_Start_Date le intnx('YEAR',&dateofbirth., &end.,'sameday') 
and event_duration ge 28 and business_area='YJU' then do;

			                      &prefix._YJ_place_&suffix. + 1;
			                                
			end;
%mend;


%macro overlap_days(start1,end1,start2,end2,days);

* four cases where there is some overlap: 
spell2 is within spell1 
spell2 finishes after spell1 
spell1 is within spell2 
spell1 starts after spell2 ;
if not ((&end1.<&start2.) or (&start1.>&end2.)) then do;
   if (&start1. <= &start2.) and  (&end1. >= (&end2.)) then &days.=(&end2.-&start2.)+1;
   else if (&start1. <= &start2.) and  (&end1. <= (&end2.)) then &days.=(&end1.-&start2.)+1;
   else if (&start1. >= &start2.) and  (&end1. <= (&end2.)) then &days.=(&end1.-&start1.)+1;
   else if (&start1. >= &start2.) and  (&end1. >= (&end2.)) then &days.=(&end2.-&start1.)+1;
end;
else  &days.=0;
%mend;




%macro FGC_FWA_count(ref_id, dateofbirth, st, end, prefix, suffix);
	* this is expected to be called in a datastep where a dataset of CYF/YJ events
	* are being processed. Can be events relating to the individual themselves
	* or it might be relating to other people (eg their siblings/caregivers);

	* creates counts of CYF notifications, police FV notifications and YJ referrals;
	* for an interval  starting at dateofbirth + st years;
	*                  ending at dateofbirth + end years;
	* prefix - is the first word in the name of the variables created;

	*          can be anything but we use child = for reference child  
	*          or othchild = sibling;

	* suffix - is the last part of the name of variables created;

	*          can be anything but at_age, or by_age or at_birth are 
	*          what was expected;
	if first.&ref_id. then
		do;
			&prefix._CP_FGC_&suffix. = 0;
			&prefix._CP_FWA_&suffix. = 0;
			&prefix._YJ_FGC_&suffix. = 0;
		end;

	if business_area_type_code in ('CNP','UNK') and intervention = 'FGC' and
		intnx('YEAR',&dateofbirth., &st.,'sameday') lt 
		startdate le intnx('YEAR', &dateofbirth., &end.,'sameday') then
			&prefix._CP_FGC_&suffix. + 1;

	if business_area_type_code in ('CNP','UNK') and intervention = 'FWA' and
		intnx('YEAR',&dateofbirth., &st.,'sameday') lt 
		startdate le intnx('YEAR', &dateofbirth., &end.,'sameday') then
			&prefix._CP_FWA_&suffix. + 1;

	if business_area_type_code in ('YJU')and intervention = 'FGC' and 
		intnx('YEAR',&dateofbirth., &st.,'sameday') lt startdate 
		le intnx('YEAR',&dateofbirth., &end.,'sameday') then
			&prefix._YJ_FGC_&suffix.+1;
%mend FGC_FWA_count;