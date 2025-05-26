# Adding Observability to the Legacy Spring Boot PetClinic Application


## Running Petclinic locally on Docker

### Build with Maven command line
```
git clone https://github.com/vjkancherla/DevSecOps-PetClinic.git
cd DevSecOps-PetClinic
./mvn clean package
```

### Create Docker image
```
docker build -t vjkancherla/petclinic:v1 .
```

### Publish Docker image
```
docker push vjkancherla/petclinic:v1
```

### Run with Docker
```
docker run -p 9080:8080 vjkancherla/petclinic:v1
```

You can then access petclinic here: [http://localhost:9080/petclinic](http://localhost:9080/petclinic)

<br>
<br>

## Running Petclinic locally on Kubernetes (K3d)

### Start K3d
```
k3d cluster create mycluster -a 1 --subnet 172.19.0.0/16
```

### Install Helm Chart
```
helm install petclinic-test --set image.repository=vjkancherla/petclinic --set image.tag=v1 ./helm-chart
```

### Create Port-Forwarding
```
k port-forward svc/petclinic 9080:8080
```

You can then access petclinic here: [http://localhost:9080/petclinic](http://localhost:9080/petclinic)


## Instrumenting the Petclinc application

### Core Architecture
```
[Legacy Java App] 
  │── Enable JMX and RMI (127.0.0.1:1099, local-only)
  └── [Sidecar: JMX Exporter]
       │── Scrapes JMX metrics by connecting to the RMI port
       └── Exposes Prometheus metrics (HTTP :9400/metrics)
            └── [Prometheus] → [Grafana]
``` 

### Design Summary
The deployment integrates a Prometheus JMX Exporter sidecar to monitor a legacy Java app (Tomcat) without code changes. 

#### Key features:

1. JMX Isolation
    - The Java app exposes JMX metrics on 127.0.0.1:1099 with local.only=true for security.
    - No authentication/SSL (safe, as JMX is container-local).

2. Sidecar Pattern
    - A bitnami/jmx-exporter sidecar runs alongside the app in the same pod.
    - Scrapes JMX metrics and converts them to Prometheus format on port 9400.
    - Configuration is mounted via a ConfigMap (exporter.yaml).

3. Prometheus Integration
    -  Auto-discovery enabled via ServiceMonitor object
    - Metrics endpoint at :9400/metrics 

4. Zero App Modifications
    - JMX activated purely through JAVA_OPTS environment variables.
    - Sidecar is decoupled—no changes to the app container required.
    - Each app replica gets its own sidecar (1:1 scaling).

### Critical Configuration Details

#### JMX enabled via JAVA_OPTS environment var on Java App (Main Container):
```
env:
- name: JAVA_OPTS
    value: >-
    -Dcom.sun.management.jmxremote
    -Dcom.sun.management.jmxremote.local.only=true
    -Dcom.sun.management.jmxremote.port=1099
    -Dcom.sun.management.jmxremote.rmi.port=1099
    -Dcom.sun.management.jmxremote.authenticate=false
    -Dcom.sun.management.jmxremote.ssl=false
    -Djava.rmi.server.hostname=127.0.0.1
```

#### JMX Exporter (Sidecar) Runs standalone JAR (not agent mode)
```
java -jar jmx_prometheus_standalone.jar 9400 /etc/jmx-exporter/exporter.yaml
```

#### JMX Exporter configuration is defined as key in a ConfigMap and then mounted as a volume
```
spec:
    volumes:
    - name: jmx-exporter-config
      configMap:
        name: jmx-exporter-config
```

```
volumeMounts:
    - name: jmx-exporter-config
      mountPath: /etc/jmx-exporter
```

### Testing

#### Install Helm Chart
```
helm install petclinic-test --set image.repository=vjkancherla/petclinic --set image.tag=v1 ./helm-chart
```

### Create Port-Forwarding to Petclinc 
```
k port-forward svc/petclinic 9080:8080
```

You can then access petclinic here: [http://localhost:9080/petclinic](http://localhost:9080/petclinic)

Browser the app to generate some metrics

### Create Port-Forwarding to JMX-Exporter
```
k port-forward svc/petclinic 9400:9400
``` 

Access the metrics here: [http://localhost:9400/metrics]([http://localhost:9400/metrics)


## Integration with Prometheus

### Run Promethues
Follow the instructions in `Observability/Prometheus/run-prometheus.txt`

### ServiceMonitor resource creates the Prometheus integration by automating metric scraping for the JMX Exporter sidecar
```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: petclinic-monitor
  labels:
    release: dev-prometheus  # Match your Prometheus instance
spec:
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  endpoints:
  - port: jmx-metrics
    interval: 15s
    path: /metrics
```

### Verify 

#### Create Port-Forwarding to Prometheus
```
k port-forward -n monitoring svc/dev-prometheus-kube-promet-prometheus 9090:9090
```

In the console, go to Status > Target and confirm that you can see PetClinc under "targets" there. It might take a couple of minutes to show up, be patient.
