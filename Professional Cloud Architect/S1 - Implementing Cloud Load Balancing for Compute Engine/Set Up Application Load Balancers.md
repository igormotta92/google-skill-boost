# Configurar Application Load Balancer no GCP

## Passo 1: Configurar Região e Zona Padrão

```bash
gcloud config set compute/region us-central1
```

**Saída no terminal:**
```
Updated property [compute/region].
```

```bash
gcloud config set compute/zone us-central1-b
```

**Saída no terminal:**
```
Updated property [compute/zone].
```

## Passo 2: Criar Instância www1

```bash
gcloud compute instances create www1 \
    --zone=us-central1-b \
    --tags=network-lb-tag \
    --machine-type=e2-small \
    --image-family=debian-11 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "Web Server: www1" | tee /var/www/html/index.html'
```

**Saída no terminal:**
```
Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-00-d99d03ef4165/zones/us-central1-b/instances/www1].
NAME: www1
ZONE: us-central1-b
MACHINE_TYPE: e2-small
PREEMPTIBLE:
INTERNAL_IP: 10.128.0.2
EXTERNAL_IP: 34.16.3.57
STATUS: RUNNING
```

## Passo 3: Criar Instância www2

```bash
gcloud compute instances create www2 \
    --zone=us-central1-b \
    --tags=network-lb-tag \
    --machine-type=e2-small \
    --image-family=debian-11 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "Web Server: www2" | tee /var/www/html/index.html'
```

**Saída no terminal:**
```
Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-00-d99d03ef4165/zones/us-central1-b/instances/www2].
NAME: www2
ZONE: us-central1-b
MACHINE_TYPE: e2-small
PREEMPTIBLE:
INTERNAL_IP: 10.128.0.3
EXTERNAL_IP: 35.225.17.198
STATUS: RUNNING
```

## Passo 4: Criar Instância www3

```bash
gcloud compute instances create www3 \
    --zone=us-central1-b  \
    --tags=network-lb-tag \
    --machine-type=e2-small \
    --image-family=debian-11 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "Web Server: www3" | tee /var/www/html/index.html'
```

**Saída no terminal:**
```
Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-00-d99d03ef4165/zones/us-central1-b/instances/www3].
NAME: www3
ZONE: us-central1-b
MACHINE_TYPE: e2-small
PREEMPTIBLE:
INTERNAL_IP: 10.128.0.4
EXTERNAL_IP: 35.202.147.131
STATUS: RUNNING
```

## Passo 5: Listar Instâncias

```bash
gcloud compute instances list
```

**Saída no terminal:**
```
NAME: www1
ZONE: us-central1-b
MACHINE_TYPE: e2-small
PREEMPTIBLE:
INTERNAL_IP: 10.128.0.2
EXTERNAL_IP: 34.16.3.57
STATUS: RUNNING

NAME: www2
ZONE: us-central1-b
MACHINE_TYPE: e2-small
PREEMPTIBLE:
INTERNAL_IP: 10.128.0.3
EXTERNAL_IP: 35.225.17.198
STATUS: RUNNING

NAME: www3
ZONE: us-central1-b
MACHINE_TYPE: e2-small
PREEMPTIBLE:
INTERNAL_IP: 10.128.0.4
EXTERNAL_IP: 35.202.147.131
STATUS: RUNNING
```

## Passo 6: Criar Firewall Rule para Network Load Balancer

```bash
gcloud compute firewall-rules create www-firewall-network-lb \
    --target-tags network-lb-tag --allow tcp:80
```

**Saída no terminal:**
```
Creating firewall...working..Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-00-d99d03ef4165/global/firewalls/www-firewall-network-lb].
Creating firewall...done.
NAME: www-firewall-network-lb
NETWORK: default
DIRECTION: INGRESS
PRIORITY: 1000
ALLOW: tcp:80
DENY:
DISABLED: False
```

## Passo 7: Testar Conectividade com Instâncias

```bash
curl http://34.16.3.57
```

**Saída no terminal:**
```
Web Server: www1
```

```bash
curl http://35.225.17.198
```

**Saída no terminal:**
```
Web Server: www2
```

```bash
curl http://35.202.147.131
```

**Saída no terminal:**
```
Web Server: www3
```

## Passo 8: Criar Instance Template

