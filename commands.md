### reload/refresh app files on deployment server
`/opt/splunk/bin/splunk reload deploy-server -timeout 300`

### list output configuration on splunk host/client (for troubleshooting)
`/opt/splunk/bin/splunk btool outputs list --debug`

### check kvstore status
`/opt/splunk/bin/splunk show kvstore-status`

### check certificate information
`openssl x509 -in cert.pem -text`

check the enddate of the cert
`openssl x509 -in cert.pem -enddate -noout`
