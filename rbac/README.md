# RBAC

RBAC - Role based access control - is comprised of a few elements that work together.

Using roles you can provide access to your cluster to users, or to applications.

Setting up these layers can be quite tricky, so I've tried to create a simplistic walkthrough on these.

The trickiest part with the manual authentication process is the lack of feedback when one step goes wrong. If you make a mistake on one of the certificates, the system won't necessarily tell you this is where the problem lies.

# User access via kubectl

These steps have helped me:

Giving an external kubectl access to your cluster.

You need these layers:

    - set up certificates
    - set up a kubeconfig which uses the ca.crt and your certificates to authenticate



## Set up certificates


First create the certificates, you can follow these steps:

Same steps for admin user:
1. Create a private key
2. Create a Certificate Signing Request
3. Then sign the certificate

Steps:

Decide on a key name:

- example: "normal"
- Your desired state will be these 3 files:
    - normal.key -> the key
    - normal.csr -> the certificate signing request
    - normal.crt -> the certificate you generate

### 1

Create key

	$ openssl genrsa -out normal.key 2048

### 2

Create csr (certificate signing request)

	$ openssl req -new -key normal.key -subj "/CN=user:normal/O=customeruser" -out normal.csr

// TODO: add more info on these:

	-subj
		/CN
		/O
	 

	If you get an error like this:

	Can't load /root/.rnd into RNG
	139802307129792:error:2406F079:random number generator:RAND_load_file:Cannot 
	open file:../crypto/rand/randfile.c:88:Filename=/root/.rnd

You might need to run this to solve the above error: 

	$ touch ~/.rnd
	
### 3

Sign against the ca.crt and ca.key using the csr you created, this is what will ultimately give you access to the cluster

	$ openssl x509 -req -in normal.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial  -out normal.crt

Note: Don't forget this part -CAcreateserial


## Set up the kubeconfig file

Creating the kubeconfig requires these elements:

    - clusters
    - users
    - contexts 

By setting a context, you create a link between a cluster and a user

Example:

kubernetes **cluster** & kubernetes-admin **user** = kubernetes-admin@kubernetes **context**


Keeping this in mind the buildings blocks for your kubeconfig will be:

    apiVersion: v1
    kind: Config
    clusters:
    -
    users:
    - 
    contexts:
    -


But that's not enough to properly authenticate someone, you still need to generate certs, which has been mapped out above, follow those steps and make sure you create the *.crt and *.key files you want to use to authenticate. 

You can either reference the files as files (but this would only be useful if you are already on the server):


    apiVersion: v1
    kind: Config
    clusters:
    - cluster
        certificate-authority: /file/path/to/ca.crt
    users:
    - user:
        client-certificate: /file/path/to/custom.crt
        client-key: /file/path/to/custom.key
    contexts:
    -

Now you might not want to reference the files directly like that. You might want to use the kubeconfig outside of the cluster so you can have access to the cluster from an external source, like your own personal computer.

You can create a kubeconfig file and use a base64 encoded copy of the *.crt files and the *.key file.

When you do this though, you need to indicate in the yaml that you are using data, and not files:

    apiVersion: v1
    kind: Config
    clusters:
    - cluster
        certificate-authority-data: base64 encoded **ca.crt** data
    users:
    - user:
        client-certificate-data: base64 encoded **custom.crt** data
        client-key-data: base64 encoded **custom.key** data
    contexts:
    -

A useful command to base64 encode the files:

    $ cat /etc/kubernetes/pki/ca.crt | base6 | tr -d "\n"
    $ cat ~/normal.crt | base6 | tr -d "\n"
    $ cat ~/normal.key | base6 | tr -d "\n"

Note: This end part: | tr -d "\n" is to remove newlines



Then remember to fill in the other information you would need for you kube config file:

Replace xxx with your data and <server-ip> with your cluster's ip

    apiVersion: v1
    kind: Config
    clusters:
    - cluster:
        certificate-authority-data: xxx
        server: https://<server-ip>:6443
    name: default-cluster
    contexts:
    - context:
        cluster: default-cluster
        namespace: default
        user: normal
    name: default-context
    current-context: default-context
    preferences: {}
    users:
    - name: normal
    user:
        client-key-data: xxx
        client-certificate-data: xxx

There are many ways you can create a kubeconfig file, even more than what I've indicated here. You can experiment with configurations. This way just helped me get to know the kubeconfig file. 

If you save that file in the location:

    ~/.kube/config

You're not done yet though. 

You have a user set up. And this user can now authenticate, but we still need to set permissions allowing the user to retrieve information or alter the cluster via kubectl.

To give a user permissions we have to create a role and a rolebinding. 

Just a quick reminder, a role and a rolebinding are namespaced. If you want broader access that is not namespaces, you can use clusterroles and clusterrolebindings.

## Roles

Setting up a simple role to start off with, just giving permission to list pods:

Example from: 

File location: rbac/roles/example-1.yaml

    # Example of User Role Binding
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
    name: read-pods
    namespace: default
    subjects:
    - kind: User
    name: normal # Name is case sensitive
    apiGroup: rbac.authorization.k8s.io
    roleRef:
    kind: Role #this must be Role or ClusterRole
    name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
    apiGroup: rbac.authorization.k8s.io



# Application access via ServiceAccount

