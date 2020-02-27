---
title: Using istioctl analyze with GitOps
subtitle:
description:
publishdate: 2020-01-05
attribution: Joe Selman (Google)
keywords: []
target_release: 1.5
---

The configuration of your service mesh is composed of a large set of custom
resources. These resources
often have complex interactions that are not easily understood in isolation. For
example, to understand how a hypothetical request might be routed to a service,
you need to consider several resource type interactions: which gateway will
handle the request, how virtual services will route that
request, and which destination rule will ultimately apply to the request. As of Istio 1.4, custom resources are [defined with OpenAPI
specs](/news/releases/1.4.x/announcing-1.4/upgrade-notes/#configuration-management)
to help prevent common configuration mistakes. However, these validations are only applied in the context of a single resource. To discover and diagnose
misconfigurations across multiple objects, you can use `istioctl analyze`.

`istioctl analyze` is a command-line tool that
analyzes Istio configuration for potential problems. By default, it runs against the
cluster configured in your `~/.kube/config` file, checking
configuration resources on the API server. Discovering such problems
in live configuration is useful, but it would be much better to catch these problems *before* they are applied to the
API server.

This article shows how to set up a workflow that uses `istioctl analyze` to
discover problems before they ever occur on your cluster. It will cover:

1. Defining a workflow for making changes to configuration
1. Setting up a git pre-commit to ensure bad configuration never makes it past your
  repository
1. Automatically running `istioctl analyze` against every pull request opened to
  your repository
1. Automatically running `istioctl analyze` against a combination of a live
  cluster and an incoming pull request

Note that this article assumes you are using at least version 1.5 of `istioctl`. You can check your version by running `istioctl version`.

## GitOps and continuous integration

If you're familiar with the term ['GitOps'](https://www.weave.works/technologies/gitops/), then you're familiar with the
concept of treating your Kubernetes resource configuration the same way that you
treat code. This allows you to establish a workflow for making and releasing
changes to configuration. In practice, this usually entails:

* Storing configuration in a version control system
* Defining a workflow for proposing changes to configuration (often
  via pull requests)
* Defining an process where automated checks and validations are automatically run against configuration changes, both
  before and after approval

GitOps workflows can be very complex; this blog post is only
concerned with running checks on incoming changes. More specifically,
this post shows how to integrate `istioctl analyze` into a broader GitOps workflow.

For now, assume that you have the following structure in your GitHub repository:

{{< text bash >}}
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
{{< /text >}}

Resources are split up by namespaces and workloads, with two apps deployed
to our cluster: a [hipster shop demo app](https://github.com/GoogleCloudPlatform/microservices-demo)
and the Bookinfo app contained within the Istio samples folder. Within each yaml
file is configuration representing both the workload as well as Istio-specific configuration.

## Setting up a pre-commit hook

An easy first step to safer configuration validation is to set up a pre-commit hook for
your local git repository. This is a simple way to prevent committing code that
has problems, but please note that it only works locally; you'll
have to setup the same script on every repository you create. For that reason,
pre-commit hooks are best used as a simple sanity check for your own workflow,
and not as a means of enforcing policy.

Before installing the hook, it's a good idea to make sure you don't have any
existing problems with your configuration. Invoke `istioctl analyze` in
file-only mode.

{{< text bash >}}
$ istioctl analyze --recursive cluster/ \
    --use-kube false \
    --all-namespaces \
    --failure-threshold Info
✔ No validation issues found when analyzing all namespaces.
{{< /text >}}

This command analyzes the files contained in the `cluster/` folder recursively. Also
specified is the `--use-kube false` argument, which tells `istioctl analyze` to not analyze
the cluster configured in your `.kube/config` file. Without this option, it
would additionally try to analyze your currently configured
default cluster. It's safe to leave out if you want to always analyze the
current live cluster on every commit, but note that your results will be more
inconsistent as the live cluster might be changing.

The `--all-namespaces` argument will ensure that `istioctl analyze` checks all namespaces. Finally,
the `--failure-threshold Info` defines what sort of problems to consider as a
failure (valid choices are `Error`, `Warn`, or `Info`). By specifying `Info`,
`istioctl analyze`
will treat any problem as a failure - feel free to adjust this to your own
situation.

The analyzer found no issues, so you can go ahead and install the
pre-commit hook. Save the contents below to a file named `pre-commit.example` at
your repository root:

{{< text shell >}}
#!/bin/bash
#
# Example pre-commit to run istioctl analyze on commit. To use, add to your git
# pre-commit hook via `ln -s ../../pre-commit.example .git/hooks/pre-commit`.

# Path to istioctl - replace with hard-coded path if you want to use a different binary.
ISTIOCTL=$HOME/bin/istioctl
# Failure threshold - valid values are Info/Warn/Error
FAILURE_THRESHOLD=Info

if ! [ -x "$(command -v $ISTIOCTL)" ]; then
    echo "$ISTIOCTL command not found. Either add it to your path, or modify"
    echo "your pre-commit hook to specify the absolute path."
    exit 1
fi

$ISTIOCTL analyze --recursive cluster/ \
    --use-kube=false \
    --all-namespaces \
    --failure-threshold $FAILURE_THRESHOLD
{{< /text >}}

You can then install the hook with a symbolic link. Assuming the `pre-commit.example`
file exists at the repository root, run:

{{< text bash >}}
$ chmod +x pre-commit.example
$ ln -s ../../pre-commit.example .git/hooks/pre-commit
{{< /text >}}

Now, whenever you commit a change to your repository, the `cluster/` directory
will be automatically analyzed. It's a good idea to test out the pre-commit hook by
introducing an error. Create a new change to expose the `productpage` service
via a gateway resource; however, introduce a typo in the name of the gateway
bound in the virtual service:

Store the configuration below in
`cluster/workloads/bookinfo/bookinfo-gateway.yaml`:

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
$ git add cluster/workloads/bookinfo/bookinfo-gateway.yaml && git commit -m "Expose bookinfo's productpage via ingress"
Error [IST0101] (VirtualService bookinfo-vs.bookinfo) Referenced gateway not found: "bookinfogateway"
Error: Analyzers found issues when analyzing all namespaces.
See https://istio.io/docs/reference/config/analysis for more information about causes and resolutions.
{{< /text >}}

From the output above you can discern:

* There was a single error with the error code `IST0101`
* The resource under question is of kind `VirtualService`, named `bookinfo-vs`
  in the `bookinfo` namespace
* The gateway that wasn't found was named `bookinfogateway`

With the above information, you should be able to discover the typo and correct
the issue.

Installing the pre-commit hook allowed you to discover the missing
reference before applying the problematic configuration to your cluster.
However, this only works for your local git repository; the next step is to
begin enforcing this check via pull requests.

## `istioctl analyze` as a continuous integration step

Shown here is an example of using `istioctl analyze` as a continuous integration
check. [GitHub actions](https://help.github.com/en/actions/getting-started-with-github-actions/about-github-actions)
is documented in this article; the actual integration of `istioctl analyze` should
be straightforward to generalize to other continuous integration platforms. The
directions below assume you are storing your cluster configuration in a GitHub
repository and have enabled GitHub actions.

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
{{< /text >}}

This defines a CI workflow that runs on every pull request to master. It creates
a job named 'analyze' that first checks out the repository via the
`actions/checkout` action, and then runs an analyze action at the path
`./.github/actions/analyze`.

Create a file in your repository named `.github/actions/analyze/action.yml` with the
contents:

{{< text yaml >}}
name: 'istioctl analyze'
description: 'Runs istioctl analyze on configuration'
runs:
  using: 'docker'
  image: 'istio/istioctl:1.5.0'
  args:
  - '-c'
  - ['analyze', '-R', 'cluster/', '--all-namespaces', '--use-kube=false', '--failure-threshold=Info']
{{< /text >}}

This is the same command run as part of the pre-commit hook; however now the command
is run whenever a pull request is opened to master. This ensures that new
changes won't introduce any issues that analyze could have caught.

### Suppress unwanted messages

You might find that some messages aren't useful in your situation. For example,
`istioctl analyze` will indicate port names that don't conform to Istio's naming
conventions. If you're in the process of migrating your application to run on
Istio, you might not want to block a deployment due to this issue:

{{< text bash >}}
$ git commit -m "Add gitlab"
Info [IST0118] (Service gitlab-postgresql.gitlab) Port name postgres (port: 5432, targetPort: postgres) doesn't follow the naming convention of Istio port.
Info [IST0118] (Service gitlab.gitlab) Port name mattermost (port: 8065, targetPort: mattermost) doesn't follow the naming convention of Istio port.
Info [IST0118] (Service gitlab.gitlab) Port name prometheus (port: 9090, targetPort: prometheus) doesn't follow the naming convention of Istio port.
Info [IST0118] (Service gitlab.gitlab) Port name registry (port: 8105, targetPort: registry) doesn't follow the naming convention of Istio port.
Info [IST0118] (Service gitlab.gitlab) Port name ssh (port: 22, targetPort: ssh) doesn't follow the naming convention of Istio port.
Info [IST0118] (Service gitlab.gitlab) Port name workhorse (port: 8005, targetPort: workhorse) doesn't follow the naming convention of Istio port.
Error: Analyzers found issues when analyzing all namespaces.
See https://istio.io/docs/reference/config/analysis for more information about causes and resolutions.
{{< /text >}}

The errors above indicate that there are a lot of ports that don't follow the
Istio naming convention. To suppress this message, take note of the analyzer
message code (IST0118) and the set of matching resources that are causing the
messages (services in the `gitlab` namespace). Then, supply a suppression using
the `--suppress` argument:

{{< text bash >}}
$ istioctl analyze --recursive cluster/ \
    --use-kube=false \
    --all-namespaces \
    --failure-threshold=Info \
    --suppress "IST0118=Service *.gitlab"
{{< /text >}}

The format for suppression rules is `<code>=<kind> <name>.<namespace>`. You can
use a `*` to pattern match and ensure that the suppression is applied across the
entire namespace.

## `istioctl analyze` against a live cluster with changes

One of the more exciting features of `istioctl analyze` is the ability to analyze
a set of local files and a live cluster together. If you provide both a file or
directory argument and the `--use-kube=true` flag, then `istioctl analyze` will
analyze the current configured cluster defined in your `.kube/config` file and
overlay the set of local changes on top of the cluster view. This is very useful
when your configuration repository might not be a complete representation of what's
deployed in your cluster.

Consider an example where you want to expose an existing service into the
default namespace. To accomplish this, you might write a virtual service like
the one below:

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: product-page
  namespace: bookinfo
spec:
  hosts:
  - productpage.default.svc.cluster.local
  http:
  - route:
    - destination:
        host: productpage.bookinfo.svc.cluster.local
{{< /text >}}

This exposes the host `productpage.bookinfo.svc.cluster.local` under the
additional name `productpage.default.svc.cluster.local`. As a result, workloads
running in the default namespace can send traffic directly to hostname
`productpage` (note that there is no suffix) and traffic will be sent to the
service named `productpage` running in the `bookinfo` namespace.

Continuing with our example, you then check in this file to your git repository.
`istioctl analyze` doesn't detect any errors. However, imagine that you decided
to analyze the change and the live cluster at the same time:

{{< text bash >}}
$ istioctl analyze --recursive cluster/ \
    --use-kube=true # Note we set this to true rather than false \
    --all-namespaces \
    --failure-threshold=Info
Error [IST0109] (VirtualService our-product-page.productpage-devel) The VirtualServices bookinfo/product-page, productpage-devel/our-product-page associated with mesh gateway define the same host */productpage.default.svc.cluster.local which can lead to undefined behavior. This can be fixed by merging the conflicting VirtualServices into a single resource.
Error [IST0109] (VirtualService product-page.bookinfo) The VirtualServices bookinfo/product-page, productpage-devel/our-product-page associated with mesh gateway define the same host */productpage.default.svc.cluster.local which can lead to undefined behavior. This can be fixed by merging the conflicting VirtualServices into a single resource.
Error: Analyzers found issues when analyzing all namespaces.
See https://istio.io/docs/reference/config/analysis for more information about causes and resolutions.
{{< /text >}}

From reading the messages above, we see that there are two virtual services in
conflict: the virtual service we introduced (`product-page.bookinfo`) and
another virtual service in the cluster (`our-product-page.productpage-devel`).
Both virtual services are attempting to define the same host name in the default
namespace, and having them both defined at the same time would lead to undefined
behavior. You can imagine that identifying this problem _before_ it occurs on
your cluster is highly desirable!

As the example above illustrates, analyzing configuration files overlaid on top
of a live cluster is useful when your git repository doesn't contain all
relevant configuration. This can happen if your team is still migrating all
configuration to the repository, or if someone made changes directly to the
cluster. Regardless, it's a good idea to introduce this check to your
continuous integration processes, but be aware of its limitations:

* You'll have to manage a Kubernetes `.kube/config` file containing read-only
  credentials to your cluster. Because this secret is available to the
  continuous integration check, be sure that you trust the incoming change to
  not accidentally or deliberately expose the credential in your continuous
  integration logs.
* The check is not necessarily reproducible; Kubernetes configuration is dynamic
  and changes over time as various controllers, users, and other processes make
  changes. It might be reasonable to run this check on a schedule to ensure you
  catch any new issues that are caused by external systems.

## Key points

This article highlighted how you can leverage `istioctl analyze` to go beyond
merely analyzing the currently-configured cluster. By setting up a GitOps-style
workflow for your Istio configuration and running `istioctl analyze` locally and
as part of a CI-workflow, you can catch problems before they ever occur in your
cluster.

Want to learn more about `istioctl analyze`? You can [read the docs here](/docs/ops/diagnostic-tools/istioctl-analyze/).

## Learn more

* [Addressing Continuous Delivery Challenges in a Kubernetes World](https://cloud.google.com/solutions/addressing-continuous-delivery-challenges-in-a-kubernetes-world)
* [GitOps: A Path to More Self-service IT](https://queue.acm.org/detail.cfm?id=3237207)

