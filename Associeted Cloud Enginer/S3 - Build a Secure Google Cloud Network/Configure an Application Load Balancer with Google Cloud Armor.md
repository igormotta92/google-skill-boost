# Passo a Passo: Configuração de HTTP Load Balancer com Firewall, Instance Groups e Cloud Armor (GCP)

## Task 1. Configurar regras de firewall para HTTP e Health Check

### Criar regra de firewall para HTTP
```bash
gcloud compute firewall-rules create default-allow-http \
    --network=default \
    --target-tags=http-server \
    --source-ranges=0.0.0.0/0 \
    --allow=tcp:80
```

### Criar regra de firewall para Health Check
```bash
gcloud compute firewall-rules create default-allow-health-check \
    --network=default \
    --target-tags=http-server \
    --source-ranges=130.211.0.0/22,35.191.0.0/16 \
    --allow=tcp
```

---

## Task 2. Configurar Instance Templates e Instance Groups

### Criar Instance Template para Região 1
```bash
gcloud compute instance-templates create us-east4-template \
    --project=qwiklabs-gcp-02-97279d553699 \
    --machine-type=e2-micro \
    --region=us-east4 \
    --network=default \
    --subnet=default \
    --tags=http-server \
    --metadata=startup-script-url=gs://spls/gsp215/gcpnet/httplb/startup.sh

# console generate
gcloud compute instance-templates create us-east4-template \
    --project=qwiklabs-gcp-02-97279d553699 \
    --machine-type=e2-micro \
    --network-interface=network-tier=PREMIUM,stack-type=IPV4_ONLY,subnet=default \
    --metadata=startup-script-url=gs://spls/gsp215/gcpnet/httplb/startup.sh,enable-oslogin=true \
    --maintenance-policy=MIGRATE \
    --provisioning-model=STANDARD \
    --service-account=283933249428-compute@developer.gserviceaccount.com \
    --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/trace.append \
    --region=us-east4 \
    --tags=http-server \
    --create-disk=auto-delete=yes,boot=yes,device-name=us-east4-template,image=projects/debian-cloud/global/images/debian-12-bookworm-v20260513,mode=rw,size=10,type=pd-balanced \
    --no-shielded-secure-boot \
    --shielded-vtpm \
    --shielded-integrity-monitoring \
    --reservation-affinity=any
```

### Criar Instance Template para Região 2 (espelhar o anterior, mudando a subrede)
```bash
gcloud compute instance-templates create us-west1-template \
    --machine-type=e2-micro \
    --region=us-west1 \
    --network=default \
    --subnet=default \
    --tags=http-server \
    --metadata=startup-script-url=gs://spls/gsp215/gcpnet/httplb/startup.sh

# console generate
gcloud compute instance-templates create us-west1-template \
    --project=qwiklabs-gcp-02-97279d553699 \
    --machine-type=e2-micro \
    --network-interface=network-tier=PREMIUM,stack-type=IPV4_ONLY,subnet=default \
    --metadata=startup-script-url=gs://spls/gsp215/gcpnet/httplb/startup.sh,enable-oslogin=true \
    --maintenance-policy=MIGRATE \
    --provisioning-model=STANDARD \
    --service-account=283933249428-compute@developer.gserviceaccount.com \
    --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/trace.append \
    --region=us-west1 \
    --tags=http-server \
    --create-disk=auto-delete=yes,boot=yes,device-name=us-west1-template,image=projects/debian-cloud/global/images/debian-12-bookworm-v20260513,mode=rw,size=10,type=pd-balanced \
    --no-shielded-secure-boot \
    --shielded-vtpm \
    --shielded-integrity-monitoring \
    --reservation-affinity=any
```

### Criar Managed Instance Group na Região 1
```bash
gcloud compute instance-groups managed create us-east4-mig \
    --project=qwiklabs-gcp-02-97279d553699 \
    --base-instance-name=us-east4-mig \
    --template=us-east4-template \
    --size=1 \
    --region=us-east4

gcloud compute instance-groups managed set-autoscaling us-east4-mig \
    --region=us-east4 \
    --min-num-replicas=1 \
    --max-num-replicas=2 \
    --target-cpu-utilization=0.8 \
    --cool-down-period=45

# console generate
gcloud beta compute instance-groups managed create us-east4-mig \
    --project=qwiklabs-gcp-02-97279d553699 \
    --base-instance-name=us-east4-mig \
    --template=projects/qwiklabs-gcp-02-97279d553699/global/instanceTemplates/us-east4-template \
    --size=1 \
    --zones=us-east4-c,us-east4-f,us-east4-b \
    --target-distribution-shape=BALANCED \
    --instance-redistribution-type=none \
    --default-action-on-vm-failure=repair \
    --action-on-vm-failed-health-check=default-action \
    --on-repair-allow-changing-zone=yes \
    --force-update-on-repair \
    --standby-policy-mode=manual \
    --list-managed-instances-results=pageless \
    --target-size-policy-mode=individual \

gcloud beta compute instance-groups managed set-autoscaling us-east4-mig \
    --project=qwiklabs-gcp-02-97279d553699 \
    --region=us-east4 \
    --mode=on \
    --min-num-replicas=1 \
    --max-num-replicas=2 \
    --target-cpu-utilization=0.8 \
    --cpu-utilization-predictive-method=none \
    --cool-down-period=45 \
    --stabilization-period=600
```

