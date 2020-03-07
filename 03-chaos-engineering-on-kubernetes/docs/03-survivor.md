# Survivor

Survivor is a simple [Go Blog](https://github.com/mikaelm1/Blog-App-Buffalo) built using [buffalo](https://github.com/gobuffalo/buffalo)

> [Original repository](https://github.com/danacr/survivor), I also added it as a [submodule](../survivor/README.md), to this repository.

I chose to fork and use this project because it Buffalo which allows me to create 27Mb docker images which contain all of the application assets.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: survivor
spec:
  replicas: 3
  selector:
    matchLabels:
      app: survivor
  template:
    metadata:
      labels:
        app: survivor
    spec:
      containers:
        - name: survivor
          envFrom:
            - secretRef:
                name: config
          image: eu.gcr.io/kill-the-cluster/survivor:stable3
          imagePullPolicy: IfNotPresent
          resources:
            limits:
              memory: "64Mi"
              cpu: "100m"
          ports:
            - containerPort: 3000
```

> Note: I moved the application to [Google Cloud Run](https://cloud.google.com/run) after the demo
