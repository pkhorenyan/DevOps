
# Section 1: Pod and Controller Behavior

## Q1 If an initContainer fails and the main container has restartPolicy: Never, what happens to the Pod?

Answer: The Pod gets stuck in Init:Error or Init:CrashLoopBackOff and never starts the main container.

With restartPolicy: Never, Kubernetes will not retry any container after failure. You have to manually delete and recreate the Pod to try again.

This catches a lot of people off guard because they assume restartPolicy only applies to the main container. It applies to everything.

## Q2 You delete pod-1 from a 3-replica StatefulSet. Do pods get renamed to fill the gap?

Answer: No. Kubernetes recreates the exact same pod-1 with the same name, same PVC binding, and same DNS entry.

That stable identity is the whole point of StatefulSets. If pods got renamed, it would break PVC bindings and corrupt the ordering guarantees.

Only during a scale-down does Kubernetes remove in reverse order, highest ordinal first.

## Q3 Does a DaemonSet automatically bypass NoSchedule taints on master or control-plane nodes?

Answer: No. This is one of the most common wrong answers I hear. DaemonSets do not get automatic NoSchedule tolerance.

You must explicitly add tolerations in the DaemonSet pod template for control-plane nodes.

What DaemonSets do get automatically are tolerations for not-ready and unreachable taints with NoExecute effect, so pods stay running during node failures.

## Q4 You push a new image while a rolling update is still in progress. Does Kubernetes wait for it to finish?

Answer: No. Kubernetes immediately starts a new rollout for the latest image and scales down the in-progress ReplicaSet.

This is by design. Kubernetes always converges towards the latest desired state, not the current in-flight one.

Practically this means you need proper CI/CD gates or you can end up triggering cascading rollouts during peak traffic.

## Q5 When a node goes NotReady, how long before Pods get evicted and can you control this per Pod?

Answer: By default, Pods are evicted after 5 minutes (300 seconds). The kube-controller-manager controls this.

For per-Pod control, set tolerationSeconds on the

