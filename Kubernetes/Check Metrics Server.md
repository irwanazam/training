To check if the Metric Server is installed in your Kubernetes cluster, you can try the following methods:

1. Check for the metric-server deployment:

```bash
kubectl get deployment metrics-server -n kube-system
```

If the Metric Server is installed, you should see output with details about the deployment.

2. Look for the metrics-server pod:

```bash
kubectl get pods -n kube-system | grep metrics-server
```

This should return one or more pods with "metrics-server" in the name if it's installed.

3. Check available API resources:

```bash
kubectl api-resources | grep metrics
```

If the Metric Server is installed and functioning, you should see entries for "metrics.k8s.io" in the output.

4. Attempt to fetch node or pod metrics:

```bash
kubectl top nodes
```

or

```bash
kubectl top pods
```

If these commands return metrics data, it indicates that the Metric Server is installed and working properly.

If you don't see the expected output from these commands, the Metric Server may not be installed or may not be functioning correctly. Would you like me to explain any of these methods in more detail?
