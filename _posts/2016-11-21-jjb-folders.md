---
layout: post
title: "Folders support in Jenkins Job Builder"
description: "How to put the Jenkins job to the folder?"
---

If you want to create and update your Jenkins Job in some folder
using [Jenkins Job Builder (jjb)][jjb], you need to put the name of the
folder into the name of the job:

```yaml
- job:
    name: "developer/zar/buildroot"
```

**Note**: at this moment Jenkins Job Builder isn't able to create
folder on demand by itself. This feature is not supported yet, but it
will be sometimes, I hope. You can track the progress on this feature
in Gerrit [here](https://review.openstack.org/#/c/134307/).

[jjb]: http://docs.openstack.org/infra/jenkins-job-builder/
