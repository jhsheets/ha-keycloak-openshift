# Overview

This example uses Keycloak 22.0.4 with MSSQL 2019, running in OpenShift, without proper clustering.

While it shares a database with all keycloak instances, cached items such as user-sessions are not 
shared within a cluter, which can result in problems.

This will run a test that shows that the nodes aren't properly clustered.


# Setup

Add `oc` command to your shell, if necessary
> On Windows:
> ```
> & crc oc-env | Invoke-Expression
> ```

Create the project
> ```
> oc login -u developer https://api.crc.testing:6443
> oc new-project no-ha-keycloak
> ```

Create a security context for MSSQL to run properly
> ```
> oc login -u kubeadmin https://api.crc.testing:6443
> oc create -f .\mssql_restrictedfsgroupscc.yaml
> oc adm policy add-scc-to-group restrictedfsgroup system:serviceaccounts:no-ha-keycloak
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
> Since this example isn't clustered, you should always see a message like this:
> ```
> ERROR: Expected session count of 2, but found 1; exiting
> ```


# Access Keycloak

You can connect to Keycloak using the following URL: 
https://no-ha-keycloak.apps-crc.testing/

> You should not attempt to login to Keycloak until you perform the Verify Cluster steps as
> logging in before it runs will mess up the test.


# Teardown

> ```
> oc delete project no-ha-keycloak
> ```
You can re-run setup once the project is torn down, but you may need to wait a few minutes
before it will allow you to recreate the namespace.
