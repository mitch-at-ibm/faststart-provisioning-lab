# FastStart Provisioning Lab

## Deploy app to Jenkins on OCP 3.11

**Note:**

For these exercises you will be asked for a `<dev-namespace>`. That value should be the name 
you claimed in the box note prefixed to `-dev` (e.g. user01-dev)

### Create a project from the template in the FastStart git org

For the lab, we need a deployable application to run the pipeline. We will use a pre-built Code Pattern
to expedite things.

1. Open a browser to https://github.com/ibm-garage-cloud/template-node-typescript
2. Click the green `Use this template` button
3. On the following page, select the FastStart git org for the owner and give the repo a name
4. Click `Create repository from template`

### Create the ConfigMap and Secret to describe the cluster

The pipeline needs certain information about the cluster to be able to push the image to the registry and 
deploy the image to the cluster. Instead of hard coding those values into the Jenkinsfile, we will store them
in a ConfigMap and Secret.

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

where:
 - `<APIKEY>` is the one provided in the box note

4. Log into the cluster following the instructions on the `Access` tab of the cluster page
   
5. Create the resources in the cluster

```
kubectl create -n <dev-namespace> -f ibmcloud-config.yaml
```

where:
 - `<dev-namespace>` should be the name you claimed in the box note prefixed to `-dev` (e.g. user01-dev)

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
kubectl create -n <dev-namespace> -f gitsecret.yaml
```

where:
 - `<dev-namespace>` should be the name you claimed in the box note prefixed to `-dev` (e.g. user01-dev)

### Create the build config

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
kubectl create -n <dev-namespace> -f buildconfig.yaml
```

where:
 - `<dev-namespace>` should be the name you claimed in the box note prefixed to `-dev` (e.g. user01-dev)

### View the pipeline in the OpenShift console

1. Open the OpenShift console for the cluster
2. Select your project/namespace (i.e. `<dev-namespace>`)
3. Select Builds -> Pipelines
4. The build pipeline that was created in the previous step should appear
5. Manually trigger the pipeline by pressing the `Build` button

### Create the webhook

1. Run the following to get the webhook details from the build config 

```
kubectl describe bc <Name> -n <dev-namespace>
```

where:
 - `<Name>` is the name used in the previous step for the build config
 - `<dev-namespace>` should be the name you claimed in the box note prefixed to `-dev` (e.g. user01-dev)

The webhook url will have a structure similar to:

`http://<openshift_api_host:port>/oapi/v1/namespaces/<namespace>/buildconfigs/<name>/webhooks/<secret>/generic`

In our case `<secret>` will be `my-secret-value`

2. Open a browser to the GitHub repo deployed in the previous step in the build config

3. Select `Settings` then `Webhooks`. Press `Add webhook`

4. Paste the webhook url from the previous step into the `Payload url`. Leave the rest of the values as the defaults and press `Add webhook`

5. Press the button to test the webhook to ensure that everything was done properly

6. Go back to your project code and push a change to one of the files
   
7. Go to the Build pipeline page in the OpenShift console to see that the build was triggered

### Bind the credentials for the cloudant database into the cluster

1. Log into the ibmcloud environment

```
ibmcloud login -r us-south -g {resource-group} [--sso]
```

2. Run the following command to bind the credentials into the cluster

```
ibmcloud ks cluster service bind --cluster <CLUSTER_NAME> --namespace <dev-namespace> --service <SERVICE_NAME> --key faststart-key
```

where:
 - `<CLUSTER_NAME>` is the name of the cluster
 - `<dev-namespace>` should be the name you claimed in the box note prefixed to `-dev` (e.g. user01-dev)
 - `<SERVICE_NAME>` is the name of the cloudant service (either faststart-one-cloudant or faststart-two-cloudant)

**Note**: the value `faststart-key` for the key works because we created that key ahead of time

3. Open the OpenShift console and select the `<dev-namespace>` project
   
4. Select Resources -> Secrets. Select the `binding-...` secret from the list
   
5. Press the `Reveal values` button to see the contents of the secret

We won't be doing it in this lab, but in order to access the Cloudant instance from an application component that secret
would be attached to the Deployment/Pod through an environment reference.

## Provision dev tools with scripts

### Look at the provided terraform modules

1. Clone the terraform modules repo - https://github.com/ibm-garage-cloud/garage-terraform-modules
2. Take a look through the way the modules are organized
3. Look at the structure of one module, e.g. https://github.com/ibm-garage-cloud/garage-terraform-modules/tree/master/generic/cluster/namespaces

Each module has the same basic structure:
- Module input variables are defined in `variables.tf`
- Module output variables are defined in `outputs.tf`
- Module logic is defined in `main.tf`
  
### Setup the terraform scripts

1. Clone the faststart iteration-zero repo - https://github.com/ibm-garage-cloud/iteration-zero-faststart
2. Copy `credentials.template` into a file named `credentials.properties`
3. In `credentials.properties`, replace <IBMCLOUD_API_KEY> with the APIKEY from the top of the box note
4. Open `terraform/settings/environment.tfvars". Set the 'cluster_name' and the 'resource_group_name' to 
match the cluster you've been assigned

### Run the terraform scripts

1. Start the docker image to create the CLI environment by running `./launch.sh`. You should see an IBM Cloud banner
2. Kick off the process by running `./runTerraform.sh`. You will see a screen that confirms the settings that will be applied. Select Y
3. The script will prompt for the dev_namespace. Provide the value used above (e.g. user01-dev)
4. In about 5 minutes the process will complete (note: the tools may not be available yet)
5. Open the Openshift console to look at the deployments and pods in your namespace

