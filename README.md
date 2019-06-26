### Kubernetes

#### Setting up

## kubernetes dashboard
kubectl proxy

http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/.

#### Create Service Account
```aidl
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
```
#### Create ClusterRoleBinding
```aidl
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
```

#### Get token
```aidl
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```

## Practice

#### redis deploy

```aidl
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: redis-master
  labels:
    app: redis
spec:
  selector:
    matchLabels:
      app: redis
      role: master
      tier: backend
  replicas: 1
  template:
    metadata:
      labels:
        app: redis
        role: master
        tier: backend
    spec:
      containers:
      - name: master
        image: k8s.gcr.io/redis:e2e  # or just image: redis
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        ports:
        - containerPoert: 6379


```
#### redis service
```aidl
apiVersion: v1
kind: Service
metadata:
  name: redis-master
  labels:
    app: redis
    role: master
    tier: backend
spec:
  ports:
  - port: 6379
    targetPort: 6379
  selector:
    app: redis
    role: master
    tier: backend

```

#### redis slave deploy & service
```aidl
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: redis-slave
  labels:
    app: redis
spec:
  selector:
    matchLabels:
      app: redis
      role: slave
      tier: backend
  replicas: 2
  template:
    metadata:
      labels:
        app: redis
        role: slave
        tier: backend
    spec:
      containers:
      - name: slave
        image: gcr.io/google_samples/gb-redisslave:v3
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        env:
        - name: GET_HOSTS_FROM
          value: dns
          # Using `GET_HOSTS_FROM=dns` requires your cluster to
          # provide a dns service. As of Kubernetes 1.3, DNS is a built-in
          # service launched automatically. However, if the cluster you are using
          # does not have a built-in DNS service, you can instead
          # access an environment variable to find the master
          # service's host. To do so, comment out the 'value: dns' line above, and
          # uncomment the line below:
          # value: env
        ports:
        - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: redis-slave
  labels:
    app: redis
    role: slave
    tier: backend
spec:
  ports:
  - port: 6379
  selector:
    app: redis
    role: slave
    tier: backend


```

## Application - front end deploy & service

```aidl
apiVersion: apps/v1
kind: Deployment
metadata:
  name: guestbook-v1
  labels:
    app: guestbook
    version: "1.0"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: guestbook
      version: "1.0"
  template:
    metadata:
      labels:
        app: guestbook
        version: "1.0"
    spec:
      containers:
      - name: guestbook
        image: ibmcom/guestbook:v1
        ports:
        - name: http-server
          containerPort: 3000    

```

#### scale
```aidl
  kubectl scale deployment guestbook-v1 --replicas=2
```
## Auto Scale
```aidl
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-scaler
spec:
  scaleTargetRef:
    kind: Deployment
    name: guestbook-v1
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```



### Version Upgrade for guestbook 2.0
* Watson Analyzer 구축

Deployment
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: analyzer
  labels:
    app: analyzer
spec:
  selector:
    matchLabels:
      app: analyzer
  replicas: 1
  template:
    metadata:
      labels:
        app: analyzer
    spec:
      containers:
      - name: analyzer
        image: ibmcom/analyzer:v1.1
        imagePullPolicy: Always
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        env:
        - name: VCAP_SERVICES_TONE_ANALYZER_API_KEY
          value: <API-KEY>
        - name : VCAP_SERVICES_TONE_ANALYZER_TOKEN_ADDRESS
          value: "https://iam.bluemix.net/identity/token"
        - name: VCAP_SERVICES_TONE_ANALYZER_SERVICE_API
          value: "https://gateway.watsonplatform.net/tone-analyzer/api"
        ports:
        - containerPort: 5000
          name: http

```

Service
```
apiVersion: v1
kind: Service
metadata:
  name: analyzer
  labels:
    app: analyzer
spec:
  # comment or delete the following line if you want to use a LoadBalancer
  # type: NodePort
  # if your cluster supports it, uncomment the following to automatically create
  # an external load-balanced IP for the frontend service.
  # type: LoadBalancer
  ports:
  - port: 80
    targetPort: 5000
    name: http
  selector:
    app: analyzer
```
##ISIO

### install
1.  Either download Istio directly from [https://github.com/istio/istio/releases](https://github.com/istio/istio/releases) or get the latest version by using curl:

    ```shell
    curl -L https://git.io/getLatestIstio | sh -
    ```

2. Change the directory to the Istio file location.

    ```shell
    cd istio-<version-number>
    ```

3. Add the `istioctl` client to your PATH. 

    ```shell
    export PATH=$PWD/bin:$PATH
    ```

4. Install Istio’s Custom Resource Definitions via kubectl apply, and wait a few seconds for the CRDs to be committed in the kube-apiserver:

    ```shell
    for i in install/kubernetes/helm/istio-init/files/crd*yaml; do kubectl apply -f $i; done
    ```
5. Change istio-ingressgateway type to `NodePort` from `LoadBalancer` 

6. Now let's install Istio demo profile into the `istio-system` namespace in your Kubernetes cluster:

    ```shell
    kubectl apply -f install/kubernetes/istio-demo.yaml
    ```
    
    
### Enable the automatic sidecar injection for the default namespace

1. Annotate the default namespace to enable automatic sidecar injection:
    
    ``` shell
    kubectl label namespace default istio-injection=enabled
    ```
    
2. Validate the namespace is annotated for automatic sidecar injection:
    
    ``` shell
    kubectl get namespace -L istio-injection
    ```
    
> Manual injectiion
> `istioctl kube-inject -f <deployt.yaml>| kubectl apply -f -`    
    
#### ingress gateway opened port    
https://istio.io/docs/tasks/traffic-management/ingress/

```aidl
istio-ingressgateway     NodePort    172.21.19.76     <none>        15020:32672/TCP,80:31380/TCP,443:31390/TCP,31400:31400/TCP,15029:31910/TCP,15030:30595/TCP,15031:32675/TCP,15032:30086/TCP,15443:30039/TCP   13m
```   

export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')


#### guest-book gateway 
```aidl
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: guestbook-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "www.guestbook.ibm.com"
```

Service
```aidl
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: virtual-service-guestbook
spec:
  hosts:
    - 'www.guestbook.ibm.com'
  gateways:
    - guestbook-gateway
  http:
    - route:
        - destination:
            host: guestbook
            subset: v1
