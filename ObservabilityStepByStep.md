# Step-by-Step Guide to Go Application Observability in Kubernetes

This guide will walk you through setting up a Go application in Kubernetes and instrumenting it for full observability using Grafana, Prometheus, Tempo, and Loki.

## Prerequisites

Before you begin, you will need the following tools installed and configured:

*   **kubectl**: The Kubernetes command-line tool.
*   **Helm**: The package manager for Kubernetes.
*   **Docker**: To build and push your application image.
*   A running Kubernetes cluster (e.g., Minikube, Kind, or a cloud provider's cluster).
*   A Docker Hub account or other container registry to store your application image.

## 1. Sample Go Application

First, let's create a simple Go application that exposes metrics, traces, and logs.

**`main.go`**
```go
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"os"
	"os/signal"
	"time"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
	"go.opentelemetry.io/otel/propagation"
	"go.opentelemetry.io/otel/sdk/resource"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.4.0"
	"go.opentelemetry.io/otel/trace"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
	httpRequestsTotal = prometheus.NewCounterVec(
		prometheus.CounterOpts{
			Name: "http_requests_total",
			Help: "Total number of HTTP requests",
		},
		[]string{"path"},
	)
	tracer = otel.Tracer("go-app")
)

func init() {
	prometheus.MustRegister(httpRequestsTotal)
}

func initTracer() (*sdktrace.TracerProvider, error) {
	exporter, err := otlptracegrpc.New(context.Background(),
		otlptracegrpc.WithInsecure(),
		otlptracegrpc.WithEndpoint("tempo:4317"), // Tempo endpoint
	)
	if err != nil {
		return nil, err
	}

	r, err := resource.Merge(
		resource.Default(),
		resource.NewWithAttributes(
			semconv.SchemaURL,
			semconv.ServiceNameKey.String("go-app"),
		),
	)
	if err != nil {
		return nil, err
	}

	tp := sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(exporter),
		sdktrace.WithResource(r),
	)
	otel.SetTracerProvider(tp)
	otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(propagation.TraceContext{}, propagation.Baggage{}))
	return tp, nil
}

func main() {
	tp, err := initTracer()
	if err != nil {
		log.Fatalf("failed to initialize tracer: %v", err)
	}
	defer func() {
		if err := tp.Shutdown(context.Background()); err != nil {
			log.Printf("Error shutting down tracer provider: %v", err)
		}
	}()

	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		ctx, span := tracer.Start(r.Context(), "hello-handler")
		defer span.End()

		span.SetAttributes(attribute.String("http.method", r.Method))
		span.SetAttributes(attribute.String("http.url", r.URL.String()))

		fmt.Fprintf(w, "Hello, World!")
		log.Println("Hello handler executed")
		httpRequestsTotal.WithLabelValues("/").Inc()
	})

	http.Handle("/metrics", promhttp.Handler())

	server := &http.Server{Addr: ":8080"}

	go func() {
		if err := server.ListenAndServe(); err != http.ErrServerClosed {
			log.Fatalf("HTTP server error: %v", err)
		}
	}()

	stop := make(chan os.Signal, 1)
	signal.Notify(stop, os.Interrupt)
	<-stop

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	if err := server.Shutdown(ctx); err != nil {
		log.Fatalf("HTTP server shutdown error: %v", err)
	}
}
```

**`go.mod`**
```
module go-app

go 1.19

require (
	github.com/prometheus/client_golang v1.14.0
	go.opentelemetry.io/otel v1.11.2
	go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc v1.11.2
	go.opentelemetry.io/otel/sdk v1.11.2
	go.opentelemetry.io/otel/trace v1.11.2
)
...
```

## 2. Containerize the Application

Create a `Dockerfile` to build the application image.

**`Dockerfile`**
```Dockerfile
FROM golang:1.19-alpine

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN go build -o /go-app

EXPOSE 8080

CMD [ "/go-app" ]
```

Build and push the image to your container registry.
```sh
docker build -t your-username/go-app .
docker push your-username/go-app
```

## 3. Kubernetes Deployment

Create a `deployment.yaml` file for the application.

**`deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: go-app
  template:
    metadata:
      labels:
        app: go-app
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: "/metrics"
        prometheus.io/port: "8080"
    spec:
      containers:
      - name: go-app
        image: your-username/go-app
        ports:
        - containerPort: 8080

---
apiVersion: v1
kind: Service
metadata:
  name: go-app
spec:
  selector:
    app: go-app
  ports:
    - name: http
      protocol: TCP
      port: 8080
      targetPort: 8080
```

Deploy the application to your cluster:
```sh
kubectl apply -f deployment.yaml
```

## 4. Install the Observability Stack

We will use Helm to install Grafana, Prometheus, Tempo, and Loki.

First, add the Grafana Helm repository:
```sh
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

**Install Loki**
```sh
helm install loki grafana/loki-stack --set grafana.enabled=false,prometheus.enabled=false
```

**Install Tempo**
```sh
helm install tempo grafana/tempo --set traces.otlp.grpc.enabled=true
```

**Install Prometheus**
```sh
helm install prometheus grafana/prometheus-community/prometheus
```

**Install Grafana**
```sh
helm install grafana grafana/grafana
```

Get the Grafana admin password:
```sh
kubectl get secret --namespace default grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

Port-forward to access the Grafana UI:
```sh
kubectl port-forward --namespace default svc/grafana 8080:80
```
You can now access Grafana at `http://localhost:8080`.

## 5. Configure Grafana Data Sources

1.  **Prometheus**:
    *   Go to `Configuration > Data Sources`.
    *   Click `Add data source` and select `Prometheus`.
    *   Set the URL to `http://prometheus-server.default.svc.cluster.local`.
    *   Click `Save & Test`.

2.  **Tempo**:
    *   Go to `Configuration > Data Sources`.
    *   Click `Add data source` and select `Tempo`.
    *   Set the URL to `http://tempo.default.svc.cluster.local:3100`.
    *   Click `Save & Test`.

3.  **Loki**:
    *   Go to `Configuration > Data Sources`.
    *   Click `Add data source` and select `Loki`.
    *   Set the URL to `http://loki.default.svc.cluster.local:3100`.
    *   Click `Save & Test`.

## 6. View Metrics, Traces, and Logs

*   **Metrics**: In Grafana, go to `Explore` and select the Prometheus data source. You can query for `http_requests_total`.
*   **Traces**: In Grafana, go to `Explore` and select the Tempo data source. You should see traces from the `go-app` service.
*   **Logs**: In Grafana, go to `Explore` and select the Loki data source. You can query for logs from your application using a query like `{app="go-app"}`.

You have now successfully set up a Go application in Kubernetes with a complete observability stack!
