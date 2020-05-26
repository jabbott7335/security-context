# Understanding Privileges and Access Controls in OpenShift

Running containers as root in **Docker** has long been considered to be bad.  Traditionally a running container would be given the same access to the host machine it runs on.  Thus a container built using a priveleged user could be a priveleged user for its hosting system.  This is fine if the container is trusted, but it must be assumed that a hacker could gain control of or run a process within a container thus gaining a beach-head for malicious or nefarious activity.

## Why Containers Might Require Privilege

Developers strive to produce applications and code that is safe and secure.  One on the central tenets on the Container Coder's Manifesto (if there were to be such a thing) is "though shall not run containers as root".  There are however reasons why containers may require privileged access internally.  Internally there are some activities restricted to privileged users. Some of these could include:

- Binding ports less than 1024 (remember containers within a pod communicate on a local network)
- Volume mounting complexities
- Accessing certain services

An argument could be made against any reason privileged users are used within a container, but many of the arguments come with a cost.  Consider briefy a project such as Istio that has a central tenet of providing an extremely secure method of running microservices within a Kubernetes environment.  A quick Google search will show you a history of arguments and discussions within the community over "why or why not" Istio has containers running as root within its project.  Needless to say its not as easy as just rebuilding a container with a quick configuration change.

You may notice when starting certain pods that sometimes during the startup you will see **init** containers pop up and then go away.  These containers are by definition short lived and can perform a myriad of special tasks that help more complex workloads start.  

- init containers can perform tasks using utilities that we would not typically want to have included in longer running workloads (these can include secure activities)
- They can help seperate duties and improve pod / container startup time
- Tasks such as using filesystems and secrets can be performed with different access can thus shelter privilege from application containers
- init containers can control how the application containers instantiate checking for preconditions and the like and thus orchestrate complex application startup


