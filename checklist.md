# How to secure your k8s clusters in the real world

This is a non-exhaustive checklist of things you can do to secure your k8s clusters.

## Attack surface
* Your nodes. 
    * If you're running k8s yourself put your nodes in a private subnet. Configure load balancers / firewalls to allow only specific traffic in and out of your subnet.
    * Enforce zero-trust networking between your nodes. See [here](https://kubernetes.io/docs/reference/ports-and-protocols/) for the list of ports you will need to explicitly allow.
    * if you're using a cloud service like AKS, EKS or GKE each configures node access in their own way. If it's not needed completely lock down access to your nodes - AKS for example doesn't allow ssh to nodes by default.
    * If you do need access to your nodes adhere to the principal of least privillage, if the cloud provider manages your nodes, lock down access using their IAM, for example GKE offers a [roles/container.nodeServiceAccount](https://cloud.google.com/kubernetes-engine/docs/how-to/iam#predefined) role. If you're managing nodes yourself, eg. using EKS self managed nodes lock down access to the autoscaling group. more here***

* Kube-api server. 
    * Lets start with the basics. Turn on authentication. This isn't as silly as you might think though. By default, the kubelet process actually accepts unauthenticated api calls (https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/)- turn this off asap
    * kubeconfig files, you can think of these a bit like database connection strings. They should be treated as secrets and stored in a suitable place such as AWS Secrets Manager or Hashicorp Vault.
    * Use RBAC and the fine-grained control that it offers. Apply this to service accounts and users and follow the principal of least privillage - enable it with the `--authorization-mode=RBAC` in kube-apiserver. For users logins, integrate with a third party auth provider for example using [dex](https://dexidp.io/docs/kubernetes/)
    * Enable encryption at rest for etcd (https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)

* Your containers.
    * Ensure that your containers don't have exploitable vulnerabilities. Use trivy or similar on every container being run in your cluster.
    * Enforce only images from whitelisted image repos using ImagePolicyWebhooks https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#imagepolicywebhook
    * Set your imagePullPolicy: Always, this ensures that a fresh image in pulled from the registry every time you run a container rather than using a cached image on the node.
    * Avoid MitM attacks by verifying image hashes, you can do this using ImagePolicyWebhooks or in Pod manifests (https://cloud.google.com/architecture/using-container-image-digests-in-kubernetes-manifests)

* Your data
    * At the end of the day this is probably what any malicious actor is after. If you're storing data in eg. an external database keep the creds in a k8s secret and inject it into pods at runtime. If your workloads are storing data, then be sure to use an encrypted storageclass and again store the creds in a secret injected at runtime, make sure that the data is also encrypted in transit if using eg. an NFS share.
    * Rotate encryptions keys, kubernetes rotates kubelet and server certs automatically - but crucially the secrets encrypting your etcd data are *not* automatically rotated - you'll need to do this manually yourself - or [let you cloud provider do it for you](https://learn.microsoft.com/en-us/azure/aks/use-kms-etcd-encryption)!
    * Seperate your etcd cluster

## Within cluster
How do we stop a malicious actor from traversing within our cluster.
* Pod to Node
    * Enforce containers running in non-privillaged mode and disallow writing to the file system, and run your containers as non-root users using [securitycontexts](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
    * If your pod needs to write to root filesystem, or make syscalls then use [apparmor](https://gitlab.com/apparmor/apparmor/-/wikis/Documentation) or [seccomp](https://kubernetes.io/docs/tutorials/security/seccomp/) to allow only what the pod requires and no more.
    * If you need to run untrusted images, use [gvisor](https://gvisor.dev/docs/) to run these within their own independant kernel. 

* Pod to Pod
    * Enforce zero-trust networking between your pods by setting default deny ingress & engress policies. Add explicit allow policies for the specific ports & protocols that your pods need to talk to each other on.
    * Implement Pod -> Pod mtls to negate MitM attacks within your cluster, if you're using a service mesh such as [Istio](https://istio.io/latest/docs/tasks/security/authentication/mtls-migration/) or [Anthos](https://cloud.google.com/service-mesh/docs/by-example/mtls) this will come out-of-the-box, otherwise you'll need to implement it yourself.

## Alterting
As security practitioners we want to be proactive rather than reactive, having logs so you can understand why an event happened is good. Having alters so that you can catch an event *as* it's happening is way better.
* Proactive monitoring
    * Always use [Falco](https://falco.org/). It comes with a comprehensive set of rule out the box which will will catch the lateral movement/privillage escalation stages of attacks, and allows for fine-grained control if there's a legitimate reason that a container needs to for example change ownership of files.
    * Use audit policies to capture logs from the kube-api server. These can be really powerful if used well, I recommend auditing every change made to a secret or service account - and setting up a simple dashboard that alerts every time one of these events happens.

Recommended further reading:
* https://kubernetes.io/blog/2018/07/18/11-ways-not-to-get-hacked/
