# gcloud-Load-Balancer
The following are the steps that I took to create a gcloud load balancer.
1. I set the default region and the default zone.
   gcloud config set compute/region us-east1
   gcloud config set compute/zone us-east1-b

2. Created a virtual machine called www1
     gcloud compute instances create www1 \
    --zone=us-east1-b \
    --tags=network-lb-tag \
    --machine-type=e2-small \
    --image-family=debian-11 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "
<h3>Web Server: www1</h3>" | tee /var/www/html/index.html'

4. Created a vitrual machine called www2
    gcloud compute instances create www2 \
    --zone=us-east1-b \
    --tags=network-lb-tag \
    --machine-type=e2-small \
    --image-family=debian-11 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "
<h3>Web Server: www2</h3>" | tee /var/www/html/index.html'

5. Created a virtual machine called www3
      gcloud compute instances create www3 \
    --zone=us-east1-b  \
    --tags=network-lb-tag \
    --machine-type=e2-small \
    --image-family=debian-11 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "
<h3>Web Server: www3</h3>" | tee /var/www/html/index.html'

6. Created a firewall rule to allow external traffic to the VM instances.
    gcloud compute firewall-rules create www-firewall-network-1b
     --target-tags network-lb-tag --allow tcp:80
   
8. Created a static external IP address for the load balancer to configure the server.
   gcloud compute addresses create network-lb-ip-1 \ --region us-east1

9. Added a legacy HTTP help check resource while configuring the server.
      gcloud compute http-health-checks create basic-check

10. Added a target pool in the same region as the instance.
    gcloud compute target-pools create www-pool \--region us-east1 --http-health-check basic-check

11. Added the instance to the pool.
      gcloud compute target-pools add-instances www-pool \ --instances www1,www2,www3
    
11. Added a forwarding rule:
    gcloud compute forwarding-rules create www-rule \
    --region us-east1 \
    --ports 80 \
    --address network-lb-ip-1 \
    --target-pool www-pool

12. Started sending traffic to the instances by doing the following commands.
    gcloud compute forwarding-rules describe www-rule --region us-east1
    IPADDRESS=$(gcloud compute forwarding-rules describe www-rule--region us-east1 --format="json" | jq -r. 34.69.234.3)
    echo $34.69.234.3
    while true; do curl -m1 $34.69.234.3; done

13. I created a HTTP load balancer.
    gcloud compute instance-templates create lb-backend-template \
   --region=Region \
   --network=default \
   --subnet=default \
   --tags=allow-health-check \
   --machine-type=e2-medium \
   --image-family=debian-11 \
   --image-project=debian-cloud \
   --metadata=startup-script='#!/bin/bash
     apt-get update
     apt-get install apache2 -y
     a2ensite default-ssl
     a2enmod ssl
     vm_hostname="$(curl -H "Metadata-Flavor:Google" \
     http://169.254.169.254/computeMetadata/v1/instance/name)"
     echo "Page served from: $vm_hostname" | \
     tee /var/www/html/index.html
     systemctl restart apache2'

14. Created a managed instance group based on the templates.
    gcloud compute instance-groups managed create lb-backend-group \ --template=lb-backend-template --size=2 --zone=us-east1-b

15. Created a fw-allow-health-check firewall rule.
    gcloud compute firewall-rules create fw-allow-health-check \
  --network=default \
  --action=allow \
  --direction=ingress \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --target-tags=allow-health-check \
  --rules=tcp:80

16. I Set up a global static external IP address for users to reach the load balancer.
    gcloud compute addresses create lb-ipv4-1 \ --ip-version=IPV4 \ --global

17. Created a health check for the load balancer
    gcloud compute health-checks create http http-basic-check \ --port 80

18. Created a backend service for the load balancer.
    gcloud compute backend-services add-backend web-backend-service \ --instance-group=lb-backend-group\ --instance-group-zone=Zone \ --global

19. Created a URL map to route the incoming requests to the default backend service.
    gcloud compute url-maps create web-map-http \ --default-service web-backend-service

20. Created a target HTTP proxy to route requests to the URL map.
    gcloud compute target-http-proxies create http-lb-proxy \ --url-map web-map-http
    
21. Created a global forwarding rule to route incoming requests to the proxy.
    gcloud compute forwarding-rules create http-content-rule \ --address=lb-ipv4-1\ --global / --target-http-proxy=http-lb-proxy \ --ports=80
    
