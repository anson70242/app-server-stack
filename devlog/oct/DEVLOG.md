# App Server Stack - DevLog
## 2025-10-28 - Day 1
**Milestone:** Project initialized. Deployed the base infrastructure on AWS Lightsail and successfully launched the Nginx container via Docker.

### System Setup
* **Host:** AWS Lightsail (`app-server-1`)

* **OS:** Ubuntu 24.04 LTS

* **Tech Stack:** Docker + Docker Compose

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