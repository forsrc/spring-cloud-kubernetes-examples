# Setting up the Environment

To play with these examples, you can install locally Kubernetes & Docker using [Minikube](https://minikube.sigs.k8s.io/docs/start/) within a Virtual Machine
managed by a hypervisor (Xhyve, Virtualbox or KVM) if your machine is not a native Unix operating system.

  
When the minikube is installed on your machine, you can start kubernetes using this command:
```
minikube start
```

You also probably want to configure your Docker client to point the minikube Docker daemon with:
```
eval $(minikube docker-env)
```

This will make sure the Docker images that you build are available to the minikube environment.

# Hello World Example

This Spring Boot application exposes an endpoint that we can call to receive a `Hello World` message as response. The application is configured using the 
@RestController, and the method returning the message is annotated with `@RequestMapping("/")`.

The uberjar of the Spring Boot application is packaged within a Docker image using the [Fabric8 Maven plugin](https://maven.fabric8.io) and next deployed top of the Kubernetes management platform as a pod 
using the replication controller created by the plugin.


Once you have the environment set up (minikube or kubectl configured against a kubernetes cluster), you can play with this Spring Boot application in the cloud using the following maven command to deploy it:
```
mvn clean package fabric8:deploy -Pkubernetes
```  
   
**Note**: Unfortunately, when you deploy using the fabric8 plugin, the readiness and liveness probes fail to point to the right actuator URL due a lack of support for Spring Boot. 
This push you to edit the generated deployment inside kubernetes and change these probes which points to "path": "/health" to  "path": "/actuator/health". 
This will make your deployment go green. This issue is already reported into the fabric8 community: https://github.com/fabric8io/fabric8-maven-plugin/issues/1178   
   
When the application has been deployed, you can access its service or endpoint url using this command:
```   
minikube service kubernetes-hello-world --url
```
  
And next you can curl the endpoint using the url returned by the previous command

``` 
curl https://IP_OR_HOSTNAME/
```

then

``` 
curl https://IP_OR_HOSTNAME/services
```     

Should return you the list of available services discovered by the DiscoveryClient     
     
``` 
mvn clean install -Pintegration
```
```yaml

cat <<EOF | kubectl apply -f -

---
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: default
  name: kubernetes-hello-world

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  namespace: default
  name: kubernetes-hello-world
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "watch", "list"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: default
  name: kubernetes-hello-world
subjects:
- kind: ServiceAccount
  name: kubernetes-hello-world
roleRef:
  kind: ClusterRole
  name: kubernetes-hello-world
  apiGroup: rbac.authorization.k8s.io

EOF

```

```yaml

cat <<EOF | kubectl apply -f -
---
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  generation: 2
  labels:
    app: kubernetes-hello-world
    group: org.springframework.cloud
    provider: fabric8
    version: 2.0.2
  name: kubernetes-hello-world
  namespace: default
spec:
  progressDeadlineSeconds: 600
  replicas: 2
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      app: kubernetes-hello-world
      group: org.springframework.cloud
      provider: fabric8
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      annotations:
      labels:
        app: kubernetes-hello-world
        group: org.springframework.cloud
        provider: fabric8
        version: 2.0.2
    spec:
      serviceAccountName: kubernetes-hello-world
      containers:
      - env:
        - name: KUBERNETES_NAMESPACE
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
        image: cloud/kubernetes-hello-world:2.0.2
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /actuator/health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 180
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        name: spring-boot
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
        - containerPort: 9779
          name: prometheus
          protocol: TCP
        - containerPort: 8778
          name: jolokia
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /actuator/health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        resources: {}
        securityContext:
          privileged: false
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
EOF

```


