
create an instance with name "nucleus-jumphost" with network:nucleus-vpc, zone: us-east1, machine-type: f1-micro and image-family: debian-9
```
gcloud compute instances create nucleus-jumphost \
          --network nucleus-vpc \
          --zone us-east1-b  \
          --machine-type f1-micro  \
    --image-family debian-9  \
          --image-project debian-cloud \
          --scopes cloud-platform \
          --no-address
```

create a cluster in the us-east1-b zone named "nucleus-backend"
```
gcloud container clusters create nucleus-backend \
          --num-nodes 1 \
          --network nucleus-vpc \
          --region us-east1
```

to authenticate the cluster run the following command
```
gcloud container clusters get-credentials nucleus-backend \
          --region us-east1
```

run the following kubectl create command to create a new Deployment hello-server from the hello-app container image:
```
kubectl create deployment hello-server \
          --image=gcr.io/google-samples/hello-app:2.0
```

to create a Kubernetes Service, which is a Kubernetes resource that lets you expose your application to external traffic with Compute Engine load balancer "LoadBalancer" at port "8080"
```
kubectl expose deployment hello-server \
          --type=LoadBalancer \
          --port 8080
```

inspect the hello-server Service by running kubectl get:
```
kubectl get service
```
create a startup script to be used by every virtual machine instance. This script sets up the Nginx server upon startup
```
cat << EOF > startup.sh
#! /bin/bash
apt-get update
apt-get install -y nginx
service nginx start
sed -i -- 's/nginx/Google Cloud Platform - '"\$HOSTNAME"'/' /var/www/html/index.nginx-debian.html
EOF
```

create an instance template, which uses the startup script with name "web-server-template" at specified machine-type and region
```
gcloud compute instance-templates create web-server-template \
          --metadata-from-file startup-script=startup.sh \
          --network nucleus-vpc \
          --machine-type g1-small \
          --region us-east1
```

create a target pool
```
gcloud compute target-pools create nucleus-pool
```

create a managed instance group using the instance template
```
gcloud compute instance-groups managed create web-server-group \
          --base-instance-name web-server \
          --size 2 \
          --template web-server-template \
          --target-pool nucleus-pool
          --region us-east1
```

configure a firewall so that you can connect to the machines on port 80 via the EXTERNAL_IP addresses
```
gcloud compute firewall-rules create web-server-firewall \
          --allow tcp:80 \
          --network nucleus-vpc
```

create a health check. Health checks verify that the instance is responding to HTTP or HTTPS traffic
gcloud compute http-health-checks create http-basic-check

define an HTTP service and map a port name to the relevant port for the instance group
```
gcloud compute instance-groups managed \
          set-named-ports web-server-group \
          --named-ports http:80 \
          --region us-east1
```

create a backend service
```
gcloud compute backend-services create web-server-backend \
          --protocol HTTP \
          --http-health-checks http-basic-check \
          --global
```

add the instance group into the backend service
```
gcloud compute backend-services add-backend web-server-backend \
          --instance-group web-server-group \
          --instance-group-region us-east1 \
          --global
```

create a default URL map that directs all incoming requests to all your instances
```
gcloud compute url-maps create web-server-map \
          --default-service web-server-backend
```

create a target HTTP proxy to route requests to your URL map
```
gcloud compute target-http-proxies create http-lb-proxy \
          --url-map web-server-map
```

create a global forwarding rule to handle and route incoming requests
```
gcloud compute forwarding-rules create http-content-rule \
        --global \
        --target-http-proxy http-lb-proxy \
        --ports 80

gcloud compute forwarding-rules list
```