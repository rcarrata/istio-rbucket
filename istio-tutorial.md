# Istio Tutorial Commands

[Istio Tutorial 1.4.x](https://redhat-developer-demos.github.io/istio-tutorial/workshop/istio-tutorial/1.4.x/)

## Prereqs environment

```
oc new-project tutorial
oc adm policy add-scc-to-user privileged -z default -n tutorial
```

```
git clone https://github.com/redhat-developer-demos/istio-tutorial
cd istio-tutorial
```

## Workarounds for ServiceMesh to run within Istioctl v1.1.9
```
cd ~/Code/RedHat/8-ISTIO/istio-1.1.9 && export ISTIO_HOME=`pwd` && export PATH=$ISTIO_HOME/bin:$PATH
&& cd - && istioctl version
```

## Deploy micros

* Customerv1

```
oc apply -f <(istioctl kube-inject -f customer/kubernetes/Deployment.yml) -n tutorial
oc create -f customer/kubernetes/Service.yml -n tutorial
oc create -f customer/kubernetes/Gateway.yml -n tutorial
```

* Preference v1

```
oc apply -f <(istioctl kube-inject -f preference/kubernetes/Deployment.yml) -n tutorial
oc create -f preference/kubernetes/Service.yml -n tutorial
```

* Recommendation v1

```
oc apply -f <(istioctl kube-inject -f recommendation/kubernetes/Deployment.yml)  -n tutorial
oc create -f recommendation/kubernetes/Service.yml  -n tutorial
```

```
istiogw=$(oc get route -n istio-system istio-ingressgateway | awk '{print $2} ' | grep -v HOST)
curl $istiogw/customer
```

```
./../scripts/run.sh http://$(kubectl get route istio-ingressgateway -n istio-system --output 'jsonpath={.status.ingress[].host}')/customer
```

* Recommendation v2

```
oc apply -f <(istioctl kube-inject -f recommendation/kubernetes/Deployment-v2.yml) -n tutorial
```

## Traffic Control

### Simple Traffic

```
istiogw=$(oc get route -n istio-system istio-ingressgateway | awk '{print $2} ' | grep -v HOST)
```

#### All users to recommendation:v2
```
kubectl create -f istiofiles/destination-rule-recommendation-v1-v2.yml -n tutorial
kubectl create -f istiofiles/virtual-service-recommendation-v2.yml -n tutorial
```

#### All users to v1 - 100%
```
kubectl create -f istiofiles/destination-rule-recommendation-v1-v2.yml -n tutorial
kubectl create -f istiofiles/virtual-service-recommendation-v1.yml -n tutorial
```

#### Canary deployment: Split traffic between v1 and v2

* [Canary Deployments Istio](https://istio.io/blog/2017/0.1-canary/#enter-istio)

Canary Deployment scenario: push v2 into the cluster but slowly send end-user traffic to it, if you continue to see success, continue shifting more traffic over time

```
kubectl apply -f ./../istiofiles/virtual-service-recommendation-v1_and_v2_75_25.yml -n tutorial
```

#### Create the virtualservice that will send 90% of requests to v1 and 10% to v2

```
kubectl apply -f ./../istiofiles/virtual-service-recommendation-v1_and_v2_50_50.yml -n tutorial
```

### Advanced Traffic

#### Smart routing based on user-agent header (Canary Deployment)

* [Route-Based On User Identity](https://istio.io/docs/tasks/traffic-management/request-routing/#route-based-on-user-identity)

```
kubectl replace -f istiofiles/virtual-service-safari-recommendation-v2.yml -n tutorial
curl -A Safari http://$(kubectl get route istio-ingressgateway -n istio-system --output 'jsonpath={.status.ingress[].host}')/customer
```

#### Set mobile users to v2

```
kubectl replace -f ./../istiofiles/virtual-service-mobile-recommendation-v2.yml -n tutorial

curl -A "Mozilla/5.0 (iPhone; U; CPU iPhone OS 4(KHTML, like Gecko) Version/5.0.2 Mobile/8J2 Safari/6533.18.5" http://$(kubectl get route istio-ingressgateway -n istio-system --output 'jsonpath={.status.ingress[].host}')/customer
```

#### Mirror Traffic - Dark Launch

* [Mirroring/Dark Launch](https://istio.io/docs/tasks/traffic-management/mirroring/)

```
kubectl create -f istiofiles/destination-rule-recommendation-v1-v2.yml -n tutorial
kubectl create -f istiofiles/virtual-service-recommendation-v1-mirror-v2.yml -n tutorial

./../scripts/run.sh http://$(kubectl get route istio-ingressgateway -n istio-system --output 'jsonpath={.status.ingress[].host}')/customer

kubectl logs `kubectl get pods -n tutorial|grep recommendation-v2|awk '{ print $1 }'` -c recommendation -n tutorial
```

#### Load Balancing

* [Load Balancing](https://istio.io/docs/concepts/traffic-management/#load-balancing-options)

By default, you will see "round-robin" style load-balancing, but you can change it up, with the
RANDOM option being fairly visible to the naked eye.

```
kubectl scale deployment recommendation-v2 --replicas=3 -n tutorial

./../scripts/run.sh http://$(kubectl get route istio-ingressgateway -n istio-system --output 'jsonpath={.status.ingress[].host}')/customer

kubectl replace -f ./../istiofiles/destination-rule-recommendation_lb_policy_app.yml -n tutorial

./../scripts/run.sh http://$(kubectl get route istio-ingressgateway -n istio-system --output 'jsonpath={.status.ingress[].host}')/customer
```


