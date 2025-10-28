# Use the official Nginx image as the base image
FROM nginx:alpine
# Remove the default Nginx configuration file
RUN rm /etc/nginx/conf.d/default.conf
# Copy the custom Nginx configuration file to the container
COPY nginx.conf /etc/nginx/conf.d
# Copy the static HTML file to the Nginx HTML directory
COPY index.html /usr/share/nginx/html