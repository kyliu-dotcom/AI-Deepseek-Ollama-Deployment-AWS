Deploying DeepSeek on AWS EC2 with NVIDIA GPU

Prerequisites
I assume you already have an AWS account and know how to launch EC2 instances, configure security groups, and create key pairs.
I also assume you already created an EC2 instance with at least type g4dn.xlarge and 100 GB disk size.
The commands below assume Amazon Linux 2023: al2023-ami-2023.11.20260514.0-kernel-6.1-x86_64.

Setup Docker and Ollama

        sudo yum update -y
        sudo yum install docker -y
        sudo usermod -a -G docker ec2-user
        sudo systemctl enable docker.service
        sudo systemctl start docker.service
        sudo chown ec2-user:docker /var/run/docker.sock
        sudo chmod 660 /var/run/docker.sock
        cat /var/run/docker.sock
        docker ps
        docker run hello-world
        docker run -d -v ollama:/root/.ollama -p 11434:11434 --name ollama ollama/ollama
        docker ps
        curl localhost:11434
        docker exec -it ollama ollama pull deepseek-r1:7b-qwen-distill-q4_K_M
        docker run -d -p 3000:8080 --add-host=host.docker.internal:host-gateway -v ollama-webui:/app/backend/data --name ollama-webui ghcr.io/ollama-webui/ollama-webui:main
        sudo yum install nginx -y
        sudo vim /etc/nginx/nginx.conf
                        location / {
                        proxy_pass http://localhost:3000;
                        }
           22  sudo nginx -t
           23  sudo systemctl enable nginx

open http://<ip>:3000/

Enable GPU for Ollama

At this point, the Ollama server runs on CPU only, so it may be slow.
To use GPU, install the NVIDIA driver and NVIDIA Container Toolkit with the commands below.

        sudo nvidia-smi
        curl -s -L https://nvidia.github.io/libnvidia-container/stable/rpm/nvidia-container-toolkit.repo | sudo tee /etc/yum.repos.d/nvidia-container-toolkit.repo
        sudo dnf install -y nvidia-container-toolkit
        sudo nvidia-ctk runtime configure --runtime=docker
        sudo nvidia-ctk runtime configure --runtime=docker
        sudo systemctl restart docker
        docker ps
        docker stop ollama
        docker rm ollama
        docker run -d --gpus all -v ollama:/root/.ollama -p 11434:11434 --name ollama ollama/ollama
        sudo dnf install nvidia-release -y
        sudo dnf install kernel-devel-$(uname -r) kernel-headers-$(uname -r) -y
        sudo dnf install nvidia-driver-cuda cuda-toolkit -y
        nvidia-smi
        docker ps -a
        docker exec -it ollama ollama pull deepseek-r1:7b-qwen-distill-q4_K_M
        docker ps -a
        sudo systemctl restart nginx
