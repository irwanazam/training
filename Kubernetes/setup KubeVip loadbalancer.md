Setting up Kube-VIP as a load balancer for the control plane in an RKE2 (Rancher Kubernetes Engine 2) cluster involves deploying Kube-VIP as a static pod on each of the control plane nodes. Kube-VIP provides a virtual IP that floats between the control plane nodes, ensuring high availability for the Kubernetes API server.

### Step-by-Step Guide to Set Up Kube-VIP for RKE2 Control Plane

#### 1. Prerequisites

- Ensure you have RKE2 installed on your control plane nodes.
- Ensure each control plane node has a unique hostname.

#### 2. Download and Install Kube-VIP

**Step 1:** SSH into each control plane node.

**Step 2:** Download the Kube-VIP binary.

```bash
curl -Lo /usr/local/bin/kube-vip https://github.com/kube-vip/kube-vip/releases/download/v0.5.5/kube-vip
chmod +x /usr/local/bin/kube-vip
```

**Step 3:** Create a Kube-VIP configuration file.

On the first control plane node, run:

```bash
sudo kube-vip manifest daemonset \
  --interface eth0 \
  --address <Virtual-IP-Address> \
  --controlplane \
  --services \
  --arp \
  --leaderElection
```

Replace `<Virtual-IP-Address>` with the virtual IP address you want to use for the Kubernetes API server.

This command generates a DaemonSet manifest for Kube-VIP.

#### 3. Configure RKE2 to Use Kube-VIP

**Step 1:** Create a `config.yaml` file for RKE2 on each control plane node with the following content:

```yaml
write-kubeconfig-mode: "0644"
tls-san:
  - <Virtual-IP-Address>
node-taint:
  - "node-role.kubernetes.io/master:NoSchedule"
disable:
  - rke2-ingress-nginx
cni: calico
token: <node-token>
server: https://<Virtual-IP-Address>:6443
```

Replace `<Virtual-IP-Address>` with the virtual IP address you configured for Kube-VIP. Replace `<node-token>` with the node token from the first control plane node.

**Step 2:** Install and start RKE2 on the first control plane node.

```bash
curl -sfL https://get.rke2.io | sh -
sudo systemctl enable rke2-server.service
sudo systemctl start rke2-server.service
```

Retrieve the node token:

```bash
cat /var/lib/rancher/rke2/server/node-token
```

**Step 3:** Copy the node token to the `config.yaml` files on the other control plane nodes and adjust the server address.

**Step 4:** Start RKE2 on the other control plane nodes.

```bash
sudo systemctl enable rke2-server.service
sudo systemctl start rke2-server.service
```

#### 4. Deploy Kube-VIP DaemonSet

**Step 1:** Apply the Kube-VIP DaemonSet manifest generated earlier on the first control plane node:

```bash
kubectl apply -f /etc/kube-vip/kube-vip.yaml
```

**Step 2:** Verify that the Kube-VIP DaemonSet is running on all control plane nodes:

```bash
kubectl get pods -n kube-system -l name=kube-vip-ds
```

#### 5. Verify the Setup

**Step 1:** Verify that the control plane nodes are up and running and that the virtual IP is assigned to one of the nodes:

```bash
kubectl get nodes
```

**Step 2:** Check the virtual IP address assignment using `ip a` or similar network tools on your control plane nodes.

```bash
ip a | grep <Virtual-IP-Address>
```

### Conclusion

By following these steps, you set up Kube-VIP as a load balancer for the RKE2 control plane. Kube-VIP provides a highly available virtual IP address that ensures the Kubernetes API server remains accessible even if one or more control plane nodes fail. This setup simplifies the high-availability configuration and avoids the need for an external load balancer.
