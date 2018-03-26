# Istio_ServiceMesh
Managing Services with Istio on Kubernetes

#### Istio Control Plane Topology


#### Install Istio

<https://istio.io/docs/setup/kubernetes/quick-start.html>

```
curl -L https://git.io/getLatestIstio | sh -
cd istio-0.6
export PATH=$PWD/bin:$PATH
kubectl apply -f install/kubernetes/istio.yaml
```

Starting Kuberenetes 1.9 and higher automatic injection od Sidecar injection is facilitated


1. Verify that the kube-apiserver process has the admission-control flag set with the MutatingAdmissionWebhook and ValidatingAdmissionWebhook admission controllers

```
kubectl api-versions | grep admissionregistration
Output: admissionregistration.k8s.io/v1beta1
```

2. Install Webhook to enable automatic Injection

```
./install/kubernetes/webhook-create-signed-cert.sh \
    --service istio-sidecar-injector \
    --namespace istio-system \
    --secret sidecar-injector-certs
```
```
kubectl apply -f install/kubernetes/istio-sidecar-injector-configmap-release.yaml
```
cat install/kubernetes/istio-sidecar-injector.yaml | \
     ./install/kubernetes/webhook-patch-ca-bundle.sh > \
     install/kubernetes/istio-sidecar-injector-with-ca-bundle.yaml
```
kubectl apply -f install/kubernetes/istio-sidecar-injector-with-ca-bundle.yaml
```

#### Istio Samples

1. Deploy Cobalt app with Blue version

The file below creates a Service, Deployment and a Ingress resource (istio)

```
---
kind: Service
apiVersion: v1
metadata:
  name: cobalt
  labels:
    app: cobalt
spec:
  selector:
    app: cobalt
  ports:
  - name: http
    protocol: TCP
    port: 80
---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: cobalt-blue
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: cobalt
        version: blue
    spec:
      containers:
        - name: cobalt
          image: gokulpch/nginx:blue
          imagePullPolicy: Always
          ports:
            - containerPort: 80
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: gateway
  annotations:
    kubernetes.io/ingress.class: "istio"
spec:
  rules:
  - http:
      paths:
      - path:
        backend:
          serviceName: cobalt
          servicePort: 80
---
```

2. Create the above configuration, injecting the sidecar manually here (if automated injection is enabled this can be ignored)

```
kubectl create -f <(istioctl kube-inject -f cobalt-blue.yaml)
```
```
service "cobalt" created
deployment "cobalt-blue" created
ingress "gateway" created
```

3. Get the URL of the ingress Gtaeway URL

```
GATEWAY_URL=$(kubectl get po -n istio-system -l istio=ingress -n istio-system -o 'jsonpath={.items[0].status.hostIP}'):$(kubectl get svc istio-ingress -n istio-system -n istio-system -o 'jsonpath={.spec.ports[0].nodePort}')
```

4. Create a Istio route defaulting all the traffic to the Blue resource

```
istioctl create -f route-cobalt-default-blue.yaml
```

```
---
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  name: cobalt-default
spec:
  destination:
    name: cobalt
  precedence: 1
  route:
  - labels:
      version: blue
    weight: 100
```

* In the configuration above "weight:100" indicates directing all the traffic to the blue version. Precednce will ot take any affect here as there is only one version created in this context.

5. Test with default blue version

```
while true; do curl http://${GATEWAY_URL}; sleep 2; done
```

```
root@ip-172-25-1-94:~/Istio_ServiceMesh# while true; do curl http://${GATEWAY_URL}; sleep 2; done
<html>
 <head>
     <title> Blue Version 1.0.0 </ title>
 </ head>
 <body bgcolor = "# 0000FF" >
     <H1> This is Blue Version 1.0.0 </ H1>
 </ body>
 </ html>
<html>
 <head>
     <title> Blue Version 1.0.0 </ title>
 </ head>
 <body bgcolor = "# 0000FF" >
     <H1> This is Blue Version 1.0.0 </ H1>
 </ body>
 </ html>
<html>
 <head>
     <title> Blue Version 1.0.0 </ title>
 </ head>
 <body bgcolor = "# 0000FF" >
     <H1> This is Blue Version 1.0.0 </ H1>
 </ body>
 </ html>
```
This returns traffic form only Blue version


6. Deploy Cobalt app with Green version and test with default green and username based traffic control

```
kubectl create -f <(istioctl kube-inject -f cobalt-green.yaml)
```

```
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: cobalt-green
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: cobalt
        version: green
    spec:
      containers:
        - name: cobalt
          image: gokulpch/nginx:green
          imagePullPolicy: Always
          ports:
            - containerPort: 80
```
* This create a deployment with Green version and uses existing service and ingress resource

* Create a Default Route forwarding traffic to gree using usernmae. The same can be tried using default green "weight 100"

```
istioctl create -f route-test-cobalt-user-green.yaml
```


```
---
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  name: cobalt-green
spec:
  destination:
    name: cobalt
  precedence: 2
  match:
    request:
      headers:
        user:
          regex: "gokulpch"
  route:
  - labels:
      version: green
```

* Test

```
while true; do curl -H "user: gokulpch" http://${GATEWAY_URL}; sleep 2; done
```

7. Test Canary traffic control A/B, Blue/Green Deployments

* Can change the weightage based on the requirement

```
---
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  name: cobalt-canary
spec:
  destination:
    name: cobalt
  precedence: 3
  route:
  - labels:
      version: blue
    weight: 70
  - labels:
      version: green
    weight: 30
```

```
istioctl create -f canary-cobalt.yaml
```

```
root@ip-172-25-1-94:~/Istio_ServiceMesh# istioctl create -f canary-cobalt.yaml
Created config route-rule//cobalt-canary at revision 842904
root@ip-172-25-1-94:~/Istio_ServiceMesh# istioctl get routerules
NAME			KIND					NAMESPACE
cobalt-canary		RouteRule.config.istio.io.v1alpha2	default
cobalt-default		RouteRule.config.istio.io.v1alpha2	default
```

```
root@ip-172-25-1-94:~/Istio_ServiceMesh# while true; do curl http://${GATEWAY_URL}; sleep 2; done
<html>
 <head>
     <title> Green Version 1.0.0 </ title>
 </ head>
 <body bgcolor = "# 00FF00" >
     <H1> This is Green Version 1.0.0 </ H1>
 </ body>
 </ html>
<html>
 <head>
     <title> Blue Version 1.0.0 </ title>
 </ head>
 <body bgcolor = "# 0000FF" >
     <H1> This is Blue Version 1.0.0 </ H1>
 </ body>
 </ html>
<html>
 <head>
     <title> Blue Version 1.0.0 </ title>
 </ head>
 <body bgcolor = "# 0000FF" >
     <H1> This is Blue Version 1.0.0 </ H1>
 </ body>
 </ html>
<html>
 <head>
     <title> Blue Version 1.0.0 </ title>
 </ head>
 <body bgcolor = "# 0000FF" >
     <H1> This is Blue Version 1.0.0 </ H1>
 </ body>
 </ html>
<html>
 <head>
     <title> Blue Version 1.0.0 </ title>
 </ head>
 <body bgcolor = "# 0000FF" >
     <H1> This is Blue Version 1.0.0 </ H1>
 </ body>
 </ html>
<html>
 <head>
     <title> Green Version 1.0.0 </ title>
 </ head>
 <body bgcolor = "# 00FF00" >
     <H1> This is Green Version 1.0.0 </ H1>
 </ body>
 </ html>
```

