# Queries
Random queries I found useful

## Core
### Get ingestion metrics for the last 7 days
idx = index\
b = bytes\
st = sourcetype\
h = host
```
index=_internal source=*license_usage.log TERM(type=Usage) earliest=-7d@d latest=@d 
| fields idx, b, st, h, _time, date_wday 
| eval h=if(len(h)=0 OR isnull(h),(SQUASHED),h) 
| bin _time span=1d 
| stats sum(b) AS b, dc(h) AS h by st idx date_wday 
| stats avg(b) as avg_b, avg(h) AS Host_Count_per_Day by idx st 
| eval Avg_MB_per_Day=round(avg_b/1024/1024,2) 
| eval Avg_GB_per_Day=round(avg_b/1024/1024/1024,2) 
| eval Avg_MB_per_Host=round(Avg_MB_per_Day/Host_Count_per_Day,2) 
| eval Host_Count_per_Day=round(Host_Count_per_Day,0) 
| rename st AS Sourcetype 
| table idx, Sourcetype, Avg_MB_per_Day, Avg_GB_per_Day, Host_Count_per_Day, Avg_MB_per_Host 
| sort idx 
```

### get logged in users
```
index=_internal sourcetype=splunkd_ui_access 
| dedup host user 
| table host user _time req_time logged in users rolling time
```

### get ufw version installed
```
index=_internal group=tcpin_connections version=* fwdType=uf 
| stats max(version) as version by hostname sourceIp guid build os arch 
| table hostname,sourceIp,guid,version,build,os,arch 
```

OR

```
index=_internal group=tcpin_connections version=* fwdType=uf 
| dedup hostname
| table hostname,sourceIp,guid,version,build,os,arch 
```


### Get host log usage
```
index=_internal source=*license_usage.log type=Usage 
| stats sum(b) as bytes by h 
| eval MB = round(bytes/1024/1024,1) 
| fields h MB 
| rename h as host 
| sort - MB
```

### Get raw log usage by index
```
index=*
 |eval raw_len=len(_raw)
 | eval raw_len_kb = raw_len/1024
 | eval raw_len_mb = raw_len/1024/1024
 | eval raw_len_gb = raw_len/1024/1024/1024
 | stats sum(raw_len) as Bytes sum(raw_len_kb) as KB sum(raw_len_mb) as MB sum(raw_len_gb) as GB by index
```

### Compare yesterday's data with today
```
index=oswinsec earliest=-1d@d 
| eval day=if(_time<relative_time(now(),"@d"),"yesterday","today")
| stats count(eval(day="yesterday")) as yesterday count(eval(day="today")) as today by host
```

### create a table of events by index for each hour for visualization
```
| tstats count where index=* by index _time span=1h prestats=t 
| timechart span=1h count by index useother=f
```
### get logged in users
```
| rest /services/authentication/httpauth-tokens splunk_server=local | stats max(updated) by userName
```

### Get sourcetypes by index
```
| eventcount summarize=false index=* 
| dedup index 
| fields index 
| map maxsearches=100 search=" | metadata type=sourcetypes index=\"$index$\" | eval index=\"$index$\"" 
| stats values(sourcetype) by index
```

### show latency of sourcetypes
```
index=* 
| eval time=_time 
| eval indextime=_indextime 
| eval latency=(indextime-time) 
| stats  avg(latency), min(latency), max(latency) by sourcetype
```

### compare todays (last 24h) event count by host to last week's
```
index=oswinsec earliest=-1d
| append 
    [ search index=oswinsec earliest=-8d latest=-7d ] 
| eval type=if(relative_time(now(), "-1d@d") > _time, "lastweek","now") 
| stats count by host type
| chart sum(count) as count by host type 
| fillnull value="0"
| sort - lastweek
```

Note: chart's syntax is: \[ **BY** \<row-split\> \<column-split\> \] OR \[ **OVER** \<row-split\> \] \[**BY** \<column-split\>\] . So in this case stats shows that each host is EITHER lastweek or now and there's a count. Chart will take the information and set the rows based on **hosts** and column of **type** with the sum at each "cell".

### Get index time (default is event time)
```
| eval indextime=strftime(_indextime,"%Y-%m-%d %H:%M:%S")
```

