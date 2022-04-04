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

for i in submariner-operator submariner-route-agent submariner-globalnet submariner-gateway submariner-networkplugin-syncer submariner-operator-index lighthouse-coredns lighthouse-agent
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

Install Submariner:

```bash
SUBMARINER_VER=$(subctl version | cut -d: -f2 | cut -dv -f2)

for i in dc1 dc2 dc3
do
labctx ${i}
  subctl cloud prepare generic --kubeconfig ${KUBE_INIT_CONFIG} --gateways 3
done

labctx dc1
subctl deploy-broker --kubeconfig ${KUBE_INIT_CONFIG} --repository ${PROXY_REGISTRY}/submariner --version ${SUBMARINER_VER}
mv broker-info.subm ${OKD_LAB_PATH}/lab-config/broker-info.subm

for i in dc1 dc2 dc3
do
  labctx ${i}
  subctl join ${OKD_LAB_PATH}/lab-config/broker-info.subm --kubeconfig ${KUBE_INIT_CONFIG} --repository ${PROXY_REGISTRY}/submariner --version ${SUBMARINER_VER} --load-balancer
done

for i in dc1 dc2 dc3
do
  labctx ${i}
  subctl uninstall --kubeconfig ${KUBE_INIT_CONFIG} --yes
  subctl cloud cleanup generic --kubeconfig ${KUBE_INIT_CONFIG}
done

subctl show all --kubeconfig ${KUBE_INIT_CONFIG}

```
