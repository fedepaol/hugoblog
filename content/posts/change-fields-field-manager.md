+++ 
draft = true
date = 2023-08-24T21:45:30+02:00
title = "Managing Kubernetes Resources with Controller Runtime's Field Manager"
description = "Explore the nuances of managing Kubernetes resources using the powerful 'controller-runtime' framework. Learn how to leverage the field manager concept to streamline updates and avoid synchronization issues."
slug = ""
authors = ["Federico Paolinelli"]
tags = []
categories = ["go", "kubernetes"]
externalLink = ""
series = []
+++

# The problem with mutated fields

Having kubernetes resources that do not sync is a common problem when dealing with a continuos delivery system such as [ArgoCD](https://argoproj.github.io/cd/) and external components that change those resources.

What happens is:

- the CD system applies a change based on some configuration
- the change is overridden by some controller
- the CD system sees a difference between the observed state and the desired state (the mutation from the step above)

and this goes on forever, unless the CD system is instructed to [ignore the differences](https://argo-cd.readthedocs.io/en/stable/user-guide/diffing/#application-level-configuration).

We hit this [issue](https://github.com/metallb/metallb/issues/1681) in MetalLB, where we inject the caBundle in the CRDs to make the conversion webhooks work.

## Setting the field manager

By declaring the manager of a given field, we can instruct the CD system that it shouldn't care about some fields
because they are handled by something else (see [the official docs](https://kubernetes.io/docs/reference/using-api/server-side-apply/#field-management)).

## Understanding the Field Manager in Controller Runtime

The [controller runtime](https://github.com/kubernetes-sigs/controller-runtime) is my go to framework when I need to write a kubernetes controller. It offers a very convenient way to
set the field manager when updating an object:

```go
    cli.Update(context.TODO(), toUpdate, client.FieldOwner("foo"))
```

By doing this, the server side will set the field owner and the CD system will be able to understand the change has to
be ignored.

## Verifying everything works

This is the major reason why I am writing this post, and the cause making this work took so much of my time (and I almost gave up).

Turns out that `kubectl` has a `--show-managed-fields` flag to set when checking a given resource. Without that, those fields will be hidden in the output to avoid bloating the yaml!

### Wrapping up

I hope this post may help whoever is looking to set the fieldmanager of a given resource using the controller-runtime, because
it took a considerable amount of my time (compared to the final, one liner solution).
