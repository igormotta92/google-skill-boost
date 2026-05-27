# Configurar Internal Load Balancer no GCP

## Passo 1: Configurar Ambiente Virtual Python

```bash
sudo apt-get install -y virtualenv
```

**Saída no terminal:**
```
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
virtualenv is already the newest version (20.25.0+ds-2).
0 upgraded, 0 newly installed, 0 removed. 2 not upgraded.
```

```bash
python3 -m venv venv
```

```bash
source venv/bin/activate
```

**Saída no terminal:**
```
(venv) student_01_ecbc8bad5405@cloudshell:~ (qwiklabs-gcp-02-e5b8a713b668)$
```

## Passo 2: backend\.sh

```bash
touch ~/backend.sh
```

### backend\.sh
Instala e configura o servidor Python que testa se números são primos:

```bash
sudo chmod -R 777 /usr/local/sbin/
sudo cat << EOF > /usr/local/sbin/serveprimes.py
import http.server

def is_prime(a): return a!=1 and all(a % i for i in range(2,int(a**0.5)+1))

class myHandler(http.server.BaseHTTPRequestHandler):
  def do_GET(s):
    s.send_response(200)
    s.send_header("Content-type", "text/plain")
    s.end_headers()
    s.wfile.write(bytes(str(is_prime(int(s.path[1:]))).encode('utf-8')))

http.server.HTTPServer(("",80),myHandler).serve_forever()
EOF
nohup python3 /usr/local/sbin/serveprimes.py >/dev/null 2>&1 &
```

## Passo 3: Criar Instance Template para Backend

```bash
gcloud compute instance-templates create primecalc \
--metadata-from-file startup-script=backend.sh \
--no-address --tags backend --machine-type=e2-medium
```

**Saída no terminal:**
```
Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-02-e5b8a713b668/global/instanceTemplates/primecalc].
NAME: primecalc
MACHINE_TYPE: e2-medium
PREEMPTIBLE:
CREATION_TIMESTAMP: 2026-05-06T15:01:22.555-07:00
```

## Passo 4: Criar Firewall Rule para Backend

```bash
gcloud compute firewall-rules create http --network default --allow=tcp:80 \
--source-ranges 10.138.0.0/20 --target-tags backend
```

**Saída no terminal:**
```
Creating firewall...working..Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-02-e5b8a713b668/global/firewalls/http].
Creating firewall...done.
NAME: http
NETWORK: default
DIRECTION: INGRESS
PRIORITY: 1000
ALLOW: tcp:80
DENY:
DISABLED: False
```

## Passo 5: Criar Managed Instance Group

```bash
gcloud compute instance-groups managed create backend \
--size 3 \
--template primecalc \
--zone us-west1-b
```

**Saída no terminal:**
```
Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-02-e5b8a713b668/zones/us-west1-b/instanceGroupManagers/backend].
NAME: backend
LOCATION: us-west1-b
SCOPE: zone
BASE_INSTANCE_NAME: backend
SIZE: 0
TARGET_SIZE: 3
INSTANCE_TEMPLATE: primecalc
AUTOSCALED: no
```

## Passo 6: Criar Health Check HTTP

```bash
gcloud compute health-checks create http ilb-health --request-path /2
```

**Saída no terminal:**
```
Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-02-e5b8a713b668/global/healthChecks/ilb-health].
NAME: ilb-health
PROTOCOL: HTTP
```

## Passo 7: Criar Backend Service para Internal Load Balancer

```bash
gcloud compute backend-services create prime-service \
--load-balancing-scheme internal --region=us-west1 \
--protocol tcp --health-checks ilb-health
```

**Saída no terminal:**
```
Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-02-e5b8a713b668/regions/us-west1/backendServices/prime-service].
NAME: prime-service
BACKENDS:
PROTOCOL: TCP
```

## Passo 8: Adicionar Backend ao Backend Service

```bash
gcloud compute backend-services add-backend prime-service \
--instance-group backend --instance-group-zone=us-west1-b \
--region=us-west1
```

**Saída no terminal:**
```
Updated [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-02-e5b8a713b668/regions/us-west1/backendServices/prime-service].
```

## Passo 9: Criar Forwarding Rule para Internal Load Balancer

```bash
gcloud compute forwarding-rules create prime-lb \
--load-balancing-scheme internal \
--ports 80 --network default \
--region=us-west1 --address 10.138.0.10 \
--backend-service prime-service
```

**Saída no terminal:**
```
Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-02-e5b8a713b668/regions/us-west1/forwardingRules/prime-lb].
```

## Passo 10: Criar Instância de Teste

```bash
gcloud compute instances create testinstance \
--machine-type=e2-standard-2 --zone us-west1-b
```

