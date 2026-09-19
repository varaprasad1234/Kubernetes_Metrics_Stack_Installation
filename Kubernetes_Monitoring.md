 You have been tasked with setting up a full observability platform for your Kubernetes infrastructure. To accomplish this, you need to set up a default StorageClass using the local path provisioner, add buckets to SeaweedFS for Loki, then set up Grafana, the Prometheus Stack, and Loki using Helm based on your organization's needs. These needs include the following:

The observability platform components deployed to a dedicated observability namespace.
Persistent data storage for Grafana, Prometheus, and AlertManager using the local path.
Public access to the Grafana UI on port 30081, and
public access to the Prometheus UI on port 30082.
Helm has already been configured, alongside SeaweedFS, which can be accessed using the IP address for either node, Worker Node 1 Public IP or Worker Node 2 Public IP on port 30080. Access key and secret key permission for the storage solution can be found in the cloud_user's home directory under seaweed_creds.

Copies of the end-state values files for each component can be found as hidden files in the Values directory. Use ls -al to view the file names to review and copy data from these files as needed.

All work should be done via the provided Workstation server, which has kubectl installed and preconfigured to work with the supplied lab environment. To confirm everything is working, use SSH to connect using the Workstation Public IP

ssh cloud_user@<Workstation Public IP>
Then run:

kubectl get nodes
All nodes should be in a Ready state upon lab start.

You will also need to use your web browser to access the SeaweedFS and Grafana UIs later on to complete this lab.
And, this lab will use vim to edit files. If you wish, you can instead use nano.
Prepare the environment
Before configuring any of the core observability components, you first need to finish up some general environmental configuration:

Creating the observability namespace
Setting up the default StorageClass
Creating the SeaweedFS buckets for Loki
Create namespace
Create the observability namespace:

kubectl create namespace observability
You can confirm the namespace's creation with:

kubectl get ns
The output will include the observability namespace.

Set the default StorageClass
Prior to setting the default StorageClass, you first need to apply the local path provisioning. This can be done with:

kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
Verify the StorageClass resource is now available:

kubectl get storageclass
Once confirmed, patch the StorageClass to make it the default:

kubectl patch storageclass local-path \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
Confirm that the StorageClass is listed as default by running kubectl get storageclass again:

kubectl get storageclass
You should see (default) next the the local-path name:

NAME
local-path (default)
Create SeaweedFS buckets
To create buckets within SeaweedFS, you can run a one-off kubectl run command, or can visit the SeaweedFS UI at <Worker Node 1 Public IP>:30080 in your browser.

You will prepend loki- to the needed bucket names so they are as follows:

loki-chunks
loki-ruler
loki-admin
This is done because admin is a restricted bucket name in SeaweedFS.

Create the buckets:

kubectl run awscli-create-buckets \
	-n seaweedfs \
	--rm -i \
	--restart=Never \
	--image=amazon/aws-cli \
	--env AWS_ACCESS_KEY_ID=observabilityaccess \
	--env AWS_SECRET_ACCESS_KEY=observabilitysecretkey \
	--env AWS_DEFAULT_REGION=us-east-1 \
	--command -- sh -c '
	  aws --endpoint-url http://seaweedfs-s3.seaweedfs.svc.cluster.local:8333 s3 mb s3://loki-chunks || true
	  aws --endpoint-url http://seaweedfs-s3.seaweedfs.svc.cluster.local:8333 s3 mb s3://loki-ruler || true
	  aws --endpoint-url http://seaweedfs-s3.seaweedfs.svc.cluster.local:8333 s3 mb s3://loki-admin || true
	  aws --endpoint-url http://seaweedfs-s3.seaweedfs.svc.cluster.local:8333 s3 ls
'
To confirm, you can navigate to <Worker Node 1 Public IP>:30080 in your browser, and look under the buckets folder.

