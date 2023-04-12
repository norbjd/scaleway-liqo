# Introduction

Steps to install liqo on Scaleway Kapsule (managed Kubernetes).

Run all commands from a clean environment using `docker`:

```shell
docker run --rm -it \
  -v $PWD/kubeconfigs:/app/kubeconfigs \
  -v $PWD/liqo-values.yaml:/app/liqo-values.yaml \
  -v $PWD/daemonset-patch.yaml:/app/daemonset-patch.yaml \
  -v $PWD/manifests:/app/manifests \
  -w /app \
  alpine:3.17.0 sh
```

# Create clusters

```shell
# install prerequisites (helm, yq, jq, kubectl, liqoctl, scw = the Scaleway CLI)
apk add helm yq jq
wget -O /usr/local/bin/kubectl https://dl.k8s.io/release/v1.24.11/bin/linux/amd64/kubectl && chmod +x /usr/local/bin/kubectl 
wget -O - "https://github.com/liqotech/liqo/releases/download/v0.8.0/liqoctl-linux-amd64.tar.gz" | tar -xz
install -o root -g root -m 0755 liqoctl /usr/local/bin/liqoctl
wget -O /usr/local/bin/scw "https://github.com/scaleway/scaleway-cli/releases/download/v2.14.0/scaleway-cli_2.14.0_linux_amd64" && chmod +x /usr/local/bin/scw

helm repo add liqo https://helm.liqo.io/
helm repo update

# configure scw CLI
mkdir -p ~/.config/scw

# enter the three required variables for interacting with Scaleway (with scw CLI)
read -s SCW_ACCESS_KEY
read -s SCW_SECRET_KEY
read -s SCW_DEFAULT_PROJECT_ID

cat > ~/.config/scw/config.yaml <<EOF
active_profile: liqo

profiles:
  liqo:
    access_key: $SCW_ACCESS_KEY
    secret_key: $SCW_SECRET_KEY
    default_organization_id: $SCW_DEFAULT_PROJECT_ID
    default_project_id: $SCW_DEFAULT_PROJECT_ID
EOF

# creates a cluster (kubernetes 1.24.11 with flannel CNI)
function create_cluster() {
    REGION="$1"
    ZONE="$REGION"-1
    NAME="$2"
    
    scw k8s cluster create name=$NAME type=kapsule cni=flannel version=1.24.11 region=$REGION
    sleep 5
    KAPSULE_ID=$(scw k8s cluster list region=$REGION -o json | jq -r --arg name "$NAME" '.[] | select(.name==$name) | .id')
    scw k8s pool create cluster-id=$KAPSULE_ID name=pool node-type=PLAY2_NANO size=2 region=$REGION zone=$ZONE
    scw k8s cluster wait $KAPSULE_ID region=$REGION
    scw k8s kubeconfig get $KAPSULE_ID region=$REGION > kubeconfigs/kubeconfig-$NAME.yaml
}

create_cluster "fr-par" "france"
create_cluster "fr-par" "france2"
create_cluster "nl-ams" "netherlands"
create_cluster "pl-waw" "poland"
```

# Install liqo

Before installing liqo, wait until cluster is ready (all nodes "Ready" + all pods started correctly).

```shell
function install_liqo() {
  NAME="$1"
  export KUBECONFIG="kubeconfigs/kubeconfig-$NAME.yaml"
  
  MASTER_URL=$(cat $KUBECONFIG | yq '.clusters[0].cluster.server')
  CLUSTER_NAME=$(cat $KUBECONFIG | yq '.clusters[0].name')
  sed -e "s~\$MASTER_URL~$MASTER_URL~" -e "s~\$CLUSTER_NAME~$CLUSTER_NAME~" liqo-values.yaml > liqo-values-cluster-$NAME.yaml
  
  helm install liqo liqo/liqo --namespace liqo --values liqo-values-cluster-$NAME.yaml --create-namespace
  
  until liqoctl status; do
    sleep 5
  done
    
  # add verbose argument (-v=5) for deployments (except liqo-proxy)
  kubectl -n liqo get deployment -l app.kubernetes.io/part-of=liqo -o name | \
    sed 's/.*\///g' | \
    grep -v liqo-proxy | \
    xargs -I {} \
    kubectl -n liqo patch deployment {} \
      --type=json \
      -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "-v=5"}]'

  # add verbose argument (-v=5) for liqo-route daemonset
  kubectl -n liqo patch daemonset liqo-route \
    --type=json \
    -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "-v=5"}]'
  
  unset KUBECONFIG
}
  
install_liqo "france"
install_liqo "france2"
install_liqo "netherlands"
install_liqo "poland"
```