**Saída no terminal:**
```
Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-02-e5b8a713b668/zones/us-west1-b/instances/testinstance].
NAME: testinstance
ZONE: us-west1-b
MACHINE_TYPE: e2-standard-2
PREEMPTIBLE:
INTERNAL_IP: 10.138.0.5
EXTERNAL_IP: 8.231.192.198
STATUS: RUNNING
```

## Passo 11: Conectar via SSH e Testar Load Balancer

```bash
gcloud compute ssh testinstance --zone us-west1-b
```

**Saída no terminal:**
```
WARNING: The private SSH key file for gcloud does not exist.
WARNING: The public SSH key file for gcloud does not exist.
WARNING: You do not have an SSH key for gcloud.
WARNING: SSH keygen will be executed to generate a key.
This tool needs to create the directory [/home/student_01_ecbc8bad5405/.ssh] before being able to generate SSH keys.

Do you want to continue (Y/n)?  Y
```

## Passo 12: Testar Conectividade com Load Balancer - Requisições de Teste

```bash
curl 10.138.0.10/2
```

**Saída no terminal:**
```
Truestudent-01-ecbc8bad5405@testinstance:~$
```

```bash
curl 10.138.0.10/4
```

**Saída no terminal:**
```
Falsestudent-01-ecbc8bad5405@testinstance:~$
```

```bash
curl 10.138.0.10/5
```

**Saída no terminal:**
```
Truestudent-01-ecbc8bad5405@testinstance:~$
```

## Passo 13: Desconectar da Instância de Teste

```bash
exit
```

**Saída no terminal:**
```
logout
Connection to 8.231.192.198 closed.
```

## Passo 14: Deletar Instância de Teste

```bash
gcloud compute instances delete testinstance --zone=us-west1-b
```

**Saída no terminal:**
```
The following instances will be deleted. Any attached disks configured to be auto-deleted will be deleted unless they are attached to any other instances or the `--keep-disks` flag is given and specifies them for keeping.
Deleting a disk is irreversible and any data on the disk will be lost.
 - [testinstance] in [us-west1-b]

Do you want to continue (Y/n)?  Y

Deleted [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-02-e5b8a713b668/zones/us-west1-b/instances/testinstance].
```

## Passo 15: Criar Frontend Script

```bash
touch ~/frontend.sh
```

```bash
sudo chmod -R 777 /usr/local/sbin/
sudo cat << EOF > /usr/local/sbin/getprimes.py
import urllib.request
from multiprocessing.dummy import Pool as ThreadPool
import http.server
PREFIX="http://10.138.0.10/" #HTTP Load Balancer
def get_url(number):
    return urllib.request.urlopen(PREFIX+str(number)).read().decode('utf-8')
class myHandler(http.server.BaseHTTPRequestHandler):
  def do_GET(s):
    s.send_response(200)
    s.send_header("Content-type", "text/html")
    s.end_headers()
    i = int(s.path[1:]) if (len(s.path)>1) else 1
    s.wfile.write("<html><body><table>".encode('utf-8'))
    pool = ThreadPool(10)
    results = pool.map(get_url,range(i,i+100))
    for x in range(0,100):
      if not (x % 10): s.wfile.write("<tr>".encode('utf-8'))
      if results[x]=="True":
        s.wfile.write("<td bgcolor='#00ff00'>".encode('utf-8'))
      else:
        s.wfile.write("<td bgcolor='#ff0000'>".encode('utf-8'))
      s.wfile.write(str(x+i).encode('utf-8')+"</td> ".encode('utf-8'))
      if not ((x+1) % 10): s.wfile.write("</tr>".encode('utf-8'))
    s.wfile.write("</table></body></html>".encode('utf-8'))
http.server.HTTPServer(("",80),myHandler).serve_forever()
EOF
nohup python3 /usr/local/sbin/getprimes.py >/dev/null 2>&1 &
```

## Passo 16: Criar Instância Frontend

```bash
gcloud compute instances create frontend --zone=us-west1-b \
--metadata-from-file startup-script=frontend.sh \
--tags frontend --machine-type=e2-standard-2
```

**Saída no terminal:**
```
Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-02-e5b8a713b668/zones/us-west1-b/instances/frontend].
NAME: frontend
ZONE: us-west1-b
MACHINE_TYPE: e2-standard-2
PREEMPTIBLE:
INTERNAL_IP: 10.138.0.6
EXTERNAL_IP: 8.231.192.198
STATUS: RUNNING
```

## Passo 17: Criar Firewall Rule para Frontend

```bash
gcloud compute firewall-rules create http2 --network default --allow=tcp:80 \
--source-ranges 0.0.0.0/0 --target-tags frontend
```

**Saída no terminal:**
```
Creating firewall...working..Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-02-e5b8a713b668/global/firewalls/http2].
Creating firewall...done.
NAME: http2
NETWORK: default
DIRECTION: INGRESS
PRIORITY: 1000
ALLOW: tcp:80
DENY:
DISABLED: False
```
