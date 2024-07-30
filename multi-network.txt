apiVersion: k8s.cni.cncf.io/v1beta1
kind: MultiNetworkPolicy
metadata:
  name: deny-by-default
  annotations:
    k8s.v1.cni.cncf.io/policy-for: vmexamples/vlan0
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress: []
