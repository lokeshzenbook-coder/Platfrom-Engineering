> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Demo Traffic Flow with Services and Ingress

> Demo explaining Kubernetes Services, ClusterIP, NodePort, and Ingress with an nginx ingress controller showing routing, service discovery, and handling ephemeral pod lifecycles.

Every Kubernetes platform engineer faces the same challenge early on: pods are ephemeral. They are created and destroyed frequently, and each new pod receives a different IP address. Relying on pod IPs for connectivity doesn't scale — services provide a stable abstraction for discovery and load balancing. This demo shows why Services are essential, how ClusterIP and NodePort services work, and how an Ingress + Ingress controller lets you route external traffic to multiple services through a single entry point.

## 1) Pods are ephemeral — deployment example

Create a Deployment named `web` (nginx) with three replicas:

```bash theme={null}
kubectl create deployment web --image=nginx --replicas=3
```

List pods and their IPs:

```bash theme={null}
kubectl get pods -o wide
```

Example output (IPs will vary):

```text theme={null}
NAME                        READY   STATUS    RESTARTS   AGE   IP            NODE
web-68d995574f-4qvnb        1/1     Running   0          2m    172.17.0.6    controlplane
web-68d995574f-nhgxz        1/1     Running   0          2m    172.17.0.7    controlplane
web-68d995574f-6hjj         1/1     Running   0          2m    172.17.0.8    controlplane
```

If you delete a pod, the Deployment controller will create a replacement pod with a different IP:

```bash theme={null}
kubectl delete pod web-68d995574f-4qvnb
kubectl get pods -o wide
```

Example replacement:

```text theme={null}
NAME                        READY   STATUS    RESTARTS   AGE   IP            NODE
web-68d995574f-4vnqv        1/1     Running   0          15s  172.17.0.9    controlplane
web-68d995574f-nhgxz        1/1     Running   0          2m   172.17.0.7    controlplane
web-68d995574f-6hjj         1/1     Running   0          2m   172.17.0.8    controlplane
```

Any client using the old pod IP (for example `172.17.0.6`) will lose connectivity. This is the primary reason we use Services.

## 2) ClusterIP service (internal cluster access)

Expose the `web` Deployment as a ClusterIP service named `web-svc` on port 80:

```bash theme={null}
kubectl expose deploy web --port=80 --target-port=80 --name=web-svc
kubectl describe svc web-svc
```

Key fields from `kubectl describe svc web-svc`:

```text theme={null}
Name:                     web-svc
Namespace:                default
Selector:                 app=web
Type:                     ClusterIP
IP:                       172.20.212.114
Port:                     80/TCP
TargetPort:               80/TCP
Endpoints:                172.17.0.9:80,172.17.0.7:80,172.17.0.8:80
```

* ClusterIP is internal-only (accessible from inside the cluster).
* The Service uses selectors (labels) to discover matching pods — the Deployment assigned the label `app=web`.

Check pod labels:

```bash theme={null}
kubectl get pods --show-labels
```

Example output:

```text theme={null}
NAME                        READY   STATUS    RESTARTS   AGE   LABELS
web-68d995574f-4vnqv        1/1     Running   0          1m   app=web,pod-template-hash=68d995574f
web-68d995574f-nhgxz        1/1     Running   0          2m   app=web,pod-template-hash=68d995574f
web-68d995574f-6hjj         1/1     Running   0          2m   app=web,pod-template-hash=68d995574f
```

## 3) NodePort service (external access for labs or single-node)

If you need external access in a lab or single-node environment (without a cloud load balancer), use NodePort:

```bash theme={null}
kubectl expose deploy web --port=80 --target-port=80 --type=NodePort --name=web-nodeport
kubectl get svc web-nodeport
```

Example:

```text theme={null}
NAME           TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)
web-nodeport   NodePort   172.20.11.172   <none>        80:30309/TCP
```

Curl the NodePort (on the node IP or `localhost` if running locally):

```bash theme={null}
curl localhost:30309
```

You should receive the NGINX default welcome page HTML.

Note: NodePort assigns a port in the 30000–32767 range. This is convenient for testing but not ideal for production (awkward ports, limited range, no TLS termination). For path-based routing, virtual hosts, and TLS termination, use Ingress (with an Ingress controller).

