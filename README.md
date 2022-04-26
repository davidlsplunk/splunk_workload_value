# splunk_workload_value
This repository stores the beginnings of Splunk Workload Value artifacts

Scheduled Search runs nightly. This creates/appends a lookup file called lkp_searchType_stats.csv
index=_audit action=search sourcetype=audittrail search_id!="rsa_*" (host=sh*.*splunk*.* OR host=si*.*splunk*.*) NOT user=cmon_user NOT user=internal_monitoring NOT user=ops_admin earliest=-1d@d latest=@d
| bin span=1d _time
| eval user = if(user="n/a", null(), user) 
| eval search_type = case( match(search_id, "^SummaryDirector_"), "summarization", match(savedsearch_name, "^_ACCELERATE_"), "acceleration", match(search_id, "^((rt_)?scheduler_|alertsmanager_)"), "scheduled", match(search_id, "\d{10}\.\d+(_[0-9A-F]{8}-[0-9A-F]{4}-[0-9A-F]{4}-[0-9A-F]{4}-[0-9A-F]{12})?$"), "ad hoc", savedsearch_name="","ad hoc", true(), "other") 
| eval search_type=case( match(savedsearch_name,"search\d"),"dashboard", match(search_id,"rt_*"),"realtime", match(search_id,"subsearch_*"),"subsearch", true(),search_type) 
| eval search=if(isnull(savedsearch_name) OR savedsearch_name=="", search, savedsearch_name) 
| stats min(_time) as _time, values(user) as user, values(_time) as time, values(info) as info, max(total_run_time) as total_run_time, first(search) as search, first(search_type) as search_type, first(apiStartTime) as apiStartTime, first(apiEndTime) as apiEndTime by search_id, host 
| where isnotnull(search) AND true() 
| search host=* (host=sh*.*splunk*.* OR host=si*.*splunk*.*) 
| stats dc(user) as count_user, dc(host) as count_host, median(total_run_time) as median_runtime, sum(total_run_time) as cum_runtime, count(search) as count, max(_time) as last_use by _time search_type 
| eval median_runtime = if(isnotnull(median_runtime), median_runtime, "-") 
| eval cum_runtime = if(isnotnull(cum_runtime), cum_runtime, "-") 
| eval last_use = strftime(last_use, "%m/%d/%Y %H:%M:%S %z") 
| fields _time, search_type, count, median_runtime, search_type, cum_runtime
| eval StartDate=strftime(_time,"%m/%d/%Y:%H:%M:%S")
| eval EndDate=strftime(relative_time(_time,"+1d"),"%m/%d/%Y:%H:%M:%S")
| sort - count 
| rename count_host as "Search Head Count", count as "Search Count", median_runtime as "Median Runtime", cum_runtime as "Cumulative Runtime", user as "User", host as "Host", search_type as "Search Type", count_user as "User Count"
