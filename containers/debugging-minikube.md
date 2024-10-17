# debugging in minikube and vscode
to debug a go microservice inside a minikube pod follow these steps:
- setup .vscode/launch.json
```json
{
    "version": "0.2.0",
    "configurations": [
      {
        "name": "Debug in Minikube",
        "type": "go",
        "request": "attach",
        "mode": "remote",
        "remotePath": "/app",  // Path inside the container
        "port": 40000,
        "host": "127.0.0.1",  // Localhost, since we are going to port-forwarded
        "cwd": "${workspaceFolder}",
        "apiVersion": 2,
        "trace": "verbose",
        "showLog": true
      }
    ]
  }
```
- change the Dockerfile of the service and optionally rename it to `Dockerfile-dlv`
```Dockerfile
FROM golang:1.23-alpine

# Install Delve for debugging
RUN apk add --no-cache git bash && \
    go install github.com/go-delve/delve/cmd/dlv@latest

WORKDIR /app

# Copy go.mod and go.sum files for dependency installation
COPY go.mod go.sum ./
RUN go mod download

# Copy the rest of the application code
COPY . .

# Build the Go app (with no optimization and function inlining)
RUN go build -gcflags "all=-N -l" -o /goapp .

# Run the application with Delve debugger in headless mode
CMD ["dlv", "exec", "/goapp", "--headless", "--listen=:40000", "--api-version=2", "--log"]
```
- change app deployment manifest and optionally rename it to `app-deployment-dlv.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: golang-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: golang-app
  template:
    metadata:
      labels:
        app: golang-app
    spec:
      containers:
      - name: golang-app
        image: <image>:<version>-dlv
        ports:
        - containerPort: 8080  # microservice port
        - containerPort: 40000 # dlv port
        # ...
---
apiVersion: v1
kind: Service
metadata:
  name: golang-app-service
spec:
  selector:
    app: golang-app
  # change to `LoadBalance` if needed in that case should use `minikube tunnel` 
  ports:
    - name: http
      port: 8080
      nodePort: 30010
    - name: debug
      port: 40000
      nodePort: 30011
  type: NodePort 
```
- config kubectl to port-forward
```bash
k port-forward pods/golang-app-... 40000:40000
```
- now we can run debugger inside vscode and create breakpoints 🎉