---
title: Using istioctl analyze with GitOps
subtitle: 
description: 
publishdate: 2020-01-05
attribution: Joe Selman (Google)
keywords: []
target_release: 1.5
---

{{< idea >}}
This is work in progress - notes are contained here.
{{< /idea >}}


## What is istioctl analyze
{{< idea >}}
(Describe analyze, what its for, how to use it, etc)
{{< /idea >}}

## Gitops
{{< idea >}}
(Describe gitops approach to managing config. Link to https://cloud.google.com/solutions/addressing-continuous-delivery-challenges-in-a-kubernetes-world)
{{< /idea >}}

## Setting up a precommit

An easy first step to safer config validation is to setup a pre-commit hook for
your local git repository. This is a simple way to prevent committing code that
has problems to your repository, but note that it only works locally; you'll
have to setup the same script on every repository you create. For that reason,
pre-commit hooks are best used as a simple sanity check for your own workflow,
and not as a means of enforcing policy.

Here's an example file you can place in your repository:

{{< text shell >}}
#!/bin/sh
#
# Example pre-commit to run istioctl analyze on commit.

# Path to istioctl - replace with hardcoded path if you want to use a different binary.
ISTIOCTL=istioctl
# Directory containing config to analyze.
ANALYZE_DIR="cluster/"

if ! [ -x "`command -v $ISTIOCTL`" ]; then
    echo "$ISTIOCTL command not found. Either add it to your path, or modify your pre-commit hook to specify the absolute path."
    exit 1
fi

find $ANALYZE_DIR \
    -type f \
    -name \*.yaml \
    -print0 | xargs -0 $ISTIOCTL analyze
{{< /text >}}

If you named this file `istio-precommit`, you can then place it in the appropriate place with a symbolic link.

{{< text bash >}}
$ ln -s istio-precommit .git/hooks/pre-commit
{{< /text>}}

Now, whenever you commit a change to your repository, the `cluster/` directory will be automatically analyzed. You can try this out by introducing an error:

{{< idea >}}
This was tested with istioctl 1.5.0-prerelease which did not support -d and has slightly different output. Review once the next pre-release is published.
{{< /idea >}}
{{< text bash >}}
$ cat <<EOF > cluster/my-bad-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-test-service
  annotations:
    a-non-existent-annotation.istio.io/not-here: test
EOF
$ git add cluster/my-bad-service && git commit -m "testing analyze pre-commit"
Warn [IST0108] (Service my-test-service.default) Unknown annotation: a-non-existent-annotation.istio.io/not-here
Error: Analyzers found issues.
See https://istio.io/docs/reference/config/analysis for more information about causes and resolutions.
{{< /text> }}
## Read more
* (Link to analyzer docs)
* (Link to https://cloud.google.com/solutions/addressing-continuous-delivery-challenges-in-a-kubernetes-world)
