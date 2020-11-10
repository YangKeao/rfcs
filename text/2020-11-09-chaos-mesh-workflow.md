# Workflow

## Summary

A built-in workflow engine with frontend support is needed for Chaos Mesh.

## Motivation

Both running chaos experiments parallelly or sequentially are important for
simulating production errors. Chaos Mesh is shipped with a feature-rich chaos
toolbox. Composing them together can make it much more powerful. With a workflow
engine, an experiment can turn from single chaos into a series of chaos with
some external logic (like checking health).

Previously, the `duration` and `scheduler` fields have enabled a very
fundamental "workflow" feature, so after implementing the workflow engine, these
fields should be deprecated.

## Detailed design

The specification for a `Workflow` custom resource can be concluded in the
following example:

```yaml
spec:
  cron: "..."
  entry: serial-chaos
  templates:
  - name: networkchaos
    type: NetworkChaos
    duration: "30m"
    spec:
      selector:
      labelSelectors:
        "component": "tikv"
      delay:
        latency: "90ms"
        correlation: "25"
        jitter: "90ms"
  - name: iochaos
    type: IOChaos
    duration: "10m"
    selector:
      labelSelectors:
        app: "etcd"
    volumePath: /var/run/etcd
    path: /var/run/etcd/**/*
    errno: 5
    percent: 50
  - name: parallel-chaos
    type: Parallel
    tasks:
    - ref: networkchaos
    - ref: iochaos
  - name: check
    type: Task
    task:
      image: 'alpine:3.6'
      command: ['/app']
    branch:
    - when: "stdout == 'continue'"
      ref: serial-chaos
    - when: “stdout == ‘continue’”
      ref: serial-chaos
  - name: serial-chaos
    type: Serial
    duration: "30m"
    tasks:
    - ref: networkchaos
    - ref: iochaos
    - ref: parallel-chaos
    - type: Suspend
      duration: "30m"
    - ref: check
```

### `templates` field

`templates` field describes all the templates. One of them is the entry of the
workflow (which is specified by the `entry` field). The `templates` is a slice
of template, each of them is a `*Chaos`, `Parallel`, `Serial` or `Task`. They
are distinguished according to the `type` field.

The following document will describe all these different kinds of `template`.

#### `*Chaos`

They are the same with the specification of already defined `*Chaos` types
without `scheduler` field. When running into this part, the workflow engine
should create a `*Chaos` resource, and delete it after the duration. If it
deletes the resource successfully, this step finishes and continues.

The created `*Chaos` resource should be a "common chaos", as Chaos Mesh doesn't
support to set duration without a scheduler.

#### `Parallel`

For a `Parallel` task, the engine should spawn these tasks at the same time, and
finish iff all the tasks have finished. The `tasks` field is a list of tasks,
which could be a reference to a template (by name), or an inline template.

You can find an example of the inline template in the `Suspend` task of
`serial-chaos`.

#### `Serial`

For a `Serial` task, the engine should run these tasks one by one. It enters the
next step iff the former one succeeded. After all these steps finished, this
task turns into the finished phase.

#### `Suspend`

For a `Suspend` task, the engine should do nothing. After a period of time
(specified by `duration` field), this task turns into the finished phase.

#### `Task`

`Task` is a special kind of template to integrate users' processes into the
workflow. In this step, the users can setup a container (with Kubernetes's pod)
to run their own process. This step finishes once the pod turns into
`PodSucceeded` phase. The description of the task could be more complicated with
more fields if needed in the future.

Combining `Parallel` and `Serial` can provide a powerful expression in task
combination. However, as it can only represent series-parallel graph, it's still
a subset of `dag`.

The `duration` field of these tasks should work as a deadline (for `Serial` and
`Parallel` tasks) or as a time specification (for `Suspend`)

### Implementation references

`Argo` is a really great workflow engine with a workflow definition. It is a
guideline for us to design and implement this feature.

#### Reconciler logic

We should manage "nodes" in status and the reconciler should act like a state
machine for these "nodes". e.g.

```yaml
- nodeName: xxxxx
  type: Container
  phase: Running
- nodeName: xxxxx
  type: ParallelTask
  phase: Succeeded
```

Creating nodes could have side-effects, such as creating containers or creating
chaos mesh resources. But updating phase (in reconciler of `Workflow`) should be
pure to keep the logic simple.

#### Structure definition

The `Workflow` structure could be really complicated. Defining it in detail with
fields could result in duplicated codes with existing `*Chaos`. Or we could set
`Workflow` as `unstructured` or set the `templates` as
`[]k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1.JSON` and decoding
it with `mapstructure` pkg.

Without defining the detailed type, we cannot get a basic validation from
kubebuilder, so that we need to write a validating webhook for `Workflow`.

## Alternatives

### Provide Argo templates

Providing Argo templates seems a good solution to integrate Chaos Mesh with a
powerful workflow engine. From the technological aspect, it could be the best
solution. However, Chaos Mesh has the ambition to become an integrated platform
for chaos engineering, which should have a workflow engine inside out of the
box.

Installing Argo for the users is also not a choice. Because the CRD is not in
namespace scope, installing Argo with Chaos Mesh for users could break existing
Argo in the users' cluster

## Unresolved questions

No. AFAIK
