# Role-based access control based on LDAP

OpenShift supports the configuration of multiple identity providers. A complete list of identity providers can be found here: [Supported Identity Providers](https://docs.openshift.com/container-platform/4.14/authentication/understanding-identity-provider.html#supported-identity-providers) For this example, we will be configuring a simple LDAP provider Red Hat Identity Management. You can use any LDAP of your choice. 

Each cluster, hub or hosted will need to be individually configured for Authentication and authorization. LDAP will handle authentication, and Kubernetes RBAC roles will handle Authorization. Because all three clusters will be leveraging the same LDAP instance for authentication we should be able to reuse some of the configuration files.

All steps below are based on [Configuring LDAP Identity Provider](https://docs.openshift.com/container-platform/4.14/authentication/identity_providers/configuring-ldap-identity-provider.html) and will require the use of the “**oc**” command line tool and a file editor. 

We will also need a “bind username and password” for connecting to the LDAP server. This is a separate user (think of it as a service account), the bind user at a minimum should have the directory user and group search privileges, this user is used to lookup the to be authenticated user in the directory and to create the initial connection to the LDAP server. This user will be referred to as the “bind\_user”, in the documentation below.

Though Red Hat OpenShift supports LDAPS, for the purpose of this example we will be using a non-encrypted port (389 in this case). Additional LDAP configuration details can be found in the official [documentation](https://docs.openshift.com/container-platform/4.14/authentication/identity_providers/configuring-ldap-identity-provider.html#identity-provider-creating-configmap_configuring-ldap-identity-provider).

The steps below also assume that the following AD groups exist to control access if these don’t exist please create it in the LDAP:* “ocpmanagment” - should contain the user “tenant0”
* “ocptenant1” - should contain the users “tenant0” and “tenant1”
* “ocptenant2” - should contain the users “tenant0” and “tenant2”This will allow us to filter the logins so that tenant0 can log into the hub and the other hosted clusters. “tenant1” will only be able to log into the “tenant1” cluster, and “tenant2” will only be able to log into the “tenant2” cluster.

 ```
**NOTE:** Self Provisioning namespaces is enabled by default for any user who can log into the cluster.
This allows users to create new namespaces on their own. If this feature is not desired, be sure to follow
the steps [Disabling project self-provisioning]
(https://docs.openshift.com/container-platform/4.14/applications/projects/configuring-project-creation.html#disabling-project-self-provisioning_configuring-project-creation)
to disable this feature
```

## LDAP Configuration for Management Cluster

**1.Create a file with the following template YAML:|                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
|   identityProviders:    - ldap:        attributes:          email:            - mail          id:            - sAMAccountName          name:            - cn          preferredUsername:            - sAMAccountName        bindDN: bind\_user        bindPassword:          name: ldap-secret        insecure: true        url: 'LDAP://\<ad\_server\_name>/\<ldap\_directory\_path>?sAMAccountName?sub?((memberOf=CN=ocpmanagement,\<ldap\_directory\_path>))'      mappingMethod: claim      name: ldap      type: LDAP |You will need to update the following sections of the template according to your LDAP setup:* \<ad\_server\_name>  - this will be the fully qualified AD server name.\
  eg: `ad.example.com`

* \<ldap\_directory\_path> - this is the search path within the LDAP directory. eg: `CN=Users,DC=ad,DC=example,DC=com`* id - the LDAP attribute identified as the users user name used for login

* Insecure: - for LDAPs set this to false and provide the CA certificate path as a config map. For LDAP set this to true.

* bindDN and bindPassword - details of the used for binging to LDAPFor additional details about the above parameters please refer to the official [documentation](https://docs.openshift.com/container-platform/4.14/authentication/identity_providers/configuring-ldap-identity-provider.html#identity-provider-ldap-CR_configuring-ldap-identity-provider)We will use this template text for the configuration of the next three sections.**
==============================================================================================================================================================================================================================================================================================================================================================================================================================================

###

## Apply LDAP bind\_user password

# **2.Create a secret to hold the password for the bind\_user, making sure to update “\<secret>” with the password for the _bind\_user_|                                                                                                                                                                                                                                                                                                                                                                      |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `$ export KUBECONFIG=kubeconfig-management``# this will create a secret used by the management cluster``$ oc create secret generic ldap-secret --from-literal=bindPassword=<secret> \``  -n openshift-config``# this will create a secret used by the tenant clusters``$ oc create secret generic ldap-secret --from-literal=bindPassword=<secret> \``  -n clusters` |**

## Apply LDAP configuration for hub cluster

# **3)1. Log into the hub cluster with the _kubeadmin_ username/password as shown here using the `oc login -u kubeadmin` command for details refer to the official [documentation](https://docs.openshift.com/container-platform/4.14/cli_reference/openshift_cli/getting-started-cli.html#cli-logging-in_cli-developer-commands).

2. Run the command `oc edit oauth cluster`

3. In the editor that opens, scroll down to the `spec:` section, and add the contents of the ldap template from step [LDAP Configuration for Management Cluster](https://docs.google.com/document/d/12IiTnsDZmbPd4XdZTIeJMmYF9AFkgBTGgbRywS1tmgk/edit#heading=h.arfpm2d7nb3i). The result should look something like this:![](https://lh7-us.googleusercontent.com/G128GY8KCWQUe3TXpWF5CoE9TYMewRuWXNk6m-u-TiximpmTVDXjEjxxzkrfTkYtUiuqiYQQO8kOxpTlopnA7CQHLyH3siKLcfjuoLdxKtWYctglC2bRcw7qHmutgSlxIv5JowTZa5eAD3RN8pSAff8)**NOTE: ONLY EDIT THE “spec” section**, do not edit any other section.4. Save the file and the cluster will update to use both the `kubeadmin` as well as `LDAP authentication`. In order to test this login with LDAP (called ldapipa in this case) in the openshift console using as shown in the below screenshot**

### ![](https://lh7-us.googleusercontent.com/_h5bVhoktzviHKu414QApDClrAz-hSyx_SEoXbS99TeiWg1_WoeTdRvI1vhaUCb1kflR2YK_56SqYBkC0dHrv0I_9TI5YJdtVGnOLql8bqFSd2iOhL7hOY45h60H1gd1s0fyewGCNOuXL7rTw38YQ_s)

# ****

###

## Apply LDAP configuration for Hosted (tenant1) Cluster

**4)1. Log into the hub cluster with the _kubeadmin_ username/password as shown here using the `oc login -u kubeadmin` command for details refer to the official [documentation](https://docs.openshift.com/container-platform/4.14/cli_reference/openshift_cli/getting-started-cli.html#cli-logging-in_cli-developer-commands).

2. Run the command `oc edit oauth cluster -n clusters`

3. Using the example YAML above, update the tenant1 hosted configuration adding lines under configuration: → oauth: section to the configuration file, making sure to update the url as described below:|                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
|   identityProviders:    - ldap:        attributes:          email:            - mail          id:            - sAMAccountName          name:            - cn          preferredUsername:            - sAMAccountName        bindDN: bind\_user        bindPassword:          name: ldap-secret        insecure: true        url: 'LDAP://\<ad\_server\_name>/\<ldap\_directory\_path>?CN=Users,DC=ad,DC=xphyrlab,DC=net?sAMAccountName?sub?((memberOf=CN=ocptenant1,CN=Users,DC=ad,DC=xphyrlab,DC=net))'      mappingMethod: claim      name: ldap      type: LDAP |*  \<ad\_server\_name>  - this will be the fully qualified AD server name.\
  eg: `ad.example.com`

* \<ldap\_directory\_path> - this is the path within the LDAP directory. eg: `CN=Users,DC=ad,DC=example,DC=com`- Don’t forget to adjust the LDAP URL based on your LDAP setup for the hosted cluster.4) Replace “ocpmanagement” with “ocptenant1” in the same URL line. The final result should look similar to this\
   ![](https://lh7-us.googleusercontent.com/ktpUU8LvRz1dDGN9D-Z-bG_K36qW5Gk7u6k9jialYKVVL5ohgAcPRg1HGz0Mz2fmJlnRdQBvMN9tnDwxPcU88dSyPh_aTfhik26khYREIMhHcuEFjEhwUNA3dqaoHB15mxucjxynN6XMhtpRSsqFabU)5. Save the file and the hosted tenant1 cluster will update to use LDAP authentication. In order to test this login with LDAP (called ldapipa in this case) in the openshift console using as shown in the below screenshot**
=================================================================================================================================================================================================================================================================================================================================================================================================================================

### ![](https://lh7-us.googleusercontent.com/_h5bVhoktzviHKu414QApDClrAz-hSyx_SEoXbS99TeiWg1_WoeTdRvI1vhaUCb1kflR2YK_56SqYBkC0dHrv0I_9TI5YJdtVGnOLql8bqFSd2iOhL7hOY45h60H1gd1s0fyewGCNOuXL7rTw38YQ_s)

# **Similarly update any other cluster to start using LDAP for authentication.**

###

## Apply RBAC for tenant0 user

# 5.First we will create a ClusterRoleBinding for the “tenant0” user. This ClusterRoleBinding file will be applied to the “management”, “tenant1” and “tenant2” clusters to ensure that this user has full cluster access.crb\_tenant0.yaml|                                                                                                                                                                                                                                                                                           |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `kind: ClusterRoleBinding``apiVersion: rbac.authorization.k8s.io/v1``metadata:``  name: tenant0admin``subjects:``  - kind: User``    apiGroup: rbac.authorization.k8s.io``    name: tenant0``roleRef:``  apiGroup: rbac.authorization.k8s.io``  kind: ClusterRole``  name: cluster-admin` |Now we will apply this Kubernetes role binding to the hub and the hosted clusters:|                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `# connect to the management cluster``$ export KUBECONFIG=kubeconfig-management``$ oc create -f crb_tenant0.yaml``clusterrolebinding.rbac.authorization.k8s.io/tenant0admin created``# connect to the tenant1 cluster``$ export KUBECONFIG=kubeconfig-tenant1``$ oc create -f crb_tenant0.yaml``clusterrolebinding.rbac.authorization.k8s.io/tenant0admin created``# similarly the above commands can be executed for any hosted cluster` |NOTE: We do not need to create any Role Bindings for tenant1 as the user is added to the cluster as a standard user, with the ability to create and delete their own project/namespaces.
