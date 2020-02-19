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

Also prepare github.com/selmanj/example-istio-cluster for public consumption.

Also note somewhere that we're assuming istioctl analyze 1.5.
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

Let's assume for now that we have the following structure in our git repository:

{{< bash >}}
$ tree .
.
├── cluster
│   ├── namespaces
│   │   ├── bookinfo.yaml
│   │   └── hipster.yaml
│   └── workloads
│       ├── bookinfo
│       │   └── bookinfo.yaml
│       └── hipstershop
│           └── kubernetes-manifests.yaml
{{< /bash >}}

We've got resources split up by namespaces and workloads, with two apps deployed
to our cluster: a [hipster-shop demo app](https://github.com/GoogleCloudPlatform/microservices-demo)
and the BookInfo app contained within the Istio samples folder. Within each yaml
file is config representing both the workload as well as Istio-specific configuration.

## Setting up a precommit

An easy first step to safer config validation is to setup a pre-commit hook for
your local git repository. This is a simple way to prevent committing code that
has problems to your repository, but please note that it only works locally; you'll
have to setup the same script on every repository you create. For that reason,
pre-commit hooks are best used as a simple sanity check for your own workflow,
and not as a means of enforcing policy.

Before installing the hook, it's a good idea to make sure we don't have any
problems contained in our configuration. Let's invoke `istioctl analyze` in
file-only mode.

{{< bash >}}
$ istioctl analyze --recursive cluster/ \
    --use-kube false \
    --all-namespaces \
    --failure-threshhold Info
✔ No validation issues found when analyzing all namespaces.
{{< /bash >}}

This command analyzes the files contained in the `cluster/` folder recursively. We also have
specified the `--use-kube false` argument, which tells analyze to analyze only
files. Without this option, it would additionally try to analyze our currently-configured
default cluster. It's safe to leave out if you want to always analyze the
current live cluster on every commit, but for now we'll disable it and show how
to add live-cluster analysis as a CI-step.

The `--all-namespaces` argument will ensure we check all namespaces. Finally,
the `--failure-threshhold Info` defines what sort of problems we consider a
failure (valid choices are `Error`, `Warn`, or `Info`). By specifying `Info` we
are treating any problem as a failure - feel free to adjust this to your own
situation.

Note that the analyzer found no issues, so we can go ahead and install the
pre-commit hook. Save the contents below to a file named `pre-commit.example`:

{{< text shell >}}
#!/bin/bash
#
# Example pre-commit to run istioctl analyze on commit. To use, add to your git
# precommit hook via `ln -s ../../pre-commit.example .git/hooks/pre-commit`.
 
# Path to istioctl - replace with hardcoded path if you want to use a different binary.
ISTIOCTL=$HOME/bin/istioctl
# Failure threshhold - valid values are Info/Warn/Error
FAILURE_THRESHHOLD=Info

if ! [ -x "$(command -v $ISTIOCTL)" ]; then
    echo "$ISTIOCTL command not found. Either add it to your path, or modify"
    echo "your pre-commit hook to specify the absolute path."
    exit 1
fi

$ISTIOCTL analyze -R cluster/ \
    --use-kube=false \
    --all-namespaces \
    --failure-threshold $FAILURE_THRESHHOLD
{{< /text >}}

You can then install the hook with a symbolic link:

{{< text bash >}}
$ ln -s ../../pre-commit.example .git/hooks/pre-commit
{{< /text>}}

Now, whenever you commit a change to your repository, the `cluster/` directory
will be automatically analyzed. Let's introduce an error to test it out. Let's
expose the `productpage` service via a `Gateway` resource; however we'll
introduce a typo in the name of the gateway bound in the `VirtualService`:

Place the below contents in `cluster/workloads/bookinfo/bookinfo-gateway.yaml`:
{{< text yaml >}}
# cluster/workloads/bookinfo/bookinfo-gateway.yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
  namespace: bookinfo
spec:
  selector:
    istio: ingressgateway 
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo-vs
  namespace: bookinfo
spec:
  hosts:
  - "*"
  gateways:
  - bookinfogateway
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        prefix: /static
    - uri:
        exact: /login
    - uri:
        exact: /logout
    - uri:
        prefix: /api/v1/products
    route:
    - destination:
        host: productpage
        port:
          number: 9080
{{< /text >}}

Note that the gateway is named `bookinfo-gateway`, but the virtual service
refers to it as `bookinfogateway`. This sort of error is very easy to make and
difficult to spot.

If you try to commit the file now, you should see an error:

{{< text bash >}}
~/s/example-istio-cluster (master|…) $ git add cluster/workloads/bookinfo/bookinfo-gateway.yaml && git commit -m "Expose bookinfo's productpage via ingress"
Error [IST0101] (VirtualService bookinfo-vs.bookinfo) Referenced gateway not found: "bookinfogateway"
Error: Analyzers found issues when analyzing all namespaces.
See https://istio.io/docs/reference/config/analysis for more information about causes and resolutions.
{{< /<text>}}

From the output above we can discern:

* There was a single error with the error code `IST0101`
* The resource under question is of kind `VirtualService`, named `bookinfo-vs`
  in the `bookinfo` namespace
* The gateway that wasn't found was named `bookinfogateway`

With the above information, you should be able to discover the typo and correct
the issue.

Installing the pre-commit hook allowed you to discover the missing
reference before applying the problematic configuration to your cluster.
However, this only works for your local git repository; the next step is to
begin enforcing this check via pull-requests.

## istioctl analyze as a continuous integration step

Shown here is an example of using `istioctl analyze` as a continuous integration
check. [GitHub actions](https://help.github.com/en/actions/getting-started-with-github-actions/about-github-actions)
is documented in this article; the actual integration of `istioctl analyze` should
be straightforward to generalize to other continuous integration platforms. The
directions below assume you are storing your cluster configuration in a GitHub
repo and have enabled GitHub actions.

Create a file in your repository named `.github/workflows/analyze.yml` with the
contents:
{{< text yaml >}}
name: Analyze CI

on:
  pull_request:
    branches: 
    - master

jobs:
  analyze:
    name: Analyze (local)
    runs-on: [ubuntu-latest]

    steps:
    - uses: actions/checkout@v2
    - uses: ./.github/actions/analyze
{{/< text >}}



{{< idea >}}
NOTES

* Document CI step, github action
  * use example-istio-cluster
  * show example of bringing in a large, vendor provided app (gitlab?)
    * with large app not really free to modify yaml (or is undersirable) so show
      how to suppress rule instead (port-names are unlabeled and it might be bad
      to label them)
    * also show suppressing via annotation (but need better example than
      port-name, maybe)
  * show example of exporting service to default namespace
    * there's another app already in the cluster exposing the same name to
      default!
    * that team didn't check in their app (those jerks)
    * with a CI step that analyzes live cluster AND PRs, you can catch issue
      before it occurs (can't expose two different services to the same name) -
      cool shit
    * CI step is also useful for migrating to gitops
* General observations about how objects can be inconsistent, so messages aren't
  necessarily stable over time - might make sense to run live cluster analysis
  as a CI job
  * Still shows how gitops is better in this case since hte gitops view is never
    "inconsistent"

{{< /idea >}}

## Read more
* (Link to analyzer docs)
* (Link to https://cloud.google.com/solutions/addressing-continuous-delivery-challenges-in-a-kubernetes-world)
