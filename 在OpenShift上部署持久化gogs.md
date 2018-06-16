#  在OpenShift上部署持久化gogs

- 准备模板文件

```
kind: Template
apiVersion: v1
metadata:
  annotations:
    description: The Gogs git server (https://gogs.io/)
    tags: instant-app,gogs,datastore
    iconClass: "icon-github"
  name: gogs-persistent
objects:
- kind: ServiceAccount
  apiVersion: v1
  metadata:
    creationTimestamp: null
    labels:
      app: ${APPLICATION_NAME}
    name: ${APPLICATION_NAME}
- kind: Service
  apiVersion: v1
  metadata:
    annotations:
      description: Exposes the database server
    name: ${APPLICATION_NAME}-postgresql
  spec:
    ports:
    - name: postgresql
      port: 5432
      targetPort: 5432
    selector:
      name: ${APPLICATION_NAME}-postgresql
- kind: DeploymentConfig
  apiVersion: v1
  metadata:
    annotations:
      description: Defines how to deploy the database
    name: ${APPLICATION_NAME}-postgresql
  spec:
    replicas: 1
    selector:
      name: ${APPLICATION_NAME}-postgresql
    strategy:
      type: Recreate
    template:
      metadata:
        labels:
          name: ${APPLICATION_NAME}-postgresql
        name: ${APPLICATION_NAME}-postgresql
      spec:
        serviceAccountName: ${APPLICATION_NAME}
        containers:
        - env:
          - name: POSTGRESQL_USER
            value: ${DATABASE_USER}
          - name: POSTGRESQL_PASSWORD
            value: ${DATABASE_PASSWORD}
          - name: POSTGRESQL_DATABASE
            value: ${DATABASE_NAME}
          - name: POSTGRESQL_MAX_CONNECTIONS
            value: ${DATABASE_MAX_CONNECTIONS}
          - name: POSTGRESQL_SHARED_BUFFERS
            value: ${DATABASE_SHARED_BUFFERS}
          - name: POSTGRESQL_ADMIN_PASSWORD
            value: ${DATABASE_ADMIN_PASSWORD}
          image: ' '
          livenessProbe:
            initialDelaySeconds: 30
            tcpSocket:
              port: 5432
            timeoutSeconds: 1
          name: postgresql
          ports:
          - containerPort: 5432
          readinessProbe:
            exec:
              command:
              - /bin/sh
              - -i
              - -c
              - psql -h 127.0.0.1 -U ${POSTGRESQL_USER} -q -d ${POSTGRESQL_DATABASE} -c 'SELECT 1'
            initialDelaySeconds: 5
            timeoutSeconds: 1
          resources:
            limits:
              memory: 512Mi
          volumeMounts:
          - mountPath: /var/lib/pgsql/data
            name: gogs-postgres-data
        volumes:
        - name: gogs-postgres-data
          persistentVolumeClaim:
            claimName: gogs-postgres-data
    triggers:
    - imageChangeParams:
        automatic: true
        containerNames:
        - postgresql
        from:
          kind: ImageStreamTag
          name: postgresql:9.5
          namespace: openshift
      type: ImageChange
    - type: ConfigChange
- kind: Service
  apiVersion: v1
  metadata:
    annotations:
      description: The Gogs server's http port
      service.alpha.openshift.io/dependencies: '[{"name":"${APPLICATION_NAME}-postgresql","namespace":"","kind":"Service"}]'
    labels:
      app: ${APPLICATION_NAME}
    name: ${APPLICATION_NAME}
  spec:
    ports:
    - name: 3000-tcp
      port: 3000
      protocol: TCP
      targetPort: 3000
    selector:
      app: ${APPLICATION_NAME}
      deploymentconfig: ${APPLICATION_NAME}
    sessionAffinity: None
    type: ClusterIP
  status:
    loadBalancer: {}
- kind: Route
  apiVersion: v1
  id: ${APPLICATION_NAME}-http
  metadata:
    annotations:
      description: Route for application's http service.
    labels:
      app: ${APPLICATION_NAME}
    name: ${APPLICATION_NAME}
  spec:
    host: ${HOSTNAME}
    to:
      name: ${APPLICATION_NAME}
- kind: DeploymentConfig
  apiVersion: v1
  metadata:
    labels:
      app: ${APPLICATION_NAME}
    name: ${APPLICATION_NAME}
  spec:
    replicas: 1
    selector:
      app: ${APPLICATION_NAME}
      deploymentconfig: ${APPLICATION_NAME}
    strategy:
      resources: {}
      rollingParams:
        intervalSeconds: 1
        maxSurge: 25%
        maxUnavailable: 25%
        timeoutSeconds: 600
        updatePeriodSeconds: 1
      type: Rolling
    template:
      metadata:
        creationTimestamp: null
        labels:
          app: ${APPLICATION_NAME}
          deploymentconfig: ${APPLICATION_NAME}
      spec:
        serviceAccountName: ${APPLICATION_NAME}
        containers:
        - image: " "
          imagePullPolicy: Always
          name: ${APPLICATION_NAME}
          ports:
          - containerPort: 3000
            protocol: TCP
          resources: {}
          terminationMessagePath: /dev/termination-log
          volumeMounts:
          - name: gogs-data
            mountPath: /data
          - name: gogs-config
            mountPath: /etc/gogs/conf
          readinessProbe:
              httpGet:
                path: /
                port: 3000
                scheme: HTTP
              initialDelaySeconds: 3
              timeoutSeconds: 1
              periodSeconds: 20
              successThreshold: 1
              failureThreshold: 3
          livenessProbe:
              httpGet:
                path: /
                port: 3000
                scheme: HTTP
              initialDelaySeconds: 3
              timeoutSeconds: 1
              periodSeconds: 10
              successThreshold: 1
              failureThreshold: 3
        dnsPolicy: ClusterFirst
        restartPolicy: Always
        securityContext: {}
        terminationGracePeriodSeconds: 30
        volumes:
        - name: gogs-data
          persistentVolumeClaim:
            claimName: gogs-data
        - name: gogs-config
          configMap:
            name: gogs-config
            items:
              - key: app.ini
                path: app.ini
    test: false
    triggers:
    - type: ConfigChange
    - imageChangeParams:
        automatic: true
        containerNames:
        - ${APPLICATION_NAME}
        from:
          kind: ImageStreamTag
          name: ${APPLICATION_NAME}:latest
      type: ImageChange
- kind: ImageStream
  apiVersion: v1
  metadata:
    labels:
      app: ${APPLICATION_NAME}
    name: ${APPLICATION_NAME}
  spec:
    tags:
    - name: "${GOGS_VERSION}"
      from:
        kind: DockerImage
        name: docker.io/${GOGS_IMAGE}:${GOGS_VERSION}
      importPolicy: {}
      annotations:
        description: The Gogs git server docker image
        tags: gogs,go,golang
        version: "${GOGS_VERSION}"
    - annotations:
        description: The Gogs git server docker image
        tags: gogs,go,golang
        version: "latest"
      from:
        kind: ImageStreamTag
        name: "${GOGS_VERSION}"
      importPolicy: {}
      name: "latest"
- kind: PersistentVolumeClaim
  apiVersion: v1
  metadata:
    name: gogs-data
  spec:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: ${GOGS_VOLUME_CAPACITY}
- kind: PersistentVolumeClaim
  apiVersion: v1
  metadata:
    name: gogs-postgres-data
  spec:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: ${DB_VOLUME_CAPACITY}
- kind: ConfigMap
  apiVersion: v1
  metadata:
    name: gogs-config
  data:
    app.ini: |
      APP_NAME = Gogs
      RUN_MODE = prod
      RUN_USER = gogs

      [database]
      DB_TYPE  = postgres
      HOST     = ${APPLICATION_NAME}-postgresql:5432
      NAME     = ${DATABASE_NAME}
      USER     = ${DATABASE_USER}
      PASSWD   = ${DATABASE_PASSWORD}
      SSL_MODE = disable

      [repository]
      ROOT = /data/repositories

      [server]
      ROOT_URL=http://${HOSTNAME}

      [security]
      INSTALL_LOCK = ${INSTALL_LOCK}

      [mailer]
      ENABLED = false

      [service]
      ENABLE_CAPTCHA = false
      REGISTER_EMAIL_CONFIRM = false
      ENABLE_NOTIFY_MAIL     = false
      DISABLE_REGISTRATION   = false
      REQUIRE_SIGNIN_VIEW    = false

      [picture]
      DISABLE_GRAVATAR        = false
      ENABLE_FEDERATED_AVATAR = true

      [webhook]
      SKIP_TLS_VERIFY = ${SKIP_TLS_VERIFY}
parameters:
- description: The name for the application.
  name: APPLICATION_NAME
  required: true
  value: gogs
- description: 'Custom hostname for http service route.  Leave blank for default hostname, e.g.: <application-name>-<project>.<default-domain-suffix>'
  name: HOSTNAME
  required: true
- description: Volume space available for data, e.g. 512Mi, 2Gi
  name: GOGS_VOLUME_CAPACITY
  required: true
  value: 1Gi
- description: Volume space available for postregs data, e.g. 512Mi, 2Gi
  name: DB_VOLUME_CAPACITY
  required: true
  value: 1Gi
- displayName: Database Username
  from: gogs
  value: gogs
  name: DATABASE_USER
- displayName: Database Password
  from: '[a-zA-Z0-9]{8}'
  value: gogs
  name: DATABASE_PASSWORD
- displayName: Database Name
  name: DATABASE_NAME
  value: gogs
- displayName: Database Admin Password
  from: '[a-zA-Z0-9]{8}'
  generate: expression
  name: DATABASE_ADMIN_PASSWORD
- displayName: Maximum Database Connections
  name: DATABASE_MAX_CONNECTIONS
  value: "100"
- displayName: Shared Buffer Amount
  name: DATABASE_SHARED_BUFFERS
  value: 12MB
- name: INSTALL_LOCK
  displayName: Installation lock
  description: 'If set to true, installation (/install) page will be disabled. Set to false if you want to run the installation wizard via web'
  value: "true"
- name: SKIP_TLS_VERIFY
  displayName: Skip TLS verification on webhooks
  description: Skip TLS verification on webhooks. Enable with caution!
  value: "true"
- name: GOGS_IMAGE
  displayName: Gogs Image
  description: The name and tag for the Gogs Image to use
  value: "wkulhanek/gogs"
  required: true
- name: GOGS_VERSION
  displayName: Gogs Image Version Tag
  description: The tag for the Gogs Image to use
  value: "latest"
  required: true
```

