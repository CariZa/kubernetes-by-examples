# kubernetes-by-examples
A collection of a ton of kubernetes examples

# Volumes

Official documentation:

https://kubernetes.io/docs/concepts/storage/volumes/



Typical structure:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ______
spec:
  containers:
  - image: ______
    name: ______
    volumeMounts:
    - mountPath: ______
      name: ______
  volumes:
  - name: ______
    ______:
        ______....
```

The fields to note with volumes on a Pod definition are:

Under the pod spec, within the image level:

```yaml
spec:
    ...
    containers:
    - image:
      ...
      volumeMounts:
        - mountPath: ______
          name: ______
      ...
    ...
```

And then right at the bottom of the pod spec there is the volumes block:

```yaml
spec: 
  ...
  volumes:
  - name: ______
    ______:
        ______....
```

So from a Pod perspective keep those 2 parts in mind. 

1. The image needs to be aware of the volume mount using a "volumeMounts" field, the name field then tells the image which volume to attach to.
2. The overall spec also creates a reference to the available volumes, and creates that link between a volume and the pod.

