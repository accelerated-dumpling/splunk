
ref:https://wiki.splunk.com/Where_do_I_configure_my_Splunk_settings%3F

### DA vs SA vs TA
ref: https://dev.splunk.com/enterprise/docs/devtools/enterprisesecurity/abouttheessolution/

* DAs typically contain dashboards and other views, along with search objects that populate them.
* SAs can contain a variety of files but typically do not contain data inputs.
* TAs often contain data inputs, as well as files that help normalize and prepare that data for display in Enterprise Security.


### Precedence
ref: https://docs.splunk.com/Documentation/Splunk/8.2.0/Admin/Wheretofindtheconfigurationfiles

#### Global vs App context
Global context refers to any configuration that would apply to indexing or monitoring (ingestion) behavior
```
admon.conf
authentication.conf
authorize.conf
crawl.conf
deploymentclient.conf
distsearch.conf
indexes.conf
inputs.conf
limits.conf, except for indexed_realtime_use_by_default
outputs.conf
pdf_server.conf
procmonfilters.conf
props.conf -- global and app/user context
pubsub.conf
regmonfilters.conf
report_server.conf
restmap.conf
searchbnf.conf
segmenters.conf
server.conf
serverclass.conf
serverclass.seed.xml.conf
source-classifier.conf
sourcetypes.conf
sysmon.conf
tenants.conf
transforms.conf  -- global and app/user context
user-seed.conf -- special case: Must be located in /system/default
web.conf
wmi.conf
```

App context refers to search related functionalities or items specific to an app
```
alert_actions.conf
app.conf
audit.conf
commands.conf
eventdiscoverer.conf
event_renderers.conf
eventtypes.conf
fields.conf
literals.conf
macros.conf
multikv.conf
props.conf -- global and app/user context
savedsearches.conf
tags.conf
times.conf
transactiontypes.conf
transforms.conf  -- global and app/user context
user-prefs.conf
workflow_actions.conf
```

#### Precedence within global context
1. System local directory -- highest priority
2. App local directories
3. App default directories
4. System default directory -- lowest priority

#### Precedence within global context, indexer cluster peers only
1. Slave-app local directories -- highest priority
2. System local directory
3. App local directories
4. Slave-app default directories
5. App default directories
6. System default directory -- lowest priority

#### Precedence within app or user context
1. User directories for current user -- highest priority
2. App directories for currently running app (local, followed by default)
3. App directories for all other apps (local, followed by default) -- for exported settings only
4. System directories (local, followed by default) -- lowest priority

#### Search time
ref: https://docs.splunk.com/Documentation/Splunk/8.2.0/Knowledge/Searchtimeoperationssequence
0. TRANSFORMS (actually index time)
1. EXTRACT
2. REPORT
3. KV-Mode (kv extractions)
4. FIELDALIAS
5. EVAL
6. LOOKUP
7. eventtypes.conf
8. tags.conf