- 使用该模板，然后创建，参数默认即可，其中域名地址（即route）要根据实际情况进行修改

- 导入is：gogs 和 postgresql, 镜像需要自行下载。

```
docker pull registry.access.redhat.com/rhscl/postgresql-95-rhel7
docker pull wkulhanek/gogs

docker tag registry.access.redhat.com/rhscl/postgresql-95-rhel7 txocp-registry.ccic-test.com.cn:5000/rhscl/postgresql-95-rhel7
docker tag wkulhanek/gogs txocp-registry.ccic-test.com.cn:5000/openshiftdemos/gogs

docker push txocp-registry.ccic-test.com.cn:5000/rhscl/postgresql-95-rhel7
docker push txocp-registry.ccic-test.com.cn:5000/openshiftdemos/gogs
```

```
oc import-image gogs --from=txocp-registry.ccic-test.com.cn:5000/openshiftdemos/gogs --confirm=true --insecure=true
```

```
oc import-image postgresql:9.5 --from=txocp-registry.ccic-test.com.cn:5000/rhscl/postgresql-95-rhel7 --confirm=true --insecure=true
```

- 创建pv: gogs-data, gogs-postgres-data

```
apiVersion: v1
kind: PersistentVolume
metadata:
  annotations:
    pv.kubernetes.io/bound-by-controller: "yes"
  name: gogs-data
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 1Gi
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: gogs-data
    namespace: ocp-service
  nfs:
    path: /vol/ThxocpJenkins/gogs-data
    server: 10.1.13.13
  persistentVolumeReclaimPolicy: Retain
```

```
apiVersion: v1
kind: PersistentVolume
metadata:
  annotations:
    pv.kubernetes.io/bound-by-controller: "yes"
  name: gogs-postgres-data
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 1Gi
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: gogs-postgres-data
    namespace: ocp-service
  nfs:
    path: /vol/ThxocpJenkins/gogs-postgres-data
    server: 10.1.13.13
  persistentVolumeReclaimPolicy: Retain
```

- 点击gogs和postgresql的DC进行部署，部署完成后点击route进行访问，注意域名解析问题，然后进行安装，安装的域名地址也需要注意填写，连接postgresql写svc域名或者IP都可以，加上端口。
- https://github.com/wkulhanek/docker-openshift-gogs

