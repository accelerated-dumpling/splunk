### input time (at universal forwarder)
ref:https://wiki.splunk.com/Where_do_I_configure_my_Splunk_settings%3F \
ref: https://docs.splunk.com/Documentation/Splunk/6.5.2/Admin/propsconf

* At first read ( CHARSET, CHECK_FOR_HEADER, INDEX_EXTRACTIONS, PREAMBLE_REGEX, FIELD_HEADER_REGEX, HEADER_FIELD_LINE_NUMBER, FIELD_DELIMETER, HEADER_FIELD_DELIMTER, FIELD_QUOTE, HEADER_FIELD_QUOTE, FIELD_NAMES, MISSING_VALUE_REGEX, JSON_TRIM_BRACES_IN_ARRAY_NAMES  )
* Linebreaking happens at parsing ( LINE_BREAKER, TRUNCATE). 
* Line Merging happens at merging ( BREAK_ONLY_BEFORE, MUST_BREAK_AFTER, SHOULD_LINEMERGE).
* Timestamping happens at typing ( DATETIME_CONFIG, TIME_FORMAT, TIME_PREFIX, MAX_TIMESTAMP_LOOKAHEAD)
* When processing binary files ( NO_BINARY_CHECK, detect_trailing_nulls )
* Checksum on checking changed data ( CHECK_METHOD )
* Source type configuration (sourcetype, invalid_cause, is_valid, unarchive_cmd, unarchive_sourcetype, LEARN_SOURCETYPE, LEARN_MODEL, maxDist, MORE_THAN, LESS_THAN)


ref: https://wiki.splunk.com/Community:HowIndexingWorks
 - Input    : Inputs data from source. Source-wide keys, such as source/sourcetypes/hosts, are annotated here. 
              The output of these pipelines are sent to the parsingQueue.
 - Parsing  : Parsing of UTF8 decoding, Line Breaking, and header is done here. This is the first place to split data stream into a single line event. 
              Note that in a UF/LWF, this parsing pipeline does "NOT" do parsing jobs. 
 - Merging  : Line Merging for multi-line events and Time Extraction for each event are done here.
 - Typing   : Regex Replacement, Punct. Extractions are done here.
 - IndexPipe: Tcpout to another Splunk, syslog output, and indexing are done here.
              In addition, this pipeline is responsible for bytequota, block signing, and indexing metrics such as thruput etc.


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
