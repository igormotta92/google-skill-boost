# Configurar Load Balancer no GCP

## Passo 1: Criar Instância www1, www2, www3

```bash
gcloud compute instances create www1 \
    --zone=europe-west1-b \
    --tags=network-lb-tag \
    --machine-type=e2-small \
    --image-family=debian-11 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "Web Server: www1" | tee /var/www/html/index.html'

gcloud compute instances create www2 \
    --zone=europe-west1-b \
    --tags=network-lb-tag \
    --machine-type=e2-small \
    --image-family=debian-11 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "Web Server: www2" | tee /var/www/html/index.html'

gcloud compute instances create www2 \
    --zone=europe-west1-b \
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

**Saída no terminal (exemplo):**
```
Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-02-dce56a6f7d09/zones/europe-west1-b/instances/www1].
NAME: www1
ZONE: europe-west1-b
MACHINE_TYPE: e2-small
PREEMPTIBLE:
INTERNAL_IP: 10.132.0.2
EXTERNAL_IP: 34.53.179.193
STATUS: RUNNING
```

## Passo 2: Criar Endereço IP Estático

```bash
gcloud compute addresses create network-lb-ip-1 \
  --region europe-west1
```

**Saída no terminal:**
```
Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-02-dce56a6f7d09/regions/europe-west1/addresses/network-lb-ip-1].
```

## Passo 3: Criar Health Check

```bash
gcloud compute http-health-checks create basic-check
```

**Saída no terminal:**
```
Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-02-dce56a6f7d09/global/httpHealthChecks/basic-check].
NAME: basic-check
HOST:
PORT: 80
REQUEST_PATH: /
```

## Passo 4: Criar Target Pool

```bash
gcloud compute target-pools create www-pool \
  --region europe-west1 --http-health-check basic-check
```

**Saída no terminal:**
```
Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-02-dce56a6f7d09/regions/europe-west1/targetPools/www-pool].
NAME: www-pool
REGION: europe-west1
SESSION_AFFINITY: NONE
BACKUP:
HEALTH_CHECKS: basic-check
```

## Passo 5: Adicionar Instâncias ao Pool

```bash
gcloud compute target-pools add-instances www-pool \
    --instances www1,www2,www3
```

**Saída no terminal:**
```
Updated [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-02-dce56a6f7d09/regions/europe-west1/targetPools/www-pool].
```

## Passo 6: Criar Regra de Forwarding

```bash
gcloud compute forwarding-rules create www-rule \
    --region europe-west1 \
    --ports 80 \
    --address network-lb-ip-1 \
    --target-pool www-pool
```

**Saída no terminal:**
```
Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-02-dce56a6f7d09/regions/europe-west1/forwardingRules/www-rule].
```

## Passo 7: Obter Endereço IP do Load Balancer

```bash
gcloud compute forwarding-rules describe www-rule --region europe-west1
```

**Saída no terminal:**
```
IPAddress: 130.211.100.194
IPProtocol: TCP
creationTimestamp: '2026-05-06T08:20:03.810-07:00'
description: ''
fingerprint: zPhaqwjcdJE=
id: '9159338832076137164'
kind: compute#forwardingRule
labelFingerprint: 42WmSpB8rSM=
loadBalancingScheme: EXTERNAL
name: www-rule
networkTier: PREMIUM
portRange: 80-80
region: https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-02-dce56a6f7d09/regions/europe-west1
selfLink: https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-02-dce56a6f7d09/regions/europe-west1/forwardingRules/www-rule
selfLinkWithId: https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-02-dce56a6f7d09/regions/europe-west1/forwardingRules/9159338832076137164
target: https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-02-dce56a6f7d09/regions/europe-west1/targetPools/www-pool
```

```bash
IPADDRESS=$(gcloud compute forwarding-rules describe www-rule --region europe-west1 --format="json" | jq -r .IPAddress)
echo $IPADDRESS
```

**Saída no terminal:**
```
130.211.100.194
```

## Passo 8: Testar Load Balancer

```bash
while true; do curl -m1 130.211.100.194; done
```

**Saída no terminal (exemplo):**
```
Web Server: www2

Web Server: www2

Web Server: www2

Web Server: www3

Web Server: www2

Web Server: www1
```

**Resultado esperado:** As requisições são distribuídas entre www1, www2 e www3.