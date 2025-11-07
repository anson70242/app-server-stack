# App Server Stack - DevLog
## 2025-11-06 - Day 3 and 2025-11-07 - Day 4

**Today Goal 1:** Switching the model to `gpt-oss-20b`
Stop and remove the old container (if it's running):

```sh
docker stop $(docker ps -q --filter ancestor=vllm/vllm-openai:latest)
```
Deploy the new container for `gpt-oss-20b`:
```sh
docker run -d --runtime nvidia --gpus all \
	--name my_vllm_container \
    -v $HOME/.cache/huggingface:/root/.cache/huggingface \
    -v /model/huggingface/hub:/model/huggingface/hub \
    --env "HUGGING_FACE_HUB_TOKEN=$HF_TOKEN" \
    --env "CUDA_VISIBLE_DEVICES=1" \
    -p 8000:8000 \
    --ipc=host \
    vllm/vllm-openai:latest \
    --model unsloth/gpt-oss-20b
```
Add `-v /model/huggingface/hub:/model/huggingface/hub` only if your system have symlink on `~/.cache/huggingface`.

---

If hugging-face model having download problem:

```sh
# Manually donwload the model first
pip uninstall hf_xet
export HF_HUB_ENABLE_HF_TRANSFER=0

# Start a new shell then:
hf download unsloth/gpt-oss-20b
```
---
Check conmtainer logs:
```sh
docker logs -f $(docker ps -q --filter ancestor=vllm/vllm-openai:latest)
```
Call the server using curl:
```sh
curl -X POST "http://54.65.190.24/llm_api/chat/completions" \
	-H "Content-Type: application/json" \
	--data '{
		"model": "unsloth/gpt-oss-20b",
		"messages": [
			{
				"role": "user",
				"content": "What is the capital of France?"
			}
		]
	}'
```



**Today Goal 2:** Add API Key

1. Use `openssl rand -hex 32` to generate a secret key

2. Add to `.env`

   ```sh
   GPU_SERVER_IP=# Your GPU Server IP
   LLM_API_KEY=f3a8...e9d2 # <-- here
   ```

3. Also add to `docker-compose.yml`

   ```sh
   environment:
   - GPU_SERVER_IP=${GPU_SERVER_IP}
   - LLM_API_KEY=${LLM_API_KEY} # <-- here
   ```

4. And `nginx/nginx.conf.template`

   ```sh
   location /llm_api/ {
       # --- API Key Check ---
       # Check if the incoming request has an 'Api-Key' HTTP Header
       # Nginx automatically converts 'Api-Key' to the $http_api_key variable
       #
       # If $http_api_key (the key from the client)
       # does not match ${LLM_API_KEY} (the key from our .env)
       # Nginx will immediately stop processing and return 401 Unauthorized
       if ($http_api_key != "${LLM_API_KEY}") {
         return 401 'Unauthorized';
       } 
       # --- End of Check ---
   
       # (If the key is correct, Nginx continues with the settings below)
   
       # Forward the request to the GPU server, rewriting the path.
       ...
   ```

5. Restart docker container:
   ```sh
   docker-compose down
   docker-compose up --build -d
   ```

6. Check if `api_key` working:

   ```sh
   curl -X POST "http://Nginx_Server_IP/llm_api/chat/completions" \
   	-H "Content-Type: application/json" \
   	-H "Api-Key: # your api key here #" \
   	--data '{
   		"model": "unsloth/gpt-oss-20b",
   		"messages": [
   			{
   				"role": "user",
   				"content": "What is the capital of France?"
   			}
   		]
   	}'
   ```
   
   

## 2025-10-29 - Day 2

**Today Goal 1:** Create `.env` for safety:

1. Create `.env` by `vim .env`:

   ```vimrc
   GPU_SERVER_IP=#Your GPU Server IP
   ```

   

2. Update `docker-compose.yml`:

   ```yaml
   version: '3.8' # commonly used version
   
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
           # Create a env var inside Nginx container
           - GPU_SERVER_IP=${GPU_SERVER_IP}
   ```

   

3. Update `nginx/nginx.conf`:

   - Rename `nginx/nginx.conf` to `nginx/nginx.conf.template`

     ```sh
     mv nginx/nginx.conf nginx/nginx.conf.template
     ```

   - Modify `nginx/nginx.conf.template`:

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

   

5. Restart docker containers

   ```sh
   docker-compose down
   docker-compose up --build -d
   ```

---

**Today Goal 2:** Set up the vLLM server, configure Nginx to forward LLM API requests to the vLLM server.

First, `ssh` to the GPU server.

- Docker Installation: 

  ```sh
  sudo apt install docker.io docker-compose -y
  ```

- Installing the NVIDIA Container Toolkit (With `apt`: Ubuntu, Debian) for the vllm container [Source](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)

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


Second, deploy the  vLLm container:

1. Export your Hugging Face token:
   ```sh
   export HF_TOKEN="#your hf token here"
   ```

2. Use vLLM's Official Docker Image

   ```sh
docker run -d --runtime nvidia --gpus all \
       -v $HOME/.cache/huggingface:/root/.cache/huggingface \
       --env "HUGGING_FACE_HUB_TOKEN=$HF_TOKEN" \
       --env "CUDA_VISIBLE_DEVICES=1" \
    -p 8000:8000 \
       --ipc=host \
       vllm/vllm-openai:latest \
       --model Qwen/Qwen3-0.6B
   ```
   
   For Multi GPU:
   
   Add `--tensor-parallel-size 2` to **split a model** into multiple GPUs like `Llama 3 70B`.
   
   Add `--data-parallel-size 2` to load the whole model into each GPU for **more throughput** like `Qwen3-0.6B`.

   ***Note: A Workstation motherboard is required for multi-GPU communication.**

   *Note: Use `--env "CUDA_VISIBLE_DEVICES=0"` or remove this line if you have only 1 GPU.

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
    version: '3.8' # commonly used version
    
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

    * For `nginx`, I created 3 files: 

        * Dockerfile, to create the nginx container.

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
        
        * nginx.conf, for custom configuration.
        
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