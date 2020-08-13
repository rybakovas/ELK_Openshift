# Overview

This document will show step-by-step to deploy a Elastic Stack Solution, AKA ELK (ElasticSearch, Logstash and Kibana), in OpenShift V 4.X. (in my case inside the IBM Cloud)

Pre-required to this Environment :

- VM Linux or Linux OS
- OC client (oc command by prompt)
- kubectl 
- OCP Admin Access
- Openshift high than V4.X

For More information about ELK Solutions please visit Elastic Site [elastic.co/](https://www.elastic.co/)
For More information about OCP and Openshift please visit RedHat Site [docs.openshift.com](https://docs.openshift.com/container-platform/4.3/welcome/index.html)

# First of all...

Connect to your OCP by command line using the command bellow

```shell
oc login --token=<OCP_TMP_TOKEN> --server=<OCP_ADDRESS:PORT>
```

Once you are connect you can run the kubectl command to create our Project (Namespace) inside our Openshift.

```shell
kubectl create -f ./namespace.yml
```

Inside the [namespace.yml](/namespace.yml) file you will find all instructions of Openshift needs to do the creation of our monitoring Namespace.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
```

The Second part is very important, once you are using Openshift we need to give privilege access to the Pods running well. 
Run the commands Bellow in you Linux prompt 

```shell
oc adm policy add-scc-to-user privileged -z default -n monitoring
oc adm policy add-scc-to-user privileged system:serviceaccount:monitoring:es-cluster
oc adm policy add-scc-to-user privileged system:serviceaccount:monitoring:filebeat
oc adm policy add-scc-to-user privileged system:serviceaccount:kube-system:metricbeat
```

Once you had ran those commands your Openshift is ready to receive your ELK Deployment. 

# About elk_stack_plus.yml file

Inside the [elk_stack_plus.yml](/elk_stack_plus.yml) you will find all instructions for Openshift do the installation / configuration of Elastic components bellow:

- [ElasticSearch](#elasticsearch) (Cluster - 3 nodes)
- [Kibana](#kibana) ( port export need be done separate)
- [Logstash](#logstash)
- [Filebeat](#filebeat)
- [MetricBeat](#metricbeat) (Inside kube-system Namespace)

Running kubectl command in your Linux will create the entire ELK Stack in your Openshift

```shell
kubectl create -f elk_stack_plus.yml
```

Lets check all components of this yml file.

## ElasticSearch

In this part we create 3 components inside the Openshift, ConfigMap, Service and StatefulSet.
The ConfigMap will help the ELK Admin to change or implement new configuration in a easy way. In our case all security options are comment, so please, check if your POC need the security part implemented as well. Check more information into [Elastic Site](https://www.elastic.co/guide/en/elasticsearch/reference/current/secure-cluster.html)

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: elasticsearch-config
  namespace: monitoring
data:
  elasticsearch.yml: |-
    cluster.name: "docker-cluster"
    network.host: 0.0.0.0
    #xpack.security.audit.enabled: true
    #xpack.monitoring.collection.enabled: true
    #xpack.security.enabled: true
```

Now lets create the Service, to do the port 9200 be open for the Openshift Pods.

```yaml
---
kind: Service
apiVersion: v1
metadata:
  name: elasticsearch
  namespace: monitoring
  labels:
    app: elasticsearch
spec:
  selector:
    app: elasticsearch
  clusterIP: None #In this case we not will set a IP for this service
  ports:
    - port: 9200
      name: rest
    - port: 9300
      name: inter-node
```

Once the Service is created the StatefulSet will create and config the ElasticSearch Pods

```yaml
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: es-cluster
  namespace: monitoring
spec:
  serviceName: elasticsearch
  replicas: 3
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:7.6.1
        volumeMounts:
        - name: elasticsearch-config
          mountPath: /usr/share/elasticsearch/config/elasticsearch.yml
          subPath: elasticsearch.yml
        resources:
            limits:
              cpu: 1000m
            requests:
              cpu: 100m
        ports:
        - containerPort: 9200
          name: rest
          protocol: TCP
        - containerPort: 9300
          name: inter-node
          protocol: TCP
        env:
          - name: cluster.name
            value: k8s-logs
          - name: node.name
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: discovery.seed_hosts
            value: "es-cluster-0.elasticsearch,es-cluster-1.elasticsearch,es-cluster-2.elasticsearch"
          - name: cluster.initial_master_nodes
            value: "es-cluster-0,es-cluster-1,es-cluster-2"
          - name: ES_JAVA_OPTS
            value: "-Xms512m -Xmx512m" #value to JVM, 512M or more
      volumes:
      - name: elasticsearch-config
        configMap:
          name: elasticsearch-config
      initContainers:
      - name: increase-vm-max-map
        image: busybox
        command: ["sysctl", "-w", "vm.max_map_count=262144"]
        securityContext:
          runAsUser: 0
          privileged: true
      - name: increase-fd-ulimit
        image: busybox
        command: ["sh", "-c", "ulimit -n 65536"]
        securityContext:
          runAsUser: 0 #For Openshift Security
          privileged: true #For Openshift Security
```

## Kibana

In this part we create 3 components inside the Openshift, ConfigMap, Service and Deployment.
The ConfigMap will help the ELK Admin to change or implement new configuration in a easy way

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-config
  namespace: monitoring
data:
  kibana.yml: |-
    server.name: kibana
    server.host: "0"
    elasticsearch.hosts: [ "http://elasticsearch:9200" ]
    #xpack.monitoring.ui.container.elasticsearch.enabled: true
    #xpack.reporting.enabled: true
    #xpack.security.audit.enabled: true
    #xpack.security.enabled: true
    #xpack.reporting.queue.timeout: 240000
    #xpack.monitoring.kibana.collection.enabled: false
    elasticsearch.username: elastic
    elasticsearch.password: changeme
```

Now lets create the Service, to do the port 5601 be open for the Openshift Pods.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: monitoring
  labels:
    app: kibana
spec:
  ports:
  - port: 5601
  selector:
    app: kibana
```

Once the Service is created the Deployment will create and config the Kibana Pod

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: monitoring
  labels:
    app: kibana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:7.6.1
        volumeMounts:
        - name: kibana-config
          mountPath: /usr/share/kibana/config/kibana.yml
          subPath: kibana.yml
        resources:
          limits:
            cpu: 1000m
          requests:
            cpu: 100m
        env:
          - name: ELASTICSEARCH_URL
            value: http://elasticsearch:9200
        ports:
        - containerPort: 5601
      volumes:
      - name: kibana-config
        configMap:
          name: kibana-config
```

**Important**, to you access the Kibana web interface you must create a new Route in your Openshift Project. By OCP is very easy, Next,Next, Finish.

## Logstash

In this part we create 4 components inside the Openshift, 2 ConfigMap (one for application configuration and other for filters and send configuration) , Service and Deployment.
The ConfigMap will help the ELK Admin to change or implement new configuration in a easy way.

```yaml
apiVersion: v1
kind: ConfigMap #Application Config
metadata:
  name: logstash-config-yaml 
  namespace: monitoring
data:
  logstash.yml: |-
    http.host: "0.0.0.0"
    xpack.monitoring.elasticsearch.hosts: [ "http://elasticsearch.monitoring.svc.cluster.local:9200" ] #Elastic IP
    xpack.monitoring.enabled: false

```

The Second ConfigMap will pass the log source, the filters and the index target (ElasticSearch)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-config
  namespace: monitoring
data:
  logstash.conf: |-
      input {
        beats {
            port => "9600" #Default port for Beats
        }
      }

      filter {

         grok {
           match => { "message" => "(?m)\[%{TIMESTAMP_ISO8601:logdate}\] %{WORD:ID} %{URIPROTO:LoggerName}%{SPACE}*%{WORD:LogLevel}%{SPACE}*%{GREEDYDATA:detail}"}
         }

      }

      output {
        # You can uncomment this line to investigate the generated events by the logstash.
        elasticsearch {
            hosts => "http://elasticsearch.monitoring.svc.cluster.local:9200" #ElasticSearch Address
            #user => "elastic"
            #password => "changeme"
            manage_template => false
            # The events will be stored in elasticsearch under previously defined index_prefix value.
            index => "%{[kubernetes][namespace]}%{[@metadata][beat]}%{+YYYY.MM.dd}"
            document_type => "%{[@metadata][type]}"
        }
      }
```

Now lets create the Service, to do the port 9600 be open for the Openshift Pods.

```yaml
kind: Service
apiVersion: v1
metadata:
  name: logstash
  namespace: monitoring
spec:
  type: NodePort
  selector:
    app: logstash
  clusterIP: None
  type: ClusterIP
  sessionAffinity: None
  ports:
  - protocol: TCP
    port: 9600
    targetPort: 9600
    name: logstash
```

The Deployment will create and config the Logstash Pod

```yaml
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: logstash
  namespace: monitoring
spec:
  template:
    metadata:
      labels:
        app: logstash
    spec:
      hostname: logstash
      containers:
      - name: logstash
        ports:
          - containerPort: 9600
            name: logstash
        image: docker.elastic.co/logstash/logstash:7.6.1
        volumeMounts:
        - name: logstash-config
          mountPath: /usr/share/logstash/pipeline/logstash.conf
          subPath: logstash.conf
        - name: logstash-config-yaml
          mountPath: /usr/share/logstash/config/logstash.yml
          subPath: logstash.yml
        command:
        - logstash
      volumes:
      # Previously defined ConfigMap object.
      - name: logstash-config
        configMap:
          name: logstash-config
      - name: logstash-config-yaml
        configMap:
          name: logstash-config-yaml
```

Done Your ELK Stack is up and Running into your Openshift.

## Filebeat

Now we need to collect the pod Logs with Filebeat. Let check the Filebeat Part.
In this part we create 5 components inside the Openshift, ConfigMap, DaemonSet, ClusterRoleBinding, ClusterRole, and ServiceAccount.

The ConfigMap will help the ELK Admin to change or implement new configuration in a easy way.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
  namespace: monitoring
  labels:
    k8s-app: filebeat
data:
  filebeat.yml: |-
    filebeat.inputs:
    - type: container
      paths:
        - /var/log/containers/*.log #Pods log path
      processors:
        - add_kubernetes_metadata:
            host: ${NODE_NAME}
            matchers:
            - logs_path:
                logs_path: "/var/log/containers"
    processors:
      - add_cloud_metadata:
      - add_host_metadata:
      - add_kubernetes_metadata:
    setup.template.name: "-index-"
    setup.template.pattern: "-index-*"
    output.logstash:
      hosts: ['${LOGSTASH_HOSTS}']
      index: "-index-"
```

**Important** For Filebeat, inside the configuration we can't have different log source, if necessary the Administrator must create a new Filebeat Deployment to collected the logs from this source. 

The DaemonSet  will create and config the Filebeat Pods

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filebeat
  namespace: monitoring
  labels:
    k8s-app: filebeat
spec:
  selector:
    matchLabels:
      k8s-app: filebeat
  template:
    metadata:
      labels:
        k8s-app: filebeat
    spec:
      serviceAccountName: filebeat
      terminationGracePeriodSeconds: 30
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:7.6.0
        args: [
          "-c", "/etc/filebeat.yml",
          "-e",
        ]
        env:
        - name: LOGSTASH_HOSTS
          value: logstash.monitoring.svc.cluster.local:9600 #Logstash address
        - name: ELASTICSEARCH_HOST
          value: elasticsearch.monitoring.svc.cluster.local #ElasticSearch address
        - name: ELASTICSEARCH_PORT
          value: "9200"
        - name: ELASTICSEARCH_USERNAME
          value: elastic
        - name: ELASTICSEARCH_PASSWORD
          value: changeme
        - name: ELASTIC_CLOUD_ID
          value:
        - name: ELASTIC_CLOUD_AUTH
          value:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        securityContext:
          runAsUser: 0
          # If using Red Hat OpenShift uncomment this:
          privileged: true
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 100Mi
        volumeMounts:
        - name: config
          mountPath: /etc/filebeat.yml
          readOnly: true
          subPath: filebeat.yml
        - name: data
          mountPath: /usr/share/filebeat/data
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: varlog
          mountPath: /var/log
          readOnly: true
      volumes:
      - name: config
        configMap:
          defaultMode: 0600
          name: filebeat-config
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: varlog
        hostPath:
          path: /var/log
      # data folder stores a registry of read status for all files, so we don't send everything again on a Filebeat pod restart
      - name: data
        hostPath:
          path: /var/lib/filebeat-data
          type: DirectoryOrCreate
```

The ClusterRoleBinding, ServiceAccount and ClusterRole are related to Openshift Cluster Security

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: filebeat
subjects:
- kind: ServiceAccount
  name: filebeat
  namespace: monitoring
roleRef:
  kind: ClusterRole
  name: filebeat
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: filebeat
  labels:
    k8s-app: filebeat
rules:
- apiGroups: [""] # "" indicates the core API group
  resources:
  - namespaces
  - pods
  verbs:
  - get
  - watch
  - list
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: filebeat
  namespace: monitoring
  labels:
    k8s-app: filebeat
```

Now your pods logs are been collected by Filebeat and send to Logstash do all work on index creation.

## MetricBeat

With Metricbeat ELK are able to take the Cluster "infrastructure" metrics, like CPU, memory, Network, etc. 
**Important** Metricbeat should be deploy inside kube-system namespace to work and Collect the information from cluster.
In this part we create 6 components inside the Openshift, 2 ConfigMap, DaemonSet, ClusterRoleBinding, ClusterRole, and ServiceAccount.

Like FileBeat, MetricBeat have also 2 ConfigMaps, one for the Application and other to Configurated what module will be used to Collect the information from Cluster

This one is for the application

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: metricbeat-daemonset-config
  namespace: kube-system
  labels:
    k8s-app: metricbeat
data:
  metricbeat.yml: |-
    metricbeat.config.modules:
      # Mounted `metricbeat-daemonset-modules` configmap:
      path: ${path.config}/modules.d/*.yml
      # Reload module configs as they change:
      reload.enabled: false

    # To enable hints based autodiscover uncomment this:
    metricbeat.autodiscover:
      providers:
        - type: kubernetes
          node: ${NODE_NAME}
          hints.enabled: true

    processors:
      - add_cloud_metadata:
      - add_docker_metadata:
      - add_host_metadata:
      - add_kubernetes_metadata:

    cloud.id: ${ELASTIC_CLOUD_ID}
    cloud.auth: ${ELASTIC_CLOUD_AUTH}

    output.elasticsearch:
      hosts: ["http://elasticsearch.monitoring.svc.cluster.local:9200"]
      username: "elastic"
      password: "changeme"
```

This is for the Modules:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: metricbeat-daemonset-modules
  namespace: kube-system
  labels:
    k8s-app: metricbeat
data:
  system.yml: |-
    - module: system
      period: 10s
      metricsets:
        - cpu
        - load
        - memory
        - network
        - process
        - process_summary
        #- core
        #- diskio
        #- socket
      processes: ['.*']
      process.include_top_n:
        by_cpu: 5      # include top 5 processes by CPU
        by_memory: 5   # include top 5 processes by memory

    - module: system
      period: 1m
      metricsets:
        - filesystem
        - fsstat
      processors:
      - drop_event.when.regexp:
          system.filesystem.mount_point: '^/(sys|cgroup|proc|dev|etc|host|lib|snap)($|/)'
  kubernetes.yml: |-
    - module: kubernetes
      metricsets:
        - node
        - system
        - pod
        - container
        - volume
      period: 10s
      host: ${NODE_NAME}
      hosts: ["https://${HOSTNAME}:10250"]
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      ssl.verification_mode: "none"
      # If using Red Hat OpenShift remove ssl.verification_mode entry and
      # uncomment these settings:
      ssl.certificate_authorities:
        - /var/run/secrets/kubernetes.io/serviceaccount/service-ca.crt
    - module: kubernetes
      metricsets:
        - proxy
      period: 10s
      host: ${NODE_NAME}
      hosts: ["localhost:10249"]
```

The DaemonSet  will create and config the Metricbeat Pods

```yaml
# Deploy a Metricbeat instance per node for node metrics retrieval
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: metricbeat
  namespace: kube-system
  labels:
    k8s-app: metricbeat
spec:
  selector:
    matchLabels:
      k8s-app: metricbeat
  template:
    metadata:
      labels:
        k8s-app: metricbeat
    spec:
      serviceAccountName: metricbeat
      terminationGracePeriodSeconds: 30
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      containers:
      - name: metricbeat
        image: docker.elastic.co/beats/metricbeat:7.6.1
        args: [
          "-c", "/etc/metricbeat.yml",
          "-e",
          "-system.hostfs=/hostfs",
        ]
        env:
        - name: ELASTICSEARCH_HOST
          value: elasticsearch.monitoring.svc.cluster.local #ElasticSearch Address
        - name: ELASTICSEARCH_PORT
          value: "9200"
        - name: ELASTIC_CLOUD_ID
          value:
        - name: ELASTIC_CLOUD_AUTH
          value:
        - name: ELASTICSEARCH_USERNAME
          value: elastic
        - name: ELASTICSEARCH_PASSWORD
          value: changeme
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        securityContext:
          runAsUser: 0
          privileged: true #For Openshift 
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 100Mi
        volumeMounts:
        - name: config
          mountPath: /etc/metricbeat.yml
          readOnly: true
          subPath: metricbeat.yml
        - name: modules
          mountPath: /usr/share/metricbeat/modules.d
          readOnly: true
        - name: dockersock
          mountPath: /var/run/docker.sock
        - name: proc
          mountPath: /hostfs/proc
          readOnly: true
        - name: cgroup
          mountPath: /hostfs/sys/fs/cgroup
          readOnly: true
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: cgroup
        hostPath:
          path: /sys/fs/cgroup
      - name: dockersock
        hostPath:
          path: /var/run/docker.sock
      - name: config
        configMap:
          defaultMode: 0600
          name: metricbeat-daemonset-config
      - name: modules
        configMap:
          defaultMode: 0600
          name: metricbeat-daemonset-modules
      - name: data
        hostPath:
          path: /var/lib/metricbeat-data
          type: DirectoryOrCreate

```

The ClusterRoleBinding, ServiceAccount and ClusterRole are related to Openshift Cluster Security

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: metricbeat
subjects:
- kind: ServiceAccount
  name: metricbeat
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: metricbeat
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: metricbeat
  labels:
    k8s-app: metricbeat
rules:
- apiGroups: [""]
  resources:
  - nodes
  - namespaces
  - events
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups: ["extensions"]
  resources:
  - replicasets
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources:
  - statefulsets
  - deployments
  - replicasets
  verbs: ["get", "list", "watch"]
- apiGroups:
  - ""
  resources:
  - nodes/stats
  verbs:
  - get
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: metricbeat
  namespace: kube-system
  labels:
    k8s-app: metricbeat
---
```

Done, now you have the ELK Stack + FileBeat + MetricBeat up and running in your Openshift.For more information about ELK please visit [elastic.co/](https://www.elastic.co/)

# Conclusion

My POC was very good and has run as expected, I was able to get logs (using MetricBeat and FileBeat ([Beats Reference](https://www.elastic.co/guide/en/beats/libbeat/current/beats-reference.html)) from my On Cloud Openshift. 
Need to add a Persistent Volume to keep all ElasticSearch/Kibana. 
