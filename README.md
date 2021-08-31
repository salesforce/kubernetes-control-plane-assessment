# Kubernetes Control Plane Assessment

This repository contains the [report of a security assessment of the Kubernetes control plane](https://github.com/salesforce/kubernetes-control-plane-assessment/raw/master/Atredis%20Partners-Salesforce-Kubernetes%20Control%20Plane%20Vulnerability%20Assessment-v1.1.pdf) as well as code and configuration samples. The overall idea of this repository is described in the [Salesforce Engineering Blog Post](https://engineering.salesforce.com/looking-at-the-kubernetes-control-plane-for-multi-tenancy-88914cd7aa89).

# Taints & Tolerations

[Taints](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) apply certain labels to nodes for which [tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) can be configured per pod. To enforce those tolerations (and no conflicting ones) on a namespace basis, the admission controller [PodTolerationRestriction](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#podtolerationrestriction) is used.

Create a namespace with toleration enforcements:
```
kubectl apply -f taints-tolerations/namespace.yaml
```

Deploy a pod to that namespace and observe scheduling decisions:
```
kubectl apply -f taints-tolerations/pod-enforced.yaml
kubectl describe pod echo -n enforced
```

# NodeSelector

[`nodeSelector`](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector) allows pods to include node constraints with the pod sepc. To enforce those constraints on a namespace basis, the admission controller [PodNodeSelector](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#podnodeselector) is used.

Enable `PodNodeSelector` for `kube-apiserver` via its manifest:

```
mkdir -p /etc/kubernetes/podnodeselector/
cat > /etc/kubernetes/podnodeselector/admission-configuration.yaml <<EOF
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodNodeSelector
  path: /adm-control/nodeselector.yaml
EOF

cat > /etc/kubernetes/podnodeselector/podnodeselector-configuration.yaml <<EOF
podNodeSelectorPluginConfig:
  clusterDefaultNodeSelector: "env=development"
  production: "env=production"
  development:
  noannotation: "env=notpresent"
EOF

# Add admission controller config to kube-apiserver manifest. Depending on your K8s deployment, you may have to apply this in a different way. 
vim /etc/kubernetes/manifests/kube-apiserver.yaml

# Add plugin and configuration
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint: 10.0.3.4:6443
[...]
    - --enable-admission-plugins=NodeRestriction,PodNodeSelector
    - --admission-control-config-file=/etc/kubernetes/podnodeselector/admission-configuration.yaml
[...]
    volumeMounts:
    - mountPath: /adm-control
      name: adm-control
      readOnly: true
[...]
  volumes:
  - hostPath:
      path: /etc/kubernetes/podnodeselector/
      type: DirectoryOrCreate
    name: adm-control
```

After the `kube-apiserver` is restarted, you can apply labels and create namespaces including annotations:
```
kubectl label node worker1 env=production
kubectl label node worker2 env=development
kubectl create namespace production
kubectl create namespace development
kubectl create namespace noannotation
kubectl patch namespace production -p '{"metadata":{"annotations":{"scheduler.alpha.kubernetes.io/node-selector":"env=production"}}}'
```

After that you can deply various pods and observer where they are deployed or rejected:
```
kubectl apply -f nodeselector/pod-dev.yaml
kubectl apply -f nodeselector/pod-prod.yaml
kubectl apply -f nodeselector/pod-conflict.yaml
```