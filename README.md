# Master Nginx-Docker Configuration for Kriya25

Application:

1. Simple Content Delivery Network (CDN)
2. Load Balancing between payment providers with weights
3. DNS Proxy for all Kriya25 websites

> [!NOTE]  
> `nginxmongo` service was added to measure the performance of the Load Balancer. Remove it or modify it as per requirement
## How to add custom domain when nginx is running in a Docker Container

To add a custom domain for the payment service and set up SSL using Let's Encrypt when running NGINX in a Docker container, follow these steps:

1. **Update nginx.conf to use the custom domain:**

```properties
events {}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    upstream payment {
        server payment_service_1:6010 weight=3;  # 75% of the requests
        server payment_service_2:6011 weight=1;  # 25% of the requests
    }
    server {
        listen 80;
        server_name your.custom.domain;

        location / {
            proxy_pass http://payment;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location /cdn/ {
            alias /usr/share/nginx/cdn/;
        }
    }
}
```

2. **Create a Docker Compose file with Certbot:**

Update your docker-compose.yml to include a Certbot container and configure NGINX for SSL:

```yaml
version: '3.8'

services:
  nginxmongo:
    image: mongo
    ports:
      - "27030:27017"
    environment:
      - MONGO_INITDB_ROOT_USERNAME=admin
      - MONGO_INITDB_ROOT_PASSWORD=password
    volumes:
      - ./mongo_data:/data/db
    networks:
      - cdn_network

  nginx:
    image: nginx:latest
    container_name: nginx_cdn
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./mime.types:/etc/nginx/mime.types:ro
      - ./content:/usr/share/nginx/cdn:ro
      - ./certbot/conf:/etc/letsencrypt
      - ./certbot/www:/var/www/certbot
    networks:
      - cdn_network

  payment_service_1:
    image: payment1
    container_name: payment_service_1
    environment:
      - MONGOURI=mongodb://admin:password@nginxmongo:27017
    ports:
      - "6010:6010"
    networks:
      - cdn_network

  payment_service_2:
    image: payment2
    container_name: payment_service_2
    environment:
      - MONGOURI=mongodb://admin:password@nginxmongo:27017
    ports:
      - "6011:6011"
    networks:
      - cdn_network

  certbot:
    image: certbot/certbot
    volumes:
      - ./certbot/conf:/etc/letsencrypt
      - ./certbot/www:/var/www/certbot
    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"
    networks:
      - cdn_network

networks:
  cdn_network:
    driver: bridge
```

3. **Obtain the SSL certificate using Certbot:**

Run the following command to obtain the SSL certificate:

```sh
docker-compose run --rm certbot certonly --webroot --webroot-path=/var/www/certbot -d your.custom.domain
```

4. **Update nginx.conf to use SSL:**

Update your nginx.conf to include the SSL configuration:

```properties
events {}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    upstream payment {
        server payment_service_1:6010 weight=3;  # 75% of the requests
        server payment_service_2:6011 weight=1;  # 25% of the requests
    }
    server {
        listen 80;
        server_name your.custom.domain;
        location / {
            return 301 https://$host$request_uri;
        }
    }

    server {
        listen 443 ssl;
        server_name your.custom.domain;

        ssl_certificate /etc/letsencrypt/live/your.custom.domain/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/your.custom.domain/privkey.pem;
        include /etc/letsencrypt/options-ssl-nginx.conf;
        ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

        location / {
            proxy_pass http://payment;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location /cdn/ {
            alias /usr/share/nginx/cdn/;
        }

        location /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }
    }
}
```

5. **Restart Docker Compose:**

Restart your Docker Compose services to apply the changes:

```sh
docker-compose down
docker-compose up -d
```

This setup will ensure that your NGINX server uses a custom domain and has SSL enabled using Let's Encrypt. The Certbot container will automatically renew the SSL certificates.