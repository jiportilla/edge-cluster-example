# Edge Cluster Service

This is an example of creating and using an edge cluster service which detects objects using deep learning models.

Based on [https://github.com/open-horizon/examples/blob/master/edge/services/operator/simple-operator/deploy/horizon/node.policy.json](https://github.com/open-horizon/examples/blob/master/edge/services/operator/simple-operator/deploy/horizon/node.policy.json)

[1. Preconditions for Using the Edge Cluster Service](#preconditions)

[2. Configuring the Edge Cluster Service](#configuring)

[3. Building and Publishing the Edge Cluster Service](#building)

[4. Using the Edge Cluster Service with Deployment Policy](#using-JSON-exporter)
![Prometheus architecture ](docs/JSON-exporter.png)

## <a id=preconditions></a> 1. Preconditions for Using the Edge Cluster Service

If you have not done so already, you must do these steps before proceeding with the Edge Cluster service:

1. Install the Horizon management infrastructure (exchange and agbot).

	*Also see [one-click Management Hub installation example](https://github.com/open-horizon/devops/blob/master/mgmt-hub/README.md)

2. Install the Horizon agent on your edge device and configure it to point to your Horizon exchange.

3. As part of the infrasctucture installation process for IBM Edge Application Manager a file called `agent-install.cfg` was created that contains the values for `HZN_ORG_ID` and the **exchange** and **css** url values. Locate this file and set those environment variables in your shell now:

```bash
eval export $(cat agent-install.cfg)
```

 - **Note**: if for some reason you disconnected from ssh or your command line closes, run the above command again to set the required environment variables.

4. In addition to the file above, an API key associated with your Horizon instance would have been created, set the exchange user credentials, and verify them:

```bash
export HZN_EXCHANGE_USER_AUTH=iamapikey:<horizon-API-key>
hzn exchange user list
```

5. Choose an ID and token for your edge node, create it, and verify it:

```bash
export HZN_EXCHANGE_NODE_AUTH="<choose-any-node-id>:<choose-any-node-token>"
hzn exchange node create -n $HZN_EXCHANGE_NODE_AUTH
hzn exchange node confirm
```
6. Register your edge cluster with the Exchange

```bash
hzn register
```


## <a id=configuring></a> 2. Configuring the Edge Cluster Service

You should complete these steps before proceeding building the Edge Cluster service:

1. Clone this git repository:

```bash
cd ~   # or wherever you want
git clone git@github.com:jiportilla/edge-cluster-example.git
cd ~/edge-cluster-example/
```
verify **helm** is installed with:

```helm version```

For example:
```
Client: &version.Version{SemVer:"v2.12.3", GitCommit:"eecf22f77df5f65c823aacd2dbd30ae6c65f186e", GitTreeState:"clean"} 
```

2. Create Helm Chart Repository:

`helm create edge-detector-cluster` 

(edge-detector-cluster should be small case letters) 

3. Update Chart.yaml if needed:

```
vi edge-detector-cluster/Chart.yaml
```

4. Update the `values.yaml` to include your deployment image, tag and ports for your service. 

```
vi edge-detector-cluster/values.yaml
```
Ensure that the repository, tag and service ports match your application's docker image properties.

```
replicaCount: 1
image:repository: codait/max-object-detector
tag: latestpull
Policy: IfNotPresent
...
service:type: 
ClusterIP
port: 5000
```

5. Modify the `deployment.yaml` and check `service.yaml` files in the templates folder according to your application as needed. Ensure that the **port number** in `deployment.yaml` matches the above port number of `values.yaml`. 

```
vi edge-detector-cluster/templates/deployment.yaml
```

With:
```
          ports:
            - name: http
              containerPort: 5000
```

6. Test if the helm chart is valid:

```
helm template edge-detector-cluster
```

If the chart is valid without any errors, this command will show you the deployment and service yaml files. It will return errors if there is an error in the helm chart.

(If you’re in a parent folder from where you can access the helm folder run `helm template edge-detector-cluster`)

You will see similar results to:

```

---
# Source: edge-detector-cluster/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: release-name-edge-detector-cluster
  labels:
    app.kubernetes.io/name: edge-detector-cluster
    helm.sh/chart: edge-detector-cluster-0.1.0
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Tiller
spec:
  type: ClusterIP
  ports:
    - port: 5000
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app.kubernetes.io/name: edge-detector-cluster
    app.kubernetes.io/instance: release-name

---
# Source: edge-detector-cluster/templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "release-name-edge-detector-cluster-test-connection"
  labels:
    app.kubernetes.io/name: edge-detector-cluster
    helm.sh/chart: edge-detector-cluster-0.1.0
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Tiller
  annotations:
    "helm.sh/hook": test-success
spec:
  containers:
    - name: wget
      image: busybox
      command: ['wget']
      args:  ['release-name-edge-detector-cluster:5000']
  restartPolicy: Never

---
# Source: edge-detector-cluster/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: release-name-edge-detector-cluster
  labels:
    app.kubernetes.io/name: edge-detector-cluster
    helm.sh/chart: edge-detector-cluster-0.1.0
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Tiller
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: edge-detector-cluster
      app.kubernetes.io/instance: release-name
  template:
    metadata:
      labels:
        app.kubernetes.io/name: edge-detector-cluster
        app.kubernetes.io/instance: release-name
    spec:
      containers:
        - name: edge-detector-cluster
          image: "codait/max-object-detector:latespull"
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 5000
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /
              port: http
          readinessProbe:
            httpGet:
              path: /
              port: http
          resources:
            {}


---
# Source: edge-detector-cluster/templates/ingress.yaml

```


## Create Helm Operator

Create Helm Operator:

1. Create helm operator with (on Mac):

```operator-sdk new edge-detector-operator --type=helm --api-version=edge-detector.com/v1 --kind=Service --helm-chart=/Users/ivanp/horizon/42tests/edge-cluster-example/edge-detector-cluster```

Results similar to:

```
INFO[0000] Creating new Helm operator 'edge-detector-operator'.
INFO[0000] Created helm-charts/edge-detector-cluster
INFO[0000] Generating RBAC rules
WARN[0000] Using default RBAC rules: failed to generate RBAC rules: failed to get server resources: Get https://kubernetes.docker.internal:6443/api?timeout=32s: EOF
INFO[0000] Created build/Dockerfile
INFO[0000] Created watches.yaml
INFO[0000] Created deploy/service_account.yaml
INFO[0000] Created deploy/role.yaml
INFO[0000] Created deploy/role_binding.yaml
INFO[0000] Created deploy/operator.yaml
INFO[0000] Created deploy/crds/edge-detector.com_v1_service_cr.yaml
INFO[0000] Generated CustomResourceDefinition manifests.
INFO[0000] Project creation complete.
```

2. Next, create the operator with:

```
cd edge-detector-operator

operator-sdk build docker.io/iportilla/edge-detector.operator_amd64:1.0.0

```

Results similar to:

```
INFO[0000] Building OCI image docker.io/iportilla/edge-detector.operator_amd64:1.0.0
Sending build context to Docker daemon  31.23kB
Step 1/3 : FROM quay.io/operator-framework/helm-operator:v0.17.0
 ---> ce3d68592219
Step 2/3 : COPY watches.yaml ${HOME}/watches.yaml
 ---> 6de4f6f579f4
Step 3/3 : COPY helm-charts/ ${HOME}/helm-charts/
 ---> 2cbad6fbf986
Successfully built 2cbad6fbf986
Successfully tagged iportilla/edge-detector.operator_amd64:1.0.0
INFO[0002] Operator build complete.
```

3. Publish image to repository with:

`docker push docker.io/iportilla/edge-detector.operator_amd64:1.0.0`


4. Update image name with:

```
vi deploy/operator.yaml

 # Replace this with the built image name
   image: docker.io/iportilla/edge-detector.operator_amd64:1.0.0
```

(Edit the REPLACE_IMAGE_NAME with docker.io/iportilla/edge-detector.operator_amd64:1.0.0 from above)


5. (Optional on the k8s compatible edge cluster) To Create and test if the files are working on the cluster:

```
. kubectl create -f deploy/service_account.yaml -n openhorizon-agent
. kubectl create -f deploy/role.yaml -n openhorizon-agent
. kubectl create -f deploy/role_binding.yaml -n openhorizon-agent
. kubectl create -f deploy/operator.yaml -n openhorizon-agent
. kubectl create -f deploy/crds/edge-detector.com_appservices_crd.yaml -n openhorizon-agent

. kubectl create -f deploy/crds/edge-detector.com_v1_appservice_cr.yaml -n openhorizon-agent
```
6. Create a tar file for operator files:

```
cd deploy/
tar -zvcf edge-detector.operator.tar.gz .
```


7. Create a Horizon Service:

```
hzn dev service new -V 1.0.0 -s edge-detector -c cluster
```

8. Update service definition file:

```
vi horizon/service.definition.json
```
Edit **OperatorYamlArchive** and add the path for the **operator.tar.gz** file from the above step

```
"clusterDeployment": {
        "operatorYamlArchive": "/Users/ivanp/horizon/42tests/edge-cluster-example/edge-detector-operator/deploy/edge-detector.operator.tar.gz"
    }
```

9. Set environment variables:

```
eval $(hzn util configconv -f horizon/hzn.json)
export ARCH=$(hzn architecture)
echo $ARCH
```

10. Publish Service to IEAM:

`hzn exchange service publish -f horizon/service.definition.json`



## <a id=building></a> 3. Building and Publishing the Edge Cluster Service

1. Change directories to the local copy:

```bash
cd ~/edge_json_exporter/
```

2. Set the values in `horizon/hzn/json` to your liking. These variables are used in the service file. They are also used in some of the commands in this procedure. After editing `horizon/hzn.json`, set the variables in your environment:

```bash
export ARCH=$(hzn architecture)
eval $(hzn util configconv -f horizon/hzn.json)
```

3. Build the docker image:

```bash
make build
```
For example, when using the default values provided in this repo [hnz.json](https://github.com/jiportilla/edge_json_exporter/blob/master/horizon/hzn.json) configuration file:
	


```bash
docker build --network="host" -t iportilla/jexporter_amd64:1.0.0 -f ./Dockerfile.amd64 .
```

3. You are now ready to publish your edge service, so that it can be deployed to edge devices. Instruct Horizon to push your docker image to your registry and publish your service in the Horizon Exchange using:

```bash
hzn exchange service publish -f horizon/service.definition.json
hzn exchange service list
```

See [Developing an edge service for devices](https://www-03preprod.ibm.com/support/knowledgecenter/SSFKVV_4.1/devices/developing/developing.html) for additional details.

## <a id=using-JSON-exporter></a> 4. Using the Edge Cluster Service with Deployment Policy


The Horizon Policy mechanism offers an alternative to using deployment patterns. Policies provide much finer control over the deployment placement of edge services. Policies also provide a greater separation of concerns, allowing edge nodes owners, service code developers, and business owners to each independently articulate their own policies. There are three main types of Horizon Policies:

1. Service Policy (may be applied to a published service in the Exchange)
2. Deployment Policy (which approximately corresponds to a deployment pattern)
3. Node Policy (provided at registration time by the node owner)

### Service Policy

Like the other two Policy types, Service Policy contains a set of `properties` and a set of `constraints`. The `properties` of a Service Policy could state characteristics of the Service code that Node Policy authors or Business Policy authors may find relevant. The `constraints` of a Service Policy can be used to restrict where this Service can be run. The Service developer could, for example, assert that this Service requires a particular hardware setup such as CPU/GPU constraints, memory constraints, specific sensors, actuators or other peripheral devices required, etc.


1. Below is the file provided in  `policy/service.policy.json` with this example:

```json
{
    "properties": [],
    "constraints": [
        "openhorizon.example == detector"
    ]
}
```

- Note this simple Service Policy provides no properties, and it states one `constraint`. This Edge Cluster Service `constraint` is one that a Service developer might add, stating that their Service must only be deployed when **openhorizon.arch** equals to **amd64**. After node registration the **openhorizon.arch** property will be set to **amd64** on amd64 cluster nodes, so this Edge Clustert Service should be compatible with our edge device.

2. If needed, run the following commands to set the environment variables needed by the `service.policy.json` file in your shell:

```bash
export ARCH=$(hzn architecture)
eval $(hzn util configconv -f horizon/hzn.json)
```

3. Next, add or replace the service policy in the Horizon Exchange for this Edge Cluster Service:

```bash
make publish-service-policy
```
For example:

```bash
hzn exchange service addpolicy -f horizon/service.policy.json edge-detector_1.0.0_amd64

```

4. View the pubished service policy attached to the **edge-detector** edge service:

```bash
hzn exchange service listpolicy edge-detector_1.0.0_amd64
```

The output should look like:

```json
{
  "properties": [
    {
      "name": "openhorizon.service.url",
      "value": "edge-detector"
    },
    {
      "name": "openhorizon.service.name",
      "value": "edge-detector"
    },
    {
      "name": "openhorizon.service.org",
      "value": "mycluster"
    },
    {
      "name": "openhorizon.service.version",
      "value": "1.0.0"
    },
    {
      "name": "openhorizon.service.arch",
      "value": "amd64"
    }
  ],
  "constraints": [
    "openhorizon.arch == amd64"
  ]
}
```

- Notice that Horizon has again automatically added some additional `properties` to your Policy. These generated property values can be used in `constraints` in Node Policies and Deployment Policies.

- Now that you have set up the published Service policy is in the exchange, we can move on to the next step of defining a Deployment Policy.


### Deployment Policy


Deployment policy (sometimes called Business Policy) is what ties together edge nodes, published services, and the policies defined for each of those, making it roughly analogous to the deployment patterns you have previously worked with.

DeploymentpPolicy, like the other two Policy types, contains a set of `properties` and a set of `constraints`, but it contains other things as well. For example, it explicitly identifies the Service it will cause to be deployed onto edge nodes if negotiation is successful, in addition to configuration variable values, performing the equivalent function to the `-f horizon/userinput.json` clause of a Deployment Pattern `hzn register ...` command. The Deployment Policy approach for configuration values is more powerful because this operation can be performed centrally (no need to connect directly to the edge node).

1. Below is the file provided in  `horizon/deployment.policy.json` with this example:

```json
{
  "label": "$SERVICE_NAME Deployment Policy",
  "description": "A super-simple sample Horizon Deployment Policy",
  "service": {
    "name": "$SERVICE_NAME",
    "org": "$HZN_ORG_ID",
    "arch": "$ARCH",
    "serviceVersions": [
      {
        "version": "$SERVICE_VERSION",
        "priority":{}
      }
    ]
  },
  "properties": [
  ],
  "constraints": [
    "openhorizon.example == detector"
  ],
  "userInput": [
      {
        "serviceOrgid": "$HZN_ORG_ID",
        "serviceUrl": "$SERVICE_NAME",
        "serviceVersionRange": "[0.0.0,INFINITY)",
        "inputs": []
      }
  ]
}
```

- This simple example of a Deployment policy provides one `constraint` **openhorizon.example** that needs to be satisfied by one of the `properties` set in the `node.policy.json` file, so this Deployment Policy should successfully deploy our Edge Cluster Service onto the edge device.

- At the end, the userInput section has the same purpose as the `horizon/userinput.json` files provided for other examples if the given services requires them. In this case the example service defines does not have configuration variables.

2. If needed, run the following commands to set the environment variables needed by the `deployment policy.json` file in your shell:

```bash
export ARCH=$(hzn architecture)
eval $(hzn util configconv -f horizon/hzn.json)

optional: eval export $(cat agent-install.cfg)
```

3. Publish this Deployment policy to the Exchange to deploy the `json.exporter` service to the edge device (give it a memorable name):

```bash
make publish-deployment-policy
```

For example:

```bash
hzn exchange deployment addpolicy -f horizon/deployment.policy.json mycluster/edge-detector_1.0.0

```

4. Verify the Deployment policy:

```bash
hzn exchange deployment listpolicy mycluster/edge-detector_1.0.0
```

- The results should look very similar to your original `deployment.policy.json` file, except that `owner`, `created`, and `lastUpdated` and a few other fields have been added.



```json
{
  "mycluster/edge-detector_1.0.0": {
    "owner": "mycluster/ivanp",
    "label": "edge-detector Deployment Policy",
    "description": "A super-simple sample Horizon Deployment Policy",
    "service": {
      "name": "edge-detector",
      "org": "mycluster",
      "arch": "amd64",
      "serviceVersions": [
        {
          "version": "1.0.0",
          "priority": {},
          "upgradePolicy": {}
        }
      ],
      "nodeHealth": {}
    },
    "constraints": [
      "openhorizon.example == detector"
    ],
    "userInput": [
      {
        "serviceOrgid": "mycluster",
        "serviceUrl": "edge-detector",
        "serviceVersionRange": "[0.0.0,INFINITY)",
        "inputs": []
      }
    ],
    "created": "2020-10-12T18:22:02.925Z[UTC]",
    "lastUpdated": "2020-10-12T19:47:59.378Z[UTC]"
  }
}
```

- Now that you have set up the Deployment Policy and the published Service policy is in the exchange, we can move on to the final step of defining a Policy for your edge device to tie them all together and cause software to be automatically deployed on your edge device.

### Node Policy


- The node registration step will be completed in this section:

1. Below is the file provided in `policy/node.policy.json` with this Edge Cluster service:

```json
{
  "properties": [
    { "name": "openhorizon.example", "value": "detector" }
  ],
  "constraints": []
}
```

- It provides values for one `property` (**penhorizon.example**), that will affect which service(s) get deployed to this edge device, and states no `constraints`.


If you completed the edge registration as indicated in step 1, run the following command to update the edge device policy:

```bash
hzn policy update -f policy/node.policy.json
```
Otherwise, perform the edge device registration as follows:

```bash
hzn register -policy f horizon/node.policy.json
```
 - **Note**: using the `-s` flag with the `hzn register` command will cause Horizon to wait until agreements are formed and the service is running on your edge node to exit, or alert you of any errors encountered during the registration process.

Next, verify an agreement is reached with

```bash
hzn agreement list
```

Expecting a similar output to:

```json

```
 
 

2. After the agreement is made, list the docker container edge service that has been started as a result:

``` bash
sudo docker ps

CONTAINER ID        IMAGE                       COMMAND                  CREATED             STATUS              PORTS                  NAMES
```

3. See the Monitoring service output:

``` bash
curl localhost:5050
```

- **Note**: Press **Ctrl C** to stop the command output.

Expect to see an output similar to:

```text
```
