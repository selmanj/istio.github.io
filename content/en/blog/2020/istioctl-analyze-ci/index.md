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

<<<<<<< HEAD
||||||| constructed merge base
https://cloud.google.com/solutions/addressing-continuous-delivery-challenges-in-a-kubernetes-world
=======
## Setting up a precommit
>>>>>>> WIP

An easy first step to safer config validation is to setup a pre-commit hook for
your local git repository. This is a simple way to prevent committing code that
has problems to your repository, but note that it only works locally; you'll
have to setup the same script on every repository you create. For that reason,
pre-commit hooks are best used as a simple sanity check for your own workflow,
and not as a means of enforcing policy.

<<<<<<< HEAD
## Setting up a precommit
## Using analyze in continuous integration

## Read more
(Link to analyzer docs)
(Link to https://cloud.google.com/solutions/addressing-continuous-delivery-challenges-in-a-kubernetes-world)

||||||| constructed merge base

## Setting up a precommit
## Using analyze in continuous integration
=======
Below is a simple pre-commit hook you can use:

{{< text bash >}}
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
{{/< text >}}

## Using analyze in continuous integration
>>>>>>> WIP
