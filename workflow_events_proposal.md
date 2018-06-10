# Proposal: Events as first class citzens of Argo workflows.

## Background
Current argo behavior has a (possibly dynamic) DAG, where tasks are triggered when the tasks that produce their input(s) have completed. There is only one kind of runtime relationship between two tasks: Task A runs once Task(s) B (`B_2`, `B_3`, … `B_n`) has finished. 

Fundamentally, Argo lacks “temporal fan out” and “temporal fan in”. Which is the ability for a task to define an arbitrary number of events through its lifetime which other tasks can respond to. This makes it unsuitable for the orchestration of tasks which have state that is expensive to store and reload.

However, many usecases want to "mix and match" the traditional argo workflows with events. Running traditional pipelines that produce events, and triggering traditional pipelines with events. This could be done with a separate CRD but for more complex networks of triggers you'll need large numbers of configs for triggers and workflows, which will hide the true semantics of your pipeline.

## Proposal Overview
Rather than create a separate CRD for events, we add semantics to Argo workflows that trigger tasks based on events from the Watch endpoints of arbitrary k8s resources. 

We then have the basis for an event based orchestration system that is extensible by custom types and modular sidecar containers.

In fact, existing Argo behavior can be thought of as syntatic sugar over this event based system.

Typical usage will be to create a task, with an “event sidecar” container from a predefined set of containers (or a custom event sidecar if one of ours does not meet their needs). The event sidecar will be responsible for monitoring the FS, network, stdout/stderr, etc effects of the user’s task container, and converting these changes into k8s API requests. 

