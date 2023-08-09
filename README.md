# [Backstage](https://backstage.io)

This is your newly scaffolded Backstage App, Good Luck!

To start the app, run:

```shell
$ yarn install
$ yarn dev
```

## Como fazer deploy com docker e GKE (localmente)

1. Clonar este repositorio ou rodar npx @backstage/create-app. depois:

```shell
$ yarn tsc
$ yarn build:backend -config ../../app-config.yaml
$ docker image build . -f packages/backend/Dockerfile --tag backstage
$ minikube start
```

```yaml
# kubernetes/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: backstage
```

```yaml
# kubernetes/postgres-secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secrets
  namespace: backstage
type: Opaque
data:
  POSTGRES_USER: YmFja3N0YWdl
  POSTGRES_PASSWORD: aHVudGVyMg==
```

```shell
$ echo -n "backstage" | base64
YmFja3N0YWdl

$ kubectl apply -f kubernetes/postgres-secrets.yaml
secret/postgres-secrets created
```

```yaml
# kubernetes/postgres-storage.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-storage
  namespace: backstage
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 2G
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: '/mnt/data'
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-storage-claim
  namespace: backstage
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2G
```

```shell
$ kubectl apply -f kubernetes/postgres-storage.yaml
persistentvolume/postgres-storage created
persistentvolumeclaim/postgres-storage-claim created
```

```yaml
# kubernetes/postgres.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: backstage
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:13.2-alpine
          imagePullPolicy: 'IfNotPresent'
          ports:
            - containerPort: 5432
          envFrom:
            - secretRef:
                name: postgres-secrets
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: postgresdb
      volumes:
        - name: postgresdb
          persistentVolumeClaim:
            claimName: postgres-storage-claim
```

```shell
$ kubectl apply -f kubernetes/postgres.yaml
deployment.apps/postgres created

$ kubectl get pods --namespace=backstage
NAME                        READY   STATUS    RESTARTS   AGE
postgres-56c86b8bbc-66pt2   1/1     Running   0          21s

$ kubectl exec -it --namespace=backstage postgres-56c86b8bbc-66pt2 -- /bin/bash
bash-5.1# psql -U $POSTGRES_USER
psql (13.2)
backstage=# \q
bash-5.1# exit
```

```yaml
# kubernetes/postgres-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: backstage
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
```

```shell
$ kubectl apply -f kubernetes/postgres-service.yaml
service/postgres created

$ kubectl get services --namespace=backstage
NAME       TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
postgres   ClusterIP   10.96.5.103   <none>        5432/TCP   29s
```

```yaml
# kubernetes/backstage-secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: backstage-secrets
  namespace: backstage
type: Opaque
data:
  GITHUB_TOKEN: VG9rZW5Ub2tlblRva2VuVG9rZW5NYWxrb3ZpY2hUb2tlbg==
```

```shell
$ kubectl apply -f kubernetes/backstage-secrets.yaml
secret/backstage-secrets created
```

```yaml
# kubernetes/backstage.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backstage
  namespace: backstage
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backstage
  template:
    metadata:
      labels:
        app: backstage
    spec:
      containers:
        - name: backstage
          image: backstage:1.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 7007
          envFrom:
            - secretRef:
                name: postgres-secrets
            - secretRef:
                name: backstage-secrets
# Uncomment if health checks are enabled in your app:
# https://backstage.io/docs/plugins/observability#health-checks
#          readinessProbe:
#            httpGet:
#              port: 7007
#              path: /healthcheck
#          livenessProbe:
#            httpGet:
#              port: 7007
#              path: /healthcheck
```

Alterar app-config.yaml e app-config.production.yaml

```yaml
backend:
  database:
    client: pg
    connection:
      host: ${POSTGRES_SERVICE_HOST}
      port: ${POSTGRES_SERVICE_PORT}
      user: ${POSTGRES_USER}
      password: ${POSTGRES_PASSWORD}
```

```shell
$ eval $(minikube docker-env)
$ yarn build-image --tag backstage:1.0.0


$ kubectl apply -f kubernetes/backstage.yaml
deployment.apps/backstage created

$ kubectl get deployments --namespace=backstage
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
backstage   1/1     1            1           1m
postgres    1/1     1            1           10m

$ kubectl get pods --namespace=backstage
NAME                                 READY   STATUS    RESTARTS   AGE
backstage-54bfcd6476-n2jkm           1/1     Running   0          58s
postgres-56c86b8bbc-66pt2            1/1     Running   0          9m

# -f to tail, <pod> -c <container>
$ kubectl logs --namespace=backstage -f backstage-54bfcd6476-n2jkm -c backstage
```

```yaml
# kubernetes/backstage-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: backstage
  namespace: backstage
spec:
  selector:
    app: backstage
  ports:
    - name: http
      port: 80
      targetPort: http
```

```shell
$ kubectl apply -f kubernetes/backstage-service.yaml
service/backstage created
```

```yaml
# kubernetes/backstage-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: backstage-ingress
  namespace: backstage
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
  spec:
    ingressClassName: nginx
  rules:
    - http:
  paths:
    - path: /
      pathType: Prefix
      backend:
        service:
          name: backstage
          port:
          number: 80
```

Alterar app-config.yaml e adicionar `upgrade-insecure-requests: false`

```yaml
# app-config.yaml
backend:
  csp:
    upgrade-insecure-requests: false
```

```shell
$ kubectl apply -f kubernetes/backstage-ingress.yaml
ingress.networking.k8s.io/backstage-ingress created

$ minikube addons enable ingress
$ minikube dashboard
```