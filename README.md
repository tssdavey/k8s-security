# How to secure your k8s clusters in the real world

It's not about if things go wrong - it's about what you do when they do.
# Defence in Depth
## Attack surface
* Your nodes. 
    * If you're running k8s yourself put your nodes in a private subnet. Configure load balancers / firewalls to allow traffic in and out of your subnet.
    * Enforce zero-trust networking between your nodes. See [here](https://kubernetes.io/docs/reference/ports-and-protocols/) for the list of ports you will need to explicitly allow.
    * if you're using a cloud service like AKS, EKS or GKE each configures node access in their own way. If it's not needed completely lock down access to your nodes - AKS for example doesn't allow ssh to nodes by default.
    * If you do need access to your nodes adhere to the principal of least privillage, if the cloud provider is managed your nodes, lock down access using their IAM, for example GKE offers a [roles/container.nodeServiceAccount](https://cloud.google.com/kubernetes-engine/docs/how-to/iam#predefined) role. If you're managing nodes yourself, eg. using EKS self managed nodes lock down access to the autoscaling group. more here***

* Kube-api server. 
    * Lets start with the basics. Turn on authentication. This isn't as silly as you might think though. By default, the kubelet process actually accepts unauthenticated api calls (https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/)- turn this off asap
    * kubeconfig files, you can think of these a bit like database connection strings. They should be treated as secrets and stored in a suitable place such as AWS Secrets Manager or Hashicorp Vault.
    * Take advantage of the fine-grained RBAC that k8d has out of the box.
    * Enable encryption at rest for etcd

* Your containers.
    * Ensure that your containers don't have exploitable vulnerabilities. Use trivy or similar
    * Enforce only images from whitelisted image repos using imagepolicywebhooks https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#imagepolicywebhook
    * Set your imagePullPolicy: Always, this ensures that a fresh image in pulled from the registry every time you run a container rather than using a cached image on the node.
    * Avoid MitM attacks by verifying image hashes, you can do this using ImagePolicyWebhooks or in Pod manifests (https://cloud.google.com/architecture/using-container-image-digests-in-kubernetes-manifests)

* Your data
    * At the end of the day this is probably what any malicious actor is after. If you're storing it in eg. an external database keep the creds in a k8s secret and inject it into pods at runtime. If your workloads are storing data, then be sure to use an encrypted storageclass and again store the creds in a secret injected at runtime, make sure that the data is also encrypted in transit if using eg. an NFS share.

## Within cluster
Here we assume that something has been compromised, how do we stop it from getting worse.
* Pod -> Node
    * Enforce containers running in non-privillaged mode using (securitycontexts)[https://kubernetes.io/docs/tasks/configure-pod-container/security-context/]
    * If your pod doesn't need to, dissalow writing to the root file system using securitycontexts, if it does need to then use apparmor or secomp for fine-grained control over which files can be edited
    * If you need to run untrusted images, use gvisor to run these within their own virtual kernel.


* Pod -> Pod 
    * Network policies. Enforce zero-trust networking between your pods by setting default deny ingress & engress policies. Add explicit allow policies for the specific ports & protocols that your pods need to talk to each other on.
    * Implement Pod -> Pod mtls to negate MitM attacks within your cluster, if you're using a service mesh such as (Istio)[https://istio.io/latest/docs/tasks/security/authentication/mtls-migration/] or (Anthos)[https://cloud.google.com/service-mesh/docs/by-example/mtls] this will come out-of-the-box, otherwise you'll need to implement it yourself.

## Alterting
As security practitioners we want to be proactive rather than reactive, having logs so you can understand why an event happened is good. Having alters so that you can catch an event *as* it's happening is way better.
* Proactive monitoring
    * Always use Falco. It comes with a comprehensive set of rule out the box which will will catch the lateral movement/privillage escalation stages of attacks, and allows for fine-grained control if there's a legitimate reason that a container needs to for example change ownership of files.
    * Use kube-api audit server logs. These can be really powerful if used well, I recommend auditing every change made to a secret or service account - and setting up a simple dashboard that alerts for example every time a kubectl get secret happens - this shouldn't be happening in your prod clusters.