### Get time difference between index and event time
```
| eval t_indextime=strftime(_indextime,"%Y-%m-%d %H:%M:%S"),t_time=strftime(_time,"%Y/%m/%d %H:%M:%S"),t_delay = _indextime - _time | search t_delay > 100
```
### User disk usage
```
| rest splunk_server=local /services/search/jobs | eval diskUsageMB=diskUsage/1024/1024 | rename eai:acl.owner as UserName | stats sum(diskUsageMB) as totalDiskUsage by UserName
 ```
 
 ```
 index=_audit search_id="*" sourcetype=audittrail action=search info=granted 
| table _time host user action info search_id search ttl 
| join search_id 
    [ search index=_internal sourcetype=splunkd quota component=DispatchManager log_level=WARN reason="The maximum disk usage quota*" 
    | dedup search_id 
    | table _time host sourcetype log_evel component username search_id, reason 
    | eval search_id = "'" + search_id + "'" 
        ]
| table _time host user action info search_id search ttl reason
```
 
### Make fake results
```
| makeresults 
| eval data="index=netvuln sourcetype=rapid7:nexpose:asset nexpose_tags=\"owner=dba;owner=systems;category=pci;category=voip;bunit=3;os=3\"" 
| rex field=data "nexpose_tags=\"(?<nexpose_tags>[\w=;]+)\"" 
| rex max_match=0 field=nexpose_tags "category=(?<category>[\w]+)" | eval category=mvjoin(category,"|")
| rex max_match=0 field=nexpose_tags "owner=(?<owner>[\w]+)" | eval owner=mvjoin(owner,"|")
| rex max_match=0 field=nexpose_tags "bunit=(?<bunit>[\w]+)" | eval bunit=mvjoin(bunit,"|")
```

### Stdev numeric outliers
```
| tstats summariesonly=t count from datamodel=test.root where nodename=root.modify by root.Process_Name _time span=1h 
| `drop_dm_object_name("root")` 
| streamstats window=24 avg(count) as avg stdev(count) as stdev 
| eval multiplier = 2 
| eval lower_bound = avg - (stdev * multiplier) 
| eval upper_bound = avg + (stdev * multiplier) 
| eval outlier = if(count < lower_bound OR count > upper_bound, 1, 0) 
| table _time upper_bound lower_bound count outlier
```

### IQR
```
| tstats summariesonly=t count from datamodel=test.root where nodename=root.modify by root.Process_Name _time span=1h 
| `drop_dm_object_name("root")` 
| eventstats median(count) as median, p25(count) as p25, p75(count) as p75 
| eval IQR = p75 - p25 
| eval multiplier = 2 
| eval lower_bound = median - (IQR * multiplier) 
| eval upper_bound = median + (IQR * multiplier) 
| eval outlier = if(count < lower_bound OR count > upper_bound, 1, 0) 
| table _ time count lower_bound upper_bound outlier
```

### Email anayslis via stoq 
```
index=main sourcetype=stoq 
| eval results = spath(_raw, "results{}") 
| mvexpand results 
| eval path=spath(results, "archivers.filedir.path"), filename=spath(results, "payload_meta.extra_data.filename"), fullpath=path."/".filename 
| search fullpath!="" 
| table filename,fullpath
```

###  Get admins from Domain admins group via MS AD Objects
```
| inputlookup AD_Obj_User 
| search cn="*domain admin*" OU="Domain admins" 
| table member 
| makemv delim="CN=" member 
| rex field=member "(?<admins>[\w]+)," 
| fields - member 
| eval admins=upper(admins)
```

### Windows - Get logins by login type
```
index=oswinsec EventCode=4624 user!=DWM-* user!=UMFD-*
| eval LogonType=case(Logon_Type="2", "Local Console Access", Logon_Type="3", "Accessing Network Folders or Files", Logon_Type="4", "Scheduled Task, Batch File, or Script", Logon_Type="5", "Service Account", Logon_Type="7", "Local Console Unlock", Logon_Type="8", "Network User Logon", Logon_Type="9", "Program launched with RunAs using /netonly switch", Logon_Type="10", "Remote Desktop via Terminal Services", Logon_Type="11", "Mobile Access or Network Domain Connection Resumed") 
| table _time host user LogonType
```

## Enterprise Security