<Callout icon="lightbulb" color="#1CB2FE">
  Ingress provides host-based and path-based routing, and TLS termination. It requires an Ingress controller to implement the routing (for example, [nginx-ingress](https://kubernetes.github.io/ingress-nginx/), [Traefik](https://traefik.io/), or other controllers).
</Callout>

## 4) Ingress controller — verify it's running

This lab uses the nginx ingress controller in namespace `ingress-nginx`. Verify controller pods and service:

```bash theme={null}
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

Example:

```text theme={null}
# controller pod
NAME                                           READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-c47b845b-bpvxz       1/1     Running   0          10m

# controller service
NAME                         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)
ingress-nginx-controller     NodePort       172.20.253.85   <none>        80:30080/TCP,443:30443/TCP
```

* In cloud environments the controller's Service is often type `LoadBalancer` with an external IP.
* In labs it may be `NodePort`, which exposes controller ports on the node(s).

All external traffic for this demo will enter via the Ingress controller service.

## 5) Create a second service — `api`

Create an `api` Deployment using HashiCorp's `http-echo` image to return a simple message. Use the `--command` flag to pass container arguments:

```bash theme={null}
kubectl create deployment api --image=hashicorp/http-echo --replicas=2 --command -- /http-echo -text="hello from kodekloud"
kubectl expose deployment api --port=5678 --target-port=5678 --name=api-svc
kubectl get svc api-svc
```

Example service:

```text theme={null}
NAME      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)
api-svc   ClusterIP   172.20.211.200  <none>        5678/TCP
```

Verify API pods:

```bash theme={null}
kubectl get pods -l app=api
```

You can test the API service from within the cluster or with `kubectl port-forward` to confirm it returns the message.

## 6) Create an Ingress resource to route to both services

Create an Ingress manifest (for example `ingress.yaml`) that routes `/api` to `api-svc:5678` and `/` to `web-svc:80`. Example:

```yaml theme={null}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: platform-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 5678
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-svc
                port:
                  number: 80
```

Apply the Ingress:

```bash theme={null}
kubectl apply -f ingress.yaml
kubectl describe ingress platform-ingress
```

You should see the Ingress scheduled for sync by the nginx controller, and the two rules listed in the description.

## 7) Test routing via the ingress controller NodePort

Use the controller NodePort (for example `30080`) to test both routes:

* Root `/` should route to `web-svc` (NGINX welcome page).
* `/api` should route to `api-svc` and return the echo message.

Example commands:

```bash theme={null}
curl localhost:30080/
curl localhost:30080/api
```

Expected responses:

* `curl localhost:30080/` returns the NGINX welcome HTML.
* `curl localhost:30080/api` returns:

```text theme={null}
hello from kodekloud
```

This demonstrates a single external entry point (the Ingress controller) routing to different internal services based on request path, without changing the application pods themselves.

## 8) The three-layer relationship

Understand the three-layer architecture and responsibilities:

* Ingress: external routing, host/path matching, TLS termination (external entry point).
* Service: stable internal discovery and load balancing (virtual IP / DNS for a set of pods).
* Pod: the actual workload (ephemeral compute).

When debugging connectivity, walk the chain from outside in:

* Can't reach the app externally? Check the Ingress resource and the Ingress controller.
* Ingress appears fine but routing is wrong? Re-check the Ingress rules and annotations.
* Service has no endpoints? Verify pod labels/selectors and pod readiness.
* Pod-level issues? Inspect pod logs and readiness/liveness probes.

Comparison: Service types

| Service Type | Use Case                                       | Notes / Example                                           |
| ------------ | ---------------------------------------------- | --------------------------------------------------------- |
| ClusterIP    | Internal cluster access                        | `kubectl expose deploy web --port=80 --name=web-svc`      |
| NodePort     | External access for labs or single-node setups | Exposes port in `30000–32767` range; `80:30309/TCP`       |
| LoadBalancer | Cloud environments for external IPs            | Provisioned by cloud provider; recommended for production |

Troubleshooting checklist

| Symptom                  | Quick checks                                                                                       |
| ------------------------ | -------------------------------------------------------------------------------------------------- |
| No external access       | Verify Ingress controller pods and service; confirm controller NodePort/LoadBalancer is accessible |
| Wrong backend served     | `kubectl describe ingress <name>` → verify rule paths and backend service names/ports              |
| Service has no endpoints | `kubectl describe svc <service>` → check `Endpoints`; ensure pods match Service selectors          |
| Pod errors               | `kubectl logs <pod>` and `kubectl describe pod <pod>`; check readiness/liveness probes             |

<Callout icon="warning" color="#FF6B6B">
  Ingress resources require a running Ingress controller. Creating an Ingress without a controller will not provide routing traffic. In cloud environments you may get a LoadBalancer service automatically for the controller; in labs you often use NodePort.
</Callout>

## Links and references

* Kubernetes Services: [https://kubernetes.io/docs/concepts/services-networking/service/](https://kubernetes.io/docs/concepts/services-networking/service/)
* Ingress and Ingress Controllers: [https://kubernetes.io/docs/concepts/services-networking/ingress/](https://kubernetes.io/docs/concepts/services-networking/ingress/)
* nginx-ingress Controller: [https://kubernetes.github.io/ingress-nginx/](https://kubernetes.github.io/ingress-nginx/)
* Traefik Ingress Controller: [https://traefik.io/](https://traefik.io/)
* HashiCorp http-echo: [https://github.com/hashicorp/http-echo](https://github.com/hashicorp/http-echo)

This walkthrough shows how Services and Ingress work together to provide stable discovery and flexible external routing, enabling reliable communication even as pods come and go.

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/989346de-0207-4837-af11-bf456d188972/lesson/1ed2dc1a-4ce9-4793-b1f3-a2c15035cfd8" />
</CardGroup>
