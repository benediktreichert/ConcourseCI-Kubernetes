# ConcourseCI / Concourse on Kubernetes
The following templates will help you deploy [ConcourseCI](http://concourse.ci) / Concourse on Kubernetes 1.7. 

This is not a copy-paste solution, please go through the files and make adjustments according to your needs, for example deploying into a different namespace.

## Concourse Web
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: concourse-web
  labels:
    app: course
    concourse: web
spec:
  strategy:
    type: Recreate
  replicas: 1
  revisionHistoryLimit: 20
  template:
    metadata:
      labels:
        app: concourse
        concourse: web
    spec:
      restartPolicy: Always
      containers:
      - image: concourse/concourse:3.3.4
        name: concourse
        args: [ web ]
        env:
        - name: postgres_username
          valueFrom:
            secretKeyRef:
              name: concourse
              key: postgres_user
        - name: postgres_password
          valueFrom:
            secretKeyRef:
              name: concourse
              key: postgres_password
        - name: CONCOURSE_BASIC_AUTH_USERNAME
          valueFrom:
            secretKeyRef:
              name: concourse
              key: concourse_auth_username
        - name: CONCOURSE_BASIC_AUTH_PASSWORD
          valueFrom:
            secretKeyRef:
              name: concourse
              key: concourse_auth_password
        - name: CONCOURSE_EXTERNAL_URL
          value: "http://concourse-web.default.svc.cluster.local:8080"
        - name: CONCOURSE_POSTGRES_DATA_SOURCE
          value: postgres://$(postgres_username):$(postgres_password)@concourse-postgres.default.svc.cluster.local:5432/concourse?sslmode=disable
        volumeMounts:
        - name: concourse-keys
          mountPath: /concourse-keys
          readOnly: true
      volumes:
      - name: concourse-keys
        secret:
          secretName: concourse
          defaultMode: 0400
---
```

## Concourse Workers
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: concourse-worker
  labels:
    app: course
    concourse: worker
spec:
  strategy:
    type: Recreate
  replicas: 3
  revisionHistoryLimit: 20
  template:
    metadata:
      labels:
        app: concourse
        concourse: worker
    spec:
      restartPolicy: Always
      containers:
      - image: concourse/concourse:3.3.4
        name: concourse
        args: [ worker ]
        env:
        - name: CONCOURSE_TSA_HOST
          value: concourse-web.default.svc.cluster.local
        volumeMounts:
        - name: concourse-keys
          mountPath: /concourse-keys
          readOnly: true
        securityContext:
          privileged: true
      volumes:
      - name: concourse-keys
        secret:
          secretName: concourse-workers
          defaultMode: 0400
```

## Service
```
apiVersion: v1
kind: Service
metadata:
  name: concourse-web
  labels:
    concourse: web
    app: concourse
spec:
  ports:
  - name: web
    port: 8080
  - name: tsa
    port: 2222
  selector:
    app: concourse
    concourse: web
```

## Secrets
```
apiVersion: v1
kind: Secret
metadata:
  name: concourse
stringData:
  concourse_auth_username: < your data >
  concourse_auth_password: < your data >
  postgres_user: < your data >
  postgres_password: < your data >
  session_signing_key: < your data >
  tsa_host_key: < your data >
  authorized_worker_keys: < your data >
---
apiVersion: v1
kind: Secret
metadata:
  name: concourse-workers
stringData:
  tsa_host_key.pub: < your data >
  worker_key: < your data >
```

## Postgres
```
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: concourse-postgres
  labels:
    concourse: db
    app: concourse
spec:
  replicas: 1
  serviceName: concourse-postgres
  podManagementPolicy: Parallel
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      name: concourse-postgres
      labels:
        concourse: db
        app: concourse
    spec:
      terminationGracePeriodSeconds: 10
      containers:
        - name: concourse-postgres
          image: postgres:9.6
          env:
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: concourse
                  key: postgres_user
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: concourse
                  key: postgres_password
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          resources:
            requests:
              memory: 400M
              cpu: 0.4
            limits:
              memory: 1G
              cpu: 2
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: ssd-normal
        resources:
          requests:
            storage: 8Gi
---
apiVersion: v1
kind: Service
metadata:
  name: concourse-postgres
  labels:
    app: concourse
spec:
  ports:
  - name: postgres-concourse
    port: 5432
  selector:
    app: concourse
    concourse: db
```
