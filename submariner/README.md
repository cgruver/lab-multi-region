### Install

Install subctl

```bash
curl -Ls https://get.submariner.io | DESTDIR=${OKD_LAB_PATH}/bin bash
```

Mirror Images

```bash
labctx

podman machine start
podman login ${LOCAL_REGISTRY}

SUBMARINER_VER=$(subctl version | cut -d: -f2 | cut -dv -f2)

for i in submariner-operator submariner-route-agent submariner-globalnet submariner-gateway submariner-networkplugin-syncer submariner-operator-index lighthouse-coredns lighthouse-agent nettest
do
  podman pull quay.io/submariner/${i}:${SUBMARINER_VER}
  podman tag quay.io/submariner/${i}:${SUBMARINER_VER} ${LOCAL_REGISTRY}/submariner/${i}:${SUBMARINER_VER}
  podman push ${LOCAL_REGISTRY}/submariner/${i}:${SUBMARINER_VER} --tls-verify=false
done
```

Prepare Kubeconfig

```bash
export KUBECONFIG=${OKD_LAB_PATH}/lab-config/cluster-kubeconfig
touch ${KUBECONFIG}

labcli --login
```

Label Nodes:

```bash
for region in 1 2 3
do
  labenv -k -d=dc${region}
  for i in 0 1 2
  do
    oc label nodes okd4-region-0${region}-worker-${i}.dc${region}.${LAB_DOMAIN} submariner.io/gateway=true --overwrite
    oc annotate nodes okd4-region-0${region}-worker-${i}.dc${region}.${LAB_DOMAIN} gateway.submariner.io/public-ip=dns:okd4-region-0${region}-worker-${i}.dc${region}.${LAB_DOMAIN} --overwrite
  done
done

```

Install Submariner:

```bash
SUBMARINER_VER=$(subctl version | cut -d: -f2 | cut -dv -f2)

for i in dc1 dc2 dc3
do
labctx ${i}
  subctl cloud prepare generic --kubeconfig ${KUBE_INIT_CONFIG} --gateways 3
done

labctx dc1
subctl deploy-broker --kubeconfig ${KUBE_INIT_CONFIG} --repository ${PROXY_REGISTRY}/submariner --version ${SUBMARINER_VER} --globalnet=false
mv broker-info.subm ${OKD_LAB_PATH}/lab-config/broker-info.subm

for i in dc1 dc2 dc3
do
  labctx ${i}
  subctl join ${OKD_LAB_PATH}/lab-config/broker-info.subm --kubeconfig ${KUBE_INIT_CONFIG} --repository ${PROXY_REGISTRY}/submariner --version ${SUBMARINER_VER} --natt=false --cable-driver libreswan --globalnet=false
done

# --natt=false --preferred-server=true --cable-driver libreswan --globalnet=false --force-udp-encaps

for i in dc1 dc2 dc3
do
  labctx ${i}
  subctl uninstall --kubeconfig ${KUBE_INIT_CONFIG} --yes
  subctl cloud cleanup generic --kubeconfig ${KUBE_INIT_CONFIG}
done

subctl show all --kubeconfig ${KUBE_INIT_CONFIG}
subctl diagnose all --kubeconfig ${KUBE_INIT_CONFIG}
subctl diagnose firewall inter-cluster ${OKD_LAB_PATH}/lab-config/okd4-region-01-dc1-${LAB_DOMAIN}/kubeconfig ${OKD_LAB_PATH}/lab-config/okd4-region-02-dc2-${LAB_DOMAIN}/kubeconfig

oc patch pod ${POD} --type json -p '[{"op": "replace", "path": "/spec/containers/0/image", "value": "${PROXY_REGISTRY}/submariner/nettest:0.12.0"}]'

quay.io/submariner/nettest:devel

```
