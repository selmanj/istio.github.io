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

The Istio control plane has a large set of custom resources that, when viewed
holistically, define the configuration of your service mesh. These resources
often have complex interactions that are not easily understood in isolation. For
example, to understand how a potential request might be routed to a service you
need to potentially understand which `Gateway` resource will handle the request;
you then need to consider how `VirtualService` resources will route that
request, and then how the `DestinationRule` will ultimately apply to the
answering service. As of Istio 1.4, custom resources are [defined with OpenAPI
specs](https://istio.io/news/releases/1.4.x/announcing-1.4/upgrade-notes/#configuration-management)
to help prevent common configuration mistakes. However these validations can
only be applied in the context of a single resource. To discover and diagnose
misconfigurations across multiple objects, we can use `istioctl analyze`.

If you haven't used it before, `istioctl analyze` is a command-line tool that
analyzes Istio configuration for problems. By default, it runs against the
default configured cluster specified in your `~/.kube/config` file, checking
resources on the api-server for files. While discovering such problems is useful,
it would be much better to catch these problems *before* they are applied to the
api-server. This article will show how to setup a workflow that will leverage
`istioctl analyze` to discover problems before they ever occur on your cluster.

## Treating configuration as code (GitOps)

{{< idea >}}
(Describe gitops approach to managing config. Link to
https://cloud.google.com/solutions/addressing-continuous-delivery-challenges-in-a-kubernetes-world)
or [https://queue.acm.org/detail.cfm?id=3237207]
{{< /idea >}}

The term 'GitOps' refers to the practice of treating configuration as code. More
specifically, this usually entails setting up a developer experience for your
configuration the same way you would for software. In general this usually
entails:

* Storing configuration in a version-control system
* Defining a workflow for proposing changes to configuration (often manifested
  via pull-requests)
* Checks and validations are automatically run against configuration changes, both
  before and after approval

GitOps workflows can be very detailed and advanced. In this article, we will
only be concerned with running checks on incoming changes. More specifically,
we'll show how to integrate `istioctl analyze` into a broader GitOps workflow. 

## Setting up a precommit

An easy first step to safer config validation is to setup a pre-commit hook for
your local git repository. This is a simple way to prevent committing code that
has problems to your repository, but please note that it only works locally; you'll
have to setup the same script on every repository you create. For that reason,
pre-commit hooks are best used as a simple sanity check for your own workflow,
and not as a means of enforcing policy.

This hook will be validating configuration the same way 

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