### Search for notable
```
index=notable 
| eval indexer_guid=replace(_bkt,".*~(.+)","\1"),event_hash=md5(_time._raw),event_id=indexer_guid."@@".index."@@".event_hash 
| eval rule_id=case(isnotnull(rule_id),rule_id,isnotnull(event_id),event_id,1=1,"unknown") 
| search rule_id=1DCA6754-AFAC-4D5D-A778-D1E1C0891BA7@@notable@@0baa56bc742901367a4ad5b8a65073f8
```

### ES lookup issue query
```
| inputlookup asset_lookup_by_str 
| eval es_lookup_type="asset_lookup_by_str" 
| inputlookup append=t asset_lookup_by_cidr 
| eval es_lookup_type=coalesce(es_lookup_type, "asset_lookup_by_cidr") 
| inputlookup append=t identity_lookup_expanded 
| eval es_lookup_type=coalesce(es_lookup_type, "identity_lookup_expanded") 
| rename _* AS es_lookup_* 
| eval es_lookup_mv_limit=25 
| foreach asset ip mac nt_host dns identity 
    [ eval count_<<FIELD>>=coalesce(mvcount(<<FIELD>>), 0), es_lookup_is_problem=mvappend(es_lookup_is_problem, if(count_<<FIELD>> >= es_lookup_mv_limit, "yes - field <<FIELD>> has over ". es_lookup_mv_limit . " entries", null()))] 
| where isnotnull(es_lookup_is_problem) 
| table es_lookup_*, count_*, *
```

### Get datamodel usage
Get the data models and tags within ES
```
| rest /services/data/models
| search NOT title=internal_* NOT title=Splunk_*
| spath input=eai:data output=foo objects{}
| fields title, eai:acl.app, acceleration, foo, eai:data
| mvexpand foo
| spath input=foo output=parent parentName
| search parent=BaseEvent
| spath input=foo constraints{}.search
| fields title, eai:acl.app constraints{}.search
```
For each datamodel and tag run (constraings[].search):
```
(`cim_Alerts_indexes`) tag=alert 
| eval raw_size=len(_raw)
| stats sum(raw_size) as totalSize_B
| eval totalSize_GB=totalSize_B/1024/1024/1024
| fields totalSize_GB
```

### Update intel list to remove an entry
```
| inputlookup ip_intel
| search ip!=x.x.x.x
| outputlookup ip_intel
```

### Rare processes
```
| tstats `security_content_summariesonly` count values(Processes.dest) as dest values(Processes.user) as user min(_time) as firstTime max(_time) as lastTime from datamodel=Endpoint.Processes by Processes.process_name 
| rename Processes.process_name as process 
| rex field=user "(?<user_domain>.*)\\\\(?<user_name>.*)" 
| `security_content_ctime(firstTime)` 
| `security_content_ctime(lastTime)` 
| search 
    [| tstats count from datamodel=Endpoint.Processes by Processes.process_name 
    | rare Processes.process_name limit=30 
    | rename Processes.process_name as process 
    | `filter_rare_process_whitelist` 
    | table process ]
```

### get AD DN activity (rw) by user
```
index=oswinsec EventCode=5136
| bin _time span=5m
| stats count by _time gpo_obj_dn user
```

### kv store metrics
```
| rest /services/server/introspection/kvstore/collectionstats
| mvexpand data
| spath input=data
| rex field=ns "(?<App>.*)\.(?<Collection>.*)"
| eval dbsize=round(size/1024/1024, 2)
| eval indexsize=round(totalIndexSize/1024/1024, 2)
| stats first(count) AS "Number of Objects" first(nindexes) AS Accelerations first(indexsize) AS "Acceleration Size (MB)" first(dbsize) AS "Collection Size (MB)" by App, Collection
```

### appendpipe / outputlookup
This query modifies the data prior to it, outputs to the lookup and then discards the data modified then continues on the query as if the appendpipe portion never happens. This is useful for performing any adhoc updates to lookups inline instead of creating a separate search for it.
```
<query>
| appendpipe [
 | inputlookup append=true mydata.csv
 | stats sum(bytes) as bytes latest(_time) as _time by user
 | where relative_time(now(), "-7d") < _time
 | outputlookup mydata.csv
 | where a=b
]
```
Note: the condition `a=b` is never true thus invalidates this whole portion and nothing actually gets appended to the parent query.
