The purpose of this tutorial is to show you how to deploy a complete demo with EMQX5 on k8s, you can get some helps from the [README](../../README.md) of this repo if you using EMQX4.

## Install EMQX-Operator
Refer [Getting Started](https://docs.emqx.com/en/emqx-operator/latest/getting-started/getting-started.html#deploy-emqx-operator) to learn how to deploy EMQX operator

## Deploy EMQX Cluster
```shell
cat << "EOF" | kubectl apply -f -
apiVersion: apps.emqx.io/v2alpha1
kind: EMQX
metadata:
  name: emqx
spec:
  image: emqx/emqx-enterprise:latest
  coreTemplate:
    spec:
      replicas: 1
      ports:
        - containerPort: 18083
          name: dashboard
  replicantTemplate:
    spec:
      replicas: 1
      ports:
        - containerPort: 18083
          name: dashboard
EOF
```

## Deploy Exporter
You need to sign in EMQX dashboard to create an API secret, then pass the API key and secret to exporter startup argument as username and password.

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: emqx-exporter
  name: emqx-exporter-service
spec:
  ports:
    - name: metrics
      port: 8085
      targetPort: metrics
  selector:
    app: emqx-exporter
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: emqx-exporter
  labels:
    app: emqx-exporter
spec:
  selector:
    matchLabels:
      app: emqx-exporter
  replicas: 1
  template:
    metadata:
      labels:
        app: emqx-exporter
    spec:
      securityContext:
        runAsUser: 1000
      containers:
        - name: exporter
          image: emqx-exporter:latest
          imagePullPolicy: IfNotPresent
          args:
            - --emqx.nodes=emqx-dashboard:18083
            - --emqx.auth-username=${paste_your_new_api_key_here}
            - --emqx.auth-password=${paste_your_new_secret_here}
          securityContext:
            allowPrivilegeEscalation: false
            runAsNonRoot: true
          ports:
            - containerPort: 8085
              name: metrics
              protocol: HTTP
          resources:
            limits:
              cpu: 100m
              memory: 100Mi
            requests:
              cpu: 100m
              memory: 20Mi
```

Save the yaml content to file `emqx.yaml`, paste your new creating API key and secret, then apply it
```shell
kubectl apply -f emqx.yaml
```

## Import Prometheus Scrape Config
Assuming that you have deployed prometheus in your k8s env by [Prometheus Operator](https://prometheus-operator.dev/) or [Kube-Prometheus](https://github.com/prometheus-operator/kube-prometheus), you need to add scrape config by define `PodMonitor` and `ServiceMonitor` CR.  

[Here](https://github.com/prometheus-operator/prometheus-operator/blob/main/Documentation/user-guides/getting-started.md) is a sample example about `PodMonitor` and `ServiceMonitor`.  
In addition, you can use cmd `kubectl explain` to see the comment about the CR spec. 

In most cases, it's easier to deploy prometheus by `Deployment` without operator if you are new for this, you can get the scrape config example from [here](../docker) 

```shell
cat << "EOF" | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: emqx-metrics
  labels:
    app: emqx-metrics
spec:
  selector:
    matchLabels:
      # the label in emqx svc
      apps.emqx.io/instance: emqx
      apps.emqx.io/managed-by: emqx-operator
  podMetricsEndpoints:
    - port: dashboard
      interval: 5s
      path: /api/v5/prometheus/stats
      relabelings:
        - action: replace
          replacement: emqx5.0
          targetLabel: cluster
        - action: replace
          replacement: emqx
          targetLabel: from
        - action: replace
          sourceLabels: ['pod']
          targetLabel: "instance"
  namespaceSelector:
    # modify the namespace if your EMQX cluster deployed in other namespace
    matchNames:
      - default

---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: emqx-exporter
  labels:
    app: emqx-exporter
spec:
  selector:
    matchLabels:
      # the label in emqx exporter svc
      app: emqx-exporter
  endpoints:
    - port: metrics
      interval: 5s
      path: /metrics
      relabelings:
        - action: replace
          replacement: emqx5.0
          targetLabel: cluster
        - action: replace
          replacement: exporter
          targetLabel: from
        - action: replace
          sourceLabels: ['pod']
          regex: '(.*)-.*-.*'
          replacement: $1
          targetLabel: "instance"
        - action: labeldrop
          regex: 'pod'
          #sourceLabels: ['pod']
  namespaceSelector:
    # modify the namespace if your exporter deployed in other namespace
    matchNames:
      - default
EOF
```

## Load Grafana Templates
Import all [templates](../../config/grafana-template) to your grafana, then browse the dashboard EMQX and enjoy yourself!