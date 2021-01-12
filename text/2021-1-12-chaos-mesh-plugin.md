# Plugin

## Summary

A plugin mechanism should be designed for the users to develop customized chaos
and scheduler to fit their requirement.

## Motivation

The extensibility for a chaos platform is always important. It's impossible for
the developers of chaos mesh to provide application layer chaos for every
softwares or platforms. According to the feedback from the users, some of them
have modified chaos mesh to inject kernel errors and run some scripts as their
need.

## Detailed design

Several different kinds of plugins are needed:

1. Selector plugin. Provide the ability to customize selectors. For example:
   select the leaders of a database cluster.

2. Chaos plugin. Provide a new type of chaos.

Multiple ways could be provided to setup a plugin: through defining a custom
resource or putting binaries in volume. All communication between the
`controller-manager` and the plugins are through grpc (over the network or over
stdin/stdout).

### Register

This chapter describes how to register a plugin. The main goal of registering is
to setup a grpc connection between `controller-manager` and the plugin, and
record this connection in a map from `(type, name)` to it.

#### Custom Resource

We need to define a new kind of custom resource, called "ChaosPlugin". The
schema of it looks like:

```yaml
kind: Plugin
metadata:
    name: pd-selector
spec:
    type: SelectorPlugin
    service:
        name: pd-selector-plugin-svc
        namespace: chaos-testing
```

Then the grpc connection can be connected through the service, and the `type`
and `name` are described in the resource. The update of the plugin can be
listened through `controller-runtime`, as a normal reconciler.

#### Binary

Binaries under `/opt/chaos-mesh/chaos-plugins` are regarded as a chaos plugin,
and the binaries under `/opt/chaos-mesh/selector-plugins` are regarded as a
selector plugin. The name of them are the corresponding filename.

They will be executed automatically when the `controller-manager` starts up. The
`stdin` and `stdout` will be used to setup the grpc connection. The `stderr`
could be used as the log output, which will be redirected to the
`controller-manager`'s log output directly.

The update of the plugin is hard to listen. `poll` to read the directory could
be a solution, or we could provide a HTTP endpoint to trigger a reload manually.

### Protocol

We need to use protobuf to describe the protocol. Some common message type
should be defined first:

```protobuf
message GVK {
    string group = 1;
    string version = 2;
    string kind = 3;
}

message NamespacedName {
    string namespace = 1;
    string name = 2;
}
```

#### Selector Plugin

Based on another [rfc](https://github.com/chaos-mesh/rfcs/pull/12), the
following specification is designed:

```protobuf
rpc Select(SelectSpec) (SelectResponse) {}

message SelectSpec {
    GVK resource_type = 1;
    google.protobuf.Any select_spec = 2;
}

message SelectResponse {
    repeated NamespacedName objects = 1;
}
```

#### Chaos Plugin

It's hard to define the protocol of chaos plugin. We need to clarify what things
should be done in `controller-manager` and what should be done in the plugin. If
the `controller-manager` only calls reconcile of the plugin, there is no need to
develop a plugin compared to a seperated controller (with a powerful SDK).
However, if the `controller-manager` does more things, like setting up the
`twophase` reconciler, it should know some information about the schema, such as
how to get the scheduler, duration, nextStart ...

Another problem is that it may want to call the `controller-manager`, an may
lead to another all to the plugin. For example, it may need to select pods, so
that it may need to call selector plugins (directly or indirectly).

I haven't came up with a good enough idea to solve this. Maybe we could try
[go-plugin](https://github.com/hashicorp/go-plugin).

