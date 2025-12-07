# frontend/Dockerfile

# 1. Base image using Nginx
FROM nginx:alpine

# 2. Remove default config (optional, but tidy)
RUN rm /etc/nginx/conf.d/default.conf || true

# 3. Copy our static website into Nginx’s standard html folder
COPY index.html /usr/share/nginx/html/index.html

# 4. Simple default server (listening on port 80)
# If in the future you add your own nginx.conf — this step can be updated
RUN printf 'server {\n\
    listen 80;\n\
    server_name _;\n\
    root /usr/share/nginx/html;\n\
    index index.html;\n\
    location / {\n\
        try_files $uri $uri/ =404;\n\
    }\n\
}\n' > /etc/nginx/conf.d/f1-frontend.conf

# 5. Expose the port
EXPOSE 80

# 6. Start nginx in the foreground
CMD ["nginx", "-g", "daemon off;"]
