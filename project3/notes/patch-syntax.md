Patch support in Kustomize has been evolving over time, so there are
actually several different ways of referring to patches. Originally,
there were [separate directives][] for strategic merge and jsonpatch
patches:

[separate directives]: https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/#customizing

```
patchesStrategicMerge:
  - path/to/patch.yaml

patchesJson6902:
- target:
    group: apps
    version: v1
    kind: Deployment
    name: my-nginx
  path: path/to/patch.yaml
```

It is now possible to use a single `patches` directive, with the type
of match determined from it's content:

```
patches:
  - path: path/to/patch.yaml
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: my-nginx
    path: path/to/patch.yaml
```

Additionally, it is possible to specify patches inline in the
`kustomization.yaml` file, rather than referring to an external file:

```
patches:
  # set the number of replicas to 3
  - patch: |-
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: example
      spec:
        replicas: 3
```

The documentation for this isn't great and is sort of scattered across
multiple documents looking at specific aspects of this support.
Relevant documents include:

- https://github.com/kubernetes-sigs/kustomize/blob/master/examples/inlinePatch.md
- https://github.com/kubernetes-sigs/kustomize/blob/master/examples/patchMultipleObjects.md
- https://github.com/kubernetes-sigs/kustomize/blob/master/examples/jsonpatch.md
