# Model-as-a-Service with Red Hat OpenShift AI Demo Setup and Walkthrough

## For RH-ers
Order the environment:
- `Red Hat Openshift AI 3`
- GPU Instance Type: `g6.8xlarge` for running one small model at a time
  - `g6.12xlarge` for running one larger model, or multiple smaller models  

*** Note *** This demo environment may require multiple attempts to provision if AWS quotas are exceeded. Retry in different regions if provisioning fails.

This environment needs to be patched and and additional resources are needed in order to support MaaS and AI Playground

### Post Environment Setup

#### RHOAI DSC config

1. Gen AI studio / Playground: two settings

`DataScienceCluster`: turn on the `OGX Operator`, which is the backend that runs the playground. The docs say to set its managementState field to Managed in the `DataScienceCluster` custom resource (CR) of the OpenShift AI Operator.

```bash
# Patch command that sets the above (and removes llamastack)

oc patch datasciencecluster default-dsc --type=merge \
  -p '{"spec":{"components":{"llamastackoperator":{"managementState":"Removed"},"ogx":{"managementState":"Managed"}}}}'
```

Turn on the Gen AI studio UI
The `gen-ai-ui` pod is already running, but the dashboard hides it until this flag is set:

```bash
oc patch OdhDashboardConfig odh-dashboard-config -n redhat-ods-applications \
  --type=merge -p '{"spec":{"dashboardConfig":{"genAiStudio":true}}}'
```

2. Enable AI GW

```bash
oc patch datasciencecluster default-dsc --type=merge \
  -p '{"spec":{"components":{"aigateway":{"managementState":"Managed"}}}}'
```

Pods can be monitored in the `redhat-ods-applications` namespace

```bash
oc rollout status deployment -n redhat-ods-applications 
oc get pods -n redhat-ods-applications
```

Look for the additional pods
```
opendatahub-ogx-operator-xxx                1/1     Running   0              90s
ogx-k8s-operator-controller-manager-xxx     1/1     Running   0              76s
ai-gateway-operator-xxx                     1/1     Running   0              24s
```

#### `AI Gateway for MaaS Prep`

1. In order to use the `AI Gateway` the `Red Hat Connectivity Link 1.4.x` operator needs to be installed (tested with `1.4.4` to match existing Authorino version installed)

```bash
oc apply -f k8s/operators/rhcl.yaml
```
or install vial the web console

Once the operator is ready, create a minimal `kuadrant-system`

```bash
oc create namespace kuadrant-system
oc apply -f k8s/ai-gw/kuadrant.yaml -n kuadrant-system 
oc rollout status deployment -n kuadrant-system 
oc get kuadrant kuadrant -n kuadrant-system -o yaml | grep -A3 ready # wait for Ready
```

Example output of last command
```bash
oc get kuadrant kuadrant -n kuadrant-system -o yaml | grep -A3 ready # wait for Ready
    message: Kuadrant is ready
    reason: Ready
    status: "True"
    type: Ready
```

2. Enable User Workload Monitoring

Without it, MaaS reports `Degraded`. Check whether it’s already on:

```bash
oc -n openshift-user-workload-monitoring get pods
```

If no pods are show, user workload monitoring has not been enabled.   
  
To enable user workload monitoring, apply the following configmap:

```bash
oc apply -f k8s/observability/configmap.yaml
```

And verify it is up and running

```bash
oc rollout status deployment -n openshift-user-workload-monitoring 
oc -n openshift-user-workload-monitoring get pods

NAME                                   READY   STATUS    RESTARTS   AGE
prometheus-operator-5d4ff97ff4-bcxcz   2/2     Running   0          37s
prometheus-user-workload-0             6/6     Running   0          35s
thanos-ruler-user-workload-0           4/4     Running   0          35s
```

3. Create default `GatewayClass`

```bash
oc apply -f k8s/ai-gw/gatewayclass.yaml
```


4. MaaS Gateway setup with default certificate

