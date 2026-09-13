> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Demo Connecting a Datasource in Grafana

> Guide showing how to configure Grafana to connect to an in-cluster Prometheus datasource, verify services, save and test the connection, and troubleshoot common issues

Grafana does not store metrics itself; it visualizes metrics collected and stored elsewhere. To visualize metrics in Grafana you must configure a datasource that points to where those metrics live. In this demo we connect Grafana to a Prometheus instance running inside the cluster.

Prerequisites

* A Prometheus stack is installed in the `monitoring` namespace (kube-prometheus-stack or similar).
* Grafana is running and reachable (exposed via NodePort in this environment).
* `kubectl` configured to access the cluster.

Verify pods and services

Run the following to confirm the Prometheus and Grafana components are running:

```bash theme={null}
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

A sample output looks like:

```bash theme={null}
# pods
NAME                                                      READY   STATUS    RESTARTS   AGE
kube-prometheus-stack-grafana-84cf9bcd57-9vm7h            2/2     Running   0          12m
kube-prometheus-stack-kube-state-metrics-567d4944f-1fpz   1/1     Running   0          12m
kube-prometheus-stack-operator-5d9b6cb8d-8hqhx             1/1     Running   0          12m
kube-prometheus-stack-prometheus-node-exporter-5r5x7      2/2     Running   0          12m
prometheus-kube-prometheus-stack-prometheus-0              2/2     Running   0          12m

# services
NAME                                                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)
kube-prometheus-stack-grafana                               NodePort    172.20.151.255  <none>        80:30080/TCP
kube-prometheus-stack-kube-state-metrics                    ClusterIP   172.20.6.143    <none>        8080/TCP
kube-prometheus-stack-operator                              ClusterIP   172.20.149.171  <none>        443/TCP
kube-prometheus-stack-prometheus                            NodePort    172.20.50.36    <none>        9090:30900/TCP,8080:30392/TCP
kube-prometheus-stack-prometheus-node-exporter              ClusterIP   172.20.93.180   <none>        9100/TCP
prometheus-operated                                         ClusterIP   None            <none>        9090/TCP
```

Quick summary of important endpoints

| Component                | Access pattern                                          | Example                                                        |
| ------------------------ | ------------------------------------------------------- | -------------------------------------------------------------- |
| Prometheus (cluster DNS) | `http://<service>.<namespace>.svc.cluster.local:<port>` | `http://prometheus-operated.monitoring.svc.cluster.local:9090` |
| Prometheus (NodePort)    | Node external access                                    | `NodePort 30900`                                               |
| Grafana (NodePort)       | Node external access                                    | `NodePort 30080`                                               |

Open Grafana and log in
Use the Grafana NodePort (or your configured ingress) to reach the Grafana UI and sign in with the credentials for your setup.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Demo-Connecting-a-Datasource-in-Grafana/grafana-login-screen-email-password-gradient.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=7976bc404d5a12f9efe5076641021b5c" alt="The image shows a Grafana login screen with fields for email or username and password against a gradient background. The &#x22;Log in&#x22; button is prominently displayed below the input fields." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Demo-Connecting-a-Datasource-in-Grafana/grafana-login-screen-email-password-gradient.jpg" />
</Frame>

After login you will arrive at Grafana’s home/dashboard page where you can add data sources and create dashboards.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Demo-Connecting-a-Datasource-in-Grafana/grafana-dashboard-interface-setup-options.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=4daae452db3197bb331448d9c376a1d3" alt="The image shows the Grafana dashboard interface, featuring a welcome message and options for setting up data sources and creating dashboards. There are also links for help and recent blog updates." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Demo-Connecting-a-Datasource-in-Grafana/grafana-dashboard-interface-setup-options.jpg" />
</Frame>

Add Prometheus as a data source

1. Click Add your first data source (or Add data source).
2. From the list, choose Prometheus.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Demo-Connecting-a-Datasource-in-Grafana/grafana-data-sources-options-interface.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=795a4ff79d605a25e7e59b05bf64aad5" alt="The image shows a Grafana interface with options to add different data sources, including Prometheus, Graphite, InfluxDB, and others. It is organized under categories like time series databases and logging & document databases." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Demo-Connecting-a-Datasource-in-Grafana/grafana-data-sources-options-interface.jpg" />
</Frame>

Configure the Prometheus connection

* Keep the default Name (or give it a descriptive name such as `prometheus-monitoring`).
* For the URL, point Grafana to the in-cluster Prometheus service. Use the full service DNS name and port (including scheme):

```bash theme={null}
http://prometheus-operated.monitoring.svc.cluster.local:9090
```

This follows the standard in-cluster DNS pattern:
`service-name.namespace.svc.cluster.local:port`

It will work regardless of the namespace where Grafana runs, provided DNS resolves and any network policies allow access.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Demo-Connecting-a-Datasource-in-Grafana/grafana-prometheus-data-source-configuration.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=8162f952936c47ecb82b8c9dc8ab4c1c" alt="The image shows the Grafana interface for configuring a Prometheus data source, including fields for connection settings and authentication methods." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Demo-Connecting-a-Datasource-in-Grafana/grafana-prometheus-data-source-configuration.jpg" />
</Frame>

Optional settings

* Toggle “Default” if you want this Prometheus instance to be the default for new dashboards.
* Leave authentication, timeouts, and other advanced settings at their defaults unless you require custom configuration (e.g., TLS, headers, or proxy).

Save and test
Click Save & Test. On success Grafana will show a green confirmation that it was able to query the Prometheus API.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Demo-Connecting-a-Datasource-in-Grafana/grafana-data-source-prometheus-configuration.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=b3c6d2ddd4c2d3e7384e684ad916a72b" alt="The image shows a Grafana data source configuration screen for Prometheus, with various settings toggles and input fields for performance and connection parameters. There's a notification indicating successful querying of the Prometheus API." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Demo-Connecting-a-Datasource-in-Grafana/grafana-data-source-prometheus-configuration.jpg" />
</Frame>

<Callout icon="lightbulb" color="#1CB2FE">
  If Save & Test fails, verify the following:

  * The service DNS and port are correct: `http://prometheus-operated.monitoring.svc.cluster.local:9090`.
  * Prometheus pods are running in the `monitoring` namespace: `kubectl get pods -n monitoring`.
  * NetworkPolicies, firewall rules, or cluster network ACLs are not preventing in-cluster requests from Grafana to Prometheus.
</Callout>

Next steps

* Create dashboards and panels that query Prometheus using PromQL.
* Import prebuilt Grafana dashboards for Kubernetes or your applications.
* Secure access to Grafana (authentication, RBAC, and TLS) before exposing it externally.

Links and references

* Grafana documentation: [https://grafana.com/docs/](https://grafana.com/docs/)
* Prometheus documentation: [https://prometheus.io/docs/](https://prometheus.io/docs/)
* Kubernetes DNS and Services: [https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/9bd090c8-8d99-4742-b50c-ae63e516e6b9/lesson/b380cd38-72e3-4794-8486-ab7f52739839" />
</CardGroup>
