# Istio Tutorial Commands

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

### Deploy micros

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

### Traffic Control

```
istiogw=$(oc get route -n istio-system istio-ingressgateway | awk '{print $2} ' | grep -v HOST)
```

* All users to recommendation:v2
```
kubectl create -f istiofiles/destination-rule-recommendation-v1-v2.yml -n tutorial
kubectl create -f istiofiles/virtual-service-recommendation-v2.yml -n tutorial
```

* All users to v1 - 100%
```
kubectl create -f istiofiles/destination-rule-recommendation-v1-v2.yml -n tutorial
kubectl create -f istiofiles/virtual-service-recommendation-v1.yml -n tutorial
```

* Canary deployment: Split traffic between v1 and v2

Canary Deployment scenario: push v2 into the cluster but slowly send end-user traffic to it, if you continue to see success, continue shifting more traffic over time

```
kubectl apply -f ./../istiofiles/virtual-service-recommendation-v1_and_v2_75_25.yml -n tutorial
```

* Create the virtualservice that will send 90% of requests to v1 and 10% to v2