In the common case, these API requests will just be CRUD requests on a custom `argo-events.v1alpha1.Event` type that conforms to the [CloudEvents spec](https://github.com/cloudevents/spec/blob/master/spec.md#context-attributes). But they could just as easily be CRUD calls on any other k8s resource (core, or custom) as long as these resources support the “watch” API verb, (we can think of these resource creations as “pipeline side-effects”)

The user then creates a second task in their pipeline, which responds to the event, by defining a task which has a Parameter that uses ValueFrom which contains an EventParameter type. This new field defines the events that the task should respond to, and the response that should be elicited by the orchestration system.

## Detailed Design Propsal
### The EventParameter type
We define a new type, and add it as a field to the argoproj.io/v1alphaX.ValueFrom

```
type EventParameter struct {    
  # Kind, namespace, and selectors for event stream ,
  EventMetadata Metadata `json:"eventMetadata,omitempty"` 
  # Field from the event object to populate the parameter with
  ObjectField string `json:"objectField,omitempty"`    
  # A policy describing the effect this event should have on this task,
  # (I think the way Argo currently works the only effects that make sense are  
  # "create" and "delete", but "update" would be nice for stateful tasks which
  # are Pods under the hood. I don't believe these types of tasks currently exist  
  # in Argo.    
  EventPolicy string `json:"eventPolicy,omitempty"`
}

type ValueFrom struct {    
  ...  
  Event FromEvent `json:"event,omitempty"` 
  ...
}
```

### Sidecar Containers
Easy creation of event streams for user tasks is critical to the overall usability of the system. We should not expose the need to make k8s API calls directly to the user. However, the relationship needs between user task containers and event streams needs to be flexible, extensible, and modular. To achieve both of these goals, we plug in to the existing sidecar system of Argo. We provide a number of “standard” sidecar containers out of the box, that create useful event streams from custom user task container behavior.

These containers should treat the user’s task container as a black box, and only interact with events produced by that container through external channels (indeed container isolation enforces this principle). To support this, we add a Parameter []Parameter field to the Sidecar type, so that Sidecars may be parameterized along with the user task.

#### Common parameters
- Event resource namespace
- Event type
- Selector labels for the events

#### Filesystem Sidecar

This sidecar watches a k8s Volume, that is attached to the user task container for changes.

This allows us to piggyback on the flexibility of the built in kubernetes storage system (Volumes, VolumeClaims, VolumeMounts), for producing events. 

Additional Parameters:
- Path(s) or filename regex in the volume to watch
- Create events on filename change, update, delete, etc. (see https://cloud.google.com/storage/docs/object-change-notification for an example)

Example Use Cases:
- Start a compression/transcoding pipeline whenever a new file appears.
- Start a new ML training job whenever new training data appears

#### `stdout`/`stderror` Sidecar 

This sidecar can emit events based on data in streams coming from the user task container

Additional Parameters:
- Stream to watch

Example Use Cases:
- Proxy logs to a central dashboard

#### Task Container Lifecycle Sidecar

This sidecar can query the local kubelet daemon for information about the user’s task container, and emit events when the status of the container changes.

Additional Parameters:
- Lifecycle events to publish

Example Use Cases
- Garbage collect an entire subdirectory when a task ends in error.
- Combined with `stdout`/`stderror` and Filesystem sidecars this can implement any existing Argo pipeline behavior. Simply emit events corresponding the Argo output, and events corresponding to the successful completion of the task, and depend on both events to start the next task.
- Update pipeline UI elements

#### Network Proxy Sidecar

This sidecar can proxy network requests from the user task container to their final destination, and emit events as a side-effect.

Additional Parameters:
- Local port to bind to
- ???
Example Use Cases:
- Aggregate, filter, or window events emitted by other sidecars, by proxying from and to the kubernetes API.

### Switching to Other Pub/Sub implementations
The implementation as currently described uses the Kuberentes API for event publishing and subscription, effectively relying on the underlying etc-d cluster of the k8s cluster for Pub/Sub capability. There may be cases when this is undesirable, in which case we should provide common capabilities that make it easy to switch out the underlying Pub/Sub infrastructure used (e.g. to Google Cloud Pub/Sub). This requires two pieces:

By convention Sidecar containers should publish events to a central service in the cluster (the address of which will be a sidecar parameter). By default this service will simply passthrough to the k8s master, but it should be swappable with a service that publishes to another Pub/Sub system.
The Argo controller needs to send watch requests to the central service, rather than the kubernetes master. The service is responsible for converting these requests into “subscriptions” if it does not passthrough to the k8s master. On creation the controller will need the cluster address of this central service.

### Example Pipelines

#### Recursive Coinflip

```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: coinflip-recursive-
spec:
  entrypoint: coinflip
  templates:
  - name: coinflip
    steps:
    # flip a coin
    - - name: flip-coin
        template: flip-coin
    # evaluate the result in parallel
    - - name: heads
        template: heads
        when: "{{steps.heads.inputs.coin}} == heads"
  - name: flip-coin
    script:
      image: python:alpine3.6
      command: [python]
      source: |
        import random
        while True:
          result = "heads" if random.randint(0,1) == 0 else "tails"
          print(result)
    inputs:
      - name: stopsignal
        valueFrom:
          eventParameter:
            metadata:
              name: stopsignal
            eventPolicy: delete
    sidecars:
      - name: stdoutwatcher
        image: argo/stdoutwatcher:1.0
        parameters:
          - stream: stdout
          - name: coin
  - name: heads
    container:
      image: alpine:3.6
      command: [sh, -c]
      args: ["echo \"it was heads\""]
    sidecars:
      - name: lifecyclewatcher
        image: argo/lifecyclewatcher:1.0
        parameters:
         - stream: success
         - name: stopsignal
     inputs:
      - name: coin
        valueFrom:
          eventParameter:
            metadata: 
              name: coin
            objectField: content
            eventPolicy: create

```

#### Explanation

- The tails step is gone. It’s only job was to requeue flip-coin, since flip coin is a potentially stateful task in this pipeline, we don’t requeue it. We stop it with events, when it should be stopped, and it continuously outputs events (note the while True line in the source field)
- Both heads and `flip-coin` have a sidecar that emits an event (`argoproj.io/v1betaX.Event` since it’s unspecified) to the kubernetes API.
- Since we care about the content of the stdout in flip-coin we made the sidecar a stdoutwatcher. It emits one event for every line in stdout, with a content field that contains the body of the line.
- Since heads is stateless, we just care about when it runs and when it completes, so we made the sidecar a lifecyclewatcher which will emit an event whenever a heads task completes successfully
- The heads step is otherwise unaltered.
- The `flip-coin` watcher emits events with `eventType: coin`, we also declare these events as outputs so that the Argo controller can use them for normal (non-stateful) tasks. 
- Since heads is a non-stateful step, we can use normal argo conditionals, the heads step will only be started if there is a coin event with content field heads 
- The heads step emits events of type stopsignal. 
- `flip-coin` takes these events as inputs. Note that the eventParameter has a different `eventPolicy: delete` which means the task will be stopped when we see an event with type stopsignal. Notably this will only happen once a heads task runs.  We don’t care about the actual value of the stopsignal only when it exists.

At first this doesn’t seem like a very compelling example. It’s quite a lot of extra config for functionally the same thing. However this is because the current `flip-coin` is stateless. Imagine instead we wished to flip a coin that was more likely to come up heads each time we flipped it and it came up tails (thus creating a distribution of flips that converged to the expected distribution more quickly). For this we need to know the number of tails flips which have occured. Our new source is:

```
import random
num_tails = 0
while True:
  if random.random() < threshold(num_tails):
    num_tails += 1
    print(“tails”)
  else:
    num_tails = 0
    print(“heads”)
 ```

To implement this in vanilla argo, we must pass a `num_tails` argument through the pipeline and recursively back into `flip-coin` even though there is no component that cares about it other than flip-coin. If initializing `num_tails` from it’s serialized value was computationally expensive this would also incur quite a performance hit.

