
# Red Hat DCI (Distributed CI) Troubleshooting Guide

**Error-1:**

_DCI job runner is failing with the following error:_

```
Reinitialized existing Git repository in /var/lib/dci-openshift-app-agent/builds/2QK-TzsKq/1/telco-solutions/ericsson-cloud-ran-test-harness/.git/
error: Could not read 6412ec0a40a19ebf256b2b0b533347cb974782ec
fatal: pack has 131 unresolved deltas
fatal: fetch-pack: invalid index-pack output
....
ERROR: Job failed: exit status 1
```
_Git log error:_

This error indicates that there is a problem with the Git repository, specifically with the pack file that contains the commit history. The error message suggests that the pack file has unresolved deltas, which means that Git is unable to read some of the objects in the repository.

When running `git log`, you may encounter the following error:

```error: Could not read 6412ec0a40a19ebf256b2b0b533347cb974782ec
$ git log

error: Could not read 6412ec0a40a19ebf256b2b0b533347cb974782ec
fatal: Failed to traverse parents of commit 966115da2e534649081ab1837764d99a2b848a45

$ pwd
/var/lib/dci-openshift-app-agent/builds/2QK-TzsKq/1/telco-solutions/ericsson-cloud-ran-test-harness
```

**Solution-1:**

To resolve the issue, you can try the following steps:

1. Delete the temporary directory created by dci job runner that is causing the issue:

   ```
   sudo su - dci-openshift-app-agent
   rm -rf /var/lib/dci-openshift-app-agent/builds/2QK-TzsKq/
   ```