### Criar Managed Instance Group na Região 2
```bash
gcloud compute instance-groups managed create us-west1-mig \
    --base-instance-name=us-west1-mig \
    --template=us-west1-template \
    --size=1 \
    --region=us-west1

gcloud compute instance-groups managed set-autoscaling us-west1-mig \
    --region=us-west1 \
    --min-num-replicas=1 \
    --max-num-replicas=2 \
    --target-cpu-utilization=0.8 \
    --cool-down-period=45

# console generate
gcloud beta compute instance-groups managed create us-west1-mig \
    --project=qwiklabs-gcp-02-97279d553699 \
    --base-instance-name=us-west1-mig \
    --template=projects/qwiklabs-gcp-02-97279d553699/global/instanceTemplates/us-west1-template \
    --size=1 \
    --zones=us-west1-b,us-west1-c,us-west1-d \
    --target-distribution-shape=BALANCED \
    --instance-redistribution-type=none \
    --default-action-on-vm-failure=repair \
    --action-on-vm-failed-health-check=default-action \
    --on-repair-allow-changing-zone=yes \
    --force-update-on-repair \
    --standby-policy-mode=manual \
    --list-managed-instances-results=pageless \
    --target-size-policy-mode=individual \

gcloud beta compute instance-groups managed set-autoscaling us-west1-mig \
    --project=qwiklabs-gcp-02-97279d553699 \
    --region=us-west1 \
    --mode=on \
    --min-num-replicas=1 \
    --max-num-replicas=2 \
    --target-cpu-utilization=0.8 \
    --cpu-utilization-predictive-method=none \
    --cool-down-period=45 \
    --stabilization-period=600
```

---

## Task 3. Configurar o Application Load Balancer

### 1. Criar o health check
```bash
gcloud compute health-checks create tcp http-health-check \
    --port 80
```

### 2. Criar backend service
```bash
gcloud compute backend-services create http-backend \
    --protocol=TCP \
    --health-checks=http-health-check \
    --global
```

### 3. Adicionar instance groups ao backend service
```bash
# Região 1
gcloud compute backend-services add-backend http-backend \
    --instance-group=us-east4-mig \
    --instance-group-region=REGION_1 \
    --balancing-mode=RATE \
    --max-rate-per-instance=50 \
    --capacity-scaler=1 \
    --global

# Região 2
gcloud compute backend-services add-backend http-backend \
    --instance-group=region-2-mig \
    --instance-group-region=REGION_2 \
    --balancing-mode=UTILIZATION \
    --max-utilization=0.8 \
    --capacity-scaler=1 \
    --global
```

http-lb-ipv4: 8.232.43.66:80
http-lb-ipv6: [2600:1901:0:5a16::]:80

### 4. Criar endereço IP externo para o frontend
```bash
gcloud compute addresses create http-lb-ipv4 \
    --ip-version=IPV4 \
    --global

gcloud compute addresses create http-lb-ipv6 \
    --ip-version=IPV6 \
    --global
```

### 5. Criar o URL map
```bash
gcloud compute url-maps create http-lb-map \
    --default-service=http-backend \
    --global
```

### 6. Criar o target proxy
```bash
gcloud compute target-http-proxies create http-lb-proxy \
    --url-map=http-lb-map \
    --global
```

### 7. Criar as regras de encaminhamento (forwarding rules)
```bash
# IPv4
gcloud compute forwarding-rules create http-content-rule-ipv4 \
    --address=$(gcloud compute addresses describe http-lb-ipv4 --global --format='value(address)') \
    --global \
    --target-http-proxy=http-lb-proxy \
    --ports=80

# IPv6
gcloud compute forwarding-rules create http-content-rule-ipv6 \
    --address=$(gcloud compute addresses describe http-lb-ipv6 --global --format='value(address)') \
    --global \
    --target-http-proxy=http-lb-proxy \
    --ports=80
```

### 8. Ativar logging no backend service
```bash
gcloud compute backend-services update http-backend \
    --enable-logging \
    --logging-sample-rate=1 \
    --global
```

---

## Task 4. Testar o Application Load Balancer

### Acessar o Load Balancer
- Acesse via navegador: `http://[LB_IP_v4]` (substitua pelo IP do LB).
- Para IPv6: `http://[LB_IP_v6]`.

### Teste de carga com siege
```bash
gcloud compute instances create siege-vm \
    --machine-type=e2-micro \
    --zone=us-east1-c

gcloud compute ssh siege-vm --zone=us-east1-c
sudo apt-get -y install siege
export LB_IP=8.232.43.66:80
siege -c 150 -t120s http://$LB_IP
```

---

## Task 5. Denylist do siege-vm com Cloud Armor

### Criar política Cloud Armor
```bash
gcloud compute security-policies create denylist-siege --description="Denylist para siege-vm"

export SIEGE_IP_EXTERNAL=$(gcloud compute instances describe siege-vm \
    --zone=us-east1-c \
    --format='get(networkInterfaces[0].accessConfigs[0].natIP)')

gcloud compute security-policies rules create 1000 \
    --security-policy=denylist-siege \
    --expression="origin.ip == '${SIEGE_IP_EXTERNAL}'" \
    --action=deny-403
```

### Associar política ao backend service
```bash
gcloud compute backend-services update http-backend \
    --global \
    --security-policy=denylist-siege
```

### Testar bloqueio
No siege-vm:
```bash
curl http://$LB_IP
# Deve retornar 403 Forbidden
siege -c 150 -t120s http://$LB_IP
# Não deve gerar output (bloqueado)
```

---

## Observações
- Substitua `REGION_1`, `REGION_2`, `REGION_3`, `ZONE_3`, `[LB_IP_v4]`, `[LB_IP_v6]`, `[SIEGE_IP_EXTERNAL]` pelos valores reais do seu ambiente.
- Algumas etapas de configuração do Load Balancer são feitas via Console GCP.
- Para logs do Cloud Armor, acesse **Network Security > Cloud Armor Policies > denylist-siege > Logs**.
