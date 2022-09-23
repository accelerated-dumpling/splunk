### reload/refresh app files on deployment server
`/opt/splunk/bin/splunk reload deploy-server -timeout 300`

### list output configuration on splunk host/client (for troubleshooting)
`/opt/splunk/bin/splunk btool outputs list --debug`

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
` setfacl -R -m u:splunk:r /var/log`

Note: may need to install *acl* packages
