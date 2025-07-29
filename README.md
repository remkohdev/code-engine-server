# code-engine-server

## Client Server Communication in IBM Cloud Code Engine


## 1. Create App Server

Go to [code-engine-server](https://github.com/remkohdev/code-engine-server) repo.

```bash
pip install -r src/requirements.txt
uvicorn --app-dir src server:app --reload
```

## 2. Create Client App
Go to [code-engine-client](https://github.com/remkohdev/code-engine-client) repo.

```bash
pip install -r src/requirements.txt
python src/client.py
  {'Hello': 'World'}
```

### 3. Build Server Image

Using Quay.io:

```bash
podman build -t code-engine-server:v1.0.0 -f src/Containerfile .
podman run -d -p 8080:8080 --name code-engine-server code-engine-server

podman login quay.io -u remkohdev_us -p "Password123"
podman tag code-engine-server:v1.0.0 quay.io/remkohdev_us/code-engine-server
podman push quay.io/remkohdev_us/code-engine-server
```

[Using IBM Container Registry](https://cloud.ibm.com/containers/registry/start):

Create a namespace in your deployment region,

```bash
export REGION=ca-tor
export NAMESPACE=c2c-code-engine
ibmcloud login -sso
ibmcloud target -g remkohdev-rg -r $REGION

ibmcloud cr region-set $REGION
  The region is set to 'ca-tor', the registry is 'ca.icr.io'.
ibmcloud cr namespace-add $NAMESPACE
  Adding namespace 'c2c-code-engine' in resource group 'remkohdev-rg' for account Build Lab - Dev in registry ca.icr.io...
  Successfully added namespace 'c2c-code-engine'
```

Push image to IBM Cloud Registry:

```bash
ibmcloud cr login
  Logging 'podman' in to 'ca.icr.io'...
  Logged in to 'ca.icr.io'.
  OK

export REPO=code-engine-server
export TAG=v1.0.0
podman tag code-engine-server:$TAG ca.icr.io/$NAMESPACE/$REPO:$TAG
podman push ca.icr.io/$NAMESPACE/$REPO:$TAG
ibmcloud cr image-list
```

## 4. Deploy Server to Code Engine

See [Deploying application to IBM Code Engine](https://www.ibm.com/docs/en/api-connect/saas?topic=agent-deploying-application-code-engine).

1. Make sure the user in Code Engine has permissions to read and write to the Container Registry.

```bash
ibmcloud iam user-policy-create remkohdev@us.ibm.com --service-name container-registry --roles Manager
ibmcloud iam user-policies remkohdev@us.ibm.com
ibmcloud iam user-policy-create remkohdev@us.ibm.com --service-name container-registry --region ca-tor --resource-type namespace --resource remkohdev-ns --roles Reader,Writer

```

```bash
ibmcloud ce project create -n c2c-code-engine --tag remkohdev
  Creating project 'c2c-code-engine'...
  ID for project 'c2c-code-engine' is '53260710-8dcf-4ef6-ad00-abb9b5ac1e14'.
  Waiting for project 'c2c-code-engine' to be active...
  Now selecting project 'c2c-code-engine'.
  OK

ibmcloud ce project select -n c2c-code-engine

export IAM_APIKEY=<Your IAM apikey>
ibmcloud ce secret create --format registry --name c2c-code-engine-secret --server ca.icr.io --username iamapikey --password $IAM_APIKEY
  Creating registry secret 'c2c-code-engine-secret'...
  OK
ibmcloud ce secret create --format registry --name c2c-code-engine-private-secret --server private.ca.icr.io --username iamapikey --password $IAM_APIKEY
  Creating registry secret 'c2c-code-engine-private-secret'...
  OK

export BUILD_CONFIG=c2c-code-engine-server-build-config

ibmcloud ce build create --name $BUILD_CONFIG --image ca.icr.io/$NAMESPACE/$REPO:$TAG  --registry-secret c2c-code-engine-secret --build-type git --source https://github.com/remkohdev/code-engine-server --context-dir /src --strategy dockerfile --dockerfile Containerfile

ibmcloud ce build get --name $BUILD_CONFIG

ibmcloud cr va-version-set v4

ibmcloud ce buildrun submit --build $BUILD_CONFIG
ibmcloud ce buildrun list

ibmcloud ce buildrun logs -f -n c2c-code-engine-server-build-config-run-250602-1442530
ibmcloud ce buildrun get -n c2c-code-engine-server-build-config-run-250602-1442530

export APP=c2c-code-engine-server
ibmcloud ce app list
ibmcloud ce app delete --name $APP

ibmcloud ce app create --name $APP --image private.ca.icr.io/$NAMESPACE/$REPO:$TAG  --registry-secret c2c-code-engine-private-secret --port 8080

ibmcloud ce app get --name $APP
ibmcloud ce app events -n $APP
ibmcloud ce app logs -f -n $APP
```

## 5. Call Code Engine Server

```bash
curl https://code-engine-server.1yg2032o6egx.ca-tor.codeengine.appdomain.cloud/hello
  {"Hello":"World"}
```

## References

https://cloud.ibm.com/docs/codeengine?topic=codeengine-project-integrations#projectintegration-cr

https://cloud.ibm.com/docs/codeengine?topic=codeengine-add-registry#add-registry-access-ce-cli

https://cloud.ibm.com/docs/codeengine?topic=codeengine-deploy-app-crimage

https://cloud.ibm.com/docs/codeengine?topic=codeengine-deploy-app-private

## Quota Limits

```bash
ibmcloud ce build create --name c2c-code-engine-server-build --source https://github.com/remkohdev/code-engine-server --context-dir /src --registry-secret c2c-code-engine-secret  --strategy buildpacks
FAILED
The storage quota limit of the IBM Container Registry has been exceeded.
```

Check quota:

```bash
ibmcloud cr quota
  Getting quotas and usage for the current month, for account 'Build Lab - Dev'...
  Quota          Limit    Used
  Pull traffic   5.0 MB   0 B
  Storage        5.0 MB   130 MB

ibmcloud cr quota-set --traffic TRAFFIC_QUOTA --storage STORAGE_QUOTA
ibmcloud cr quota-set --storage 1000
  Setting quotas: 'storage: 1000'...
  FAILED
  The requested 'Storage' quota exceeds the pricing plan limit.
  The limit is 512 megabytes.
  ➜  code-engine-client git:(main) ibmcloud cr quota-set --storage 512 
  Setting quotas: 'storage: 512'...
  OK
```

Remove older images:

```bash
ibmcloud cr retention-policy-set --images 1 $NAMESPACE
ibmcloud cr images
  Listing images...
  Repository  Digest  Namespace  Created  Size  Security status
    ca.icr.io/c2c-code-engine/code-engine-server

ibmcloud cr retention-run --images 1 c2c-code-engine   
  Retrieving images to delete from namespace 'c2c-code-engine' in registry ca.icr.io'...
  Image     Tags
  ca.icr.io/c2c-code-engine/code-engine-server@sha256:00c7443ab3d6d8467b57708e3461f730a957786f640cf1d318ae8431dd0ac85d   
  Found 1 images to delete.
  Do you want to continue and delete the images? [y/N]> y
  Deleting 1 images...
  Successfully deleted 1 images.
  OK
```


## Todos

* Run client on Code Engine
* [Optional] Containerize


