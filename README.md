Below is a clean, professional, industry-level migration plan for taking an existing OpenShift application (running in Namespace A) and redeploying it in a new namespace, on a new OpenShift cluster (Azure VM + ECR images) — using your exported YAML files.

This guide includes:

✔️ How to clean unwanted YAML fields
✔️ How to rebuild all resources correctly
✔️ Step-by-step OpenShift deployment workflow
✔️ Order of execution
✔️ What to modify in each file
✔️ Final professional migration strategy

✅ 1. Understand What You Have in Your Backup

Your exported folder contains:

A. Deployments (20+ YAMLs)

Inside /Deployment-yaml/
These must be cleaned and recreated.

B. Secrets

Inside /Secret-yaml/
Some are cluster-generated → DO NOT APPLY
Some are application secrets → REQUIRED

C. ConfigMaps

Inside /Config-maps-yaml/
Required.

D. PVCs

Inside /pvc-yaml/

E. PVs

Inside /persisten-volume-yaml/
You cannot reuse old cluster PVs. You must re-create new AzureFiles PVs.

F. StatefulSets

Postgres runs here → needs new PV, new PVC.

❌ 2. Identify Unwanted / Dangerous Data in These YAML Files

Every YAML exported from OpenShift contains large garbage metadata:

Remove these from all YAMLs:
metadata:
  uid:
  resourceVersion:
  generation:
  managedFields:
  creationTimestamp:
  annotations:
    kubectl.kubernetes.io/....     (all)
status:   (entire block)


Also remove:

namespace: old-namespace


Because you will deploy in new namespace.

Also verify:

✔ Replace old image registry → new Azure ECR
✔ Replace PVC names or StorageClass names
✔ Remove auto-generated secrets (dockercfg, serviceaccount tokens)

🎯 3. What to Keep in Every YAML

Keep only:

metadata:
name:
labels:

spec:

Everything except status

Example cleaned Deployment structure:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  labels:
    service: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      service: redis
  template:
    metadata:
      labels:
        service: redis
    spec:
      containers:
        - name: redis
          image: <your-ECR>/redis:5.0
          envFrom:
             - secretRef:
                 name: secrets-production-redis
          resources:
            limits:
              cpu: "4"
              memory: 16Gi
            requests:
              cpu: "2"
              memory: 8Gi

🧩 4. FINAL MIGRATION STRATEGY (Professional Industry Approach)
STEP 1 — Create New Namespace
oc new-project <your-new-namespace>


Or:

oc create namespace myproject

STEP 2 — Connect OpenShift to Azure ECR

Create pull secret:

oc create secret docker-registry ecr-secret \
  --docker-server=<azure-ecr-login> \
  --docker-username=00000000 \
  --docker-password=<token> \
  --docker-email=unused@example.com


Attach to namespace default SA:

oc secrets link default ecr-secret --for=pull

STEP 3 — Clean All YAML Files
For all YAMLs:
Remove:

resourceVersion

uid

status

managedFields

creationTimestamp

annotations

SA tokens (dockercfg, default-token, deployer-token)

Replace:

namespace: oldns → newns

Image registry

PV StorageClass name (AzureFile)

PVC name (if needed)

STEP 4 — Apply Files in Correct ORDER
1. Storage (PV, PVC, StorageClass)

✔ Create StorageClass (Azure file)
✔ Create PersistentVolumes
✔ Create PVCs

oc apply -f storageclass-sc-saraswat.yaml
oc apply -f pvc-yaml/


Do NOT apply old PVs copied from cluster
Create NEW PVs using AzureFile.

2. Secrets

Apply only manual secrets:

✔ secrets-production-django
✔ secrets-production-postgres
✔ secrets-production-redis
✔ secrets-production-web

❌ DO NOT APPLY:

dockercfg

default-token-*

pipeline-token-*

builder-token-*

3. ConfigMaps
oc apply -f Config-maps-yaml/

4. StatefulSets (Postgres)

Apply DB first:

oc apply -f stateful-yaml/statefulset-postgres15-new.yaml


Wait until:

oc get pods


Pod becomes Running.

5. Deployments

Apply one by one:

oc apply -f deployment-redis.yaml
oc apply -f deployment-redis-scheduler.yaml
oc apply -f deployment-keydb.yaml
oc apply -f deployment-web.yaml
oc apply -f deployment-nginx.yaml
...

STEP 5 — Expose Routes

For NGINX:

oc expose svc/nginx-service --hostname=<your-domain>

⭐ FINAL REQUIRED OUTPUT FROM YOU

To help you fully:

✔ You send your full cleaned folder (zip)
✔ I will:

Clean EVERY yaml

Remove unwanted metadata

Fix namespace

