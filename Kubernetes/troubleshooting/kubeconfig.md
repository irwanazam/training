If you can only run `kubectl get nodes` with the `--kubeconfig` flag, it indicates that the `KUBECONFIG` environment variable is not set correctly. Here’s how to resolve this issue so that you don't have to specify the `--kubeconfig` flag every time.

### Setting the KUBECONFIG Environment Variable

You need to set the `KUBECONFIG` environment variable to point to your RKE2 kubeconfig file. Here's how you can do it:

1. **Find the Kubeconfig File**

   The default location for the RKE2 kubeconfig file is `/etc/rancher/rke2/rke2.yaml`.

2. **Set the Environment Variable**

   You can set the environment variable temporarily for the current session:

   ```sh
   export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
   ```

   To verify that it’s set correctly, you can print the value:

   ```sh
   echo $KUBECONFIG
   ```

   Now, you should be able to run `kubectl` commands without specifying the `--kubeconfig` flag:

   ```sh
   kubectl get nodes
   ```

3. **Persisting the Setting**

   To make this change permanent, you can add the `export` command to your shell's configuration file. For example, if you are using `bash`, you can add it to your `.bashrc` or `.bash_profile` file:

   ```sh
   echo 'export KUBECONFIG=/etc/rancher/rke2/rke2.yaml' >> ~/.bashrc
   source ~/.bashrc
   ```

   If you are using `zsh`, add it to your `.zshrc` file:

   ```sh
   echo 'export KUBECONFIG=/etc/rancher/rke2/rke2.yaml' >> ~/.zshrc
   source ~/.zshrc
   ```

### Verify the Configuration

After setting the `KUBECONFIG` environment variable, you should be able to use `kubectl` commands without specifying the `--kubeconfig` flag:

```sh
kubectl get nodes
```

### Troubleshooting

If you still encounter issues, here are some additional checks:

1. **Check Permissions**

   Ensure that the kubeconfig file has the correct permissions and is readable by the user you are executing the commands with:

   ```sh
   ls -l /etc/rancher/rke2/rke2.yaml
   ```

   If needed, adjust the permissions:

   ```sh
   sudo chmod 644 /etc/rancher/rke2/rke2.yaml
   ```

2. **Verify Content**

   Ensure the content of your kubeconfig file is correct and points to the correct server and user credentials.

3. **Multiple Configurations**

   If you have multiple kubeconfig files, you can merge them into one file. To do this, set the `KUBECONFIG` variable to a list of kubeconfig files:

   ```sh
   export KUBECONFIG=/path/to/first/config:/path/to/second/config
   ```

By following these steps, you should be able to configure your environment so that you can use `kubectl` without repeatedly specifying the `--kubeconfig` flag.
