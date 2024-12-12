# Steps to Run Independent Temporal UI and UI Server in Helm Chart

This document outlines the steps to run Temporal UI and Temporal UI Server as independent components using a custom Helm chart.

## 1. Pull Latest Version of Temporal UI

First, clone the repository for the latest version of Temporal UI from GitHub:

```bash
git clone https://github.com/kiet-git/temporal_ui
```

## 2. Build Docker Images for `ui-server` and `ui`

### Build the `ui-server` Image

```bash
cd server
docker build -t <your-registry>/temporal-ui-server:v1.0.0 .
```

### Build the `ui` Image

```bash
cd ..
docker build -t <your-registry>/temporal-ui:v1.0.0 .
```

## 3. Push Docker Images to Your Registry

Login to your Docker registry:

```bash
docker login
```

Push the images to your registry:

```bash
docker push <your-registry>/temporal-ui-server:v1.0.0
docker push <your-registry>/temporal-ui:v1.0.0
```

## 4. Create ConfigMap for Environment Variables

Before deploying, create a ConfigMap to store the environment variables needed for the UI. This is done by running the following command:

```bash
kubectl create configmap temporal-ui-env --from-env-file=.env
```

Ensure that the `.env` file contains the necessary environment variables for your Temporal UI application.

Env example:

```bash
VITE_TEMPORAL_PORT="7233"
VITE_API="http://localhost:8080"
VITE_MODE="development"
VITE_TEMPORAL_UI_BUILD_TARGET="local"
```

## 5. Create Custom Helm Templates

Create custom Helm templates for both `ui-server` and `ui` in the `templates` directory of your Helm chart.

### `custom-ui-server-service.yaml`

```yaml
{{- if .Values.uiserver.enabled }}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ui-server
  labels:
    app: ui-server
spec:
  replicas: {{ .Values.uiserver.replicaCount }}
  selector:
    matchLabels:
      app: ui-server
  template:
    metadata:
      labels:
        app: ui-server
    spec:
      containers:
        - name: ui-server
          image: "{{ .Values.uiserver.image.repository }}:{{ .Values.uiserver.image.tag }}"
          imagePullPolicy: {{ .Values.uiserver.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.uiserver.service.port }}
          env:
            {{- range .Values.uiserver.additionalEnv }}
            - name: {{ .name }}
              value: {{ .value }}
            {{- end }}

---
apiVersion: v1
kind: Service
metadata:
  name: ui-server
spec:
  ports:
    - port: {{ .Values.uiserver.service.port }}
      targetPort: {{ .Values.uiserver.service.port }}
  selector:
    app: ui-server
{{- end }}
```

### `custom-ui-service.yaml`

```yaml
{{- if .Values.ui.enabled }}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ui
  labels:
    app: ui
spec:
  replicas: {{ .Values.ui.replicaCount }}
  selector:
    matchLabels:
      app: ui
  template:
    metadata:
      labels:
        app: ui
    spec:
      containers:
        - name: ui
          image: "{{ .Values.ui.image.repository }}:{{ .Values.ui.image.tag }}"
          imagePullPolicy: {{ .Values.ui.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.ui.service.port }}
          envFrom:
            {{- if .Values.ui.additionalEnvFrom }}
              {{- range .Values.ui.additionalEnvFrom }}
                - configMapRef:
                    name: {{ .configMapRef.name }}  # Reference the ConfigMap name
              {{- end }}
            {{- else }}
              # Optionally, if no envFrom values are provided, provide defaults
            {{- end }}
---
apiVersion: v1
kind: Service
metadata:
  name: ui
spec:
  ports:
    - port: {{ .Values.ui.service.port }}
      targetPort: {{ .Values.ui.service.port }}
  selector:
    app: ui
{{- end }}
```

## 6. Modify `values.yaml`

Modify the `values.yaml` file to enable the UI and UI Server and configure other necessary settings.

### Example `values.yaml` configuration

```yaml
uiserver:
  enabled: true
  replicaCount: 1
  image:
    repository: <your-registry>/temporal-ui-server
    tag: v1.0.0
    pullPolicy: Always
  service:
    type: ClusterIP
    port: 8080
    annotations: {}
  ingress:
    enabled: false
    annotations: {}
    hosts:
      - "/"
    tls: []
  additionalEnv:
    - name: TEMPORAL_ADDRESS
      value: "192.168.194.197:7233"
    - name: TEMPORAL_CLI_ADDRESS
      value: "192.168.194.197:7233"
    - name: TEMPORAL_CORS_ORIGINS
      value: "http://localhost:3000"

ui:
  enabled: true
  replicaCount: 1
  image:
    repository: <your-registry>/temporal-ui
    tag: v1.0.0
    pullPolicy: Always
  service:
    type: ClusterIP
    port: 3000
    annotations: {}
  additionalEnvFrom:
    - configMapRef:
        name: temporal-ui-env

# Disable default web UI
web.enabled: false

# Optionally limit resources for other services
prometheus.enabled: false
grafana.enabled: false
cassandra.config.cluster_size: 1
elasticsearch.replicas: 1
```

## 7. Install the Helm Chart

Run the following command to install the Temporal UI and UI Server:

```bash
helm install temporaltest ./charts/temporal --timeout 15m -f charts/temporal/values.yaml
```

## 8. Expose Ports for Access

Use `kubectl` to forward the ports of the UI and UI Server to your local machine:

```bash
kubectl port-forward services/ui 3000:3000  # UI
kubectl port-forward services/ui-server 8080:8080  # UI Server
```

## 9. Access the UI and UI Server

After setting up port forwarding, you can access the Temporal UI and UI Server by visiting the following URLs:

- [Temporal UI](http://localhost:3000/)
- [Temporal UI Server](http://localhost:8080/)
