# FastStart Provisioning Lab

## Deploy app to Jenkins on OCP 3.11

### Create a project from the template in the FastStart git org

1. Open a browser to 

### Create the ConfigMap and Secret to describe the cluster

1. Log into the ibmcloud cli

```
ibmcloud login -r us-south -g {resource-group} [--sso]
```

where:
 - `resource-group` is your assigned resource group (e.g. `faststart-one`)
 - `--sso` may be needed if you are configured for SSO login

2. Get the cluster information

```
ibmcloud ks cluster get --cluster {cluster-name}
```

3. Copy the following into a file named ibmcloud-config.yaml and update the values from the output of the previous command

```
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    app: ibmcloud-config
    group: catalyst-tools
  name: ibmcloud-config
data:
  APIURL: 'https://cloud.ibm.com'
  CLUSTER_NAME: <CLUSTER_NAME>
  CLUSTER_TYPE: openshift
  INGRESS_SUBDOMAIN: <INGRESS_SUBDOMAIN>
  REGION: us-south
  REGISTRY_NAMESPACE: <REGISTRY_NAMESPACE>
  REGISTRY_URL: us.icr.io
  RESOURCE_GROUP: <RESOURCE_GROUP>
  SERVER_URL: <MASTER_URL>
  TLS_SECRET_NAME: <TLS_SECRET_NAME>
---
apiVersion: v1
kind: Secret
metadata:
  labels:
    app: ibmcloud-config
    group: catalyst-tools
  name: ibmcloud-apikey
type: Opaque
stringData:
  APIKEY: <APIKEY>
  REGISTRY_USER: iamapikey
```

4. Log into the cluster following the instructions on the `Access` tab of the cluster page
   
5. Create the resources in the cluster

```
kubectl create -n {dev-namespace} -f ibmcloud-config.yaml
```

### Create git secret

1. Copy the following into a file called `gitsecret.yaml` and update the <Name>, <Git-Repo-URL>, <Git-Username>, and <Git-PAT>

```
apiVersion: v1
metadata:
  annotations:
    build.openshift.io/source-secret-match-uri-1: https://github.com/ibm-garage-cloud/*
  labels:
    jenkins.io/credentials-type: usernamePassword
  name: <Name>
type: kubernetes.io/basic-auth
stringData:
  url: <Git-Repo-URL>
  username: <Git-Username>
  password: <Git-PAT>
```

where:
 - `Name` is in the format of `{git org}.{git repo}`
 - `Git-Repo-URL` is the https url to the git repository
 - `Git-Username` is the username that has access to the git repo
 - `Git-PAT` is the personal access token of the git user

2. Log into the cluster following the instructions on the `Access` tab of the cluster page

3. Create the secret in the cluster

```
kubectl create -n {dev-namespace} -f gitsecret.yaml
```

### Create build config

1. Copy the following into a file called `buildconfig.yaml` and update the <Name>, <Secret>, <Git-Repo-URL>, and <Namespace>

```
apiVersion: v1
kind: BuildConfig
metadata:
  name: <Name>
spec:
  triggers:
  - type: GitHub
    github:
      secret: my-secret-value
  source:
    git:
      uri: <Git-Repo-URL>
      ref: master
  strategy:
    jenkinsPipelineStrategy:
      jenkinsfilePath: Jenkinsfile
      env:
      - name: CLOUD_NAME
        value: openshift
      - name: NAMESPACE
        value: <Namespace>
```

where:
 - `Name` is in the same name as the git secret 
 - `Git-Repo-URL` is the https url to the git repository
 - `Namespace` is the dev namespace where the app will be deployed

2. Log into the cluster following the instructions on the `Access` tab of the cluster page

3. Create the buildconfig in the cluster

```
kubectl create -n {dev-namespace} -f buildconfig.yaml
```