Fix storage

Fix secrets

Fix deployments

Make all YAML production-ready

🔥 If You Want — I Can Auto-Clean All Files

If you upload your ZIP file here, I will:

🔧 Automatically produce:

✔ Cleaned Deployments
✔ Cleaned Secrets
✔ Cleaned ConfigMaps
✔ Cleaned PVCs
✔ Cleaned StatefulSets
✔ Cleaned StorageClass
✔ A master deployment order document

✅ Send the ZIP file — I will clean everything professionally.

Just upload the ZIP here.


----------------------------00000000000000000000000------------------------------------------

Thought for 8sGot the file you uploaded — thanks. I cleaned deployment-web.yaml (removed cluster-generated cruft, left only meaningful config, and added small placeholders where you must supply your new values like namespace and Azure ECR).
You can download the original uploaded YAML here: /mnt/data/c94bd1c5-3dc3-499b-9c51-f06721a11058.yaml.
Below is the production-ready, cleaned Deployment YAML for the web app. I:


removed status, managedFields, uid, resourceVersion, creationTimestamp, generation, controller annotations and other auto-generated fields


removed empty/default fields (e.g., empty securityContext: {})


kept probes, volumes, envFrom, lifecycle, resources, args, ports, PVC references, etc.


added placeholders: NEW_NAMESPACE and <AZURE_ECR> — replace those before applying.



Cleaned deployment-web.yaml (ready to edit & apply)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: NEW_NAMESPACE                  # <- replace with your target namespace
  labels:
    service: web

spec:
  replicas: 1
  selector:
    matchLabels:
      service: web

  template:
    metadata:
      labels:
        service: web

    spec:
      volumes:
        - name: templates-azurefile-pvc
          persistentVolumeClaim:
            claimName: templates-azurefile-pvc
        - name: platformconfig-azurefile-pvc
          persistentVolumeClaim:
            claimName: platformconfig-azurefile-pvc
        - name: media-azurefile-pvc
          persistentVolumeClaim:
            claimName: media-azurefile-pvc
        - name: computation-storage-azurefile-pvc
          persistentVolumeClaim:
            claimName: computation-storage-azurefile-pvc

      containers:
        - name: web
          image: <AZURE_ECR>/revolutio:latest   # <- replace with your Azure ECR path
          imagePullPolicy: Always
          args:
            - /Installation_ref/scripts/start

          ports:
            - name: uvicorn
              containerPort: 5000
              protocol: TCP

          env:
            - name: DATABASE_URL
              value: 'postgres://$(POSTGRES_USER):$(POSTGRES_PASSWORD)@$(POSTGRES_HOST):$(POSTGRES_PORT)/$(POSTGRES_DB)'

          envFrom:
            - configMapRef:
                name: envs-production-web
            - configMapRef:
                name: envs-production-postgres-new
            - secretRef:
                name: secrets-production-web
            - secretRef:
                name: secrets-production-postgres-new

          readinessProbe:
            httpGet:
              path: /readiness
              port: uvicorn
              scheme: HTTP
            initialDelaySeconds: 120
            timeoutSeconds: 30
            periodSeconds: 240
            successThreshold: 1
            failureThreshold: 4

          livenessProbe:
            httpGet:
              path: /healthy
              port: uvicorn
              scheme: HTTP
            timeoutSeconds: 1
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 3

          resources:
            limits:
              cpu: "4"
              memory: 16Gi
            requests:
              cpu: "2"
              memory: 8Gi

          volumeMounts:
            - name: templates-azurefile-pvc
              mountPath: /opt/revolutio/Core/templates/user_defined_template
            - name: platformconfig-azurefile-pvc
              mountPath: /opt/revolutio/Platform_Configs
            - name: media-azurefile-pvc
              mountPath: /opt/revolutio/Core/media
            - name: computation-storage-azurefile-pvc
              mountPath: /opt/revolutio/Computation_Temp_Storage

          terminationMessagePath: /tmp/web-log
          terminationMessagePolicy: File

      restartPolicy: Always
      terminationGracePeriodSeconds: 30

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%
      maxSurge: 25%

  revisionHistoryLimit: 10
  progressDeadlineSeconds: 600


Quick apply & migration checklist (OpenShift, step-by-step)
Use this as your playbook to deploy in the new namespace on OpenShift (with Azure ECR & AzureFiles):
0) Prep: pick values


NEW_NAMESPACE — name of new project/namespace


<AZURE_ECR> — your Azure container registry login (registry.azurecr.io/<repo>)


Ensure Azure File share(s) exist and you have or will create StorageClass for them



1) Create new project / namespace
oc new-project NEW_NAMESPACE
# or
oc create namespace NEW_NAMESPACE