# Peering

Example with `france` and `france2` clusters (`france` is consumer, `france2` is provider):

```shell
export KUBECONFIG=kubeconfigs/kubeconfig-france.yaml

# patch kube-system daemonsets to avoid pods from these daemonsets to start on virtual nodes
# https://github.com/liqotech/liqo/issues/1398: DaemonSet pods scheduled on virtual-nodes and result in OffloadingBackOff
# https://docs.liqo.io/en/v0.8.0/usage/reflection.html#pods-offloading
kubectl -n kube-system get daemonset -o name | \
  sed 's/.*\///g' | \
  xargs -I {} \
    kubectl -n kube-system patch daemonset {} --patch-file daemonset-patch.yaml

CLUSTER_NAME_TO_PEER=france2

PEER_CMD=$(liqoctl generate peer-command --kubeconfig kubeconfigs/kubeconfig-$CLUSTER_NAME_TO_PEER.yaml --only-command)
echo $PEER_CMD
$PEER_CMD --timeout 30s --verbose

liqoctl status peer --verbose

# to unpeer:
# liqoctl unpeer out-of-band $CLUSTER_NAME_TO_PEER

unset PEER_CMD CLUSTER_NAME_TO_PEER KUBECONFIG
```

# Tests

Run a simple hello world (2 pods + 1 service).

```shell
export KUBECONFIG=kubeconfigs/kubeconfig-france.yaml

kubectl create namespace liqo-demo
liqoctl offload namespace liqo-demo # just adds annotation "liqo.io/scheduling-enabled=true"?

# simple hello world (2 pods, 1 per cluster) + service
kubectl apply -f manifests/hello-world.yaml -n liqo-demo

kubectl get pod -n liqo-demo -o wide

# check request goes to either nginx-local or nginx-remote
kubectl run --image=curlimages/curl curl -n default -it --rm --restart=Never -- curl --silent liqo-demo.liqo-demo.svc.cluster.local | grep 'Server'

kubectl delete -f manifests/hello-world.yaml -n liqo-demo

unset KUBECONFIG
```

# Cleanup

To clean everything, just run clusters cleanup, no need to uninstall liqo before :)

## Liqo

```shell
function uninstall_liqo() {
  NAME="$1"
  export KUBECONFIG="kubeconfigs/kubeconfig-$NAME.yaml"
  
  helm uninstall -n liqo liqo && kubectl delete namespace liqo && liqoctl uninstall --purge
  
  unset KUBECONFIG
}

uninstall_liqo "france"
uninstall_liqo "france2"
uninstall_liqo "netherlands"
uninstall_liqo "poland"
```

## Clusters

```shell
function delete_cluster() {
    REGION="$1"
    NAME="$2"
    
    scw k8s cluster delete $(scw k8s cluster list region=$REGION -o json | jq -r --arg name "$NAME" '.[] | select(.name==$name) | .id') region=$REGION with-additional-resources=true
}

delete_cluster "fr-par" "france"
delete_cluster "fr-par" "france2"
delete_cluster "nl-ams" "netherlands"
delete_cluster "pl-waw" "poland"
```

# Issues

It looks like we can't peer clusters living on different regions:

- `fr-par` => `nl-ams`
- `fr-par` => `pl-waw`
- `nl-ams` => `pl-waw`

See https://liqo-io.slack.com/archives/C014Z7F25KP/p1681143824293009.