Use the lab page's Worker Node 1 Public IP. As previously mentioned, you can use Node 2 as well.
It's not secure, which is okay for this lab, so you can click Continue to site on the warning pop-up. This pop-up may differ a bit, depending on your browser
The following can also be done to similarly confirm:

kubectl run awscli-list-buckets \
  -n seaweedfs \
  --rm -i \
  --restart=Never \
  --image=amazon/aws-cli \
  --env AWS_ACCESS_KEY_ID=observabilityaccess \
  --env AWS_SECRET_ACCESS_KEY=observabilitysecretkey \
  --env AWS_DEFAULT_REGION=us-east-1 \
  --command -- aws \
    --endpoint-url http://seaweedfs-s3.seaweedfs.svc.cluster.local:8333 \
    s3api list-buckets
This should return something similar to the following:

    "Buckets": [
        {
            "Name": "loki-admin",
            "CreationDate": "2026-05-14T14:33:24+00:00"
        },
        {
            "Name": "loki-chunks",
            "CreationDate": "2026-05-14T14:33:22+00:00"
        },
        {
            "Name": "loki-ruler",
            "CreationDate": "2026-05-14T14:33:23+00:00"
        }
Install and configure Grafana
Next, you need to set up Grafana, a data visualization platform whose UI will work as a base of operations for the other components in the observability platform. This should be done via Helm, using a values file. But first, you need to create a Kubernetes Secret to store the access information for the UI.

Create Kubernetes Secret
Create the Kubernetes Secret in the observability namespace via the command line. Update the password to use a more secure password if desired:

kubectl create secret generic grafana-admin-credentials -n observability --from-literal=admin-user=admin --from-literal=admin-password='averylongpassword'
Note: Keep this password in mind, as you will need to use it when logging in to Grafana.

You can confirm the secret was created by running:

kubectl get secrets -n observability
It should return:

NAME                        TYPE     DATA   AGE
grafana-admin-credentials   Opaque   2      26s
Create Values file
Move into the Values folder from the cloud_user's home directory:

cd Values
Using your preferred text editor, create a file called grafana-values.yaml. This guide will use vim:

vim grafana-values.yaml
Call the admin parameters via the Kubernetes Secret you just created, and expose the Grafana UI on port 30081 using the NodePort option:

admin:
  existingSecret: grafana-admin-credentials
  userKey: admin-user
  passwordKey: admin-password

service:
  type: NodePort
  nodePort: 30081
  
persistence:
  enabled: true
  storageClassName: local-path
  accessModes:
    - ReadWriteOnce
  size: 5Gi
Note:

A copy of this values file can be found at /home/cloud_user/Values/.grafana-values.yaml.
When using vim, to paste in the YAML contents while maintaining the required spacing, first issue :set paste. Do this later in the lab when needed as well.
Save and exit the file.

Install Grafana
Add the Grafana Community Helm repo:

helm repo add grafana-community https://grafana-community.github.io/helm-charts
helm repo update
Using Helm, install Grafana into the observability namespace:

helm install grafana grafana-community/grafana \
  -n observability \
  -f grafana-values.yaml
Access Grafana
Finally, confirm Grafana is working by doing the following:

In a new browser tab, access the site at <Worker Node 1 Public IP>:30081.

It will take a few seconds for the site to load.

Enter the admin username, and the password you created earlier and stored in the Kubernetes Secret.

Close any informational pop-ups that may appear.

Leave the UI up in a tab while completing the rest of the lab; it will be used later.

Install and configure the Prometheus stack
You now need to perform the same steps to deploy the Prometheus stack, which includes Prometheus, AlertManager, the Node Exporter, and kube-state-metrics for metrics and monitoring. Ensure the Prometheus UI is exposed on port 30082, and that persistence is enabled from Prometheus and AlertManager using the local path provisioner.

Create Values file
Create a prom-stack-values.yaml file under the same Values directory using your preferred text editor:

vim prom-stack-values.yaml
Put in the following contents:

grafana:
  enabled: false

prometheus:
  prometheusSpec:
    serviceMonitorSelectorNilUsesHelmValues: false
    retention: 24h
    retentionSize: 4GB
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: local-path
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 5Gi
  service:
    type: NodePort
    nodePort: 30082

alertmanager:
  enabled: true
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: local-path
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 5Gi

kube-state-metrics:
  enabled: true

nodeExporter:
  enabled: true
Notice that Grafana is disabled since it was installed separately. Also notice the subtle differences in configuration between Prometheus and AlertManager with setting up storage–notably the use of storageSpec versus storage. You can view a full list of configurations for this file by running the command helm show values oci://ghcr.io/grafana-community/helm-charts/grafana outside of your text editor, if desired.

Note that a copy of this values file can be found at /home/cloud_user/Values/.prom-stack-values.yaml.

Save and exit the file.

Install the Prometheus Stack
Add the Prometheus Community Helm chart:

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
Install the Prometheus stack into the observability namespace using the values file you created:

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  -n observability \
  -f prom-stack-values.yaml
Once finished, you can confirm the installation by checking the status of the pods used by Prometheus:

kubectl -n observability get pods -l "release=kube-prometheus-stack"
Wait for a bit until all pods display as being READY, re-running the get command periodically until they all show 1/1. This may take about 30 seconds.

Add as Grafana data source
Once the pods are up and running, return to the Grafana UI to add Prometheus as a data source.

From the UI (close any pop-ups!), if needed in the upper-left first click the ☰ button.

Do this if needed later in the lab.

In the left-menu, navigate to Connections, then click Add new connection.

Search for and then click Prometheus.

In the upper-right, click Add new data source.

On the prometheus page, in the Connection section, enter the following:

Prometheus server URL: http://kube-prometheus-stack-prometheus:9090
Scroll to the bottom of the page and click Save & test.

A green box will verify things worked, displaying

Successfully queried the Prometheus API.

Install and configure Loki
Finally, the preliminary steps for log collection should be set up by installing and configuring Loki. Remember the bucket names you created earlier for this step:

loki-chunks
loki-ruler
loki-admin
Create Kubernetes Secret
Loki will need access to SeaweedFS, so you first need to create a Kubernetes Secret under the observability namespace to use the access and secret key information for SeaweedFS. This is separate from the Kubernetes Secret you can see under the Manifests directory, which was used to set up the Kubernetes Secret under the seaweedfs namespace.

Reminder: Kubernetes Secrets cannot cross namespaces.

Create the Secret:

kubectl create secret generic seaweedfs-s3-config \
  -n observability \
  --from-literal=AWS_ACCESS_KEY_ID=observabilityaccess \
  --from-literal=AWS_SECRET_ACCESS_KEY=observabilitysecretkey
Confirm by viewing the available secrets in the observability namespace:

kubectl get secrets -n observability
The secret you just created, seaweedfs-s3-config, will be listed.

Note that the Prometheus stack has created a number of secrets too, which will also be listed.

Create values file
Using your preferred text editor, create and open loki-values.yaml. You should still be working from the Values directory:

vim loki-values.yaml
Configure Loki to work as a single binary with no replicas, and pull the secrets data from the seaweedfs-s3-config secret. Note that under storage the bucket names should match the buckets set up at the beginning of the lab.

deploymentMode: SingleBinary

singleBinary:
  replicas: 1
  persistence:
    enabled: true
    storageClass: local-path
    size: 5Gi
  extraArgs:
    - -config.expand-env=true
  extraEnvFrom:
    - secretRef:
        name: seaweedfs-s3-config

read:
  replicas: 0

write:
  replicas: 0

backend:
  replicas: 0

chunksCache:
  enabled: false

resultsCache:
  enabled: false

loki:
  auth_enabled: false

  commonConfig:
    replication_factor: 1

  schemaConfig:
    configs:
      - from: "2024-01-01"
        store: tsdb
        object_store: s3
        schema: v13
        index:
          prefix: loki_index_
          period: 24h

  storage:
    type: s3
    bucketNames:
      chunks: loki-chunk
      ruler: loki-ruler
      admin: loki-admin
    s3:
      endpoint: seaweedfs-s3.seaweedfs.svc.cluster.local:8333
      region: us-east-1
      accessKeyId: ${AWS_ACCESS_KEY_ID}
      secretAccessKey: ${AWS_SECRET_ACCESS_KEY}
      s3ForcePathStyle: true
      insecure: true

  storage_config:
    tsdb_shipper:
      active_index_directory: /var/loki/tsdb-index
      cache_location: /var/loki/tsdb-cache

gateway:
  enabled: false

monitoring:
  dashboards:
    enabled: true
  rules:
    enabled: true
  serviceMonitor:
    enabled: true
Note that a copy of this values file can be found at /home/cloud_user/Values/.prom-stack-values.yaml.

Save and exit the file.

Install Loki
Loki uses the same Grafana Community Helm chart that was used to install Grafana itself, so no additional charts need to be added. Instead, directly install Loki into the observability namespace using the values file just created:

helm install loki grafana-community/loki \
  -n observability \
  -f loki-values.yaml
You can then confirm Loki is working by enabling port forwarding so you can send a test log directly from the workstation:

kubectl port-forward --namespace observability svc/loki 3100:3100 &
If you get an error: unable to forward port because pod is not running. Current status=Pending message, re-issue the command.

Then, url curl to send a log:

curl -H "Content-Type: application/json" -XPOST -s "http://127.0.0.1:3100/loki/api/v1/push"  \
--data-raw "{\"streams\": [{\"stream\": {\"job\": \"test\"}, \"values\": [[\"$(date +%s)000000000\", \"fizzbuzz\"]]}]}"
You can then confirm via a separate curl command:

curl "http://127.0.0.1:3100/loki/api/v1/query_range" --data-urlencode 'query={job="test"}' | jq .data.result
This should return:

[
  {
    "stream": {
      "detected_level": "unknown",
      "job": "test",
      "service_name": "test"
    },
    "values": [
      [
        "1778773131000000000",
        "fizzbuzz"
      ]
    ]
  }
]
Add as Grafana data source
Once the pods are up and running, return to the Grafana UI to add Loki as a data source.

From the left menu under Connections, click Add new connection.

Search for and click Loki

Click Add new data source (in the upper-right).

Under Connection, in the URL field enter http://loki:3100

Scroll to the bottom of the page and click Save & test.

A green box will verify:

Data source successfully connected.

Additionally, view the log you sent via the command line. From the left-menu click Explore.

From the drop-down towards the upper-left, select loki.

(This drop-down will likely initially be displaying prometheus.)

then selecting loki from the data source dropdown at the top (it will default to prometheus).

Towards the right of the query builder, beside Builder, select Code.

Input the following into the query bar:

{job="test"}
Run the query by pressing Shift+Enter.

Under Logs, you should see a single fizzbuzz log.

Additional Resources
You have been tasked with setting up a full observability solution for your Kubernetes infrastructure. This solution should be hosted on Kubernetes itself and leverage the Helm package manager for installation and configuration, with SeaweedFS as the supported object storage solution already running on the cluster.

Your first steps to achieve this goal are to set up, configure, and install Grafana, Prometheus (and related tools, such as AlertManager), and Loki on the existing cluster, ensuring Prometheus and Loki function as a data sources within the Grafana UI. Prometheus and AlertManager should also use the local path for storage, which will need to be configured as the default storage class.

A Kubernetes cluster has been provided, alongside a “workstation” server with kubectl installed and access to the cluster. Perform the lab from the workstation server. You will also need to use your browser to access the Grafana UI.

Once you start the lab, credentials for SeaweedFS can be found in the seaweed-creds.txt file in the cloud_user’s home directory on the Workstation server.