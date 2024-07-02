When configuring RKE2 (Rancher Kubernetes Engine 2) for high availability, you need to set the `tls-san` field in the `config.yaml` 
to include the IP address or hostname of the load balancer that will front your control plane nodes. This means you need to create the load balancer first and then use its IP address or hostname in the `tls-san` configuration.

Here’s a step-by-step guide on how to set up RKE2 with a load balancer, assuming you are using HAProxy as the load balancer:

### Step-by-Step Guide to Set Up RKE2 with HAProxy

#### 1. Install and Configure HAProxy

**Step 1:** Set up a VM or server for HAProxy.

**Step 2:** Install HAProxy on the server.

```bash
sudo apt update
sudo apt install haproxy -y
```

**Step 3:** Configure HAProxy by editing the `/etc/haproxy/haproxy.cfg` file.

```cfg
frontend k8s-api
    bind *:6443
    mode tcp
    option tcplog
    default_backend k8s-api-backend

backend k8s-api-backend
    mode tcp
    balance roundrobin
    option tcp-check
    server master1 <master1-ip>:6443 check
    server master2 <master2-ip>:6443 check
    server master3 <master3-ip>:6443 check
```

Replace `<master1-ip>`, `<master2-ip>`, and `<master3-ip>` with the IP addresses of your RKE2 control plane nodes.

**Step 4:** Restart HAProxy to apply the configuration changes.

```bash
sudo systemctl restart haproxy
```

**Step 5:** Verify that HAProxy is running.

```bash
sudo systemctl status haproxy
```

#### 2. Configure RKE2 Control Plane Nodes

**Step 1:** On each control plane node, create or edit the `config.yaml` file for RKE2.

```yaml
write-kubeconfig-mode: "0644"
tls-san:
  - <load-balancer-ip-or-hostname>
node-taint:
  - "node-role.kubernetes.io/master:NoSchedule"
disable:
  - rke2-ingress-nginx
cni: calico
```

Replace `<load-balancer-ip-or-hostname>` with the IP address or hostname of your HAProxy load balancer.

**Step 2:** Install RKE2 on each control plane node.

```bash
curl -sfL https://get.rke2.io | sh -
```

**Step 3:** Start RKE2 on the first control plane node and retrieve the join token.

```bash
sudo systemctl enable rke2-server.service
sudo systemctl start rke2-server.service
```

Get the node token to join the other control plane nodes.

```bash
cat /var/lib/rancher/rke2/server/node-token
```

**Step 4:** Start RKE2 on the other control plane nodes using the token.

Edit the `config.yaml` on each additional control plane node to include the token.

```yaml
token: <node-token>
server: https://<load-balancer-ip-or-hostname>:6443
```

Start RKE2 on the additional control plane nodes.

```bash
sudo systemctl enable rke2-server.service
sudo systemctl start rke2-server.service
```

#### 3. Verify the Setup

**Step 1:** Verify that the control plane nodes are up and running.

```bash
kubectl get nodes
```

This command should list all the control plane nodes and show them in the `Ready` state.

### Conclusion

By following these steps, you set up HAProxy as a load balancer in front of your RKE2 control plane nodes. This configuration ensures high availability for your Kubernetes API server. The `tls-san` field in the RKE2 `config.yaml` file allows the control plane nodes to recognize and accept the load balancer's IP address or hostname, ensuring proper TLS communication.
