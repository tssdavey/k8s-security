"Everything fails, all the time"
As security professionals we need to plan for stuff to go wrong. 
We cannot follow a set of steps and then be certain that our systems are secure, instead we have to adhere to principles in every facet of our work - in partcular for this blog - defence in depth, and least privilege.

Kubernetes was designed knowing this, and provides a user with the capability to implement these principles at various stages. 
This blog provides an outline of 20 different ways that kuberenets provides

In general terms there are two ways that malicious actors can get into your clusters.
* The front door, getting access to the kube-api
* The back door, establishing a foothold in a pod or node and traversing to something important, such as a secret or persistant storage.

So if I implement this checklist my clusters are secure, right?
