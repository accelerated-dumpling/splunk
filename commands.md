### reload/refresh app files on deployment server
`/opt/splunk/bin/splunk reload deploy-server -timeout 300`

### list output configuration on splunk host/client (for troubleshooting)
`/opt/splunk/bin/splunk btool outputs list --debug`

### stale pid file
occasionally you'll get the error:

&nbsp; couldn't send SIGTERM to pid 2972: Operation not permitted
&nbsp; Couldn't send SIGTERM to some splunk helpers. [FAILED]
&nbsp; Error: Unable to stop splunk helpers.

You'll need to remove the stale pid at `$SPLUNK_HOME/var/run/splunk/splunkd.pid`


### check kvstore status
`/opt/splunk/bin/splunk show kvstore-status`

### clean kvstore (remove everything)
```
./splunk stop
./splunk clean kvstore --local
./splunk start
```

### check certificate information
`openssl x509 -in cert.pem -text`

check the enddate of the cert<br />
`openssl x509 -in cert.pem -enddate -noout`

### show decrypted
`./splunk show-decrypted --value '<value>'`


### ACL on /var/log 
` setfacl -R -m u:splunk:rx /var/log`

Note: may need to install *acl* packages


### clean index

```
./splunk stop
./splunk clean eventdata -index <index>
./splunk start
```
