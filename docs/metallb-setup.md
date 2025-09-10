# MetalLB Setup

This document describes how to install and configure **MetalLB** to provide LoadBalancer support in the home Kubernetes cluster.  
MetalLB allows services of type `LoadBalancer` to be assigned real IP addresses from the local LAN.

## 1. Prerequisites

- A working **k3s cluster** (master + workers ready).  
- **kubectl** configured on the master node.  
- A free IP range on the LAN that can be used for load-balanced services.  

### Cluster Network Info

- Subnet: `192.168.2.0/24`  
- Router/Gateway: `192.168.2.254`  
- Cluster nodes: `192.168.2.31–40`  
- **Reserved for MetalLB**: choose a range outside of node/DHCP allocations.  
  - Example: `192.168.2.41–192.168.2.60`

## 2. Install MetalLB

Create the namespace:

```bash
kubectl create namespace metallb-system
```

Apply the MetalLB manifests:

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml
```

Verify pods are running:

```bash
kubectl -n metallb-system get pods
```

## 3. Configure IP Address Pool

Create an IPAddressPool that defines the range of IPs MetalLB can assign:

```yaml
# metallb-ipaddresspool.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-address-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.2.41-192.168.2.60
```

Apply it:

```bash
kubectl apply -f metallb-ipaddresspool.yaml
```

## 4. Configure L2 Advertisement

MetalLB uses Layer 2 mode in this cluster. Create an L2Advertisement:

```yaml
# metallb-l2advertisement.yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec: {}
```

Apply it:

```bash
kubectl apply -f metallb-l2advertisement.yaml
```

## 5. Verify Functionality

Deploy a test service with type LoadBalancer:

```yaml
kubectl create deployment hello --image=nginx
kubectl expose deployment hello --type=LoadBalancer --port=80
```

Apply and check:

```bash
kubectl get svc hello
```

You should see an external IP from the pool (e.g. 192.168.2.41).
Access it directly in your browser: http://192.168.2.41

### Cleanup (optional)

Once we'vc verified it's working we can remove this service

#### 1. Delete the Service and Deployment

Remove the service and deployment

```bash
kubectl delete svc hello
kubectl delete deploy hello
```

#### 2. Verify Cleanup

Check that the resources are gone:

```bash
kubectl get svc
kubectl get deploy
```

There should be no `hello` service or deployment left in the namespace.

## 6. Expose Longhorn UI via MetalLB

Instead of using kubectl port-forward, expose the Longhorn frontend with a fixed LoadBalancer IP.

Create a new Service:

```yaml
# longhorn-frontend-lb.yaml
apiVersion: v1
kind: Service
metadata:
  name: longhorn-frontend-lb
  namespace: longhorn-system
spec:
  type: LoadBalancer
  loadBalancerIP: 192.168.2.41
  selector:
    app: longhorn-ui
  ports:
    - port: 80
      targetPort: 8000
```

Apply it:

```bash
kubectl apply -f longhorn-frontend-lb.yaml
```

Now open the Longhorn UI at:
👉 http://192.168.2.41

(Adjust the loadBalancerIP if you want a different fixed address within your MetalLB pool.)

## 7. Best Practices

- Reserve the MetalLB IP range in your DHCP server (Pi-hole) to avoid conflicts.
- Use DNS entries pointing to the MetalLB-assigned IPs for friendlier access (e.g. app.local).
- Monitor MetalLB pods in metallb-system for health.
