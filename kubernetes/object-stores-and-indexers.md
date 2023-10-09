> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [leftasexercise.com](https://leftasexercise.com/2019/07/11/understanding-kubernetes-controllers-part-ii-object-stores-and-indexers/)

> In the last post, we have seen that our sample controller uses Listers to retrieve the current state ......

![][img-0]

In the last post, we have seen that our sample controller uses **Listers** to retrieve the current state of Kubernetes resources. In this post, we will take a closer look at how these Listers work.

Essentially, we have already seen how to use the Kubernetes Go client to retrieve informations on Kubernetes resources, so we could simply do that in our controller. However, this is a bit inefficient. Suppose, for instance, you are using multiple worker threads as we do it. You would then probably retrieve the same information over and over again, creating a high load on the API server. To avoid this, a special class of Kubernetes informers – called **index informers** can be used which build a thread-safe object store serving as a cache. When the state of the cluster changes, the informer will not only invoke the handler functions of our controller, but also do the necessary updates to keep the cache up to date. As the cache has the additional ability to deal with indices, it is called an **Indexer**. Thus at the end of todays post, the following picture will emerge.

![][img-1]

In the remainder of this post, we will discuss indexers and how they interact with an informer in more detail, while in the next post, we will learn how informers are created and used and dig a little bit into their inner workings.

Watches and resource versions
-----------------------------

Before we talk about informers and indexers, we have to understand the basic mechanisms that clients can use to keep track of the cluster state. To enable this, the [Kubernetes API](https://kubernetes.io/docs/reference/using-api/api-concepts/) offers a mechanism called a **watch**. This is maybe explained best using an example.

To follow this example, we assume that you have a Kubernetes cluster up and running. We will use curl to directly interact with the API. To avoid having to add tokens or certificates to our request, we will use the kubectl proxy mechanism. So in a separate terminal, run

```
$ kubectl proxy


```

You should see a message that the proxy is listening on a port (typically 8001) on the local host. Any requests sent to this port will be forwarded to the Kubernetes API server. To populate our cluster, let us first start a single HTTPD.

```
$ kubectl run alpine --image=httpd:alpine


```

Then let us use curl to get a list of running pods in the default namespace.

```
$ curl localhost:8001/api/v1/namespaces/default/pods
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/namespaces/default/pods",
    "resourceVersion": "6834"
  },
  "items": [
    {
      "metadata": {
        "name": "alpine-56cf65bbfc-tzqqx",
        "generateName": "alpine-56cf65bbfc-",
        "namespace": "default",
        "selfLink": "/api/v1/namespaces/default/pods/alpine-56cf65bbfc-tzqqx",
        "uid": "584ddf85-5f8d-11e9-80c0-080027696a3f",
        "resourceVersion": "6671",
--- REDACTED ---


```

As expected, you will get a JSON encoded object of type _PodList_. The interesting part is the data in the metadata. You will see that there is a field _resourceVersion_. Essentially, the resource version is a number which increases over time and that uniquely identifies a certain state of the cluster.

Now the Kubernetes API offers you the option to request a **watch**, using this resource version as a starting point. To do this manually, enter

```
$ curl -v localhost:8001/api/v1/namespaces/default/pods?watch=1&resourceVersion=6834


```

Looking at the output, you will see that this request returns a HTTP response with the transfer enconding “chunked”. This is specified in [RFC 7230](https://tools.ietf.org/html/rfc7230#section-4.1) and puts the client into streaming mode, i.e. the connection will remain open and the API server will continue to send updates in small chunks. This will move curl into the background, but curl will continue to print the received data to the terminal. If you now create additional pods in your cluster or delete existing pods, you will continue to see notifications being received, informing you about the events. Each notification consists of a type (ADDED, MODIFIED, ERROR or DELETED) and an object – the layout of the message is described [here](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.13/#watchevent-v1-meta).

This gives us a way to obtain a complete picture of the clusters state in an efficient manner. We first use an ordinary API request to list all resources. We then remember the resource version in the response and use that resource version as a starting point for a watch. Whenever we receive a notification about a change, we update our local data accordingly. And essentially, this is exactly what the combination of informer and indexer are doing.

Caching mechanisms and indexers
-------------------------------

An indexer is any object that implements the interface _cache.Indexer_. This interface in turn is derived from _cache.Store_, so let us study that first. Its definition is in [store.go](https://github.com/kubernetes/client-go/blob/master/tools/cache/store.go).

```
type Store interface {
	Add(obj interface{}) error
	Update(obj interface{}) error
	Delete(obj interface{}) error
	List() []interface{}
	ListKeys() []string
	Get(obj interface{}) (item interface{}, exists bool, err error)
	GetByKey(key string) (item interface{}, exists bool, err error)
	Replace([]interface{}, string) error
	Resync() error
}


```

So basically a store is something to which we can add objects, retrieve them, update or delete them. The interface itself does not make any assumptions about keys, but when you create a new store, you provide a **key function** which extracts the key from an object and has the following signatures.

`type KeyFunc func(obj interface{}) (string, error)`

Working with stores is very convenient and easy, you can find a short example that stores objects representing books in a store [here](https://github.com/christianb93/kubernetes-client-examples/blob/master/example4/main.go).

Let us now verify that, as the diagram above claims, both, the informer and the lister have a reference to the same indexer. To see this, let us look at the creation process of our [sample controller](https://github.com/kubernetes/sample-controller/blob/master/controller.go).

When a new controller is created by the function _NewController_, this function accepts a _DeploymentInformer_ and a _FooInformer_. These are interfaces that provide access to an actual informer and a lister for the respective resources. Let us take the _FooInformer_ as an example. The actual creation method for the Lister looks as follows.

```
func (f *fooInformer) Lister() v1alpha1.FooLister {
	return v1alpha1.NewFooLister(f.Informer().GetIndexer())
}


```

This explains how the link between the informer and the indexer is established. The communication between informer and indexer is done via the function _handleDeltas_ which receives a list of _Delta_ objects as defined in [delta_fifo.go](https://github.com/kubernetes/client-go/blob/master/tools/cache/delta_fifo.go) (we will learn more about how this works in the next post). If we look at this function, we find that it does not only call all registered handler functions (with the help of a _processor_), but also calls the methods _Add_, _Update_ and _Delete_ on the store, depending on the type of the delta.

We now have a rather complete picture of how our sample controller works. The informer uses the Kubernetes API and its mechanism to watch for changes based on resource versions to obtain updates of the cluster state. These updates are used to maintain an object store which reflects the current state of the cluster and to invoke defined event handler functions. A controller registers its functions with the informer to be called when a resource changes. It can then access the object store to easily retrieve the current state of the resources and take necessary actions to drive the system towards the target state.

What we have not yet seen, however, is **how** exactly the magical informer works – this will be the topic of our next post.