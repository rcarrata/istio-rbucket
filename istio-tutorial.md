# Istio Tutorial Commands

[Istio Tutorial 1.4.x](https://redhat-developer-demos.github.io/istio-tutorial/workshop/istio-tutorial/1.4.x/)

## 0. Installation (Maistra 1.0.8 / Service Mesh 1.0 / Istio 1.1.17)

```
oc apply -f installation/servicemesh-namespace.yml
```

```
oc apply -f installation/elasticsearch-operator.yml
oc apply -f installation/jaegar-operator.yml
oc apply -f installation/kiali-operator.yml
oc apply -f installation/servicemesh-operator.yml
```

```
oc apply -f installation/servicemesh-controlplane.yml
oc apply -f installation/servicemesh-memberrole.yml
```

## 1. Prereqs environment

```
oc new-project tutorial
oc adm policy add-scc-to-user privileged -z default -n tutorial
```

* Workarounds for ServiceMesh to run within Istioctl v1.1.9

```
curl -O https://github.com/istio/istio/releases/download/1.1.17/istio-1.1.17-linux.tar.gz | tar xz
cd istio-1.1.17 && export ISTIO_HOME=`pwd` && export PATH=$ISTIO_HOME/bin:$PATH && cd - && istioctl version
```

* Git clone tutorial

```
git clone https://github.com/redhat-developer-demos/istio-rbucket
cd istio-rbucket
```



## 1. Deploy micros

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

## 2. Traffic Control

### 2.1 Simple Traffic

```
istiogw=$(oc get route -n istio-system istio-ingressgateway | awk '{print $2} ' | grep -v HOST)
```

#### 2.1.1 All users to recommendation:v2
```
kubectl create -f istiofiles/destination-rule-recommendation-v1-v2.yml -n tutorial
kubectl create -f istiofiles/virtual-service-recommendation-v2.yml -n tutorial
```

#### 2.1.2 All users to v1 - 100%
```
kubectl create -f istiofiles/destination-rule-recommendation-v1-v2.yml -n tutorial
kubectl create -f istiofiles/virtual-service-recommendation-v1.yml -n tutorial
```

#### 2.1.3 Canary deployment: Split traffic between v1 and v2

* [Canary Deployments Istio](https://istio.io/blog/2017/0.1-canary/#enter-istio)

Canary Deployment scenario: push v2 into the cluster but slowly send end-user traffic to it, if you continue to see success, continue shifting more traffic over time

```
kubectl apply -f ./../istiofiles/virtual-service-recommendation-v1_and_v2_75_25.yml -n tutorial
```

#### 2.1.4 Create the virtualservice that will send 90% of requests to v1 and 10% to v2

```
kubectl apply -f ./../istiofiles/virtual-service-recommendation-v1_and_v2_50_50.yml -n tutorial
```

### 2.2 Advanced Traffic

#### 2.2.1 Smart routing based on user-agent header (Canary Deployment)

* [Route-Based On User Identity](https://istio.io/docs/tasks/traffic-management/request-routing/#route-based-on-user-identity)

```
kubectl replace -f istiofiles/virtual-service-safari-recommendation-v2.yml -n tutorial
curl -A Safari http://$(kubectl get route istio-ingressgateway -n istio-system --output 'jsonpath={.status.ingress[].host}')/customer
```

#### 2.2.2 Set mobile users to v2

```
kubectl replace -f ./../istiofiles/virtual-service-mobile-recommendation-v2.yml -n tutorial

curl -A "Mozilla/5.0 (iPhone; U; CPU iPhone OS 4(KHTML, like Gecko) Version/5.0.2 Mobile/8J2 Safari/6533.18.5" http://$(kubectl get route istio-ingressgateway -n istio-system --output 'jsonpath={.status.ingress[].host}')/customer
```

#### 2.2.3 Mirror Traffic - Dark Launch

* [Mirroring/Dark Launch](https://istio.io/docs/tasks/traffic-management/mirroring/)

```
kubectl create -f istiofiles/destination-rule-recommendation-v1-v2.yml -n tutorial
kubectl create -f istiofiles/virtual-service-recommendation-v1-mirror-v2.yml -n tutorial

./../scripts/run.sh http://$(kubectl get route istio-ingressgateway -n istio-system --output 'jsonpath={.status.ingress[].host}')/customer

kubectl logs `kubectl get pods -n tutorial|grep recommendation-v2|awk '{ print $1 }'` -c recommendation -n tutorial
```

#### 2.2.4 Load Balancing

* [Load Balancing](https://istio.io/docs/concepts/traffic-management/#load-balancing-options)

By default, you will see "round-robin" style load-balancing, but you can change it up, with the
RANDOM option being fairly visible to the naked eye.

```
kubectl scale deployment recommendation-v2 --replicas=3 -n tutorial

./../scripts/run.sh http://$(kubectl get route istio-ingressgateway -n istio-system --output 'jsonpath={.status.ingress[].host}')/customer

kubectl replace -f ./../istiofiles/destination-rule-recommendation_lb_policy_app.yml -n tutorial

./../scripts/run.sh http://$(kubectl get route istio-ingressgateway -n istio-system --output 'jsonpath={.status.ingress[].host}')/customer
```

## 3. Service Resiliency

[Retries Istio](https://istio.io/docs/concepts/traffic-management/#retries)

















