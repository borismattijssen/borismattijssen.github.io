---
layout: post-no-feature
title: "Kubernetes: Controllers, Informers, Reflectors and Stores"
description: "Kubernetes offers these powerful structures to get a local representation of the API server's resources."
category: articles
tags: ["software", "kubernetes"]
mathjax: true
typefix:
   indent: true
date: 2016-05-22T20:41:00+02:00
---

*F* or my bachelor thesis, I am working with Kubernetes. My partner and I are working on a [system](//github.com/nov1n/kubernetes-workflow){:target="_blank"} to run workflows on Kubernetes. We really like the Kubernetes ideology of seeing the entire system as a control system. That is, the system constantly tries to move its current state to a desired state. The worker units that guarantee the desired state are called [controllers](//kubernetes.io/docs/user-guide/replication-controller/#alternatives-to-replication-controller){:target="_blank"}. Since we want to implement our own workflow controller we decided to look at the *JobController*. That's where we stumbled upon this piece of candy:

{% highlight go linenos %}
jm.jobStore.Store, jm.jobController = framework.NewInformer(
  &cache.ListWatch{
    ListFunc: func(options api.ListOptions) (runtime.Object, error) {
      return jm.kubeClient.Batch().Jobs(api.NamespaceAll).List(options)
    },
    WatchFunc: func(options api.ListOptions) (watch.Interface, error) {
      return jm.kubeClient.Batch().Jobs(api.NamespaceAll).Watch(options)
    },
  },
  〜
  framework.ResourceEventHandlerFuncs{
    AddFunc: jm.enqueueController,
    UpdateFunc: 〜
    DeleteFunc: jm.enqueueController,
  },
)
{% endhighlight %}

We weren't sure how it exactly worked, but it seemed to return a store of Jobs given a *list* and *watch* function. In this case the list and watch functions on line 4 and 7 are direct calls to the API server, via the job client. Awesome right? You feed it a list and watch interface to the API server. The Informer automagically syncs the upstream data to a downstream store and even offers you some handy event hooks.

At some point we noticed some behavior we didn't expect. For example: *UpdateFunc* was called every 30 seconds, while there actually were no updates on the upstream. This is when I decided to dig into the workings of *Informers*, *Controllers*, *Reflectors*, and *Stores*. I'll start by explaining how Controllers work, then I'll explain how Controllers use a Reflector and a *DeltaFIFO* store internally, and lastly how Informers are just a convenient wrapper to sync your upstream with a downstream store.

Please note that Kubernetes is under heavy development. The code base will probably have changed between the point of writing and you reading this article. I will refer to code at this [blob](//github.com/kubernetes/kubernetes/tree/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg){:target="_blank"}.

## Controllers
When we want to know how informers work, we need to know how controllers work. An example of how controllers work is given in [controller_test.go](//github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/controller/framework/controller_test.go){:target="_blank"}:
{% highlight go linenos anchorlinenos lineanchors=example %}
// source simulates an apiserver object endpoint.
source := framework.NewFakeControllerSource()

// This will hold the downstream state, as we know it.
downstream := cache.NewStore(framework.DeletionHandlingMetaNamespaceKeyFunc)

// This will hold incoming changes. Note how we pass downstream in as a
// KeyLister, that way resync operations will result in the correct set
// of update/delete deltas.
fifo := cache.NewDeltaFIFO(cache.MetaNamespaceKeyFunc, nil, downstream)

// Let's do threadsafe output to get predictable test results.
deletionCounter := make(chan string, 1000)

cfg := &framework.Config{
	Queue:            fifo,
	ListerWatcher:    source,
	ObjectType:       &api.Pod{},
	FullResyncPeriod: time.Millisecond * 100,
	RetryOnError:     false,

	// Let's implement a simple controller that just deletes
	// everything that comes in.
	Process: func(obj interface{}) error {
		// Obj is from the Pop method of the Queue we make above.
		newest := obj.(cache.Deltas).Newest()

		if newest.Type != cache.Deleted {
			// Update our downstream store.
			err := downstream.Add(newest.Object)
			if err != nil {
				return err
			}

			// Delete this object.
			source.Delete(newest.Object.(runtime.Object))
		} else {
			// Update our downstream store.
			err := downstream.Delete(newest.Object)
			if err != nil {
				return err
			}

			// fifo's KeyOf is easiest, because it handles
			// DeletedFinalStateUnknown markers.
			key, err := fifo.KeyOf(newest.Object)
			if err != nil {
				return err
			}

			// Report this deletion.
			deletionCounter <- key
		}
		return nil
	},
}

// Create the controller and run it until we close stop.
stop := make(chan struct{})
defer close(stop)
go framework.New(cfg).Run(stop)

// Lets add a few objects to the source.
testIDs := []string{"a-hello", "b-controller", "c-framework"}
for _, name := range testIDs {
	// Note that these pods are not valid-- the fake source doesnt
	// call validation or anything.
	source.Add(&api.Pod{ObjectMeta: api.ObjectMeta{Name: name}})
}

// Lets wait for the controller to process the things we just added.
outputSet := sets.String{}
for i := 0; i < len(testIDs); i++ {
	outputSet.Insert(<-deletionCounter)
}

for _, key := range outputSet.List() {
	fmt.Println(key)
}
{% endhighlight %}

The example above will produce the following output:
{% highlight bash %}
a-hello
b-controller
c-framework
{% endhighlight %}

Let's see how this works! Line 2 declares a source. This source will usually be a client to the API server. For now it's a fake source, such that we can control its behavior. Line 5 declares the downstream store, which we'll use to have a local representation of the source. Line 10 declares a DeltaFIFO queue, which is used to keep track of the differences between source and downstream. To configure the controller we give it the FIFO queue, the source, and a process loop.

The process loop contains the logic to bring the current state of the system to the desired state. The process function receives an *obj*, which is an array of  *[Delta](//github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/client/cache/delta_fifo.go#L532){:target="_blank"}*s from the FIFO queue. In our example we check if the delta is of any type other then *Deleted*. In that case the *Object* belonging to the delta is added to the downstream and the Object is removed from the source. If the delta is of type *Deleted*, the Object is removed from the downstream and a message is sent on the *deletionCounter* channel.

When I ran the example for the first time, I was a bit confused by `if newest.Type != cache.Deleted` on line 28. Why not simply check if type is *Added*? I decided to put the following print statement on line 27: `fmt.Printf("[%v] %v\n", newest.Object.(*api.Pod).Name, newest.Type)`. This is the output I got:
{% highlight bash %}
[a-hello] Sync
[b-controller] Sync
[c-framework] Sync
[a-hello] Deleted
[b-controller] Deleted
[c-framework] Deleted
{% endhighlight %}
So we are actually getting *Sync* events when something is added. Why? Let's find out!

### Controller Run
On line 61 from the example the controller is created and run. Let's have a look at the *Run* method:
{% highlight go linenos %}
// Run begins processing items, and will continue until a value is sent down stopCh.
// It's an error to call Run more than once.
// Run blocks; call via go.
func (c *Controller) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	r := cache.NewReflector(
		c.config.ListerWatcher,
		c.config.ObjectType,
		c.config.Queue,
		c.config.FullResyncPeriod,
	)
  〜
	r.RunUntil(stopCh)
	wait.Until(c.processLoop, time.Second, stopCh)
}
{% endhighlight %}

First a new reflector is created. We give it both the *ListerWatcher* (source) and *Queue* (DeltaFIFO). Then *RunUntil* is called on the reflector, which is a non-blocking call. Lastly the processLoop is called using *[wait.Until](//github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/util/wait/wait.go#L46){:target="_blank"}*. The processLoop drains the FIFO queue and calls the Process function with the popped Deltas from the queue:
{% highlight go linenos %}
func (c *Controller) processLoop() {
	for {
		obj := c.config.Queue.Pop()
		err := c.config.Process(obj)
		〜
	}
}
{% endhighlight %}

We now know how the *Process* function from the example is called, but we still don't know how stuff gets in the FIFO Queue. To understand how this works, we'll have to dig into the Reflector.

## Reflectors
According to the comment in [reflector.go](//github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/client/cache/reflector.go), a "Reflector watches a specified resource and causes all changes to be reflected in the given store". In our example the resource is the *source* from {% include line.html no="18" snippet="example"%} and the store is the DeltaFIFO from line 10.
<!-- From line 32 we get that if we want to create a controller, we'll have to give in a *framework.Config*. -->
