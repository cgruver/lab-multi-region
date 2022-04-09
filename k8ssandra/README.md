## K8ssandra Install


### Intall Cert Manager:

```bash
mkdir -p ${OKD_LAB_PATH}/k8ssandra-work-dir/cert-manager-install

CERT_MGR_VER=v1.8.0

podman machine start

podman login -u openshift-mirror ${LOCAL_REGISTRY}

for image in cert-manager-cainjector cert-manager-controller cert-manager-webhook
do
  podman pull quay.io/jetstack/${image}:${CERT_MGR_VER}
  podman tag quay.io/jetstack/${image}:${CERT_MGR_VER} ${LOCAL_REGISTRY}/jetstack/${image}:${CERT_MGR_VER}
  podman push --tls-verify=false ${LOCAL_REGISTRY}/jetstack/${image}:${CERT_MGR_VER}
done

wget -O ${OKD_LAB_PATH}/k8ssandra-work-dir/cert-manager-install/cert-manager.yaml https://github.com/jetstack/cert-manager/releases/download/${CERT_MGR_VER}/cert-manager.yaml

cat <<EOF > ${OKD_LAB_PATH}/k8ssandra-work-dir/cert-manager-install/kustomization.yaml
resources:
- ${OKD_LAB_PATH}/k8ssandra-work-dir/cert-manager-install/cert-manager.yaml
images:
- name: quay.io/jetstack/cert-manager-cainjector
  newTag: ${CERT_MGR_VER}
  newName: ${LOCAL_REGISTRY}/jetstack/cert-manager-cainjector
- name: quay.io/jetstack/cert-manager-controller
  newTag: ${CERT_MGR_VER}
  newName: ${LOCAL_REGISTRY}/jetstack/cert-manager-controller
- name: quay.io/jetstack/cert-manager-webhook
  newTag: ${CERT_MGR_VER}
  newName: ${LOCAL_REGISTRY}/jetstack/cert-manager-webhook
EOF

kustomize build ${OKD_LAB_PATH}/k8ssandra-work-dir/cert-manager-install > ${OKD_LAB_PATH}/k8ssandra-work-dir/cert-manager-install.yaml

for cluster in dc1 dc2 dc3
do
  labenv -k -d=${cluster}
  oc create -f ${OKD_LAB_PATH}/k8ssandra-work-dir/cert-manager-install.yaml
done
```

### Install K8ssandra Operator

```bash
CASS_VER=v1.10.1
K8SSANDRA_VER=v1.0.1

mkdir -p ${OKD_LAB_PATH}/k8ssandra-work-dir/control-plane
mkdir -p ${OKD_LAB_PATH}/k8ssandra-work-dir/data-plane

podman pull docker.io/k8ssandra/cass-operator:${CASS_VER}
podman pull docker.io/k8ssandra/k8ssandra-operator:${K8SSANDRA_VER}


podman tag docker.io/k8ssandra/cass-operator:${CASS_VER} ${LOCAL_REGISTRY}/k8ssandra/cass-operator:${CASS_VER}
podman tag docker.io/k8ssandra/k8ssandra-operator:${K8SSANDRA_VER} ${LOCAL_REGISTRY}/k8ssandra/k8ssandra-operator:${K8SSANDRA_VER}


podman push --tls-verify=false ${LOCAL_REGISTRY}/k8ssandra/cass-operator:${CASS_VER}
podman push --tls-verify=false ${LOCAL_REGISTRY}/k8ssandra/k8ssandra-operator:${K8SSANDRA_VER}

cat <<EOF > ${OKD_LAB_PATH}/k8ssandra-work-dir/control-plane/kustomization.yaml
namespace: k8ssandra-operator
resources:
- github.com/k8ssandra/k8ssandra-operator/config/deployments/control-plane?ref=v1.0.1
images:
- name: k8ssandra/k8ssandra-operator
  newTag: ${K8SSANDRA_VER}
  newName: ${LOCAL_REGISTRY}/k8ssandra/k8ssandra-operator
- name: k8ssandra/cass-operator
  newTag: ${CASS_VER}
  newName: ${LOCAL_REGISTRY}/k8ssandra/cass-operator
EOF

cat <<EOF > ${OKD_LAB_PATH}/k8ssandra-work-dir/data-plane/kustomization.yaml
namespace: k8ssandra-operator
resources:
- github.com/k8ssandra/k8ssandra-operator/config/deployments/data-plane?ref=v1.0.1
images:
- name: k8ssandra/k8ssandra-operator
  newTag: ${K8SSANDRA_VER}
  newName: ${LOCAL_REGISTRY}/k8ssandra/k8ssandra-operator
- name: k8ssandra/cass-operator
  newTag: ${CASS_VER}
  newName: ${LOCAL_REGISTRY}/k8ssandra/cass-operator
EOF

kustomize build ${OKD_LAB_PATH}/k8ssandra-work-dir/control-plane > ${OKD_LAB_PATH}/k8ssandra-work-dir/k8ssandra-control-plane.yaml
kustomize build ${OKD_LAB_PATH}/k8ssandra-work-dir/data-plane > ${OKD_LAB_PATH}/k8ssandra-work-dir/k8ssandra-data-plane.yaml

labenv -k -d=dc1
oc create -f ${OKD_LAB_PATH}/k8ssandra-work-dir/k8ssandra-control-plane.yaml
labenv -k -d=dc2
oc create -f ${OKD_LAB_PATH}/k8ssandra-work-dir/k8ssandra-data-plane.yaml
labenv -k -d=dc3
oc create -f ${OKD_LAB_PATH}/k8ssandra-work-dir/k8ssandra-data-plane.yaml

```

