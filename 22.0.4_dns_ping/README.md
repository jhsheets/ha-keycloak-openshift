# Overview

This example uses Keycloak 22.0.4 with MSSQL 2019, running in OpenShift, and clustring the
Keycloak instances using [DNS_PING](http://www.jgroups.org/manual5/index.html#_dns_ping) 
for cluster discovery. 
> We set `DNS_PING` mode by setting the `KC_CACHE_STACK` environment variable to `kubernetes`
> and specifying the `jgroups.dns.query` to point to a headless service's DNS that we create in 
> `keycloak.yaml`

Here's a Medium article describing how this works:
https://medium.com/@nishada/keycloak-clustering-on-kubernetes-ec3d6a99fc33

There are comments in the `yaml` files that should help explain things a bit more.


# Setup

Add `oc` command to your shell, if necessary
> On Windows:
> ```
> & crc oc-env | Invoke-Expression
> ```

Create the project
> ```
> oc login -u developer https://api.crc.testing:6443
> oc new-project ha-keycloak
> ```

Create a security context for MSSQL to run properly
> ```
> oc login -u kubeadmin https://api.crc.testing:6443
> oc create -f .\mssql_restrictedfsgroupscc.yaml
> oc adm policy add-scc-to-group restrictedfsgroup system:serviceaccounts:ha-keycloak
> ```
> Note: if you see any *AlreadyExists* errors, you can safely ignore them.
> Some of these settings only need to be created once per OpenShift setup, but it doesn't
> hurt to try to recreate them.

Setup containers
> ```
> oc login -u developer https://api.crc.testing:6443
> oc create secret generic mssql --from-literal=SA_PASSWORD="Password!23"
> oc apply -f .\mssql.yaml
> oc apply -f .\keycloak.yaml
> oc apply -f .\cluster_test.yaml
> ```


# Verify Cluster

When you perform the Setup, it will create a container that will test if Keycloak's cluster
is properly configured.
> Note: It's very important that you do not attempt to login to Keycloak - via website
> or any other means - before the test runs. This test captures the number of logged-in
> sessions and compares them to what it should be. If you logged-in before the test ran,
> then perform a teardown and rerun the setup.

Verify the cluster is setup
> ```
> oc logs $(oc get pods -o name | select-string cluster-test)
> ```
> If you see this error: 
> ```
> Error from server (BadRequest): container "cluster-test" in pod "cluster-test-xxx" is waiting to start: PodInitializing
> ```
> then just wait a minute and try again
> 
> If everything's worked correctly, then you should see this message at the bottom:
> ```
> SUCCESS: all tests passed
> ```
>
> If you see any sort of error message at the bottom, then it's not properly clustered.


# Access Keycloak

You can connect to Keycloak using the following URL: 
https://ha-keycloak.apps-crc.testing/

> You should not attempt to login to Keycloak until you perform the Verify Cluster steps as
> logging in before it runs will mess up the test.


# Teardown

> ```
> oc delete project ha-keycloak
> ```
You can re-run setup once the project is torn down, but you may need to wait a few minutes
before it will allow you to recreate the namespace.


# Notes

## Keycloak StatefulSet

The reason why I use a StatefulSet to deploy Keycloak in this example is because doing so
gives every pod a deterministic name. 

This allows me to generate a setup where I can create NodePort's so that my `cluster_test.yaml`
tests can connect directly to each pod and verify that they're clustered properly.

I've also seen [this post](https://jimops.io/highly-available-keycloak-on-kubernetes) on the
internet where the author references  
[this helm chart](https://github.com/codecentric/helm-charts/blob/master/charts/keycloak/README.md) 
that states that there's a 23-character limit on the DNS names in Keycloak, and stateful sets
can be used to generate a short name to fall within this limit. These notes seem to reference
`jboss` which is no longer used in new versions of Keycloak, but I don't know if this character
limit still exists or not.

## Infinispan Cache Owners

We create a `ConfigMap` with a file in it calle `cache-ispn.xml`. This file controls the Infinispan
caches used by Keycloak. The distributed caches have a field called `owners` which controls how
many copies of the data are created in different clustered nodes.

Here's a simple table to help explain how the number of `replicas` and Infinispan's cache `owners` 
influence potential data loss
> Data loss here means things stored in the distributed cache, such as session data.
> IE: users might not appear authenticated, etc. Data persisted to the database is not lost.

|Replicas |Owners | Min. Nodes | Description |
|---------|-------|------------|-------------|
|1        |1      |1           |Any node loss will result in data loss|
|2        |1      |2           |Data not copied. Any node loss will result in data loss|
|2        |2      |1           |Data copied to 2 nodes. Can lose 1 node w/o data loss|
|3        |1      |3           |Data not copied. Any node loss will result in data loss|
|3        |2      |2           |Data copied to 2 nodes. Can lose 1 node w/o data loss|
|3        |3      |1           |Data copied to 3 nodes. Can lose 2 nodes w/o data loss|

The 'Minimum Nodes' is the minimum number of required nodes that must be running to prevent data loss.

You can verify the current minimum number of nodes in keycloak by performing the following:

1. Ensure both keycloak instances/pods are up and have been running for a few *minutes*
2. Open https://ha-keycloak.apps-crc.testing/metrics in your browser
3. Search for `vendor_cache_manager_keycloak_cache_sessions_statistics_required_minimum_number_of_nodes`
4. Verify the end of this line ends with `1.0`
	> Note: this is if you're using 2 replicas with 2 owners. If you change the values in `keycloak.yaml`
	> and apply the changes, then you may see a different value. See the chart above for expected values