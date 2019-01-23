# How-to install and configure PKS to use LDAP with RBAC on the PKS API and Kubernetes clusters

## Setup Ldap Server:

`cd ldap-server`
`docker build -t ldap-server .`
`docker run -d -p 1389:389 -p 1636:636 ldap-server slapd  -h "ldap://0.0.0.0:389  ldaps://0.0.0.0:636" -d 3 -f /ldap/slapd.conf`

You should have now an LDAP Server running on port 1389 (no TLS) or 1636 (TLS enabled).
Now import the sample data:

`ldapadd -v -x -D "cn=admin,dc=example,dc=com" -w mypassword  -H ldap://localhost:1389 -f import.ldif`

After importing the data you'll have the following groups and it's members (the password for all users is `mypassword`):

superadmins: user1
readonly: user2
ops: user3, user4
appdev: user5

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

## Create a Kubernetes cluster with the PKS API

TO-DO

## Create a Namespace, Role and RoleBinding as part of your Kubernetes cluster

TO-DO

## Switch between users by generating new `kubeconfig` files

This script will do all the hard work to generate a `kubeconfig` to make it really easy to switch between users within your Kubernetes cluster.

`kubectl config view --raw -o json | jq -r '.clusters[] | select(.name == "'$(kubectl config current-context)'") | .cluster."certificate-authority-data"' | base64 --decode > ca-cert.crt`
`./scripts/get-pks-k8s-config.sh --API=api.pks.moscow.cf-app.com --CLUSTER=k8s.moscow.cf-app.com --USER=user1 --NS=k8s --CERT=ca-cert.crt`

Used the following article to get credentials for the multiple users:
https://community.pivotal.io/s/article/script-to-automate-generation-of-the-kubeconfig-for-the-kubernetes-user
