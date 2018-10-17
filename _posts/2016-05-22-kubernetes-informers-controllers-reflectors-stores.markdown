---
layout: post-no-feature
title: "Kubernetes: Controllers, Informers, Reflectors and Stores"
description: "Kubernetes offers these powerful structures to get a local representation of the API server's resources."
category: articles
tags: ["software", "kubernetes"]
mathjax: true
comments: true
date: 2016-05-22T20:41:00+02:00
---

*F* or my bachelor thesis, I am working with Kubernetes. My partner and I are working on a [system](//github.com/nov1n/kubernetes-workflow){:target="_blank"} to run workflows on Kubernetes. We really like the Kubernetes ideology of seeing the entire system as a control system. That is, the system constantly tries to move its current state to a desired state. The worker units that guarantee the desired state are called [controllers](//kubernetes.io/docs/user-guide/replication-controller/#alternatives-to-replication-controller){:target="_blank"}. Since we want to implement our own workflow controller we decided to look at the *JobController*. That's where we stumbled upon this piece of candy:

{% highlight go linenos %}
jm.jobStore.Store, jm.jobController = framework.NewInformer(
  &cache.ListWatch{
    ListFunc: func(options api.ListOptions) (runtime.Object, error) {
      // Direct call to the API server, using the job client
      return jm.kubeClient.Batch().Jobs(api.NamespaceAll).List(options)
    },
    WatchFunc: func(options api.ListOptions) (watch.Interface, error) {
      // Direct call to the API server, using the job client
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

We weren't sure how it exactly worked, but it seemed to return a store of Jobs given a *list* and *watch* function. In this case the list and watch functions on lines 5 and 9 are direct calls to the API server, using the job client. Awesome right? You feed it a list and watch interface to the API server. The Informer automagically syncs the upstream data to a downstream store and even offers you some handy event hooks.

At some point we noticed some behavior we didn't expect. For example: *UpdateFunc* was called every 30 seconds, while there actually were no updates on the upstream. This is when I decided to dig into the workings of *Informers*, *Controllers*, *Reflectors*, and *Stores*. I'll start by explaining how Controllers work, then I'll explain how Controllers use a Reflector and a *DeltaFIFO* store internally, and lastly how Informers are just a convenient wrapper to sync your upstream with a downstream store.

<span class="nice-italic">Please note that Kubernetes is under heavy development. The code base will probably have changed between the point of writing and the time you're reading this article. I will refer to code at this [blob](//github.com/kubernetes/kubernetes/tree/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg){:target="_blank"}.</span>

## Controllers
When we want to know how informers work, we need to know how controllers work. An example of how controllers work is given in [controller_test.go](//github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/controller/framework/controller_test.go){:target="_blank"}:
{% highlight go linenos anchorlinenos lineanchors=example %}
// source simulates an apiserver object endpoint.
// This will be the resource for the Reflector.
source := framework.NewFakeControllerSource()

// This will hold the downstream state, as we know it.
downstream := cache.NewStore(framework.DeletionHandlingMetaNamespaceKeyFunc)

// This will hold incoming changes. Note how we pass downstream in as a
// KeyLister, that way resync operations will result in the correct set
// of update/delete deltas.
// This will be the store for the Reflector.
fifo := cache.NewDeltaFIFO(cache.MetaNamespaceKeyFunc, nil, downstream)

// Let's do threadsafe output to get predictable test results.
deletionCounter := make(chan string, 1000)

// Configure the controller with a source, the FIFO queue and a Process function.
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

Let's see how this works! Line 3 declares a source. This source will usually be a client to the API server. For now it's a fake source, such that we can control its behavior. Line 6 declares the downstream store, which we'll use to have a local representation of the source. Line 12 declares a *DeltaFIFO* queue, which is used to keep track of the differences between source and downstream. To configure the controller we give it the FIFO queue, the source, and a process function.

The process loop contains the logic to bring the current state of the system to the desired state. The process function receives an *obj*, which is an array of  *[Delta](//github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/client/cache/delta_fifo.go#L532){:target="_blank"}*s from the FIFO queue. In our example we check if the delta is of any type other than *Deleted*. In that case the *Object* belonging to the delta is added to the downstream and the Object is removed from the source. If the delta is of type *Deleted*, the Object is removed from the downstream and a message is sent on the *deletionCounter* channel.

When I ran the example for the first time, I was a bit confused by `if newest.Type != cache.Deleted` on line 31. Why not simply check if type is *Added*? I decided to put the following print statement on line 30: `fmt.Printf("[%v] %v\n", newest.Object.(*api.Pod).Name, newest.Type)`. This is the output I got:
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
On line 64 from the example the controller is created and run. Let's have a look at the *Run* method:
{% highlight go linenos %}
// Run begins processing items, and will continue until a value is sent down stopCh.
// It's an error to call Run more than once.
// Run blocks; call via go.
func (c *Controller) Run(stopCh <-chan struct{}) {
  defer utilruntime.HandleCrash()
  // Create a new Reflector
  r := cache.NewReflector(
    c.config.ListerWatcher,
    c.config.ObjectType,
    c.config.Queue,
    c.config.FullResyncPeriod,
  )
  〜
  // Call RunUntil, which is non-blocking
  r.RunUntil(stopCh)
  // Call the processLoop. When it's finished we wait a second and call it again.
  // This loops infinitly until we receive on stopCh.
  wait.Until(c.processLoop, time.Second, stopCh)
}
{% endhighlight %}

First a new reflector is created. We give it both the *ListerWatcher* (source) and *Queue* (DeltaFIFO). Then, *RunUntil* is called on the reflector, which is a non-blocking call. Lastly the processLoop is called using *[wait.Until](//github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/util/wait/wait.go#L46){:target="_blank"}*. The processLoop drains the FIFO queue and calls the Process function with the popped Deltas from the queue:
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
According to the comment in [reflector.go](//github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/client/cache/reflector.go){:target="_blank"}, a "Reflector watches a specified resource and causes all changes to be reflected in the given store". In our example the resource is the *source* from line 3 and the *store* is the DeltaFIFO from line 12.

When the reflector was created, its *RunUntil* method was called. Let's look at what it does:
{% highlight go linenos %}
func (r *Reflector) RunUntil(stopCh <-chan struct{}) {
  〜
  go wait.Until(func() {
    if err := r.ListAndWatch(stopCh); err != nil {
      utilruntime.HandleError(err)
    }
  }, r.period, stopCh)
}
{% endhighlight %}

So it keeps calling ListAndWatch, until the `stopCh` receives a message. Now let's dive into *[ListAndWatch](//github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/client/cache/reflector.go#L281){:target="_blank"}* where all the real magic happens:
<a name="list-and-watch" />
{% highlight go linenos %}
// ListAndWatch first lists all items and get the resource version at the moment of call,
// and then use the resource version to watch.
// It returns error if ListAndWatch didn't even try to initialize watch.
func (r *Reflector) ListAndWatch(stopCh <-chan struct{}) error {
  〜
  resyncCh, cleanup := r.resyncChan()
  defer cleanup()

  // Explicitly set "0" as resource version - it's fine for the List()
  // to be served from cache and potentially be delayed relative to
  // etcd contents. Reflector framework will catch up via Watch() eventually.
  options := api.ListOptions{ResourceVersion: "0"}
  // Do initial List.
  list, err := r.listerWatcher.List(options)
  〜
  listMetaInterface, err := meta.ListAccessor(list)
  〜
  // Get resource version from the list.
  resourceVersion = listMetaInterface.GetResourceVersion()
  items, err := meta.ExtractList(list)
  〜
  if err := r.syncWith(items, resourceVersion); err != nil {
    return fmt.Errorf("%s: Unable to sync list result: %v", r.name, err)
  }
  // Set the resource version such that Watch can use it.
  r.setLastSyncResourceVersion(resourceVersion)

  // Never ending loop, unless some unexpected error occur. The function will
  // then return the error causing the loop to stop. I omitted the error handling
  // for better readability. When the ListAndWatch returns, it will get called
  // again by RunUntil.
  for {
    options := api.ListOptions{
      ResourceVersion: resourceVersion,
      〜
    }
    w, err := r.listerWatcher.Watch(options)
    〜
    if err := r.watchHandler(w, &resourceVersion, resyncCh, stopCh); err != nil {
      〜
      if err != errorResyncRequested {
        return nil
      }
    }
    if r.canForceResyncNow() {
      〜
      if err := r.store.Resync(); err != nil {
        return err
      }
      cleanup()
      resyncCh, cleanup = r.resyncChan()
    }
  }
}
{% endhighlight %}

The comment above the method says: "ListAndWatch first lists all items and get[s] the resource version at the moment of call, and then use[s] the resource version to watch". So basically this is what happens:
![Reflector timeline]({{ site.url }}/images/Reflector Time Diagram.png)

At time <span style="color:#f44d82">t=0</span> we list the ListerWatcher -- which is the upstream source in our case. From the List we get the highest *resourceVersion* of all items in the list. At <span style="color:#4d82f4">t>0</span> we keep watching the ListerWatcher for changes newer than the obtained resourceVersion. <span class="nice-italic">If you don't know what resource versions are, check the [API conventions](//github.com/kubernetes/kubernetes/blob/master/docs/devel/api-conventions.md#concurrency-control-and-consistency){:target="_blank"}.</span>

So how does that translate to the code of ListAndWatch? At line 14 (<span style="color:#f44d82">t=0</span>) *List* gets called. The *items* from the list are passed to the syncWith method. SyncWith calls store.Replace with the items of the list:
{% highlight go linenos %}
func (r *Reflector) syncWith(items []runtime.Object, resourceVersion string) error {
  found := make([]interface{}, 0, len(items))
  for _, item := range items {
    found = append(found, item)
  }
  return r.store.Replace(found, resourceVersion)
}
{% endhighlight %}

<span class="nice-italic">I'll explain how store.Replace works later</span>

After the initial list we end up in a never ending for loop (<span style="color:#4d82f4">t>0</span>). In this loop we call the *[watchHandler](//github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/client/cache/reflector.go#L369){:target="_blank"}* method with a *[watch.Interface](//github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/watch/watch.go#L26){:target="_blank"}*. The contents of watchHandler is the following piece of blocking code:

{% highlight go linenos %}
// watchHandler watches w and keeps *resourceVersion up to date.
func (r *Reflector) watchHandler(w watch.Interface, resourceVersion *string, resyncCh <-chan time.Time, stopCh <-chan struct{}) error {
  〜
  defer w.Stop()
  〜
  for {
    select {
    case <-stopCh:
      return errorStopRequested
    // When resyncPeriod nanoseconds have passed.
    case <-resyncCh:
      return errorResyncRequested
    case event, ok := <-w.ResultChan():
      〜
      // Catch watch events.
      switch event.Type {
      case watch.Added:
        r.store.Add(event.Object)
      case watch.Modified:
        r.store.Update(event.Object)
      case watch.Deleted:
        〜
        r.store.Delete(event.Object)
      〜
      }
    }
  }
  〜
  return nil
}
{% endhighlight %}

The interesting code is on lines 14 - 23. Watch events from the upstream are caught and corresponding *store* methods are called. Another piece of interesting code is on lines 11 - 12. When a message is received on *resyncCh*, the watchHandler returns. So how do messages end up in resyncCh?

When we take a look back to the code of [ListAndWatch](#list-and-watch), we find the answer on lines 6 - 7. Line 6 calls the *[resyncChan](//github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/client/cache/reflector.go#L233){:target="_blank"}* method, which returns a timer channel that receives a message after r.resyncPeriod nanoseconds. This channel is passed to watchHandler, such that it returns after r.resyncPeriod ns. When watchHandler returns, the ListAndWatch method can finally reach the call to the *[canForceResyncNow](//github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/client/cache/reflector.go#L269){:target="_blank"}* method. CanForceResyncNow returns true if we're [close enough](//github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/client/cache/reflector.go#L88){:target="_blank"} to a next planned periodic resync. In that case *store.Resync* is called. So in the end the situation looks like this:

![Reflector timeline]({{ site.url }}/images/Reflector Time Diagram2.png)

So now we know when the Reflector calls the DeltaFIFO *store*. Let's figure out how the store works!

## DeltaFIFO Store
According to the comments in [delta_fifo.go](//github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/client/cache/delta_fifo.go){:target="_blank"}, a "DeltaFIFO is a producer-consumer queue, where a Reflector is intended to be the producer, and the consumer is whatever calls the Pop() method". In our case the *processLoop* method of our controller is the consumer.

From its definition we get that the DeltaFIFO holds a *queue*, which is an array of strings, and an *items* map, whose keys correspond to the strings in the *queue*:
{% highlight go linenos %}
type DeltaFIFO struct {
  〜
  items map[string]Deltas
  queue []string
  〜
}
{% endhighlight %}

*items* Maps a key to an array of *Delta*s. *Delta* tells you what change happened (Added, Updated, Deleted, Sync) and the object's state after that change.

Let's have a look at the Add, Update, Delete, Replace, and Resync methods that were called from the Reflector.

### Add, Update, and Delete
The Add, Update and Delete methods all call the *[queueActionLocked](//github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/client/cache/delta_fifo.go#L273){:target="_blank"}* method with the corresponding *[DeltaType](//github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/client/cache/delta_fifo.go#L513){:target="_blank"}*s.
In queueActionLocked the given *obj* gets inserted in the queue.

### Replace
[Replace](//github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/client/cache/delta_fifo.go#L393){:target="_blank"} gets a list of items. It enqueues each item with a *Sync* DeltaType. It then uses the *knownObject* -- which is a reference to the downstream store in our case -- to see if items were deleted. If so *Deleted* events get enqueued.
<span class="nice-italic">Note that this is the reason why we got Sync events in the controller example. The three pods were inserted before the reflector started at <span style="color:#f44d82">t=0</span> (maybe not always the case, because starting the reflector and inserting pods in the store are parallel tasks). So the three pods were found during the initial list, which called the store.Replace method. Since store.Replace only fires Sync events for new items, we didn't find any Added events.</span>

### Resync
[Resync](//github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/client/cache/delta_fifo.go#L459){:target="_blank"} will send a Sync event for all items in *knownObjects* -- which is our downstream store.

## Recap on Controller, Reflector and Store
Okay, I get that that was a lot of new information. Let's try to clear up what we've just learned.

The controller:

* has a reference to the FIFO queue;
* has a reference to the ListerWatcher (the upstream source in our case);
* is responsible for consuming the FIFO queue;
* has a process loop, which is responsible for getting the system to a desired state;
* creates a Reflector.

The reflector:

* has a reference to the same FIFO queue (called *store* internally);
* has a reference to the same ListerWatcher;
* lists and watches the ListerWatcher;
* is responsible for producing the FIFO queue's input;
* is responsible for calling the Resync method on the FIFO queue every resyncPeriod ns.

The FIFO queue:

* has a reference to the downstream store;
* has a queue of Deltas for objects that were listed and watched by the Reflector.

## Informer
We now know how the Controller, Reflector and FIFO Queue work together to stay in sync with the upstream source. So let's have a look at how the *Informer* uses these concepts to sync the upstream source to the downstream source. According to the comments [controller.go](//github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/controller/framework/controller.go){:target="_blank"}, "NewInformer returns a cache.Store and a controller for populating the store while also providing event notifications". This is how the *[NewInformer](//github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/controller/framework/controller.go#L198){:target="_blank"}* function looks:

{% highlight go linenos %}
func NewInformer(
  lw cache.ListerWatcher,
  objType runtime.Object,
  resyncPeriod time.Duration,
  h ResourceEventHandler,
) (cache.Store, *Controller) {
  // This will hold the client state, as we know it.
  // This is our downstream store.
  clientState := cache.NewStore(DeletionHandlingMetaNamespaceKeyFunc)

  // This will hold incoming changes. Note how we pass clientState in as a
  // KeyLister, that way resync operations will result in the correct set
  // of update/delete deltas.
  fifo := cache.NewDeltaFIFO(cache.MetaNamespaceKeyFunc, nil, clientState)

  cfg := &Config{
    Queue:            fifo,
    ListerWatcher:    lw,
    ObjectType:       objType,
    FullResyncPeriod: resyncPeriod,
    RetryOnError:     false,

    Process: func(obj interface{}) error {
      // from oldest to newest
      for _, d := range obj.(cache.Deltas) {
        switch d.Type {
        case cache.Sync, cache.Added, cache.Updated:
          if old, exists, err := clientState.Get(d.Object); err == nil && exists {
            if err := clientState.Update(d.Object); err != nil {
              return err
            }
            h.OnUpdate(old, d.Object)
          } else {
            if err := clientState.Add(d.Object); err != nil {
              return err
            }
            h.OnAdd(d.Object)
          }
        case cache.Deleted:
          if err := clientState.Delete(d.Object); err != nil {
            return err
          }
          h.OnDelete(d.Object)
        }
      }
      return nil
    },
  }
  return clientState, New(cfg)
}
{% endhighlight %}

So it's basically a controller with some boilerplate code to sync events from the FIFO queue to the downstream store. It takes a ListerWatcher and a ResourceEventHandler, which looked like this in the JobController source code:
{% highlight go linenos %}
jm.jobStore.Store, jm.jobController = framework.NewInformer(
  &cache.ListWatch{
    ListFunc: func(options api.ListOptions) (runtime.Object, error) {
      // Direct call to the API server, using the job client
      return jm.kubeClient.Batch().Jobs(api.NamespaceAll).List(options)
    },
    WatchFunc: func(options api.ListOptions) (watch.Interface, error) {
      // Direct call to the API server, using the job client
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

The *Process* function deals with all the Delta events. It calls the corresponding Add, Update, and Delete methods on the downstream store -- which is called *clientState* in this code. <span class="nice-italic">Note that ResourceEventHandlerFuncs has no SyncFunc. Therefore AddFunc or UpdateFunc is called when a Sync event is received -- even when an object isn't updated but only resynced.</span>

---

I hope this article has given you a clear representation of the Controller, Informer, Reflector and Store concepts used in Kubernetes, that make your life so much nicer. Please feel free to comment or critize :)

Also, many kudos to [@nov1n](https://github.com/nov1n){:target="_blank"} who helped me with this research.
