"Everything fails, all the time"
And as security professionals we need to plan for stuff to go wrong. That's why implement layers of defence - or defense in depth.

There is no silver bullet for kubernetes, but it's a well designed system, and is designed with this in mind.

In general terms there are two ways that malicious actors can get into your clusters.
* The front door, getting access to the kube-api
* The back door, establishing a foothold in a pod or node and traversing to something important, such as a secret or persistant storage.

So if I implement this checklist my clusters are secure, right?