## Create ClientConfigs

```bash
mkdir -p ${OKD_LAB_PATH}/k8ssandra-work-dir/kubeconfig
CONTROL_PLANE_KUBE=$(labcli -d=dc1 --kube | grep -v domain:)
for i in dc2 dc3
do
  REGION_KUBE=$(labcli -d=${i} --kube | grep -v domain:)
  sa_secret=$(oc --kubeconfig ${REGION_KUBE} -n k8ssandra-operator get serviceaccount k8ssandra-operator -o jsonpath='{.secrets[0].name}')
  sa_token=$(oc --kubeconfig ${REGION_KUBE} -n k8ssandra-operator get secret $sa_secret -o jsonpath='{.data.token}' | base64 -d)
  ca_cert=$(oc --kubeconfig ${REGION_KUBE} -n k8ssandra-operator get secret $sa_secret -o jsonpath="{.data['ca\.crt']}")
  cluster=$(oc --kubeconfig ${REGION_KUBE} config view -o jsonpath="{.contexts[0].context.cluster}")
  cluster_addr=$(oc --kubeconfig ${REGION_KUBE} config view -o jsonpath="{.clusters[0].cluster.server}")

SECRET_FILE=${OKD_LAB_PATH}/k8ssandra-work-dir/kubeconfig/kubeconfig

cat << EOF > ${SECRET_FILE}
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: ${ca_cert}
    server: ${cluster_addr}
  name: ${cluster}
contexts:
- context:
    cluster: ${cluster}
    user: k8ssandra-operator
  name: ${cluster}-k8ssandra
current-context: ${cluster}-k8ssandra
kind: Config
preferences: {}
users:
- name: k8ssandra-operator
  user:
    token: ${sa_token}
EOF

oc --kubeconfig ${CONTROL_PLANE_KUBE} -n k8ssandra-operator create secret generic ${cluster}-config --from-file="${SECRET_FILE}"

cat << EOF | oc --kubeconfig ${CONTROL_PLANE_KUBE} apply -n k8ssandra-operator -f -
apiVersion: config.k8ssandra.io/v1beta1
kind: ClientConfig
metadata:
  name: ${cluster}-k8ssandra
spec:
  contextName: ${cluster}-k8ssandra
  kubeConfigSecret:
    name: ${cluster}-config
EOF

done

oc --kubeconfig ${CONTROL_PLANE_KUBE} -n k8ssandra-operator scale deployment k8ssandra-operator --replicas=0
sleep 3
oc --kubeconfig ${CONTROL_PLANE_KUBE} -n k8ssandra-operator scale deployment k8ssandra-operator --replicas=1
```

## Deploy Cluster

```bash
for i in dc1 dc2 dc3
do
  REGION_KUBE=$(labcli -d=${i} --kube | grep -v domain:)
  oc --kubeconfig ${CONTROL_PLANE_KUBE} adm policy add-scc-to-user privileged -z default -n k8ssandra-operator
done

labenv -k -d=dc1

cat <<EOF | oc -n k8ssandra-operator apply -f -
apiVersion: k8ssandra.io/v1alpha1
kind: K8ssandraCluster
metadata:
  name: k8ssandra-cluster
spec:
  cassandra:
    serverVersion: "4.0.1"
    storageConfig:
      cassandraDataVolumeClaimSpec:
        storageClassName: rook-ceph-block
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 20Gi
    config:
      jvmOptions:
        heapSize: 512M
    networking:
      hostNetwork: false 
    datacenters:
      - metadata:
          name: dc1
        size: 3
        stargate:
          size: 1
          heapSize: 256M
      - metadata:
          name: dc2
        k8sContext: okd4-region-02-k8ssandra
        size: 3
        stargate:
          size: 1
          heapSize: 256M
      - metadata:
          name: dc3
        k8sContext: okd4-region-03-k8ssandra
        size: 3
        stargate:
          size: 1
          heapSize: 256M
EOF
```

docker.io/library/busybox:1.34.1


## Delete Cluster

```bash
labenv -k -d=dc1
oc delete K8ssandraCluster k8ssandra-cluster -n k8ssandra-operator
```

## Delete ClientConfigs

```bash
CONTROL_PLANE_KUBE=$(labcli -d=dc1 --kube | grep -v domain:)
for i in dc1 dc2 dc3
do
  REGION_KUBE=$(labcli -d=${i} --kube | grep -v domain:)
  cluster=$(oc --kubeconfig ${REGION_KUBE} config view -o jsonpath="{.contexts[0].context.cluster}")
  oc --kubeconfig ${CONTROL_PLANE_KUBE} -n k8ssandra-operator delete secret ${cluster}-config
  oc --kubeconfig ${CONTROL_PLANE_KUBE} -n k8ssandra-operator delete ClientConfig ${cluster}-k8ssandra
done
oc --kubeconfig ${CONTROL_PLANE_KUBE} -n k8ssandra-operator scale deployment k8ssandra-operator --replicas=0
sleep 3
oc --kubeconfig ${CONTROL_PLANE_KUBE} -n k8ssandra-operator scale deployment k8ssandra-operator --replicas=1
```

## Delete K8ssandra

```bash
labenv -k -d=dc1
oc delete -f ${OKD_LAB_PATH}/k8ssandra-work-dir/k8ssandra-control-plane.yaml
labenv -k -d=dc2
oc delete -f ${OKD_LAB_PATH}/k8ssandra-work-dir/k8ssandra-data-plane.yaml
labenv -k -d=dc3
oc delete -f ${OKD_LAB_PATH}/k8ssandra-work-dir/k8ssandra-data-plane.yaml
```