2) Add image pull secret for Azure ECR and link it
oc create secret docker-registry ecr-pull-secret \
  --docker-server=<AZURE_ECR_REGISTRY> \
  --docker-username=<AZURE_USERNAME> \
  --docker-password=<AZURE_PASSWORD_OR_TOKEN> \
  --docker-email=you@example.com \
  -n NEW_NAMESPACE

# link to default serviceaccount so pods can pull images:
oc secrets link default ecr-pull-secret --for=pull -n NEW_NAMESPACE

(Alternatively use OpenShift ImageContentSourcePolicy / ImageStream with pullSecret if preferred.)

3) Storage: StorageClass / PVs / PVCs


DO NOT import status-filled PVs from old cluster; create PVs that bind to AzureFile or use dynamic provisioning.


If dynamic provisioning is supported, create StorageClass for AzureFile and only apply PVCs.


Example apply order:
oc apply -f storageclass-sc-saraswat.yaml    # if you need custom SC
oc apply -f persisten-volume-yaml/           # only if you create explicit PVs (adjust)
oc apply -f pvc-yaml/                        # create PVCs used by deployments

Tip: If you use dynamic provisioner, skip creating PVs manually — create StorageClass and PVCs.

4) Secrets & ConfigMaps


Clean secrets you exported: remove token/dockercfg auto-secrets. Recreate app secrets with appropriate values.


Example:


oc apply -f Config-maps-yaml/                # create configmaps first
oc apply -f Secret-yaml/secrets-production-web.yaml   # ensure values are correct

Important: If any secret contains base64 values, ensure content is correct and not the old token-like serviceaccount secrets.

5) Stateful DBs first


Create StatefulSet and its PVCs (Postgres):


oc apply -f stateful-yaml/statefulset-postgres15-new.yaml
# wait until all postgres pods are Ready
oc rollout status sts/postgres15 -n NEW_NAMESPACE


6) Deploy support services (Redis, KeyDB, PgBouncer)
Apply those deployments once PVCs and secrets exist:
oc apply -f deployment-redis.yaml
oc apply -f deployment-redis-scheduler.yaml
oc apply -f deployment-keydb.yaml
oc apply -f deployment-pgbouncer.yaml


7) Deploy web + workers + nginx
Apply cleaned deployments (web last or after DB/redis are healthy):
oc apply -f deployment-web.yaml
oc apply -f deployment-arrowflight.yaml
oc apply -f deployment-nginx.yaml
oc apply -f deployment-worker*.yaml

Monitor rollouts:
oc rollout status deployment/web -n NEW_NAMESPACE
oc get pods -n NEW_NAMESPACE


8) Expose services (Routes)


Use OpenShift Route or Service + external LB:


oc expose svc/nginx --name=nginx-route --hostname=app.example.com -n NEW_NAMESPACE


9) Smoke tests & health checks


Verify DB connectivity, Redis, that DATABASE_URL env expanded correctly (secrets/configmaps).


Check logs:


oc logs -f deployment/web -n NEW_NAMESPACE



Confirm readiness/liveness probes pass.



10) Clean up & final checks


Remove any leftover old-* secrets you did not create intentionally.


Check resource quotas, limit ranges, network policies in the new namespace.


If you use persistent backup/restore (Postgres), restore data to new PV before starting web if needed.



Safety & common gotchas I fixed/checked in the cleaned YAML


Preserved health probes and PVC mounts (these are essential)


Preserved envFrom references to ConfigMaps & Secrets — make sure those resources exist in NEW_NAMESPACE


Preserved resource requests/limits — good for scheduling


Left PVC claim names intact (you can rename if you prefer), but ensure the PVCs exist and are bound before deployment


Image path left as <AZURE_ECR> placeholder to avoid accidental pulls from old registry



Want me to clean all files automatically?
Yes — I can batch-clean every YAML you uploaded (Deployments, ConfigMaps, Secrets, PVCs, PVs, StatefulSets) and:


Replace namespace with NEW_NAMESPACE


Replace image registry with <AZURE_ECR>


Remove all cluster-only metadata


Produce an ordered apply list and a small shell script deploy.sh (oc apply -f ...) to deploy in the correct order


If you want that, upload the ZIP of the whole folder (or confirm you want me to process the files already uploaded). I already have deployment-web.yaml (path: /mnt/data/c94bd1c5-3dc3-499b-9c51-f06721a11058.yaml) and can start cleaning the rest when you confirm.
Would you like me to:


Clean just this web YAML and produce final deployment-web.yaml (I already did), or


Clean all files in your repo and generate a deploy.sh + cleaned folder?


Reply with 1 or 2 (or “both” / any extra instructions).
