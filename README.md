# TLS Rotation
Instructions on how to rotate TLS CA and certs for a Tectonic cluster.

## Generate new certs and patches

#### Prepare

**CAUTION**: Before rotation, it's preferable to back up your cluster:
1. On an etcd node, take a backup of the current state.
2. [Run bootkube recover](https://coreos.com/tectonic/docs/latest/troubleshooting/bootkube_recovery_tool.html) to extract the existing state from etcd. Copy this back to your working machine as a precaution, in case the control plane goes down.
3. Download the current kubeconfig and assets.zip.

#### Prerequisite

- `ssh-agent` is required for ssh'ing into the nodes to update files.
- `jq`
- `KUBECONFIG` kubeconfig of the cluster.
- `BASE_DOMAIN` base domain of the cluster, you might be able to retrieve it from the server address in the kubeconfig, e.g. `https://${CLUSTER_NAME}-api.${BASE_DOMAIN}:443`
- `CLUSTER_NAME` name of the cluster, you might be able to retrieve it from the server address in the kubeconfig, e.g. `https://${CLUSTER_NAME}-api.${BASE_DOMAIN}:443`

#### Run

```shell
export KUBECONFIG=PATH_TO_KUBECONFIG
export BASE_DOMAIN=example.com
export CLUSTER_NAME=my-cluster

./gencerts.sh generated
```

**CAUTION**: PLEASE save the generated assets somewhere, it's IMPORTANT for future reference!

## Rotate Etcd CA and certs

#### Prerequisite

- `KUBECONFIG` kubeconfig of the cluster.
- `MASTER_IPS` List of public IPs of the master nodes, separated by a space.
- `ETCD_IPS` List of private IPs of the etcd nodes, seperated by a space.
- `SSH_KEY` The ssh key for login in the master nodes.
- The etcd members MUST be in healthy state before rotating the CA and certs.

#### Run

```shell
export KUBECONFIG=PATH_TO_KUBECONFIG
export MASTER_IPS="IP1 IP2 ..."
export ETCD_IPS="IP1 IP2 ..."
export SSH_KEY="/home/.ssh/id_rsa"

./rotate_etcd.sh
```

## Rotate CA and certs in the cluster

#### Prerequisite

- `KUBECONFIG` kubeconfig of the cluster.
- `MASTER_IPS` List of public IPs of the master nodes, separated by a space.
- `WORKER_IPS` List of private IPs of the worker nodes, separated by a space.
- `SSH_KEY` The ssh key for login in the master nodes.

#### Run

```shell
export KUBECONFIG=PATH_TO_KUBECONFIG
export MASTER_IPS="IP1 IP2 ..."
export WORKER_IPS="IP1 IP2 ..."
export SSH_KEY="/home/.ssh/id_rsa"

./rotate_cluster.sh
```

#### NOTE
In order to rotate the kubelet certs, the kubeconfig on host needs to be updated.
If you are on AWS platform, a program is provided for this task:
```shell
./aws/update_kubeconfig --tfstate=PATH_TO_TFSTATE_FILE --kubeconfig=./generated/auth/kubeconfig
```

**PLEASE MAKE SURE THE KUBECONFIG IS UPDATED CORRECTLY, OTHERWISE THE ROTATION WILL FAIL!**

E.g. if you are on AWS you can run the following command to verify the kubeconfig:
```shell
aws s3 cp s3://S3_BUCKET_NAME/kubeconfig /tmp/kubeconfig
diff /tmp/kubeconfig generated/auth/kubeconfig
```

## Reboot Cluster

After updating the CA and certs of the API server, we need to restart all the pods
to ensure they refresh their service account.

This can be done by rebooting all the nodes in the cluster.

A script (`./utils/reboot_helper.sh`) is provided to make the step easier.
Once the reboot is done, the pods will come back eventually.
At this point, the CA rotation is fully completed.


## Verification

After the above steps, you can verify the success of the CA and certs rotation:

- Old `kubeconfig` should **NOT** be able to contact the API server, however the new kubeconfig should be able to talk to it.
- All nodes are expected to be `Ready`, all pods are expected to be `Running`
- Try to fetch the logs of `kube-apiserver`, `kube-scheduler` and `kube-controller-namager`, they should all be running correctly without spitting errors.
  E.g. ```kubectl logs -lk8s-app=kube-scheduler -n kube-system`
- You should be able to login the `Tectonic Console` and create test pods.
