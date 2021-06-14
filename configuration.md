## Setting up environment
1. license master
2. deployment server
3. cluster master
4. indexer(s)
5. search head(s)

### configure SSL
TODO: find article that identifies usage of 2 different types of cert \
ref: https://docs.splunk.com/Documentation/Splunk/8.2.0/Security/SecureSplunkWebusingasignedcertificate \
ref: https://docs.splunk.com/Documentation/Splunk/8.2.0/Security/ConfigureSplunkforwardingtousesignedcertificates

1) Ensure ca chain cert path is correct otherwise certificate may not have worked properly and data to indexers may not work.
2) data certificate uses a different cert to configure different set of encryption algorithms for efficient usage of host resources

* web.conf
  * web certificate
* server.conf
  * data certificate for logs
* outputs.conf
  *  data certificate for universal fowarder


### deployment server
* attach to lm
* create default packages into deployment apps
  * outputs configuration
  * universal forwarder configuration
    * server.conf
      * disable management port
```
#Disable management port to prevent remote (or local) config.
#Set to UFW ONLY
[httpServer]
disableDefaultPort = true
```
.
    * outputs.conf
```
[tcpout]
defaultGroup = production_ssl
#defaultGroup = production_non_ssl

[tcpout:production_ssl]
server = idx1:9997, idx2:9997
disabled = 0
useSSL = true

[tcpout-server://idx1:9997]
clientCert = $SPLUNK_HOME/etc/apps/prod_outputs/local/uf_data_cert.pem
sslCertPath = $SPLUNK_HOME/etc/apps/prod_outputs/local/uf_data_cert.pem
sslRootCAPath = $SPLUNK_HOME/etc/apps/prod_outputs/local/certchain.pem
sslPassword = <ssl pw>
sslVerifyServerCert = true
sslCommonNameToCheck = idx1.localdomain
useClientSSLCompression = true
useACK = true

[tcpout-server://idx2:9997]
clientCert = $SPLUNK_HOME/etc/apps/prod_outputs/local/uf_data_cert.pem
sslCertPath = $SPLUNK_HOME/etc/apps/prod_outputs/local/uf_data_cert.pem
sslRootCAPath = $SPLUNK_HOME/etc/apps/prod_outputs/local/certchain.pem
sslPassword = <ssl pw>
sslVerifyServerCert = true
sslCommonNameToCheck = idx2.localdomain
useClientSSLCompression = true
useACK = true

[tcpout:production_non_ssl]
server=idx1.localdomain:9998,idx2.localdomain:9998
disabled = 1

[tcpout:dev]
server=dev:9997
disabled = 0


```
.
      * identifies possible destinations to send logs
  * indexes
    * indexes.conf
      * define indexes here instead of through search head ui
      * define volume size for hot,warm,cold/frozen indexes
        * maxTotalDataSizeMB
        * maxVolumeDataSizeMB
```
#for indexers
# One Volume for Hot and Cold
[volume:primary]
path = /hot
# ~70 GB
maxVolumeDataSizeMB = 70000

[volume:secondary]
path = /cold
# ~140 GB
maxVolumeDataSizeMB = 140000

[volume:_splunk_summaries]
path = /hot
# ~ 1GB
# maxVolumeDataSizeMB = 1000
```

```
# for search heads
# One Volume for Hot and Cold
[volume:primary]
path = /opt/splunk/var/lib/splunk

[volume:secondary]
path = /opt/splunk/var/lib/splunk

[volume:_splunk_summaries]
path = /opt/splunk/var/lib/splunk

```
.
    * define where to store indexes (if different from default)
    
```
### example
[osnix]
coldPath = volume:secondary/osnix/colddb
homePath = volume:primary/osnix/db
thawedPath = $SPLUNK_DB/osnix/thaweddb
```
  * license master
    * server.conf
```
[license]
master_uri = https://<lm url>:8089
```
.
    * define where to connect for license master
  * authentication
    * define where to connect for domain authentication
    * authentication.conf
```
[authentication]
authType = LDAP
authSettings = ad_auth

[cov_ad_auth]
SSLEnabled = 1
bindDN = CN=splunk_service,DC=localdomain
bindDNpassword = <splunk_service pw>
groupBaseDN = <path where splunk groups wil be>
groupMappingAttribute = dn
groupMemberAttribute = member
groupNameAttribute = cn
host = adserver.localdomain
port = 636
realNameAttribute = cn
userBaseDN = <path to where users are located>
userNameAttribute = sAMAccountName
timelimit = 15
network_timeout = 20
anonymous_referrals = 0

[roleMap_ad_auth]
admin = Splunk Admins
```
  * deployment client
    * define phone home interval to reduce network noise
    * deploymentclient.conf
```
[deployment-client]
# 15 minutes
phoneHomeIntervalInSecs = 900
```

### cluster master
* cm attached to deployment server; indexers attached to cluster master
* deployment server deploys apps to cm's apps
* cm deploys packages to indexers via cluster-apps
* create default packages for indexers
  * indexes.conf
```
[default]
repFactor = auto

[_introspection]
repFactor = 0

# ~50 GB
maxTotalDataSizeMB = 50000



# One Volume for Hot and Cold
[volume:primary]
path = /hot
# ~70 GB
maxVolumeDataSizeMB = 70000

[volume:secondary]
path = /cold
# ~140 GB
maxVolumeDataSizeMB = 140000

[volume:_splunk_summaries]
path = /hot
# ~ 1GB
# maxVolumeDataSizeMB = 1000
```
   * inputs.conf
```
# BASE SETTINGS
[splunktcp-ssl://9997]

[splunktcp://9998]

# SSL SETTINGS
# [SSL]
# rootCA = $SPLUNK_HOME/etc/auth/cacert.pem
# serverCert = $SPLUNK_HOME/etc/auth/server.pem
# password = password
# requireClientCert = false
# If using compressed = true, it must be set on the forwarder outputs as well.
# compressed = true
```
   * server.conf
```
[clustering]
mode = slave
master_uri = <cm url>:8089
pass4SymmKey = <p4skey>

# Provide a port on which the peers can chat.
[replication_port://9887]
disabled = false

[license]
master_uri = <lmurl>:8089
```
   * web.conf
```
# In clustered environments, all searches are brokered by the cluster master
# (cf. org_cluster_search_base). This means that connecting directly to an
# indexer to perform a search is ill advised (and may get inconsistent
# results). Further, the configuration of a clustered indexer is managed by
# the master itself (apply cluster-bundle), so this is another reason to
# disable the Splunk UI.

[settings]
startwebserver = false
```

### indexer
* attached to cluster master
* gets apps from cluster master
