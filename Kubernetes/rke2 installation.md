Installing RKE2 (Rancher Kubernetes Engine 2) in a high-availability (HA) configuration on Ubuntu involves several steps. Here's a detailed step-by-step guide to set up an HA RKE2 cluster with three control plane nodes and one or more worker nodes.

### Prerequisites

1. **Ubuntu Servers:** You need at least three Ubuntu servers for the control plane nodes and one or more for worker nodes. Ensure they have internet access.
2. **Root Access:** Ensure you have root access on all servers.
3. **Unique Hostnames:** Each node should have a unique hostname.
4. **Open Ports:** Ensure required ports are open (e.g., 6443 for the Kubernetes API server).

### Step 1: Set Up Hostnames and Update System

Update your system packages and set unique hostnames.

```sh
# Set the hostname (replace with appropriate hostname for each node)
sudo hostnamectl set-hostname <hostname>

# Update system packages
sudo apt update && sudo apt upgrade -y
```

### Step 2: Install RKE2 on All Nodes

1. **Download and Install RKE2**

```sh
curl -sfL https://get.rke2.io | sudo sh -
```

2. **Enable and Start the RKE2 Server on Control Plane Nodes**

On each control plane node, enable and start the RKE2 server service.

```sh
sudo systemctl enable rke2-server.service
sudo systemctl start rke2-server.service
```

### Step 3: Configure High Availability

1. **Generate a Token**

On the first control plane node, generate a token that will be used for joining other nodes.

```sh
sudo cat /var/lib/rancher/rke2/server/token
```

Copy this token for use on other nodes.

2. **Configure the First Control Plane Node**

Create the RKE2 configuration file `/etc/rancher/rke2/config.yaml`:

```yaml
# First control plane node
node-name: "control-plane-1"
write-kubeconfig-mode: "0644"
token: "<YOUR_GENERATED_TOKEN>"
tls-san:
  - "<LOAD_BALANCER_IP>"
```

Replace `<YOUR_GENERATED_TOKEN>` with the token generated earlier and `<LOAD_BALANCER_IP>` with the IP address of your load balancer.

Restart the RKE2 service to apply the configuration:

```sh
sudo systemctl restart rke2-server.service
```

3. **Configure Additional Control Plane Nodes**

On each additional control plane node, create the RKE2 configuration file `/etc/rancher/rke2/config.yaml`:

```yaml
# Additional control plane nodes
node-name: "control-plane-2" # Change for each node
server: "https://<LOAD_BALANCER_IP>:9345"
token: "<YOUR_GENERATED_TOKEN>"
```

Restart the RKE2 service to apply the configuration:

```sh
sudo systemctl restart rke2-server.service
```

### Step 4: Configure Worker Nodes

On each worker node, create the RKE2 configuration file `/etc/rancher/rke2/config.yaml`:

```yaml
# Worker nodes
server: "https://<LOAD_BALANCER_IP>:9345"
token: "<YOUR_GENERATED_TOKEN>"
```

Enable and start the RKE2 agent service:

```sh
sudo systemctl enable rke2-agent.service
sudo systemctl start rke2-agent.service
```

### Step 5: Set Up a Load Balancer

You need to set up a load balancer to distribute traffic among the control plane nodes. You can use HAProxy, Nginx, or any other load balancer. Here is an example using HAProxy.

1. **Install HAProxy**

```sh
sudo apt install haproxy -y
```

2. **Configure HAProxy**

Edit the HAProxy configuration file `/etc/haproxy/haproxy.cfg`:

```plaintext
frontend kubernetes-frontend
    bind *:6443
    option tcplog
    default_backend kubernetes-backend

backend kubernetes-backend
    balance roundrobin
    server control-plane-1 <CONTROL_PLANE_1_IP>:6443 check
    server control-plane-2 <CONTROL_PLANE_2_IP>:6443 check
    server control-plane-3 <CONTROL_PLANE_3_IP>:6443 check
```

Replace `<CONTROL_PLANE_1_IP>`, `<CONTROL_PLANE_2_IP>`, and `<CONTROL_PLANE_3_IP>` with the IP addresses of your control plane nodes.

3. **Restart HAProxy**

```sh
sudo systemctl restart haproxy
```

### Step 6: Verify the Cluster

1. **Get the Kubeconfig File**

On the first control plane node, copy the `kubeconfig` file to your local machine:

```sh
sudo cat /etc/rancher/rke2/rke2.yaml
```

2. **Set the KUBECONFIG Environment Variable**

On your local machine, set the `KUBECONFIG` environment variable to point to the downloaded `kubeconfig` file:

```sh
export KUBECONFIG=~/path/to/rke2.yaml
```

3. **Check the Cluster Nodes**

```sh
kubectl get nodes
```

You should see all your control plane and worker nodes listed.

### Conclusion

You now have an RKE2 high-availability cluster set up on Ubuntu servers. The control plane nodes are balanced by a load balancer, and the worker nodes are connected to the cluster. This setup ensures high availability and fault tolerance for your Kubernetes cluster.
