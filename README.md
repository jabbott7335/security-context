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

## Security Context

**OpenShift** and Kubernetes provide us a way manage privilege and access control for a container or pod that we wish to run within our Kubernetes cluster.  What initially comes to mind is that we can control the user and group IDs, but there are many other aspects that are also controlled such as filesystem access, SELinux, system call filtering, privilege escalation control and the list goes on. If you are curious you have likely already seen that if you log into a typical pod a create a file, that file will be tagged with a strange looking UID such as: `1000590000`.  This user ID comes from a OpenShift project level setting `sa.scc.uid-range` that can be seen using the `oc describe` against your project.  Managing the user ID used for all of the workload within a project not only provides a basic foundation for a multi layer security strategy, it also helps with such activities as persistent storage management (an article unto itself).

When we set **security context** for certain aspects of the pod / container we are defining how the workload will run and interact within our environment.  And example pod definition for discussion is shown:

```
...
kind: Pod
metadata:
  name: your-pod-here
spec:
  securityContext:
    runAsUser: 1234
    runAsGroup: 5678
    fsGroup: 9877
...
```
The security context definition for the pod dictates that each of the processes running within the pod will run via the user specified by 'runAsUser' as '1234', the group will be `5678` and so forth.  If the `runAsGroup` is omitted the primary group of `root(0)` is used by default (not in itself a security issue).  Both **pod level** and **container level** security context are available.  However, when the same values are set in each, the container level context takes priority.  The security context settings give us a framework for configuring these properties, but more is required to safeguard our environment and prevent accidental lapses in configuration.

## Security Context Constraints

OpenShift prevents us from accidentally granting privileged containers by applying **security context constraints** to the workload running within our project / namespace.  By default a namespace is given `restricted` as an **scc** policy.  This restricted policy will resemble the following:

```
Name:						restricted
Priority:					<none>
Access:
  Users:					<none>
  Groups:					system:authenticated
Settings:
  Allow Privileged:				false
  Allow Privilege Escalation:			true
  Default Add Capabilities:			<none>
  Required Drop Capabilities:			KILL,MKNOD,SETUID,SETGID
  Allowed Capabilities:				<none>
  Allowed Seccomp Profiles:			<none>
  Allowed Volume Types:				configMap,downwardAPI,emptyDir,persistentVolumeClaim,projected,secret
  Allowed Flexvolumes:				<all>
  Allowed Unsafe Sysctls:			<none>
  Forbidden Sysctls:				<none>
  Allow Host Network:				false
  Allow Host Ports:				false
  Allow Host PID:				false
  Allow Host IPC:				false
  Read Only Root Filesystem:			false
  Run As User Strategy: MustRunAsRange
    UID:					<none>
    UID Range Min:				<none>
    UID Range Max:				<none>
  SELinux Context Strategy: MustRunAs
    User:					<none>
    Role:					<none>
    Type:					<none>
    Level:					<none>
  FSGroup Strategy: MustRunAs
    Ranges:					<none>
  Supplemental Groups Strategy: RunAsAny
    Ranges:					<none>
```

Cluster administrators are able to manage these policies.  The **security context constraint** ensures that nothing can run within the project that does not meet the requirements specified.  Thus, it will stop a developer from unwittingly violating a your policy that may ban containers from running as a privileged user, regardless of how they were built.   
