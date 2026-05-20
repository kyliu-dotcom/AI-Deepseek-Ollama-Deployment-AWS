# Deploying DeepSeek on AWS EC2 with NVIDIA GPU Acceleration

## Prerequisites

- AWS account with EC2 access
- Ability to launch EC2 instances, configure security groups, and create key pairs
- EC2 instance: **type g4dn.xlarge** or higher with **100 GB disk** minimum
- OS: **Amazon Linux 2023** (al2023-ami-2023.11.20260514.0-kernel-6.1-x86_64)

## Setup Docker and Ollama

### 1. Install Docker

```bash
sudo yum update -y
sudo yum install docker -y
sudo usermod -a -G docker ec2-user
sudo systemctl enable docker.service
sudo systemctl start docker.service
sudo chown ec2-user:docker /var/run/docker.sock
sudo chmod 660 /var/run/docker.sock
```

### 2. Verify Docker Installation

```bash
cat /var/run/docker.sock
docker ps
docker run hello-world
```

### 3. Deploy Ollama Container

```bash
docker run -d \
  -v ollama:/root/.ollama \
  -p 11434:11434 \
  --name ollama \
  ollama/ollama
```

### 4. Verify Ollama is Running

```bash
docker ps
curl localhost:11434
```

### 5. Pull DeepSeek Model

```bash
docker exec -it ollama ollama pull deepseek-r1:7b-qwen-distill-q4_K_M
```

### 6. Deploy Ollama Web UI

```bash
docker run -d \
  -p 3000:8080 \
  --add-host=host.docker.internal:host-gateway \
  -v ollama-webui:/app/backend/data \
  --name ollama-webui \
  ghcr.io/ollama-webui/ollama-webui:main
```

### 7. Setup Nginx Reverse Proxy

```bash
sudo yum install nginx -y
sudo vim /etc/nginx/nginx.conf
```

Add the following configuration in the `location /` block:

```nginx
location / {
  proxy_pass http://localhost:3000;
}
```

Validate and enable Nginx:

```bash
sudo nginx -t
sudo systemctl enable nginx
```

### 8. Access the Web UI

Open your browser and navigate to:

```
http://<your-ec2-ip>:3000/
```

## Enable GPU for Ollama

At this point, Ollama runs on CPU only and may be slow. Follow these steps to enable GPU acceleration.

### 1. Check NVIDIA GPU

```bash
sudo nvidia-smi
```

### 2. Install NVIDIA Container Toolkit

```bash
curl -s -L https://nvidia.github.io/libnvidia-container/stable/rpm/nvidia-container-toolkit.repo | \
  sudo tee /etc/yum.repos.d/nvidia-container-toolkit.repo

sudo dnf install -y nvidia-container-toolkit
```

### 3. Configure Docker Runtime for GPU

```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

### 4. Stop and Remove the Current Ollama Container

```bash
docker ps
docker stop ollama
docker rm ollama
```

### 5. Deploy Ollama with GPU Support

```bash
docker run -d \
  --gpus all \
  -v ollama:/root/.ollama \
  -p 11434:11434 \
  --name ollama \
  ollama/ollama
```

### 6. Install NVIDIA Drivers and CUDA Toolkit

```bash
sudo dnf install nvidia-release -y
sudo dnf install kernel-devel-$(uname -r) kernel-headers-$(uname -r) -y
sudo dnf install nvidia-driver-cuda cuda-toolkit -y
```

### 7. Verify GPU Installation

```bash
nvidia-smi
```

### 8. Pull DeepSeek Model Again

```bash
docker ps -a
docker exec -it ollama ollama pull deepseek-r1:7b-qwen-distill-q4_K_M
```

### 9. Restart Nginx

```bash
sudo systemctl restart nginx
```

---

Your DeepSeek deployment is now ready with GPU acceleration! Access it at `http://<your-ec2-ip>:3000/`
