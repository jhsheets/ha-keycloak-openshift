# Overview
This is an example of deploying multiple instances of Keycloak in a high-availability cluster 
to an Red Hat OpenShift kubernetes environment.


# Setup OpenShift

This example uses RedHat OpenShift 4.x as a kubernetes backend.

For testing purposes, I'll be using a RedHat OpenShift Local setup on my own machine
to make it easy to follow.

If you wish to run this on a production OpenShift cluster, you'll need to use the correct
URLs and logins to your cluster, but everything should work.

## Download 
I'm using RedHat's OpenShift Local 2.x, which is a single-node cluster
version of OpenShift 4.x, which can be used for local development and testing. 

OpenShift Local runs on Linux, Windows and MacOS.

Here's the documentation:
https://access.redhat.com/documentation/en-us/red_hat_openshift_local

1. Create a RedHat account: 
	> https://www.redhat.com/wapps/ugc/register.html
2. Signup for OpenShift local: 
	> https://developers.redhat.com/download-manager/link/3868678
3. Get an OpenShift Local key
	> https://console.redhat.com/openshift/create/local

## Install
Run `crc setup` to create an OpenShift Local cluster VM on your machine. 

## Start
1. Run `crc start'
	> If you're having issues, try using `crc start --log-level debug`
2. It will take some time for the VM to launch
3. Once started, it will print out the command to login to the OpenShift CLI (using `oc`) or the webpage
	1. Copy the credentials
	2. It may print out a command for you to run, so that `oc` works. If it does, copy and run the command in the shell you plan to use
4. You can login to the OpenShift Web Console with this URL: 
	> https://console-openshift-console.apps-crc.testing/



# Understanding Keycloak Clustering

When you have multiple instances of Keycloak running, you must point them all to use the same database.
This is where most of Keycloak's information is stored in the database; 
your realm settings, users, clients, etc.

However, some information is stored in caches, such as user sessions. 
If you were to login to one instance of keycloak - which creates your session - and then get directed 
to another instance that doesn't know about your session, it wouldn't know you've authenticated, and 
it would prompt you to re-authenticate. To solve this, we must synchronize the necssary cache data
between instances.

Keycloak uses [Infinispan](https://infinispan.org/) for caching, which in turn uses 
[JGroups](http://www.jgroups.org/) which is a Java library for clustering. By using Infinispan, 
Keycloak can enable some of its caches to be 'distributed'. These are caches which contain
information that needs to be shared between all the nodes in the cluster in order to work properly.

Here's Keycloak's notes on Distributed caching:
https://www.keycloak.org/server/caching)

Here's an old Keycloak guide on the different forms of Discovery that JGroup's uses for connecting 
Keycloak instances, which has a lot of good information:
https://www.keycloak.org/2019/05/keycloak-cluster-setup
> Note that this documentation came out when Keycloak still used JBoss so some of its information
> is out-of-date. It also doesn't mention `DNS_PING` for kubernetes.


# Keycloak Clustering Notes

Here are some of the different forms of generic service discovery and clustering supported By Keycloak 
through JGroups

- [DNS_PING](http://www.jgroups.org/manual5/index.html#_dns_ping) discovery works in Kubernets/OpenShift,
and it's one of the easiest ways to get it working in these environments. However, I don't believe This
methodology would work in most other environments.

- [MPING](http://www.jgroups.org/manual5/index.html#MPING) uses multicasting, which is usually disabled
in kubernetes setups by default. You can enable Multicasting in OpenShift, but you need to be a 
[cluster-admin](https://docs.openshift.com/container-platform/4.8/networking/ovn_kubernetes_network_provider/enabling-multicast.html).
If you have the option of enabling multicasting and have different, multiple setups (ex: Kubernetes, 
Docket, etc) and want to use the same configuration in all of them, then this is a possible option.

- [JDBC Ping](http://www.jgroups.org/manual5/index.html#_jdbc_ping) is a another option.
[Here](https://github.com/ivangfr/keycloak-clustered) is an example where an author has setup 
Keycloak with this methodology, customizing the Infinispan settings to write each Keycloak 
instance's IP to a database. This methodology seems like would be tricky to setup properly in Kubernetes,
as you would need some way to pass in each pod's IP address as an environment variable.