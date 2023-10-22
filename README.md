# knative-on-docker-desktop
 Install and try knative serving and eventing on Docker Desktop for Windows built-in Kubernetes. For Docker Desktop on Mac/Linux, `kind` or `minicube` are required.

### 1. Enable Kubernetes feature on Docker Desktop
- Go to Settings | Kubernetes 
- Tick Enable kubernetes
- Click Apply & restart

### 2. Install KnativeCLI
- Download the cli for windows from https://github.com/knative/client/releases
- Copy the exe file to preferred directory/folder
- Rename the exe file to `kn.exe`
- Add the folder to the env PATH
- Test with get knative version command
```
kn version
```

### 3. Install Knative serving 
Follow steps on https://knative.dev/docs/install/yaml-install/serving/install-serving-with-yaml/
- Install custom resoure definitions
```
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.11.1/serving-crds.yaml
```

- Install the core components 
```
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.11.1/serving-core.yaml
```

- Install Istio
```
kubectl apply -l knative.dev/crd-install=true -f https://github.com/knative/net-istio/releases/download/knative-v1.11.0/istio.yaml
kubectl apply -f https://github.com/knative/net-istio/releases/download/knative-v1.11.0/istio.yaml
```

- Install the Knative Istio controller
```
kubectl apply -f https://github.com/knative/net-istio/releases/download/knative-v1.11.0/net-istio.yaml
```

- Fetch the External IP address or CNAME
```
kubectl --namespace istio-system get service istio-ingressgateway
```
```
-->
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                                      AGE
istio-ingressgateway   LoadBalancer   10.104.251.9   localhost     15021:32243/TCP,80:31363/TCP,443:32447/TCP   9m28s
```
- Verify
```
kubectl get pods -n knative-serving
```

- Temp domain example.com (run in Git bash)
```
 kubectl patch configmap/config-domain \
      --namespace knative-serving \
      --type merge \
      --patch '{"data":{"example.com":""}}'
```

- Install the components needed to support HPA-class autoscaling
```
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.11.1/serving-hpa.yaml
```

### 4. Install Knative Eventing
Follow steps on https://knative.dev/docs/install/yaml-install/eventing/install-eventing-with-yaml/ for Knative Eventing
- Install the required custom resource definitions
```
kubectl apply -f https://github.com/knative/eventing/releases/download/knative-v1.11.4/eventing-crds.yaml
```

- Install the core components of Eventing
```
kubectl apply -f https://github.com/knative/eventing/releases/download/knative-v1.11.4/eventing-core.yaml
```

- Verify
```
kubectl get pods -n knative-eventing
```

- Install InMemory Channel
```
kubectl apply -f https://github.com/knative/eventing/releases/download/knative-v1.11.5/in-memory-channel.yaml
```

- Install mt channel broker
```
kubectl apply -f https://github.com/knative/eventing/releases/download/knative-v1.11.5/mt-channel-broker.yaml
```

### 5. Try with Hello World Service
- Run
```
kn service create hello --image ghcr.io/knative/helloworld-go:latest --port 8080 --env TARGET=World
```

- Check Knative services running
```
kubectl get ksvc
```
```
-->
NAME    URL                                LATESTCREATED   LATESTREADY   READY   REASON
hello   http://hello.default.example.com   hello-00001     hello-00001   True
```

- Hit the internal url (use bash shell)
```
curl -H "Host: hello.default.example.com" http://localhost
```
```
-->
Hello World!
```

- Check SCALING TO ZERO (POD)
```
watch kubectl get pods
```
Wait a bit and we will see pod get terminated and removed. If we hit the service again, the pod will get instantiated and serve again.

- Delete the service
```
kn service delete 'hello'
```

### 6. Cloud event player
- Create the CloudEvents Player Service
```
kn service create cloudevents-player --image quay.io/ruben/cloudevents-player:latest
```

- Create MTC broker
```
kn broker create mtc-broker
```

- Sink binding the service and the broker
```
kn source binding create ce-player-binding --subject "Service:serving.knative.dev/v1:cloudevents-player" --sink broker:mtc-broker
```

- Create trigger
```
kn trigger create cloudevents-trigger --sink cloudevents-player  --broker mtc-broker
```

- Send event from http://cloudevents-player.default.example.com we will see the player display the sending message and also the receiving message.

### 7. Clean up
- Delete trigger
```
kn trigger delete cloudevents-trigger
```

- Delete source binding
```
kn source binding delete ce-player-binding
```

- Delete MTC broker
```
kn broker delete mtc-broker
```

- Delete service
```
kn service delete 'cloudevents-player'
```

### 7. Some other commands
- Knative
```
kubectl get kservice
kubectl get route
kubectl get configuration
kubectl get revision
kubectl get deploy
kubectl -n knative-serving get cm config-autoscaler
kubectl -n knative-serving describe cm config-autoscaler
```
- Broker
```
kubectl describe broker.eventing.knative.dev/mtc-broker
kubectl describe SinkBinding
kubectl describe configmap config-br-default-channel -n knative-eventing
kubectl describe configmap config-br-defaults -n knative-eventing
kubectl get configmap config-br-default-channel -n knative-eventing -o yaml
kubectl get configmap config-br-defaults -n knative-eventing -o yaml
```

- Trigger
```
kubectl describe trigger.eventing.knative.dev/cloudevents-trigger
```

### References
- Greate video: https://www.youtube.com/watch?v=PSnVGk73CjQ&list=PLBvjNj5-9WtHD7rbY-0K5bx2nzbNhGGVm&index=3
- Knative website: https://knative.dev/docs/
- Step-by-step tutorial: https://opensource.com/article/21/2/knative-eventing
