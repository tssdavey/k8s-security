# How to secure your k8s clusters in the real world

# Defence in Depth
## Attack surface
* Your nodes. 
    * Where are they and who has access to them. If you're running nodes yourself think about which segments they're deployed into and prefer private        subnets.
    * Enforce zero-trust networking between your nodes. See (https://kubernetes.io/docs/reference/ports-and-protocols/) for the list of ports you will need to explicitly allow.
    * if you're using a cloud service like AKS, EKS or GKE each configures node access in their own way, for example AKS doesn't even have ssh to nodes enabled by default - think about which user & service accounts have access to your nodes, and always ahere to the principal of least privilage. For example GKE offers fine-grained RBAC control, with the roles/container.nodeServiceAccount role.

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
    * Run your containers unrpivillaged
    * Don't allow writing to root filesystem
    * Use apparmor or secomp for fine-grained control
    * Use gvisor if you don't trust images


* Pod -> Pod 
    * Network policies. Enforce zero-trust networking between your pods by setting default deny ingress & engree policies. Add explicit allow policies for the specific ports & protocols that your pods need to talk to each other on.
    * Implement Pod -> Pod mtls to negate MitM attacks within your cluster. 

## Alterting
As security practitioners we want to be proactive rather than reactive, having logs so you can understand why an event happened is good. Having alters so that you can catch an event *as* it's happening is way better.
* Proactive monitoring
    * Use Falco. It comes with a comprehensive set of tools out the box which will will catch the lateral movement/privillage escalation stages of attacks, and allows for fine-grained control if there's a legitimate reason that a container needs to for example change ownership of files.
    * Use kube-api audit server logs. These can be really powerful if used well, I recommend auditing every change made to a secret or service account - and setting up a simple dashboard that alerts for example every time a kubectl get secret happens - this shouldn't be happening in your prod clusters.
