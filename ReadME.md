# MXM
## Brief
Your topic will be centralised application monitoring with Prometheus and Grafana. You need to ensure your presentation fits the following use case:
You have multiple projects running on different Kubernetes (k8s) clusters. These clusters are isolated for security reasons, as each application is owned by different customers. To improve the efficiency of our team, we need to build a centralised monitoring system that is ready to use and easy to plug in or out. Since the logic and behavior of each application may differ, you must consider how to export metrics to the central monitoring system (another k8s cluster) with low cohesion and easy decoupling, allowing for future tailored implementations within the application cluster.
## Considerations:
You need to account for potential edge cases, such as Kubernetes (k8s) clusters hosted on different cloud providers.
You must address networking issues, including 
- VPNs
- VPCs
- similar concerns.
## Solution dump
- **(For low cohesion)** Since projects are spread across multiple clusters and cloud, each project cluster would have their own VPC networks. To establish a communication between clusters in different networks, we establish **VPC sharing** or **VPN**. This will create a communication tunnel that is private and traffic does not go through open internet. 
- **(For easy decoupling)** After connections are established, We create **Service Mesh(ISTIO)**. Reason for not using inbuilt k8s Services is that they only offer communication within a cluster and it does not provide any encryption nor observability on the service communication. Service mesh establishes communication between clusters.
- **(easy to plug in or out)** Next we deploy **Metrics Extractor** on each pod to extract app metrics and convert that into prometheus understandable format. This can be a DaemonSet in k8s which is easy to plug in when we create additional pods.
- We create a new cluster in GKE, and deploy **Prometheus** to pull the metrics from all the clusters and store it in a time series DB.
- We also deploy **Grafana** in the same cluster to convert the stored metrics into visual graphs and we observe.
- InfluxDB or Big Query for persisting metrics.
## VPN 
- Deploy on all clusters as a pod
- setup video - [Quick Start](https://www.wireguard.com/quickstart/)
- add custom routes and gateways for traffic management.
- modify firewall rules — enable port, add traffic rules. (research more on this)
- Config
  ```bash
  [Interface]
  Address = 10.13.13.1
  ListenPort = 51820
  PrivateKey = <private-key>
  
  PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth+ -j MASQUERADE
  PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth+ -j MASQUERADE
  
  [Peer]
  PublicKey = <public-key>
  AllowedIPs = 10.13.13.2/32
  ```
- store as ConfigMaps
  ```yml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: wireguard-configmap
    namespace: wireguard
  data:
    PUID: "1000"
    PGID: "1000"
    TZ: "America/New_York"
    SERVERPORT: "51820"
    PEERS: "1"
    PEERDNS: "10.4.0.10"
    ALLOWEDIPS: "0.0.0.0/0, ::/0"
  ```
- IP forwarding & NAT
  ```bash
  sudo nano /etc/sysctl.conf
  
  # IP FORWARD
  net.ipv4.ip_forward=1
     
  # NAT
  iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
  ```
## Service Mesh & Prometheus
- K8s RBAC role binding
- firewall rule 
  ```bash
  $ gcloud compute firewall-rules create istio-multicluster-pods \
  --allow=tcp,udp,icmp,esp,ah,sctp \
  --direction=INGRESS \
  --port=15890 \
  --source-ranges="${ALL_CLUSTER_CIDRS}" \	# ips
  --target-tags="${ALL_CLUSTER_NETTAGS}" --quiet # tags
  ```
- install Istio
- label to namespace `—istio-injection=enabled`
- Istio automatically detects the services and endpoints in that cluster through pilot.
- Identity, mTLS(Auth), Encryption(Security)
  - CA X.509 certificates
- ServiceEntry & DestinationRule for multi cloud setup
  ```yml
  # IngressGateway
  apiVersion: networking.istio.io/v1alpha3
  kind: Gateway
  metadata:
    name: my-gateway
  spec:
    selector:
      istio: ingressgateway
   servers:
      port:
      number: 80
      name: http
      protocol: HTTP
      hosts:
  
  # Service Entry to connect SMs
    apiVersion: networking.istio.io/v1alpha3
    kind: ServiceEntry
    metadata:
      name: gcp-service-mesh-entry
    spec:
      hosts:
      - "*.gcp-service-mesh.com"
      location: MESH_EXTERNAL
      ports:
      - number: 443
        name: https
        protocol: HTTPS
      resolution: DNS
  
    --- 
  
    # Destination Rule for service req timeouts & circuit breaking
    apiVersion: networking.istio.io/v1
    kind: DestinationRule
    metadata:
      name: ext-res-dr
    spec:
      host: *.gcp-service-mesh.com
      trafficPolicy:
        connectionPool:
          tcp:
            connectTimeout: 1s
      subsets:
        trafficPolicy:
          connectionPool:
            tcp:
              maxConnections: 100
  
  
    ---
  
  
    # Retry count and interval to prevent service call overwhelming
    apiVersion: networking.istio.io/v1
    kind: VirtualService
    metadata:
      name: ratings
    spec:
      hosts:
      - "*.gcp-service-mesh.com"
      http:
      - route:
        - destination:
            host: ratings
            subset: v1
        retries:
          attempts: 3
          perTryTimeout: 2s
  ```  
- Metrics
  - [x] CPU Utilisation
  - [x] Memory Utilisation
  - [x] Network Utilisation
  - [x] Disk I/O Utilisation
  - [x] Number of incoming requests
  - [x] job durations
  - [x] number of fails.
  - Available by default
  - HTTP, HTTP/2, and gRPC Metrics
  - **COUNTER** total req success - `istio_requests_total` with **success filter** 200
  - **COUNTER** total req failure - `istio_requests_total`with **fail filter** 403, 500
  - **DISTRIBUTION** req duration - `istio_request_duration_milliseconds`
  - **DISTRIBUTION** payload size - `istio_request_bytes`
  - **DISTRIBUTION** res size - `istio_response_bytes`
  - **COUNTER** TCP con open & close - `istio_tcp_connections_opened_total` & `istio_tcp_connections_closed_total`
- Metrics types
  - gauges
  - counters
  - summaries
  - histogram
- Setup custom metrics, Define `Telemetry`
  ```yml
  apiVersion: telemetry.istio.io/v1alpha1
  kind: Telemetry
  metadata:
    name: namespace-metrics
  spec:
    metrics:
      - providers:
        - name: prometheus
      - name: my_custom_metric
        type: COUNTER
        description: "Description of your custom metric"
        dimensions:
          source_app: source.app
          destination_app: destination.app
          request_path: request.path
        expression: "1"
      - overrides:
        - match:
            metric: REQUEST_COUNT
            mode: CLIENT_AND_SERVER
          tagOverrides:
            request_operation:
              value: istio_operationId
            request_host:
              value: "request.host"
            destination_port:
              value: "string(destination.port)"
  ```
- Prometheus node exporter with Volume and Volume Mounts
  ```yaml
  # node exporter to deploy on each sidecar
  apiVersion: apps/v1
  kind: DaemonSet
  spec:
    selector:
      matchLabels:
        app: node-exporter
    template:
      metadata:
        labels:
          app: node-exporter
      spec:
        containers:
        - name: node-exporter
          image: prom/node-exporter:v1.3.1
          args:
          - --path.sysfs=/host/sys
          - --path.procfs=/host/proc
          - --path.rootfs=/host/root
          ports:
          - containerPort: 9100
          volumeMounts:
          - name: sys
            mountPath: /host/sys
            readOnly: true
          - name: root
            mountPath: /host/root
            readOnly: true
        volumes:
        - name: sys
          hostPath:
            path: /sys
        - name: root
          hostPath:
            path: /
  ```
- Prometheus Server
  ```yml
  global:
    scrape_interval: 15s
  scrape_configs:
    - job_name: 'prometheus'
      scrape_interval: 15s
      static_configs:
        - targets: ['localhost:9090']
  remote_write:
    - url: "http://localhost:9091/write"
  ```
## Big Query
- BQ table
  ```bash
  BQ_DATASET_NAME=prometheus
  BQ_TABLE_NAME=metrics
  GCP_PROJECT_ID=<your-gcp-project-id>
  
  bq --location=EURPOPE mk --dataset $GCP_PROJECT_ID:$BQ_DATASET_NAME
  bq mk --table \
    --schema ./bq-schema.json \
    --time_partitioning_field timestamp \
    --time_partitioning_type DAY $GCP_PROJECT_ID:$BQ_DATASET_NAME.$BQ_TABLE_NAME
  ```
- open source adapter - [KohlsTechnology/prometheus_bigquery_remote_storage_adapter: Prometheus remote storage adapter for Google's BigQuery](https://github.com/KohlsTechnology/prometheus_bigquery_remote_storage_adapter)
  ## Grafana
  ```promql
  # CPU
  100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
  
  # Memory
  100 * (1 - ((node_memory_MemAvailable_bytes{} or node_memory_MemFree_bytes{}) / node_memory_MemTotal_bytes{}))
  
  # Network Incoming
  rate(node_network_receive_bytes_total[5m])
  
  # Network Outgoing
  rate(node_network_transmit_bytes_total[5m])
  
  # Disk
  rate(node_filesystem_avail_bytes[5m])
  
  # Total requests
  sum(rate(istio_requests_total[5m]))
  
  # Job duration
  rate(istio_request_duration_milliseconds[5m])
  ```
- Custom metrics
  ```yml
  # app pod annotations
  annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "<metrics-port>"
  prometheus.io/path: "/metrics"
  
  # istio config to merge metrics
  meshConfig:
  enablePrometheusMerge: true
  
  PodMonitor & ServiceMonitor config
  apiVersion: monitoring.coreos.com/v1
  kind: PodMonitor
  metadata:
    name: custom-metrics
  spec:
    selector:
      matchLabels:
        app: your-app-label
    podMetricsEndpoints:
  - port: metrics-port-name
    path: /metrics
  ```
  ## Presentation Notes:
- Intro —  2m
- VPN — 3m
- Service Mesh — 8m
- Data & Visualise — 2m
- Edge — 2m
- Future  — 2m
- Cost — 1m
- Q&A — 20m