```

#### test with curl 
```aidl
curl  -HHost:www.guestbook.ibm.com http://168.1.148.142:$INGRESS_PORT
curl  -HHost:www.guestbook.ibm.com http://168.1.148.142:$INGRESS_PORT/env
```

#### Destination Rule
```aidl

apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: destination-rule-guestbook
spec:
  host: guestbook
  subsets:
  - name: v1
    labels:
      version: "1.0"
  - name: v2
    labels:
      version: "2.0"

```

### VS for canary, VS for Blue Grean deploy



## Monitoring

### Istio telemetry

Istio's tracing and metrics features are designed to provide broad and granular insight into the health of all services. Istio's role as a service mesh makes it the ideal data source for observability information, particularly in a microservices environment. As requests pass through multiple services, identifying performance bottlenecks becomes increasingly difficult using traditional debugging techniques. Distributed tracing provides a holistic view of requests transiting through multiple services, allowing for immediate identification of latency issues. With Istio, distributed tracing comes by default. This will expose latency, retry, and failure information for each hop in a request.

You can read more about how [Istio mixer enables telemetry reporting](https://istio.io/docs/concepts/policy-and-control/mixer.html).

### Configure Istio to receive telemetry data

1. Verify that the Grafana, Prometheus, Kiali and Jaeger add-ons were installed successfully. All add-ons are installed into the `istio-system` namespace.

    ```shell
    kubectl get pods -n istio-system
    kubectl get services -n istio-system
    ```

2. Configure Istio to automatically gather telemetry data for services that run in the service mesh. Create a rule to collect telemetry data.
    ```shell
    cd ../../plans/
    kubectl create -f guestbook-telemetry.yaml
    ```

1. Obtain the guestbook endpoint to access the guestbook. You can access the guestbook via the external IP for your service as guestbook is deployed as a load balancer service. Get the EXTERNAL-IP of the guestbook service via output below:

    ```shell
    kubectl get service guestbook -n default
    ```

Go to this external ip address in the browser to try out your guestbook.

4. Generate a small load to the app.

    ```shell
    for i in {1..20}; do sleep 0.5; curl http://<guestbook_IP>/; done
    ```

## View guestbook telemetry data

#### Jaeger

1. Establish port forwarding from local port 16686 to the Tracing instance:

```shell
kubectl port-forward -n istio-system \
$(kubectl get pod -n istio-system -l app=jaeger -o jsonpath='{.items[0].metadata.name}') \
16686:16686 &
```
2. In your browser, go to `http://127.0.0.1:16686`
3. From the **Services** menu, select either the **guestbook** or **analyzer** service.
4. Scroll to the bottom and click on **Find Traces** button to see traces.

Read more about [Jaeger](https://www.jaegertracing.io/docs/)

#### Grafana

1. Establish port forwarding from local port 3000 to the Grafana instance:

    ```shell
    kubectl -n istio-system port-forward \
      $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') \
      3000:3000 &
    ```

2. Browse to http://localhost:3000 and navigate to the `Istio Service Dashboard` by clicking on the Home menu on the top left, then Istio, then Istio Service Dashboard.

3. Select guestbook in the Service drop down.

4. In a different tab, visit the guestbook application and refresh the page multiple times to generate some load, or run the load script you used previously. Switch back to the Grafana tab.

This Grafana dashboard provides metrics for each workload. Explore the other dashboard provided as well. 

Read more about [Grafana](http://docs.grafana.org/).

#### Prometheus

```shell
kubectl -n istio-system port-forward \
  $(kubectl -n istio-system get pod -l app=prometheus -o jsonpath='{.items[0].metadata.name}') \
  9090:9090 &
```
* Browse to http://localhost:9090/graph, and in the “Expression” input box, enter: `istio_request_bytes_count`. Click Execute and then select Graph.

* Then try another query: `istio_requests_total{destination_service="guestbook.default.svc.cluster.local", destination_version="2.0"}`

#### Kiali

Kiali is an open-source project that installs on top of Istio to visualize your service mesh.

```shell
kubectl -n istio-system port-forward \
$(kubectl -n istio-system get pod -l app=kiali -o jsonpath='{.items[0].metadata.name}') \
20001:20001 &
```

2. Browse to [http://localhost:20001/kiali/](http://localhost:20001/kiali/), and login with `admin` for both username and password.




## Helm chart

#### install

```
curl -LO https://get.helm.sh/helm-v2.14.1-linux-amd64.tar.gz
tar -zxvf helm-v2.14.1-linux-amd64.tar.gz
export PATH=$PWD/linux-amd64:$PATH 

```
```
$ kubectl create serviceaccount --namespace kube-system tiller

$ kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller

$ helm init --service-account tiller --upgrade
```
#### Sample create


#### Upgrade, history, rollback


https://github.com/IBM/helm101/tree/master/charts/guestbook