```bash
gcloud compute instance-templates create lb-backend-template \
   --region=us-central1 \
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
```

**Saída no terminal:**
```
Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-00-d99d03ef4165/global/instanceTemplates/lb-backend-template].
NAME: lb-backend-template
MACHINE_TYPE: e2-medium
PREEMPTIBLE:
CREATION_TIMESTAMP: 2026-05-06T10:49:31.068-07:00
```

## Passo 9: Criar Managed Instance Group

```bash
gcloud compute instance-groups managed create lb-backend-group \
   --template=lb-backend-template --size=2 --zone=us-central1-b
```

**Saída no terminal:**
```
Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-00-d99d03ef4165/zones/us-central1-b/instanceGroupManagers/lb-backend-group].
NAME: lb-backend-group
LOCATION: us-central1-b
SCOPE: zone
BASE_INSTANCE_NAME: lb-backend-group
SIZE: 0
TARGET_SIZE: 2
INSTANCE_TEMPLATE: lb-backend-template
AUTOSCALED: no
```

## Passo 10: Criar Firewall Rule para Health Check

```bash
gcloud compute firewall-rules create fw-allow-health-check \
  --network=default \
  --action=allow \
  --direction=ingress \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --target-tags=allow-health-check \
  --rules=tcp:80
```

**Saída no terminal:**
```
Creating firewall...working..Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-00-d99d03ef4165/global/firewalls/fw-allow-health-check].
Creating firewall...done.
NAME: fw-allow-health-check
NETWORK: default
DIRECTION: INGRESS
PRIORITY: 1000
ALLOW: tcp:80
DENY:
DISABLED: False
```

## Passo 11: Criar Endereço IP Estático Global

```bash
gcloud compute addresses create lb-ipv4-1 \
  --ip-version=IPV4 \
  --global
```

**Saída no terminal:**
```
Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-00-d99d03ef4165/global/addresses/lb-ipv4-1].
```

## Passo 12: Descrever Endereço IP

```bash
gcloud compute addresses describe lb-ipv4-1 \
  --format="get(address)" \
  --global
```

**Saída no terminal:**
```
34.160.31.170
```

## Passo 13: Criar Health Check HTTP

```bash
gcloud compute health-checks create http http-basic-check \
  --port 80
```

**Saída no terminal:**
```
Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-00-d99d03ef4165/global/healthChecks/http-basic-check].
NAME: http-basic-check
PROTOCOL: HTTP
```

## Passo 14: Criar Backend Service

```bash
gcloud compute backend-services create web-backend-service \
  --protocol=HTTP \
  --port-name=http \
  --health-checks=http-basic-check \
  --global
```

**Saída no terminal:**
```
Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-00-d99d03ef4165/global/backendServices/web-backend-service].
NAME: web-backend-service
BACKENDS:
PROTOCOL: HTTP
```

## Passo 15: Adicionar Backend ao Backend Service

```bash
gcloud compute backend-services add-backend web-backend-service \
  --instance-group=lb-backend-group \
  --instance-group-zone=us-central1-b \
  --global
```

**Saída no terminal:**
```
Updated [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-00-d99d03ef4165/global/backendServices/web-backend-service].
```

## Passo 16: Criar URL Map

```bash
gcloud compute url-maps create web-map-http \
    --default-service web-backend-service
```

**Saída no terminal:**
```
Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-00-d99d03ef4165/global/urlMaps/web-map-http].
NAME: web-map-http
DEFAULT_SERVICE: backendServices/web-backend-service
```

## Passo 17: Criar HTTP Target Proxy

```bash
gcloud compute target-http-proxies create http-lb-proxy \
    --url-map web-map-http
```

**Saída no terminal:**
```
Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-00-d99d03ef4165/global/targetHttpProxies/http-lb-proxy].
NAME: http-lb-proxy
URL_MAP: web-map-http
```

## Passo 18: Criar Forwarding Rule

```bash
gcloud compute forwarding-rules create http-content-rule \
   --address=lb-ipv4-1\
   --global \
   --target-http-proxy=http-lb-proxy \
   --ports=80
```

**Saída no terminal:**
```
Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-00-d99d03ef4165/global/forwardingRules/http-content-rule].
```

**Resultado esperado:** O Application Load Balancer está pronto para distribuir requisições para o backend service nas diferentes regiões e instâncias.