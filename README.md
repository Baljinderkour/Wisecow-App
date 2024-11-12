# Wisecow-App
QA trainee assesment
git clone https://github.com/nyrahul/wisecow.git
cd wisecow
import subprocess

# Helper function to run shell commands
def run_command(command, cwd=None):
    try:
        result = subprocess.run(command, shell=True, check=True, cwd=cwd, capture_output=True, text=True)
        print(result.stdout)
    except subprocess.CalledProcessError as e:
        print(f"Error running command: {e.stderr}")

# Set your Docker Hub username and the name of your Kubernetes domain
dockerhub_username = "<baljinderkour>"
app_name = "wisecow-app"
domain_name = "wisecow.com"

# Step 1: Clone the Wisecow Repository
print("Cloning the Wisecow repository...")
run_command("git clone https://github.com/nyrahul/wisecow.git")
project_directory = "wisecow"

# Step 2: Create Dockerfile
print("Creating Dockerfile...")
dockerfile_content = """
FROM python:3.9-slim
WORKDIR /app
COPY . /app
RUN pip install --no-cache-dir -r requirements.txt
EXPOSE 8000
CMD ["python", "app.py"]
"""
with open(f"{project_directory}/Dockerfile", "w") as f:
    f.write(dockerfile_content)

# Step 3: Build Docker Image
print("Building Docker image...")
run_command(f"docker build -t {app_name} .", cwd=project_directory)

# Step 4: Tag and Push Docker Image to Docker Hub
print("Tagging and pushing Docker image to Docker Hub...")
run_command(f"docker tag {app_name} {dockerhub_username}/{app_name}:latest")
run_command(f"docker push {dockerhub_username}/{app_name}:latest")

# Step 5: Create Kubernetes Deployment YAML
print("Creating Kubernetes deployment YAML...")
deployment_yaml = f"""
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {app_name}
spec:
  replicas: 2
  selector:
    matchLabels:
      app: {app_name}
  template:
    metadata:
      labels:
        app: {app_name}
    spec:
      containers:
      - name: {app_name}
        image: {dockerhub_username}/{app_name}:latest
        ports:
        - containerPort: 8000
"""
with open("deployment.yaml", "w") as f:
    f.write(deployment_yaml)

# Step 6: Create Kubernetes Service YAML
print("Creating Kubernetes service YAML...")
service_yaml = f"""
apiVersion: v1
kind: Service
metadata:
  name: {app_name}-service
spec:
  selector:
    app: {app_name}
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
  type: LoadBalancer
"""
with open("service.yaml", "w") as f:
    f.write(service_yaml)

# Step 7: Apply Deployment and Service YAML to Kubernetes
print("Applying deployment and service to Kubernetes...")
run_command("kubectl apply -f deployment.yaml")
run_command("kubectl apply -f service.yaml")

# Step 8: Install Cert-Manager for TLS (Only needed once per cluster)
print("Installing Cert-Manager...")
run_command("kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.6.1/cert-manager.yaml")

# Step 9: Create Ingress YAML with TLS Configuration
print("Creating Kubernetes ingress YAML for TLS...")
ingress_yaml = f"""
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {app_name}-ingress
  annotations:
    cert-manager.io/issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - {domain_name}
    secretName: {app_name}-tls
  rules:
  - host: {domain_name}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: {app_name}-service
            port:
              number: 80
"""
with open("ingress.yaml", "w") as f:
    f.write(ingress_yaml)

# Step 10: Apply Ingress YAML
print("Applying ingress to Kubernetes...")
run_command("kubectl apply -f ingress.yaml")

# Final message
print("Deployment complete! Visit your application at https://{domain_name}")
