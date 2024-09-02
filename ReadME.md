### Brief
Your topic will be centralised application monitoring with Prometheus and Grafana. You need to ensure your presentation fits the following use case:
You have multiple projects running on different Kubernetes (k8s) clusters. These clusters are isolated for security reasons, as each application is owned by different customers. To improve the efficiency of our team, we need to build a centralised monitoring system that is ready to use and easy to plug in or out. Since the logic and behavior of each application may differ, you must consider how to export metrics to the central monitoring system (another k8s cluster) with low cohesion and easy decoupling, allowing for future tailored implementations within the application cluster.

---

### Considerations:
You need to account for potential edge cases, such as Kubernetes (k8s) clusters hosted on different cloud providers.
You must address networking issues, including 
- VPNs
- VPCs
- similar concerns.

---

### Solution 1
- **(For low cohesion)** Since projects are spread across multiple clusters and cloud, each project cluster would have their own VPC networks. To establish a communication between clusters in different networks, we establish **VPC sharing** or **VPC peering** or **VPN**. This will create a communication tunnel that is private and traffic does not go through open internet. 
- **(For easy decoupling)** After connections are established, We create **Service Mesh(ISTIO)**. Reason for not using inbuilt k8s Services is that they only offer communication within a cluster and it does not provide any encryption nor observability on the service communication. Service mesh establishes communication between clusters.
- **(easy to plug in or out)** Next we deploy **Metrics Extractor** on each pod to extract app metrics and convert that into prometheus understandable format. This can be a DaemonSet in k8s which is easy to plug in when we create additional pods.
- We create a new cluster in GKE, and deploy **Prometheus** to pull the metrics from all the clusters and store it in a time series DB.
- We also deploy **Grafana** in the same cluster to convert the stored metrics into visual graphs and we observe.
- Additional : 
  - PromQL
  - Dashboards
  - Service Discovery
  - Proxies
  - Admission Controllers
  - mTLS
  - Security
  - Canary updates
