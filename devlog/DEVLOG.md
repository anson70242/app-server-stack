# App Server Stack - DevLog
## 2025-10-29 - Day 2

**Today Goal 1:** Create `.env` for safety:

1. Create `.env` by `vim .env`:

   ```vimrc
   GPU_SERVER_IP=#Your GPU Server IP
   ```

   

2. Update `docker-compose.yml`:

   ```yaml
   version: '3.8' # commendlly used version
   
   services:
     nginx:
       build: ./nginx # indicate the path to the Dockerfile
       ports:
         - 80:80 # map host port 80 to container port 80
       # restart when the container when it stops unexpectedly
       restart: unless-stopped 
       # For .env
       environment:
           # read gpu server ip from .env
           # Create a env_ver inside Nginx container
           - GPU_SERVER_IP=${GPU_SERVER_IP}
   ```

   

3. Update `nginx/nignx.conf`:

   - Rename `nginx/nignx.conf` to `nginx/default.conf.template`

     ```sh
     mv nginx/nginx.conf nginx/nginx.conf.template
     ```

   - Modify `nginx/default.conf.template`:

     ```conf
     server {
         listen 80; # Listen on port 80
         location / {
             # Serve static files from the specified directory
             root /usr/share/nginx/html;
             # Default file to serve
             index index.html index.htm;
         }
         
         # Location block for forwarding API requests to the vLLM server
         location /llm_api/ {
             # Forward the request to the GPU server, rewriting the path.
             # Example: /llm_api/chat -> /v1/chat
             proxy_pass http://${GPU_SERVER_IP}:8000/v1/;
     
             # Pass the original 'Host' header from the client to the upstream server
             proxy_set_header Host $host;
             
             # Pass the client's real IP address to the upstream server
             proxy_set_header X-Real-IP $remote_addr;
             
             # Pass the chain of IPs the request has gone through (standard proxy header)
             proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
             
             # Pass the original protocol (http or https) used by the client
             proxy_set_header X-Forwarded-Proto $scheme;
         }
     }
     ```

     

4. Update `nginx/Dockerfile`:

   From:

   ```dockerfile
   COPY nginx.conf /etc/nginx/conf.d
   ```

   To:

   ```dockerfile
   COPY nginx.conf.template /etc/nginx/templates/nginx.conf.template
   ```

   

5. Restart docker container

   ```sh
   docker-compose down
   docker-compose up --build -d
   ```

---

**Today Goal 2:** Setup the vllm server, configure nginx to forward LLM api request to vllm server.

First, `ssh` to the GPU server.

- Docker Installation: 

  ```sh
  sudo apt install docker.io docker-compose -y
  ```

- Installing the NVIDIA Container Toolkit (With `apt`: Ubuntu, Debian) for vllm container [Source](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)

  1. Install the prerequisites for the instructions below:

     ```sh
     sudo apt-get update && sudo apt-get install -y --no-install-recommends \
        curl \
        gnupg2
     ```

  2. Configure the production repository:

     ```sh
     curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
       && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
         sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
         sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
     ```

  3. Update the packages list from the repository:

     ```sh
     sudo apt-get update
     ```

  4. Install the NVIDIA Container Toolkit packages:
     ```sh
     export NVIDIA_CONTAINER_TOOLKIT_VERSION=1.18.0-1
       sudo apt-get install -y \
           nvidia-container-toolkit=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
           nvidia-container-toolkit-base=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
           libnvidia-container-tools=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
           libnvidia-container1=${NVIDIA_CONTAINER_TOOLKIT_VERSION}
     ```

- Configuration

  1. Configure the container runtime by using the `nvidia-ctk` command:

     ```sh
     sudo nvidia-ctk runtime configure --runtime=docker
     ```

     The `nvidia-ctk` command modifies the `/etc/docker/daemon.json` file on the host. The file is updated so that Docker can use the NVIDIA Container Runtime.

  2. Restart the Docker daemon:

     ```sh
     sudo systemctl restart docker
     ```


Second, deploy vllm container:

1. Export your huggingface token:
   ```sh
   export HF_TOKEN="#your hf token here"
   ```

