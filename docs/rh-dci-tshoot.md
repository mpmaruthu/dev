
# Red Hat DCI Troubleshooting Guide
Error-1: Git Log Error

When running `git log`, you may encounter the following error:

```error: Could not read 6412ec0a40a19ebf256b2b0b533347cb974782ec
$ git log

error: Could not read 6412ec0a40a19ebf256b2b0b533347cb974782ec
fatal: Failed to traverse parents of commit 966115da2e534649081ab1837764d99a2b848a45

$ git branch -a
$ git pull origin dev
fatal: unable to access 'https://gitlab.consulting.redhat.com/telco-solutions/ericsson-cloud-ran-test-harness.git/':
error setting certificate verify locations:
CAfile: /var/lib/dci-openshift-app-agent/builds/2QK-TzsKq/1/telco-solutions/ericsson-cloud-ran-test-harness.tmp/CI_SERVER_TLS_CA_FILE
CApath: none
$ pwd
/var/lib/dci-openshift-app-agent/builds/2QK-TzsKq/1/telco-solutions/ericsson-cloud-ran-test-harness
```
This error indicates that the Git repository is in a bad state, possibly due to a corrupted commit or an issue 
with the certificate verification.

To resolve this issue, you can try the following steps:
