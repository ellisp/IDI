* Set macro variables;
%let version=20151012;

* for IDI refresh version control;
%let date=15102015;

* for dataset version control;
%let sensor=31Dec2014;

* Global sensor data cut of date;
%let first_anal_yr=1988;

* first calendar year of analysis;
%let last_anal_yr=2014;

* Last calendar year of analysis;
%let firstage=0;

* start creating variables from birth to age 1;
%let lastage=25;
%let cohort_start=&first_anal_yr;

* start creating variables till age 24;
* Start of monthly array count;
%let start='01Jan1993'd;

* include A&I standard library and macros;
%include "\\wprdfs08\TreasuryData\MAA2013-16 Citizen pathways through human services\Common Code\Std_macros_and_libs\Std_macros.txt";
%include "\\wprdfs08\TreasuryData\MAA2013-16 Citizen pathways through human services\Common Code\Std_macros_and_libs\Std_libs.txt";
%include "\\wprdfs08\TreasuryData\MAA2013-16 Citizen pathways through human services\Common Code\Std_macros_and_libs\BDD_rules_macros.txt";
%include "\\wprdfs08\TreasuryData\MAA2013-16 Citizen pathways through human services\Common Code\Std_macros_and_libs\CYF_rules_macros.txt";
%include "\\wprdfs08\TreasuryData\MAA2013-16 Citizen pathways through human services\Common Code\Std_macros_and_libs\Education_rules_macros.txt";
%include "\\wprdfs08\TreasuryData\MAA2013-16 Citizen pathways through human services\Common Code\Std_macros_and_libs\Correction_rules_macros.txt";
%include "\\wprdfs08\TreasuryData\MAA2013-16 Citizen pathways through human services\Common Code\Std_macros_and_libs\get_mother.sas";
%include "\\wprdfs08\TreasuryData\MAA2013-16 Citizen pathways through human services\Common Code\Std_macros_and_libs\get_caregivers.sas";
libname Project "\\wprdfs08\TreasuryData\MAA2013-16 Citizen pathways through human services\SarahT\2015_2_cohortanalysis\test";

/*libname Project "\\wprdfs08\RODatalab\MAA2013-16  Citizen pathways through human services\B16\rerun_15102015";*/

* "01-Population definition" or "01_Youth population as at 2014" or other user defined code will creates the population. Once the population dataset is created all sbsequent codes require macro population. 
The statement below sets population macro;
%let population=project.population1988_2014;

proc sort data=&population;
	by snz_uid;
run;