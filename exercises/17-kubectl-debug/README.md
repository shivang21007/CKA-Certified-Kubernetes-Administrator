# Exercise 17 — kubectl debug: Pod and Node

> **Medium** | ~15 min | Domain: Troubleshooting (30%) | **Updated May 2026**
>
> Related: [README — Troubleshooting](../../README.md#domain-2--troubleshooting-30)

Use `kubectl debug` to troubleshoot running pods and access node-level resources. This is GA in v1.35 and directly relevant to CKA troubleshooting tasks.

## Conventions

Before debugging, understand the trap architecture:

- **Primary Trap:** Debug container started without `--target` (isolated namespace instead of shared)
- **Secondary Trap (Gotcha):** Node debug without `chroot /host` (sees container filesystem, not node)
- **Validation Criteria:** Your debug session shows:
  - Pod debug: processes from target container visible in debug pod
  - Node debug: node filesystem mounted at `/host`, kubelet systemd status accessible
  - Both: no errors when listing processes or checking services
- **Scoring:** Full credit for successful debug + root cause identification. Partial for debug success alone.

## Tasks

### Part A: Debug a pod

1. Create a namespace called `exercise-17`
2. Create a pod named `broken-app` with image `nginx:1.27`
3. Attach a debug container to the running pod using `kubectl debug`:
   - Image: `busybox:1.36`
   - Target the `broken-app` container to share process namespace
4. From inside the debug container, list the processes (you should see the nginx process)
5. Check the filesystem and network from the debug container
6. Exit the debug session

### Part B: Debug a node

7. Run `kubectl debug` against a node to get a shell with host filesystem access
8. Use `chroot /host` to access the real node root
9. Check kubelet status from the debug session
10. Exit the debug session

## Hints

<details>
<summary>Stuck? Click to reveal hints</summary>

- Pod debug: `k debug <pod> -it --image=busybox:1.36 --target=<container>`
- Node debug: `k debug node/<node-name> -it --image=busybox:1.36`
- The `--target` flag shares process namespace with the target container
- Node debug mounts the host filesystem at `/host`
- Use `chroot /host` to run node commands directly

</details>

## What tripped me up

> **Pod Debug Trap:** I ran `k debug broken-app -it --image=busybox:1.36` without `--target=broken-app`. The debug container started, but `ps aux` only showed my busybox shell — no nginx processes. Without `--target`, the debug container gets its own process namespace. You NEED `--target=<container>` to share process namespace with the container you're debugging. This is the most common k8s 1.35 gotcha: `--target` is not optional; it's the difference between isolated debugging and actual container inspection.
>
> **Node Debug Trap:** Node debugging without `chroot /host` leaves you in the debug pod's filesystem, not the real node. `systemctl status kubelet` fails because systemd isn't mounted. After `chroot /host`, you get full node access. Common mistake: spending 5 minutes trying to find config files that don't exist in the container root.
>
> **May 2026 Gotcha (Image Availability):** If you specify `--image=busybox:1.36` but the node doesn't have it cached, `kubectl debug` will pull it. On exam nodes with slow network or no direct internet access, this can timeout silently. Fallback: use `--image=ubuntu:22.04` or whatever is already cached. Check: `crictl images` on the node first.
>
> **Debug Container Port Forwarding Gotcha:** From inside a debug pod in the pod namespace, you can `curl localhost:80` to test the target container's port. But from node debug, `localhost` is the node, not a specific service. You need to either `curl <service-ip>` (find with `k get svc`) or `curl <pod-ip>:port` (find in target pod). Make sure you understand the network stack level you're debugging.
>
> **Multiple Containers Trap:** If the pod has multiple containers, `--target` specifies which ONE. If you `--target=container-a` but the pod's issue is in `container-b`, you won't see the broken process. Always verify which container the issue is in before attaching the debug pod.
>
> **Resource Limits on Debug Pod:** kubectl debug respects node resource limits. If a node is under resource pressure, you might not be able to spawn a debug pod. This is exam-realistic: you want to debug a problem but the cluster is too broken to create debug resources. Verify: `k describe node` for pressure conditions before failing debug as "not working."

## Verify

```bash
# Part A: process list should show nginx
# Inside debug container:
ps aux

# Part B: kubelet check
# Inside node debug (after chroot /host):
systemctl status kubelet
```

## Cleanup

```bash
k delete ns exercise-17
```

<details>
<summary>Solution</summary>

### Part A

```bash
k create ns exercise-17

k run broken-app -n exercise-17 --image=nginx:1.27

# Wait for running
k get pod broken-app -n exercise-17 -w

# Debug with shared process namespace
k debug broken-app -n exercise-17 -it --image=busybox:1.36 --target=broken-app

# Inside the debug container:
ps aux                      # should show nginx master + worker processes
ls /proc/1/root/etc/nginx/  # access the target container filesystem
wget -qO- localhost:80      # test connectivity from inside
exit
```

### Part B

```bash
# Get a node name
k get nodes
# e.g., node-1

k debug node/node-1 -it --image=busybox:1.36

# Inside:
chroot /host

# Now you have real node access
systemctl status kubelet
journalctl -u kubelet --no-pager | tail -20
crictl ps
cat /etc/kubernetes/manifests/kube-apiserver.yaml | head -20

exit  # exit chroot
exit  # exit debug pod
```

</details>
