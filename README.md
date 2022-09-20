# how to secure your k8s clusters in the real world

# Defence in Depth
## Attack surface
* Your nodes. 
    * Where are they and who has access to them. If you're running nodes yourself think about which segments they're deployed into and prefer private        subnets.
    * Enforce zero-trust networking between your nodes. See (https://kubernetes.io/docs/reference/ports-and-protocols/) for the list of ports you will need to explicitly allow.
    * if you're using a cloud service like AKS, EKS or GKE each configures node access in their own way, for example AKS doesn't even have ssh to nodes enabled by default - think about which user & service accounts have access to your nodes, and always ahere to the principal of least privilage. For example GKE offers fine-grained RBAC control, with the roles/container.nodeServiceAccount role.

* Kube-api server. 
    * Lets start with the basics. Turn on authentication. This isn't as silly as you might think though. By default, the kubelet process actually accepts unauthenticated api calls (https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/)- turn this off asap
    * kubeconfig files, these are your crown jewels. So don't leave them on bastion boxes, they should be treated as secrets and stored in a suitable place such as AWS Secrets Manager or Hashicorp Vault.
    * Take advantage of the fine-grained RBAC that k8d has out of the box.

* Your containers, we'll only cover how to stop someone 
    * Your supply chain, use trivy.
    * 

* Your data
    * At the end of the day this is probably what any malicious actor is after. If you're storing it in eg. an external database keep the creds in a k8s secret and inject it into pods at runtime. If your workloads are storing data, then be sure to use an encrypted storageclass and again store the creds in a secret injected at runtime, make sure that the data is also encrypted in transit if using eg. an NFS share.

## Out of pod
* Pod -> node
* Pod -> pod
* Pod -> api server
## Within cluster
* Network policies
* 

## Monitoring
* Reactive monitoring aka auditing
* Proactive monitoring
- 
