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
  ```
  http 30003 http 30003
  ```

After that one can find the FQDN for the node and use below to see the page in browser.
  ```
  http://<fqdn-for-node-lb>:30003/
  ```

Note both should use LB and FQDN for `Node`, Not `Master`.

3. Self-healing

  Delete a random pod and see Kubernetes creates a new one.
  ```
  kubectl -s http://<fqdn-for-master-lb>:8080 delete pod frontend-1287392616-6kk6q; kubectl -s http://<fqdn-for-master-lb>:8080 get pods | grep frontend
  pod "frontend-1287392616-6kk6q" deleted
  frontend-1287392616-3zmqa   1/1       Running       0          27m
  frontend-1287392616-6kk6q   1/1       Terminating   0          27m
  frontend-1287392616-8lki8   0/1       Terminating   0          1s
  frontend-1287392616-92fgg   1/1       Running       0          27m
  frontend-1287392616-my2gy   0/1       Terminating   0          1s
  frontend-1287392616-r9sfx   0/1       Pending       0          1s

  kubectl -s http://<fqdn-for-master-lb>:8080 get pods | grep frontend
  frontend-1287392616-3zmqa   1/1       Running   0          28m
  frontend-1287392616-92fgg   1/1       Running   0          28m
  frontend-1287392616-r9sfx   1/1       Running   0          29s
  ```


4. Scaling

  ```
  kubectl -s http://<fqdn-for-master-lb>:8080 scale deployment/frontend --replicas=6; kubectl -s http://<fqdn-for-master-lb>:8080 get pods | grep frontend
  deployment "frontend" scaled
  frontend-1287392616-3zmqa   1/1       Running             0          32m
  frontend-1287392616-7dd1z   0/1       Terminating         0          0s
  frontend-1287392616-92fgg   1/1       Running             0          32m
  frontend-1287392616-p1x6m   0/1       ContainerCreating   0          0s
  frontend-1287392616-r9sfx   1/1       Running             0          4m
  frontend-1287392616-z7jjv   0/1       ContainerCreating   0          0s

  kubectl -s http://<fqdn-for-master-lb>:8080 get pods | grep frontend
  frontend-1287392616-3zmqa   1/1       Running   0          32m
  frontend-1287392616-92fgg   1/1       Running   0          32m
  frontend-1287392616-p1x6m   1/1       Running   0          8s
  frontend-1287392616-pjf35   1/1       Running   0          8s
  frontend-1287392616-r9sfx   1/1       Running   0          4m
  frontend-1287392616-z7jjv   1/1       Running   0          8s

  kubectl -s http://<fqdn-for-master-lb>:8080 scale deployment/frontend --replicas=3; kubectl -s http://<fqdn-for-master-lb>:8080 get pods | grep frontend
  deployment "frontend" scaled
  frontend-1287392616-3zmqa   1/1       Running       0          33m
  frontend-1287392616-92fgg   1/1       Running       0          33m
  frontend-1287392616-lnuj6   0/1       Pending       0          0s
  frontend-1287392616-p1x6m   1/1       Terminating   0          1m
  frontend-1287392616-pjf35   1/1       Terminating   0          1m
  frontend-1287392616-r9sfx   1/1       Running       0          6m
  frontend-1287392616-z7jjv   1/1       Terminating   0          1m

  kubectl -s http://<fqdn-for-master-lb>:8080 get pods | grep frontend
  frontend-1287392616-3zmqa   1/1       Running       0          33m
  frontend-1287392616-92fgg   1/1       Running       0          33m
  frontend-1287392616-lnuj6   1/1       Terminating   0          5s
  frontend-1287392616-r9sfx   1/1       Running       0          6m
  frontend-1287392616-u8mjt   1/1       Terminating   0          4s

  kubectl -s http://<fqdn-for-master-lb>:8080 get pods | grep frontend
  frontend-1287392616-3zmqa   1/1       Running   0          36m
  frontend-1287392616-92fgg   1/1       Running   0          36m
  frontend-1287392616-r9sfx   1/1       Running   0          8m
  ```

5. Rolling update
  ```
  kubectl -s http://<fqdn-for-master-lb>:8080 create -f frontend-canary.yaml
  kubectl -s http://<fqdn-for-master-lb>:8080 get pods -L track
  NAME                               READY     STATUS    RESTARTS   AGE       TRACK
  backend-3180978658-93bi0           1/1       Running   0          50m       <none>
  frontend-1287392616-3zmqa          1/1       Running   0          39m       stable
  frontend-1287392616-92fgg          1/1       Running   0          39m       stable
  frontend-1287392616-r9sfx          1/1       Running   0          11m       stable
  frontend-canary-1722551660-obhmi   1/1       Running   0          18s       canary

  kubectl -s http://<fqdn-for-master-lb>:8080  get deployments frontend-canary
  NAME              DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
  frontend-canary   1         1         1            1           2m

  kubectl create -f frontend-canary.yaml
  vim frontend.yaml # update simple-node:v1 to simple-node:v2
  kubectl apply -f frontend.yaml
  ```

Hit the frontend service a few times to verify that only one pod is updated to chenglong/simple-node:v2
  ```
  $ curl http://<fqdn-for-node-lb>:30003
  This request is served by frontend-canary-1722551660-obhmi
  You have viewed this page 1877 times!
  Server Time: 2017-03-28T17:46:43.299Z

  $ curl http://<fqdn-for-node-lb>:30003
  This request is served by frontend-1287392616-r9sfx. You have viewed this page 1930 times!
  ```

Since the Canary Release is working fine, I want to roll out to all pods.
  ```
  vim frontend.yaml
            #image: chenglong/simple-node:v1
            image: chenglong/simple-node:v2
  kubectl -s http://<fqdn-for-master-lb>:8080 apply -f frontend.yaml
  ```

Find out details of one of the none-canary pods. Note that the image is chenglong/simple-node:v2 not v1.
  ```
  kubectl -s http://<fqdn-for-master-lb>:8080 describe pods frontend-1389432169-t89d8
  Name:   frontend-1389432169-t89d8
  Namespace:  default
  ...
  Containers:
    nodejs:
      Container ID: docker://9f3aded8f61cf6326ffcb4aa02699b4fd11ef713b0e879237e31040b79c0b60f
      Image:    chenglong/simple-node:v2
  ...
  ```

Delete Canary deployment
  ```
  kubectl -s http://<fqdn-for-master-lb>:8080 delete deployments frontend-canary
  ```

If this release is not ideal, we could easily roll back
  ```
  kubectl -s http://<fqdn-for-master-lb>:8080 rollout undo deployment/frontend
  ```

Confirm it is using v1 by describing one of the frontend pods.
  ```
  kubectl -s http://<fqdn-for-master-lb>:8080 describe pods frontend-1287392616-2kakj
  Name:   frontend-1287392616-2kakj
  ...
  Containers:
    nodejs:
      Container ID: docker://4373cb8ae9f8ac78f715e3a1a35ca652a8c20eb7452f92731a648860cd44a142
      Image:    chenglong/simple-node:v1
  ...
  ```