[node.kubernetes.io/not-ready:NoExecute](https://node.kubernetes.io/not-ready:NoExecute)

and unreachable:NoExecute tolerations.

Critical system pods often tolerate longer. Latency-sensitive apps sometimes go much shorter to fail fast and reschedule quickly.

## Q6 Can two containers in the same Pod bind to the same port?

Answer: No. All containers in a Pod share one network namespace, which means they share the same port space.

The second container to bind will get an 'address already in use' error immediately.

This is a real gotcha when designing multi-container Pods. Sidecars and proxies must use their own dedicated ports.

## Q7 With ReadWriteOnce access mode, can multiple Pods on the same node use the PVC at the same time?

Answer: ReadWriteOnce means the volume can be mounted read-write by a single node, not a single Pod.

Whether multiple Pods on that node can actually use it depends entirely on the CSI driver and storage backend.

If you want guaranteed single-Pod access, use ReadWriteOncePod which was added in Kubernetes 1.22. That gives you an explicit contract.

## Q8 What happens to HPA if the metrics server goes down during a traffic spike?

Answer: HPA stops all scaling decisions, both up and down, and holds at the current replica count until metrics come back.

It will log 'unable to get metrics' events. But your application is still taking load and you are stuck.

The fix is running metrics-server in HA mode, or using Prometheus Adapter as a more resilient backend with multiple replicas.

## Q9 Can you kubectl port-forward to a CrashLoopBackOff Pod?

Answer: Technically yes, but it will break every time the container crashes and restarts.

For real debugging, use kubectl debug to attach an ephemeral container, or kubectl logs --previous to see what happened in the last run.

CrashLoopBackOff under port-forward pressure is not a reliable debugging setup. Fix the crash first.

## Q10 What happens to mounted tokens and API access if you delete a ServiceAccount while Pods are still running?

Answer: Running Pods keep their existing tokens and continue working until those tokens expire, roughly 1 hour for projected service account tokens.

After expiry, the kubelet tries to rotate the token, fails because the ServiceAccount is gone, and the Pod loses API server access.

The Pod keeps running but any API calls start returning 401. This is a quiet failure that is easy to miss in blue-green deployments.

# Section 2: Production Troubleshooting

## Q11 Can strict anti-affinity rules create a scheduling deadlock?

Answer: Yes. A classic one: you have 3 nodes, a required anti-affinity rule saying no two pods on the same node, and you try to schedule 4 replicas. The 4th sits Pending forever.

Use preferredDuringSchedulingIgnoredDuringExecution for non-critical spread constraints. Save required for things like keeping DB replicas in separate AZs.

Always test your affinity rules against your minimum node count, not your current fleet size.

## Q12 A Job has parallelism: 3 and one Pod fails with restartPolicy: Never. Does the Job create a replacement?

Answer: Yes. The Job controller creates a new Pod to maintain the desired parallelism level.

With restartPolicy: Never, the failed Pod stays in Failed state for log inspection, and the Job keeps creating fresh Pods.

Watch your backoffLimit setting. The default is 6, which means a misbehaving job can flood your cluster with 6 rounds of failed Pods.

## Q13 Can you modify resource requests and limits on a running Pod?

Answer: In Kubernetes 1.27+, In-Place Pod Vertical Scaling allows modifying CPU without a restart in most cases. Memory is more restricted.

On older clusters you must delete and recreate the Pod.

During OOM, the kernel kills processes that exceed their memory limits. At the node level, BestEffort pods are evicted first, then Burstable, and Guaranteed pods only when they exceed their own limits.

## Q14 If a NetworkPolicy only specifies ingress rules, is egress traffic blocked?

Answer: No. Egress is only blocked if you explicitly include 'Egress' in policyTypes.

If you add policyTypes: [Ingress, Egress] but write no egress rules, then all outbound traffic is denied.

One mistake I see all the time: teams apply a default-deny policy and forget to allow DNS egress to port 53. This silently breaks all service discovery.

## Q15 If a shared Persistent Volume gets corrupted, can it cascade failures across multiple namespaces?

Answer: For ReadWriteMany volumes backed by NFS or EFS, multiple PVCs across namespaces can reference the same underlying storage.

If that storage fails, every Pod using it fails at the same time, across every namespace.

This is a real blast radius concern in multi-tenant clusters. Separate storage pools per tenant is the right answer for anything sensitive.

# Section 3: Debugging Real Scenarios

## Q16 Your Pod is in CrashLoopBackOff but logs show nothing. Where do you start?

Answer: Start with kubectl describe pod and look at the Events section. It often tells you about OOMKilled, probe failures, or image issues before the container writes a single log line.

Use kubectl logs --previous to see output from the last crash. Check if memory limits are too tight because OOM kills are silent.

Temporarily remove liveness probes to see if they are killing the container prematurely. A misconfigured probe on a slow-starting app is a very common silent killer.

## Q17 A StatefulSet Pod will not recreate properly after deletion. How do you fix this without losing data?

Answer: First check PVC status. If it is stuck in Terminating, a finalizer is blocking it. Only remove the finalizer manually after confirming there is no active I/O.

Verify the storage class is still available and the target node can reach the volume. Zonal storage like EBS is tied to specific AZs.

Force-deleting the Pod should be a last resort. Always confirm PVC integrity before you do that.

## Q18 Cluster Autoscaler is not scaling up even though Pods are Pending. What do you check?

Answer: Most common reason: Pods have no resource requests. CA completely ignores Pods with zero requests.

Also check if your node group has already hit its max size, and look at scheduling constraints like nodeAffinity or PodAffinity rules that no new node can satisfy.

Check CA logs for 'cannot expand' messages. They are usually very specific. Also verify IAM permissions for calling cloud provider APIs.

## Q19 A NetworkPolicy is breaking cross-namespace service communication. How do you debug and fix it?

Answer: Start with a default-deny baseline, then add explicit allow rules. Use namespaceSelector to target the source namespace and podSelector for specific Pods.

You need both an ingress allow on the destination and an egress allow on the source side. Missing one side is the most common mistake.

Remember DNS. You must explicitly allow egress to kube-dns on port 53 UDP and TCP or service name resolution silently fails for everything.

## Q20 A microservice needs to connect to an external database through a VPN inside the cluster. How do you architect this?

Answer: Deploy VPN gateway Pods as a Deployment with anti-affinity across availability zones, fronted by a ClusterIP Service. Apps connect through the Service, not directly to VPN pods.

Store VPN credentials in Kubernetes Secrets. Apply NetworkPolicy to restrict VPN access to only the namespaces that need it.

Add a database proxy layer like PgBouncer or RDS Proxy in front. It handles connection pooling and hides the VPN complexity from application code completely.

# Section 4: Security and Architecture

## Q21 How do you isolate tenants on a shared EKS cluster with proper security, quotas, and observability?

Answer: Layer your isolation. Use namespace-level ResourceQuotas and LimitRanges for compute governance, NetworkPolicies with default-deny per namespace for traffic, and RBAC with least-privilege roles scoped to each namespace.

For node-level isolation of sensitive tenants, use node taints combined with tolerations and nodeAffinity to dedicate node groups.

Tag all resources with tenant labels and filter Prometheus queries and Grafana dashboards by those labels. For regulated industries, separate Fargate profiles per tenant is the cleanest model.

## Q22 The kubelet keeps restarting on a specific node. How do you isolate the issue?

Answer: Cordon the node first so nothing new gets scheduled there. Then check system resources with top, df, and iostat.

Look at kubelet logs via journalctl -u kubelet -f and check disk pressure. A full /var/lib/kubelet or /var/lib/containerd directory will crash the kubelet reliably.

Check for OOM killer events in kernel logs. Persistent hardware issues like disk I/O errors often show up as intermittent kubelet crashes. If you suspect hardware, replace the node.

## Q23 A critical production Pod keeps getting evicted due to node memory pressure. How do you prevent this?

Answer: Set equal requests and limits to get Guaranteed QoS class. Guaranteed Pods are the last ones evicted, only when they exceed their own limits.

Add a high-priority PriorityClass and create a PodDisruptionBudget to limit voluntary disruptions.

Reserve system resources on nodes using kubelet flags like --system-reserved and --kube-reserved so node overhead does not eat into pod memory silently.

## Q24 An application needs TCP and UDP on the same port number. How do you configure this in Kubernetes?

Answer: Kubernetes Services cannot expose the same port for both TCP and UDP in a single Service object. You need two separate Service objects.

If you need a single external IP for both protocols, check whether your cloud provider's load balancer supports mixed-protocol. AWS NLB does, but support varies across providers.

Standard Ingress controllers only handle HTTP and HTTPS. UDP and raw TCP must go through Services directly, not through Ingress.

## Q25 A rolling update caused downtime even though you had it configured. What advanced strategies fix this?

Answer: Most common cause: the readiness probe passed before the app was actually ready, or the app did not handle SIGTERM gracefully so in-flight requests were dropped.

Fix with a preStop hook sleep to allow load balancer deregistration, terminationGracePeriodSeconds longer than your request timeout, and maxUnavailable: 0 with maxSurge: 1.

For truly zero-downtime-critical services, combine with canary deployments using Argo Rollouts or Flagger so you can validate traffic before full promotion.

# Section 5: Performance and Advanced Topics

## Q26 Your Istio Envoy sidecar is using more CPU and memory than the actual application. How do you optimize?

Answer: Start by right-sizing using actual observed data from Prometheus. Look at envoy_server_total_connections and container_memory_usage_bytes before changing anything.

Disable features you are not using. Distributed tracing has overhead. Access logging for internal service-to-service traffic adds up fast.

Use Sidecar resources to limit the scope of Envoy's xDS configuration. By default, Envoy learns about every service in the mesh. That is expensive. Scope it to what each workload actually needs.

## Q27 You are building a Kubernetes operator. How do you design the CRD and reconciliation loop?

Answer: Keep spec and status separate. Spec is what the user declares. Status is what the operator writes. Never mix them.

Make your reconciliation loop completely idempotent. Running it twice on the same CR must produce the same result. Use owner references so child resources get garbage-collected when the CR is deleted.

For errors, use exponential backoff with a rate limiter. Without this, a broken reconciler will hammer the API server and cause cluster-wide problems.

## Q28 Multiple nodes are showing high disk I/O because of container logs. How do you address this?

Answer: Configure log rotation in kubelet using --container-log-max-size and --container-log-max-files. Add ephemeral-storage limits to pods so a single chatty container cannot fill the node disk.

The most common cause of log volume explosions in production is debug-level logging that was never changed before deployment. Raise log levels to WARN or ERROR.

Route logs to a centralized system like Loki or CloudWatch via a Fluent Bit DaemonSet. Offload logs before they accumulate locally.

## Q29 Your etcd cluster performance is degrading. What are the root causes and how do you fix it?

Answer: etcd is extremely sensitive to disk latency. The number one cause is slow fsync on the WAL. etcd needs under 10ms disk latency consistently.

Monitor etcd_disk_wal_fsync_duration_seconds. If your p99 is above 10ms, you have a disk problem. Use dedicated NVMe SSDs or provisioned IOPS volumes.

Also configure --auto-compaction-retention to prevent unbounded DB growth, and deploy etcd across 3 or 5 nodes in separate AZs for HA.

## Q30 How do you enforce that all images in the cluster must come from a trusted internal registry?

Answer: Use OPA Gatekeeper or Kyverno with a policy that validates the image field against an allowlist of registry prefixes. This runs at admission time so unauthorized images are rejected before scheduling.

Pair this with ImagePolicyWebhook if you need richer logic like checking digests or scanning results.

Also apply NetworkPolicies to restrict egress to public registries from inside the cluster. This is your second layer in case admission control is bypassed.