```bash
# Find the wildcard certificate secret
export CERT_SECRET=$(oc get ingresscontroller default -n openshift-ingress-operator \
  -o jsonpath='{.spec.defaultCertificate.name}{"\n"}')
oc get secret $CERT_SECRET -n openshift-ingress

# Domain of cluster
export DOMAIN=$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')
echo $DOMAIN

# Apply using environment variables
envsubst < k8s/ai-gw/maas-gateway.yaml | oc apply -f -
```

Check gateway status example:
```bash
oc rollout status deployment -n openshift-ingress 
oc get gateway -n openshift-ingress maas-default-gateway 

NAME                   CLASS               ADDRESS                     PROGRAMMED   AGE
maas-default-gateway   openshift-default   xxx.xxx.elb.amazonaws.com   True         42s
```

Create the default gateway TLS secret as a copy of the wildcard certificate:

```bash
oc extract secret/$CERT_SECRET -n openshift-ingress --to=/tmp/wildcard --confirm
oc create secret tls default-gateway-tls -n openshift-ingress \
  --cert=/tmp/wildcard/tls.crt --key=/tmp/wildcard/tls.key
rm -rf /tmp/wildcard
```

Configure Authorino TLS

```bash
oc annotate service authorino-authorino-authorization -n kuadrant-system \
  service.beta.openshift.io/serving-cert-secret-name=authorino-server-cert --overwrite

oc patch authorino authorino -n kuadrant-system --type=merge -p \
  '{"spec":{"listener":{"tls":{"enabled":true,"certSecretRef":{"name":"authorino-server-cert"}}}}}'

oc -n kuadrant-system set env deployment/authorino \
  SSL_CERT_FILE=/etc/ssl/certs/openshift-service-ca/service-ca-bundle.crt \
  REQUESTS_CA_BUNDLE=/etc/ssl/certs/openshift-service-ca/service-ca-bundle.crt
```
#### `DB for MaaS Prep`

MaaS uses PostgreSQL to manage API keys. The 3.5 doc says the database is required for API key lifecycle management, and the 3.4 migration notes describe it as storing subscription, API key, and usage data.

```bash
oc create namespace maas-postgres 

export DB_PASS=$(openssl rand -hex 16)

oc create secret generic maas-postgres-creds -n maas-postgres \
  --from-literal=POSTGRESQL_USER=maas \
  --from-literal=POSTGRESQL_PASSWORD="$DB_PASS" \
  --from-literal=POSTGRESQL_DATABASE=maasdb

oc apply -f k8s/ai-gw/maas-db.yaml -n maas-postgres 
oc rollout status deployment/maas-postgres -n maas-postgres
```

Smoke test db access

```bash
oc exec -n maas-postgres deploy/maas-postgres -- \
  psql -U maas -d maasdb -c 'select version();'
```

### Enable MaaS on DSC

```bash
oc patch datasciencecluster default-dsc --type=merge -p \
  '{"spec":{"components":{"aigateway":{"managementState":"Managed","modelsAsAService":{"managementState":"Managed"}}}}}'
sleep 2
oc rollout status deployment -n redhat-ods-applications
```

Wait for the infrastructure namespace

```bash
oc get maastenantconfig default-tenant -n models-as-a-service \
  -o jsonpath='{.status.infraNamespace}{"\n"}'
```
This should return `redhat-ai-gateway-infra`

Create the secret for db credentials and restart `maas-api`

```bash
export INFRA_NS=redhat-ai-gateway-infra   # or whatever step 2 printed

export DB_PASS=$(oc get secret maas-postgres-creds -n maas-postgres -o jsonpath='{.data.POSTGRESQL_PASSWORD}' | base64 -d)

oc create secret generic maas-db-config -n $INFRA_NS \
  --from-literal=DB_CONNECTION_URL="postgresql://maas:${DB_PASS}@maas-postgres.maas-postgres.svc.cluster.local:5432/maasdb?sslmode=disable"

oc rollout restart deployment/maas-api -n $INFRA_NS
oc rollout status deployment/maas-api -n $INFRA_NS
```

