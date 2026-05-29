# Exercise 28 — Complex NetworkPolicy (Multi-Namespace, Traffic Control)

> Related: [NetworkPolicy skeleton](../../skeletons/networkpolicy.yaml) | [README — Networking](../../README.md#domain-5--services--networking-13)

Create multi-namespace NetworkPolicies with ingress/egress rules with careful label matching and debugging. Understand that NetworkPolicy enforcement depends on the CNI (Container Network Interface) plugin your cluster uses — not all CNIs enforce all policy rules equally.

## Important: CNI-Aware Testing

NetworkPolicy resources exist across all Kubernetes clusters, but whether they're *enforced* depends on your CNI:

- **Enforcing CNIs**: Calico, Cilium, Weave, Kube-router
- **Non-enforcing CNIs**: flannel, kubenet (pass-through)

If your CNI doesn't enforce policies, the NetworkPolicy objects exist but traffic flows freely. Exam clusters use enforcing CNIs (usually Calico). For testing locally, verify your CNI supports NetworkPolicy before debugging.

## Tasks

1. Create 3 namespaces: `frontend`, `backend`, `database`
2. Deploy apps:
   - Frontend (nginx): labels `tier=frontend`
   - Backend (busybox): labels `tier=backend`
   - Database (redis): labels `tier=database`
3. Create NetworkPolicies:
   - Allow frontend → backend (only port 8080)
   - Allow backend → database (only port 6379)
   - Deny all other traffic initially (prefer-deny ingress)
   - Allow DNS egress (UDP 53) to kube-dns for all pods
4. **Explicitly test connectivity** to confirm policies are enforced:
   - Frontend CAN reach backend on 8080 only
   - Frontend CANNOT reach database
   - Backend CAN reach database on 6379
   - Backend CANNOT reach frontend
5. Verify DNS still works from all namespaces (tests egress rules)
6. If policies don't enforce, verify CNI supports NetworkPolicy

## Key Learning

- NetworkPolicies are pod label selectors, not namespace selectors alone
- Ingress rules applied to target pods
- Egress rules applied to source pods
- Must explicitly allow DNS egress or service discovery breaks
- Debugging: use `k describe pod` and check labels carefully
- Exam tests label matching precision

## Hints

<details>
<summary>Stuck? Click to reveal hints</summary>

- **Check CNI enforcement first:** `k get pods -n kube-system | grep -i cni` or `kubectl describe node` to see CNI plugin
- For cross-namespace policies: use `namespaceSelector` with namespace labels (label namespaces first!)
- DNS rule: egress with `to: []` and `ports: [{protocol: UDP, port: 53}]` (empty `to` means all destinations)
- Test connectivity with **explicit port**: `k exec <pod> -- curl http://service.namespace.svc.cluster.local:8080`
- Debug: `k describe networkpolicy <name>` shows selectors AND rules clearly
- **Order matters:** ingress applied to destination pod, egress to source pod. Both must allow for traffic to flow.
- **Timeouts mean denial:** If `curl` hangs with no response, the policy is likely dropping packets (good sign policies work!)
- **CNI not enforcing?** Policies may exist but not be enforced. Verify with `k get networkpolicies` and check node's CNI plugin.

</details>

## What tripped me up

> **Source Pod Egress Rules Required Too**
>
> Created correct ingress rules on backend but forgot that the source pod (frontend) ALSO needs egress rules allowing it out. Even with a perfect backend ingress rule, if frontend has no matching egress, traffic is denied at the source. NetworkPolicy works in both directions: destination (ingress) AND source (egress) must both allow.

> **DNS Fails Without Explicit Egress**
>
> All pods couldn't resolve service names — `nslookup` timed out. The problem: I created ingress/egress rules for app traffic but forgot to explicitly allow DNS egress (UDP port 53 to kube-dns). ServiceName resolution broke. Always add `allow-dns-egress` rules unless you're deliberately blocking all egress.

> **Labels Must Match Exactly (Case-Sensitive, Whitespace)**
>
> Policies compiled and looked right, but traffic was still blocked. Turned out I had a typo in a label selector: `tier: backend` in the policy but the pod was labeled `tier: Backends` (capital B). Label matching is case-sensitive and whitespace matters. A character difference silently fails — no error message, just traffic blocked. Always double-check label selectors with `kubectl get pods --show-labels`.

> **Policies May Not Enforce on All CNIs**
>
> Created perfect NetworkPolicies on a test cluster and nothing was enforced. I tested with flannel CNI, which doesn't enforce NetworkPolicy (it doesn't implement the enforcement layer). Switched to Calico and everything worked as expected. CKA exam uses Calico, but know which CNIs actually enforce policies: Calico, Cilium, Weave enforce; flannel and kubenet do not.

> **Port Ranges: Understand endPort Behavior**
>
> Created a policy allowing port 8080, then tested with 8081 — got blocked (correct). But when I added `endPort: 8090`, the range included 8080-8090. Some CNIs don't support endPort or have quirks negotiating range boundaries. Test the exact port range and understand your CNI's support level by checking ingress port count constraints.

## Verify

```bash
# Pods are running in correct namespaces
k get pods -A

# NetworkPolicies exist
k get networkpolicies -A

# From frontend, test connectivity
k exec -it <frontend-pod> -n frontend -- curl http://backend.<backend>.svc.cluster.local:8080
# Should succeed

# From frontend, try database (should fail)
k exec -it <frontend-pod> -n frontend -- curl http://database.<database>.svc.cluster.local:6379
# Should timeout/fail

# Test DNS works
k exec -it <any-pod> -- nslookup kubernetes.default
# Should succeed
```

## Cleanup

```bash
k delete ns frontend backend database
```

<details>
<summary>Solution</summary>

```bash
# Create namespaces
k create ns frontend backend database

# Label namespaces for cross-ns selection
k label ns frontend name=frontend
k label ns backend name=backend
k label ns database name=database

# Deploy frontend
k run frontend-web -n frontend --image=nginx:1.27 --labels=tier=frontend

# Deploy backend
k run backend-api -n backend --image=busybox:1.36 --command sleep 3600 --labels=tier=backend

# Deploy database
k run database-redis -n database --image=busybox:1.36 --command sleep 3600 --labels=tier=database

# Create services
k expose pod frontend-web -n frontend --port=80 --type=ClusterIP
k expose pod backend-api -n backend --port=8080 --type=ClusterIP
k expose pod database-redis -n database --port=6379 --type=ClusterIP

# Allow frontend → backend
cat <<EOF | k apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: backend
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
    ports:
    - protocol: TCP
      port: 8080
EOF

# Allow backend → database
cat <<EOF | k apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-database
  namespace: database
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: backend
    ports:
    - protocol: TCP
      port: 6379
EOF

# Allow DNS egress from all namespaces
for ns in frontend backend database; do
cat <<EOF | k apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: $ns
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to: []
    ports:
    - protocol: UDP
      port: 53
EOF
done

# Test connectivity
# Frontend → Backend (should work)
k exec -it $(k get po -n frontend -o name | head -1) -n frontend -- curl -m 2 http://backend-api.backend.svc.cluster.local:8080

# Frontend → Database (should fail)
k exec -it $(k get po -n frontend -o name | head -1) -n frontend -- curl -m 2 http://database-redis.database.svc.cluster.local:6379
```

</details>
