# Cloudrun_sidecar_container

use cloudrun container as reserver proxy and routed traffic to sidecar container having main application on it named wikijs

## In this scenario, we are using Cloud Run with a sidecar container. Nginx is running on the main application container as a reverse proxy and it will forward the traffic to the sidecar container where our application is running.

## Dockerfile for creating nginx images

    FROM nginx
    COPY docs.disearch.conf /etc/nginx/conf.d/default.conf


### docs.disearch.conf

    gzip on;
    gzip_disable "msie6";
    
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_buffers 16 8k;
    gzip_http_version 1.1;
    gzip_types text/plain text/css application/json application/javascript application/x-javascript text/xml application/xml application/xml+rss text/javascript;
    
    server {
        listen 8080;
        server_name docs.disearch.ai;
    
        access_log /var/log/nginx/access.log;
    
        location / {
            proxy_pass http://localhost:3000;  # Assuming your WikiJS container listens on port 3000
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    
        location /protected/ {
            if ($token_present = 0) {
                return 403;
            }
            proxy_pass http://localhost:3000/protected/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    
        location /_assets/ {
            proxy_pass http://localhost:3000/_assets/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
     
## NOW BUILD , TAG AND PUSH IMAGE to gcr

    docker build -t docs-proxy:latest .
    docker tag docx-proxy:latest gcr.io/disearch/docs-proxy:latest
    docker push gcr.io/disearch/docs-proxy:latest

    now your images will be showing in gcr under docs-proxy repositary. you can use this in cloud run as main container.

### sidecar container image.

    pull wikijs images from public repo in localmachine and push it on GCP gcr. So we can use this image in sidecar container. 
 
    docker pull ghcr.io/requarks/wiki:2
    docker tag requarks/wiki:2 gcr.io/disearch/wikijs-image:latest
    docker push gcr.io/disearch/wikijs-image:latest

    now your images will be showing in gcr under wikijs-image repositary. you can use this in cloud run as sidecar container.    

### cloudrun service

    #Container Sections for main ngnix container 
     #ingress container
     Container Image URL --------> select nginx(docs-proxy) image from gcr
     Container name      --------> docs-proxy-1
     Container port      --------> 8080    
     Resources           ---------> memory 512MB , CPU 1 (DEFUAlT)

    #Container Sections for sidecar container where main application is running
     click on "ADD Container" 
     Container Image URL --------> select wikijs-image image from gcr 
     Resources           ---------> memory 512MB , CPU 1 (DEFUAlT) 
     Depends on          ----------> docs-proxy-1

       #side car contiainer env section(variable&secrets)  --> by using this information our wikijs application running as sidecar will connect with cloud SQL database
       #Environment variables
       DB_TYPE: postgres
       DB_HOST: 10.90.240.4  --> DB PRIVATE IP
       DB_PORT: 5432
       DB_USER: postgres
       DB_PASS: M<2ZUQhV8-z^YKt}0OYcBvZ)%.j6N*
       DB_NAME: postgres          -------------->  in my case postgres datebase is already created that is why i am using it again here. if you face "migration table" is already created error. then you need to create a new database in cloud SQL. for this access the cloud SQL database with this command "psql -h <db-publicip-address> -U postgres" (make sure to make you cloud SQL public so you can take and use the public ip address of cloud sql when you are going to access the cloud sql through local terminal using command  or through web(because your traffic will be going through the public internet so you need a public ip)) and Because we have created our database in a way that no unknown user can access the database. only the authorized user can access it. so you need to authorized you public ip in cloud sql first before accessing it through terminal(steps: get your public ip(ISP ip)> then go to gcp cloud sql> connection > network > Authorized networks > give&save ip address without /32) 
 
                   
         then usr this command again "psql -h <db-publicip-address> -U postgres"  and give password. With this you can successfully access the cloud SQL.  once you get accessed cloud sql, use command to create database having name wikijs. command "CREATE DATABASE wikijs;". 
       
    #Requests
    Request timeout:300
    Maximum concurrent requests per instance: 80
    
    #CPU allocation and pricing
    CPU is only allocated during request processing: selected
    CPU is always allocated: not selected

    # Execution Environment
    DefaultL selected
    first generation: not selected
    secend generation: not selected
   
    #Revision autoscaling
    Minimum number of instances: 1 
    Maximum number of instances: 100

    #Startup CPU boost
    Start containers faster by allocating more cpu during startup time: selected

    # Cloud SQL connection
     cloud SQL instance 1: disearch:us-east1:search-db: selected ----> so my side car container having main wikijs application can connect with gcp SQL postgres database.  we have given (db user , password , host and bd name), we will be giving this information to side car container from envirmental variable. so our wikijs application can connect with postgress database.
    
    #cloudrun networking section
     Connect to a VPC for outbound traffic: selected
      Send traffic directly to a VPC: selected
       network: default
       Subnet:  default
     Traffic routing: selected
       Route all traffic to the VPC: selected
      
    #cloudrun security section
    Service account(default) 
      Compute Engine default service account   

    now click on deploy. ---> at this step you cloud run main and side car container should be running.

### DOMAIN MAPPING THROUGH CLOUD RUN

    The Benefit for using DOMAIN MAPPING THROUGH CLOUD RUN is. it will automatically set SSL certificate with your domain. you no need to do any extra step to setting the SSL certificate.  what you have done in case of loadbalancer+cloudrun

    step: go to cloud run service > click on Manage domain > then click on ADD MAPPING > then select a service to map(in my case it is disearch)> then click on cloud run domain mapping> then give verified domain(your domain like "disearch.ai")> and give subdomain "docs" > then continue 

    it will give you a CNAME. copy the CNAME and switch your project to "disearch" and go to cloud DNS > select zone disearch > select ADD standard >

    and give

    DNS NAME: docs(subdomain)
    Resource record type: CNAME 
    TTL: 5 
    TTL unit: minutes
    Canonical name: give copied CNAME  and then click on create 

now once the domain creation is done. go to browser and test. nginx container will redirect the traffic to wikijs running in side car container.
    