Dashboard flags to enable MaaS management

```bash
oc patch OdhDashboardConfig odh-dashboard-config -n redhat-ods-applications --type=merge -p \
'{"spec":{"dashboardConfig":{"modelAsService":true,"vLLMDeploymentOnMaaS":true}}}'
```

For when you are setting up differnt TRLP for models and users

### MaaS Operations in RHOAI

First make sure out namespace is labeled properly
```bash
oc label namespace my-first-model opendatahub.io/dashboard=true  
```

Open up the RHOAI Web Console

Stop the running model via the UI to free up GPU resources

#### Deploy the model

1. Open the OpenShift AI dashboard and go to `Projects → my-first-model → Deployments → Deploy model`.
2. `Model details`
  - `Model location`: Existing connection
  - Connection and model path: the same as your existing `llama-32-3b-instruct`.
  - Model type: `Generative AI model`.
  - Click `Next`.
3. Model deployment
  - Name: `llama-32-3b-maas`
  - `Deeployment method`: `LLM inference service`
  - Hardware profile: `gpu-profile` (not default)
  - Accelerator configuration -> Manual selection: `vLLM NVIDIA CUDA GPU LLMInferenceServiceConfig Single Node`
  - Replicas: `1`
  - Click `Next`.
4. Advanced settings
  - Check `Publish as MaaS`.
  - If it’s offered, check Add as `AI asset endpoint`.
  - Click `Add custom runtime arguments` and paste:

```
--dtype=half
--max-model-len=32768
--gpu-memory-utilization=0.95
--enable-auto-tool-choice
--tool-call-parser=llama3_json
--chat-template=/opt/app-root/template/tool_chat_template_llama3.2_json.jinja
```
  - Click `Next`.
5. `Review`, then click `Deploy model`.

Once the model status is ready, check the inference service

```bash
oc get llminferenceservice -n my-first-model 
```

Example output
```
NAME                  URL                                                                   READY   REASON   AGE
llama-32-3b-maas-v2   https://maas.apps.ocp.<domain>.com/my-first-model/llama-32-3b-maas    True             32m
```

### Managing MaaS for consumption

#### Set up group and user for MaaS consumers (admin is the only account I have at the moment)

```bash
oc adm groups new maas-users admin 
oc get group maas-users
```

Note: you could set up multiple groups such as:
- `maas-users-gold-tier`
- `maas-users-silver-tier`
- `maas-users-bronze-tier`

#### Set up subscription tiers

1. Go to Settings → MaaS governance → Create subscription and fill it in:
  - Name: `sandbox`
  - Priority: `0`
  - Groups: `maas-users`, or `system:authenticated`
  - Models: click `Add models`, select `llama-32-3b-maas`, then click   Add token limit . Use `10000` tokens per `1 minute` (or whatever you want for testing).
2. Check `Create authorization policy`.
3. Click `Create subscription`.

Why create an authorization policy?

A `subscription` (MaaSSubscription) defines which models should be available to certain groups on request, but it does not grant access to those models on its own.


An `authorization policy` (MaaSAuthPolicy) is a separate resource that authorizes specific groups to be able to access model endpoints through the API gateway.


Both resources are needed in order to consume model endpoints through the A(P)I gateway

#### Create an API key
1. Go to `Gen AI studio → API keys → Create API key`.
2. Select the `sandbox` subscription.
3. Click `Create`, and copy the `sk-oai-...` key right away. It’s shown only once!

##### Test your key
```bash
KEY=sk-oai-...
HOST="maas.$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')"

curl -s -H "Authorization: Bearer $KEY" "https://${HOST}/maas-api/v1/models" | jq

curl -s -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  "https://${HOST}/v1/chat/completions" \
  -d '{"model":"publishers/my-first-model/models/llama-32-3b-maas","messages":[{"role":"user","content":"hello"}],"max_tokens":20}' | jq
```

