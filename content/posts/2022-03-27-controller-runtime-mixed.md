---
title: "Mixing cluster scoped resources and namespaced resources with kubebuilder and controller runtime"
date: 2022-03-27T14:22:36+02:00
---
# Out of the box

If we look at the [kubebuilder tutorial](https://book.kubebuilder.io/cronjob-tutorial/empty-main.html), it's possible to define a namespace (or a set of namespaces, when using [MultiNamespacedCacheBuilder](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/cache#MultiNamespacedCacheBuilder)) for our controller to listen on.

If we don't set the namespace, our controller is meant to be cluster scoped and will listen (and need priviledges) to the resources in all the namespaces.

## The problem

While working on MetalLB, we had to add an alternative password field of type _secret reference_. In other words, MetalLB has to be still cluster scoped for some resources (the services for example), while having access only to instances living in its namespace for other types of resources (i.e. secrets). This, in order to try to observe the least priviledge principle.

### The solution

Despite not being documented, a bit of GitHub archeology helped me find [this PR](https://github.com/kubernetes-sigs/controller-runtime/pull/1435) where it was introduced a per-object selector to the controller-runtime's manager cache, and [this other PR](https://github.com/kubernetes-sigs/controller-runtime/pull/1602) where, by adding a namespace selector on a given object, would have translated all the actions on that particular object as namespaced.

By mixing the two, and selecting the namespace, it is now possible to have our controller using controller-runtime select both namespaced resources and cluster scoped resources.

### The code

This exempt is taken from MetalLB:

```go
	namespaceSelector := cache.ObjectSelector{
		Field: fields.ParseSelectorOrDie(fmt.Sprintf("metadata.namespace=%s", cfg.Namespace)),
	}

	mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
		Scheme:         scheme,
		Port:           9443,
		LeaderElection: false,
		NewCache: cache.BuilderWithOptions(cache.Options{
			SelectorsByObject: map[client.Object]cache.ObjectSelector{
				&metallbv1alpha1.AddressPool{}:     namespaceSelector,
				&metallbv1beta1.AddressPool{}:      namespaceSelector,
				&metallbv1beta1.BFDProfile{}:       namespaceSelector,
				&metallbv1beta1.BGPAdvertisement{}: namespaceSelector,
				&metallbv1beta1.BGPPeer{}:          namespaceSelector,
				&metallbv1beta1.IPPool{}:           namespaceSelector,
				&metallbv1beta1.L2Advertisement{}:  namespaceSelector,
				&metallbv1beta2.BGPPeer{}:          namespaceSelector,
			},
		}),
	})
```

By doing this, even if the controller is declared as cluster scoped (no namespace is added when creating the Manager), a selected set of resources will be namespace scoped and it will be possible to give local (`Role`) permissions to those.
