# Setting container sysctls in TKGI

This is a quick walkthrough of setting sysctls in containers running on TKGI clusters.

**Prerequisites**

* [pks cli](https://network.pivotal.io/products/pivotal-container-service/)
* [kubectl](https://kubernetes.io)

**Start with an existing TKGI cluster**

1. verify the cluster name connectivity
```bash
pks clusters
```
```
PKS Version    Name    k8s Version  Plan Name  UUID                                  Status     Action
1.7.2-build.9  bruce1  1.16.14      medium     7391cd98-0135-4657-a91d-44d029d0441f  succeeded  UPDATE
```
```bash
kubectl version
```
```
Client Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.2", GitCommit:"52c56ce7a8272c798dbc29846288d7cd9fbae032", GitTreeState:"clean", BuildDate:"2020-04-16T11:56:40Z", GoVersion:"go1.13.9", Compiler:"gc", Platform:"darwin/amd64"}
Server Version: version.Info{Major:"1", Minor:"16", GitVersion:"v1.16.14+vmware.1", GitCommit:"bbf52cd6cf83a50507e943b717ed321d383a37b5", GitTreeState:"clean", BuildDate:"2020-08-19T23:48:38Z", GoVersion:"go1.13.15", Compiler:"gc", Platform:"linux/amd64"}
```

**Deploy a DaemonSet that sets sysctls on the Node hosts**

1.  Deploy a config map defining the sysctls you want to set:
```bash
kubectl deploy -f ./custom-sysctl-cm.yaml
```
2.  Deploy a daemonset that reads the configmap and updates host sysctls
```bash
kubectl deploy -f ./custom-sysctl-ds.yaml
```
3.  (optional) If you have access to the Bosh instance managing this cluster, you can verify that the sysctls are updated on a worker node
```bash
bosh -d <deployment-id> ssh worker/0 -c "cat /proc/sys/net/core/somaxconn /proc/sys/vm/max_map_count /proc/sys/net/ipv4/tcp_retries2"
```

4.  deploy a job to print the sysctls from within a container:
```bash
kubectl deploy -f ./job-no-sysctl.yaml
```

5. When the job completes, look at the logs.  How  do they compare with what is in the configmap? 
```bash
kubectl logs -l job-name=echo-sysctl
```

6.  Delete the job when you're done:
```bash
kubectl delete job echo-sysctl
```


Notice that only the non-namespaced sysctls are inherited from the host.  In this case, only `vm.max_map_count: 262000` is in that category.  That's because it can't be set in a pod spec because one pod changing this value would affect everything else running on the node.  Therefore, for non-namespaced sysctls we take must take the DaemonSet approach whereby we intentionally set the value on the Node knowing that all its conatiners will inherit it.

However, the other two sysctls are namespaced and therefore get the default values from the Linux `init_net` namespace regardless of the value on the node.  In order to change those, we must inform `kubelet` that authorized pods can adjust those values.

**Use a TKGI K8s profile to set a custom kubelet flag**
1.  Create the k8s policy
```bash
pks create-kubernetes-policy ./cucstom-sysctl-k8s-profile.json
```
```


Warning: Experimental_customizations is an Evaluation feature and is intended for evaluation and test purposes only. Product support and future availability are not guaranteed for Evaluation features. Do not use this feature in a production environment.

Kubernetes profile custom-sysctl-profile successfully created

```
Notice the warning.  If there's evidence that this approach adds value for enterprise workloads, we can add this kubelet flag to the `non-experimental` category.

2.  Update the cluster, applying the policy
```bash
pks update-cluster bruce1 --kubernetes-profile custom-sysctl-profile
```
TKGI will zero-downtime update your cluster with the new settings.

3.  Once again if you have Bosh access you can ssh into a worker node and verify the flags passed to `kubelet`.


**Deploy the RBAC assets to allow certain pods to set certain sysctls**

Look at these files to see how sysctl permissions are propagated to the `default` serviceaccount in the `default` namespace
```bash
kubectl apply -f ./custom-sysctl-psp.yml
kubectl apply -f ./custom-sysctl-cluster-role.yaml
kubectl apply -f ./custom-sysctl-role-binding.yaml
```

**Deploy a job that sets the sysctls for the pod and view the logs.**
```
kubectl deploy  -f ./job-with-sysctl.yml
kubectl logs -l job-name=echo-sysctl
kubectl  delete job echo-sysctl
```

Notice that the 2 additional sysctls are now set:
```
net.core.somaxconn: 64000
vm.max_map_count: 262000
net.ipv4.tcp_retries2: 13
```