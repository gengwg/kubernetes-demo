## Orchestrating Docker with Kubernetes

This is a demo app to showcase orchestrating Docker with Kubernetes. You can find more at [my post](https://chengl.com/orchestrating-docker-with-kubernetes/).

### Usage

1. Run backend

  ```
  kubectl create -f backend.yaml
  ```
According to the best practices, create a Service before corresponding Deployments so that the scheduler can spread the pods comprising the Service. So the redis service is created before deployment and they are defined in backend.yaml.


2. Run frontend

  ```
  kubectl create -f frontend.yaml
  ```

Note I used `type: NodePort`, because our cloud doe not provide public IP.
The solution is to map the node port to container port,
then in LB configure virtual port to node port.
For example:

  http 30003 http 30003

After that one can find the FQDN for the node and use below to see the page in browser.

http://k8s.prod.crmk8s.presidio.prod.walmart.com:30003/

Note both should use LB and FQDN for `Node`, Not `Master`.

3. Self-healing

  Delete a random pod and see Kubernetes creates a new one.
  ```
  kubectl delete pod frontend-2747139405-bk4ul; kubectl get pods
  ```

4. Scaling

  ```
  kubectl scale deployment/frontend --replicas=6; kubectl get pods
  ```

5. Rolling update
  ```
  kubectl create -f frontend-canary.yaml
  vim frontend.yaml # update simple-node:v1 to simple-node:v2
  kubectl apply -f frontend.yaml
  ```

