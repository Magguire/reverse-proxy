# Dockerized Nginx Reverse Proxy Service

### Steps to run a  dockerized nginx reverse proxy.

1. Create custom nginx.conf file to include extra logging formats. 

```
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;


    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
    	 	      '"$http_user_agent" "$http_x_forwarded_for"';



    log_format custom_format '$remote_addr - $remote_user [$time_local] "$request" '
                             'status=$status body_bytes_sent=$body_bytes_sent '
                             '"$http_referer" "$http_user_agent" '
                             'request_body=$request_body '
                             'request_uri=$request_uri '
                             'args=$args '
                             'authorization=$http_authorization';

    
    log_format json_format '$remote_addr - '
                           'user $remote_user '
	                   '[$time_local] - '
                           '"$request" '
			   '"status $status" '
                           '"Bytes $body_bytes_sent"   '
                           '"Referer $http_referer"    '
                           '"Agent $http_user_agent" \n'
			   '{'
			   '"Body":$request_body, '
                           '"Authorization":$http_authorization, '
                           '"Args":$args'
                            '}\n';


    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;
    
    # include all conf extension files in conf.d folder.
    include /etc/nginx/conf.d/*.conf;
}
```

2. Create default.conf file with server configuration and location paths. For this example, we shall use localhost as server and map mpesa daraja b2c API to the path 'mpesa/b2c' as follows:

```
server {
	listen 80;
	server_name localhost;


	# Logging files
     	access_log /logs/nginx/access.log; # uses default main logging
     	access_log /logs/nginx/requests.log json_format; # uses customized json_format logging
     	error_log  /logs/nginx/error.log; # uses default main logging

     
	location /mpesa/b2c {
             #proxy_ssl_server_name on;
             proxy_pass https://sandbox.safaricom.co.ke/mpesa/b2c/v3/paymentrequest;
             #proxy_buffering off;
             proxy_http_version 1.1;
             proxy_set_header Upgrade $http_upgrade;
             proxy_set_header Connection "upgrade";
             proxy_set_header X-Real-IP $remote_addr;
             proxy_set_header X-Forwarded-Host $host;
             #proxy_set_header X-Forwarded-Port $server_port;

             # Enable reading request body for logging purposes
            client_body_buffer_size 128k;
            client_max_body_size 128k;
            client_body_in_single_buffer on;
        }
    }

```


3.  Create nginx folder under /var/log/ locally if it is not available where the logs will be stored.

4.  Create docker-compose.yaml file. Map nginx.conf, default.conf and /var/log/ accordingly.

```
services:
  nginx:
    restart: always
    container_name: nginx-proxy
    image: nginx:latest
      #ports:
      #- "80:80"
      #- "443:443"
    network_mode: host
    volumes:
      # - local:container
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./default.conf:/etc/nginx/conf.d/default.conf
      - /var/log:/logs
```


5. Run the container with the following command:

```
docker compose up --build -d
```


### Points to note

- In order to have readable request logs, you can use the following to dump the logs to another file:


```
sed -e 's/\\x22/"/g' -e 's/\\x0A//g' -e 's/  */ /g' requests.log > cleaned_requests.log
```


The above command uses sed package to:

- Replace \x22 with "
- Remove \x0A 
- Remove extra spaces.



- To stop the docker container, run the following command:

```
docker compose down
```


- nginx.conf and default.conf files can be edited to include other custom logging formats and location paths respectively as needed. After saving changes, stop  and rerun the container to rewrite the files.