2. Use vLLM's Official Docker Image

   Create a volume `hf-cache` to avoid permission problem.

   ```sh
   docker volume create hf-cache
   ```

   ```sh
   docker run -d --runtime nvidia --gpus all \
       -v hf-cache:/root/.cache/huggingface \
       --env "HUGGING_FACE_HUB_TOKEN=$HF_TOKEN" \
       --env "CUDA_VISIBLE_DEVICES=1" \
       -p 8000:8000 \
       --ipc=host \
       vllm/vllm-openai:latest \
       --model Qwen/Qwen3-0.6B
   ```

   For Multi GPU:

   Add `--tensor-parallel-size 2` to split model into multiple GPU like `Llama 3 70B`.

   Add `--data-parallel-size 2` to load whole model into each GPU for more **Throughput** like `Qwen3-0.6B`.

   ***Note: A Workstation motherboard is required for multi-gpu commication.**

3. Testing:

   ```sh
   curl http://GPU_Server_IP:8000/v1/chat/completions \
       -H "Content-Type: application/json" \
       -d '{
           "model": "Qwen/Qwen3-0.6B",
           "messages": [
               {"role": "user", "content": "Hello! What is your name?"}
           ]
       }'
   ```

   Testing Nginx forwarding:

   ```sh
   curl http://Nginx_Server_IP/llm_api/chat/completions \
       -H "Content-Type: application/json" \
       -d '{
           "model": "Qwen/Qwen3-0.6B",
           "messages": [
               {"role": "user", "content": "Hello! What is your name?"}
           ]
       }'



## 2025-10-28 - Day 1

**Milestone:** Project initialized. Deployed the base infrastructure on AWS Lightsail and successfully launched the Nginx container via Docker.

### System Setup
* **Host:** AWS Lightsail (`app-server-1`)

* **OS:** Ubuntu 24.04 LTS

* **Tech Stack:** Docker + Docker Compose

  * Docker Installation: 

    ```sh
    sudo apt install docker.io docker-compose -y
    ```
  
  * Docker Compose file:
  
    ```yaml
    version: '3.8' # commendlly used version
    
    services:
      nginx:
        build: ./nginx # indicate the path to the Dockerfile
        ports:
          - 80:80 # map host port 80 to container port 80
        # restart when the container when it stops unexpectedly
        restart: unless-stopped 
    ```
  
    
  


### Progress
* **Service (`nginx`):** Nginx is now functioning. It's serving a static `index.html` test page, accessible from the public IP (Port 80).

    * For `nginx`, I created 3 file: 

        * Dockerfile, create the nginx continer.

            ```dockerfile
            # Use the official Nginx image as the base image
            FROM nginx:alpine
            # Remove the default Nginx configuration file
            RUN rm /etc/nginx/conf.d/default.conf
            # Copy the custom Nginx configuration file to the container
            COPY nginx.conf /etc/nginx/conf.d
            # Copy the static HTML file to the Nginx HTML directory
            COPY index.html /usr/share/nginx/html
            ```
        
        * nginx.conf, for custom configuraction.
        
            ```conf
            server {
                listen 80; # Listen on port 80
                location / {
                    # Serve static files from the specified directory
                    root /usr/share/nginx/html;
                    # Default file to serve
                    index index.html index.htm;
                }
            }
            ```
        
        * index.html, for a static html page, for testing.
        
            ```html
            <h1>Nginx is working!</h1>
            <p>This is the test page from your app-server-1.</p>
            ```
        
            

* **Version Control:** Initialized the `app-server-stack` project and pushed it to a private GitHub repository.

* **Vim Setup:** Configured the `.vimrc` file to automatically handle different indentations:
    * `yaml` files: 2 spaces
    * `nginx` / `conf` files: 4 spaces
    * Default: 4 spaces
    ```vimrc
    filetype plugin indent on
    
    set tabstop=4
    set shiftwidth=4
    set expandtab
    set number
    
    autocmd FileType yaml setlocal tabstop=2 shiftwidth=2 expandtab
    autocmd FileType conf setlocal tabstop=4 shiftwidth=4 expandtab
    autocmd FileType nginx setlocal tabstop=4 shiftwidth=4 expandtab
    ```

### Next Steps
* Begin building the `mcp_server` (FastAPI) service.