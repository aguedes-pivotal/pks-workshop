# How-to install and configure PKS to use LDAP with RBAC on the PKS API and Kubernetes clusters

## Setup Ldap Server:

You'll need a VM with Docker installed to run the LDAP Server - this could be a new compute insntace within GCP or another VM on your vSphere environment. Bare in mind that the PKS UAA VM will need access to this VM on port 1636 (TLS) or 1386 (no TLS).

```
cd ldap-server
docker build -t ldap-server .
docker run -d -p 1389:389 -p 1636:636 ldap-server slapd  -h "ldap://0.0.0.0:389  ldaps://0.0.0.0:636" -d 3 -f /ldap/slapd.conf
```

You should have now an LDAP Server running on port 1389 (no TLS) or 1636 (TLS enabled).
Now import the sample data:

`ldapadd -v -x -D "cn=admin,dc=example,dc=com" -w mypassword  -H ldap://localhost:1389 -f import.ldif`

After importing the data you'll have the following groups and it's members (the password for all users is `mypassword`):

```
superadmins: user1
readonly: user2
ops: user3, user4
appdev: user5
```

## Configuring PKS to integrate with LDAP:

Now make sure you setup PKS to use LDAP by going to OpsMan > PKS Tile > UAA and selecting the authentication mechanism to be LDAP Server:

```
Server URL: ldaps://IP_ADDRESS:1636
LDAP Username: cn=admin,dc=example,dc=com
LDAP Password: mypassword
User Search Base: ou=people,dc=example,dc=com
User Search Filter: cn={0}
Group Search Base: ou=groups,dc=example,dc=com
Group Search Filter: uniqueMember={0}
Server SSL Cert: Use the contents of the LdapCA.crt file part of the ldap-server configuration
Email Attribute: mail
LDAP Refferals: Automatically follow any referrals
```

Also make sure you have `Enable UAA as OIDC provider` ticked otherwise you won't have the integration back to the Kubernetes clusters, only to the PKS API.

## Grant PKS API management permissions to LDAP group

Steps were taken from: https://docs.pivotal.io/runtimes/pks/1-3/manage-users.html#uaa-admin-login

Get the `Pks Uaa Management Admin Client` secret from `Ops Manager -> PKS Tile -> Credentials -> Pks Uaa Management Admin Client`.

```
uaac target https://PKS_API_URL:8443 --skip-ssl-validation
uaac token client get admin -s ADMIN-CLIENT-SECRET
uaac group map cn=superadmins,ou=groups,dc=example,dc=com --name pks.clusters.manage
```

In this case I'm mapping any users part of the `superadmins` LDAP group as PKS API admins: `user1`.

## Create a Kubernetes cluster with the PKS API

```
pks login -u user1 -p mypassword -a api.pks.moscow.cf-app.com -k
pks create-cluster k8s --external-hostname k8s.moscow.cf-app.com --plan PLAN
```

## Create a Namespace, Role and RoleBinding as part of your Kubernetes cluster

We will now create a namespace called `appdev`, a Role called `namespace-admin` and a RoleBinding binding the Role `namespace-admin` to the LDAP group `appdev`.

#### First let's create the new namespace:
Run `kubectl apply -f roles/00_namespace-appdev.yml`

You can check if the namespace was created successfully, you can run `kubectl get namespaces`.

#### Now let's create a Role with the following rules: `apiGroups: ["*"]`, `resources: ["*"]` and `verbs: ["*"]`.
Run `kubectl apply -f roles/01_namespace-admin-role.yml`

You can check if the role was created by running `kubectl get roles`.

#### The last step is to bind the Role created on the previous step to an LDAP group called `appdev`:
Run `kubectl apply -f roles/02_rb_appdev-admins.yml`

You can check if the RoleBinding has been created by running `kubectl get rolebindings`.

## Switch between users by generating new `kubeconfig` files

This script will do all the hard work to generate a `kubeconfig` to make it really easy to switch between users within your Kubernetes cluster.

#### To download the ca-cert from the current PKS cluster, run the following:
`kubectl config view --raw -o json | jq -r '.clusters[] | select(.name == "'$(kubectl config current-context)'") | .cluster."certificate-authority-data"' | base64 --decode > ca-cert.crt`

#### To switch to `user5` which is part of the `appdev` LDAP group, run the following command:
`./scripts/get-pks-k8s-config.sh --API=api.pks.moscow.cf-app.com --CLUSTER=k8s.moscow.cf-app.com --USER=user5 --NS=appdev --CERT=ca-cert.crt`

Used the following arcticle provided the script to get PKS Kubernetes credentials:
https://community.pivotal.io/s/article/script-to-automate-generation-of-the-kubeconfig-for-the-kubernetes-user

## Checking permissions of the new user

We have granted the users on the group `appdev` full access to the `appdev` namespace. So try a few different things to understand if the permissions are correct:

Getting the pods from the `appdev` namespace should work: `kubectl get pods`
Getting a list of all the namespaces shouldn't work: `kubectl get namespaces`

