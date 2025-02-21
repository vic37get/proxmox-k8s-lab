# proxmox-k8s-lab

## Comandos de criação da VM Template:

- Primeiro comando:
```bash
apt install libguestfs-tools -y
```

- Comando para personalizar a imagem:
```bash
virt-customize --add jammy-server-cloudimg-amd64.img \
 --install qemu-guest-agent
```


- Comando para criar a máquina virtual:
```bash
qm create <NUMERO DA MÁQUINA> \
 --name <NOME DA MÁQUINA> \
 --numa 0 \
 --ostype l26 \
 --cpu cputype=host \
 --cores 3 \
 --sockets 2 \
 --memory 4096 \
 --net0 virtio,bridge=vmbr1,tag=11
```

- Comando para importar o disco:
```bash
qm importdisk 150 /tmp/jammy-server-cloudimg-amd64.img VOL-01
```

- Setando o disco:
```bash
qm set 150 --scsihw virtio-scsi-pci --scsi0 VOL-01:vm-150-disk-0
```

- Habilitando o Cloudinit:
```bash
qm set 150 --ide2 local-lvm:cloudinit
```

- Colocando o scsi0 como disco de boot:
```bash
qm set 150 --boot c --bootdisk scsi0
```

- Ajustando o monitor:
```bash
qm set 150 --serial0 socket --vga serial0
```

- Habilitar o agent:
```bash
qm set 150 --agent enabled=1
```

- Redimensionando o disco:
```bash
qm disk resize 150 scsi0 +30G
```

- Adicionando dependencias a serem instaladas:
```bash
qm set 150 --cicustom "vendor=local:snippets/vendor.yaml"
```

- Template:
```bash
qm template 150
```

## Criação do Cluster HA

- Habilitar o modo HA na máquina master e criar o primeiro node:
```bash
microk8s enable ha-cluster
microk8s add-node
```
Ele vai devolver algo parecido como 
```bash
<IP-MESTRE>:25000/<TOKEN>
```
- Após isso, conectar as outras máquinas ao cluster dessa forma:
```bash
microk8s join <IP-MESTRE>:25000/<TOKEN>
```

- Visualizando se as VMs estão no cluster, na primeira VM execute:
```bash
microk8s kubectl get nodes
```
- Vai aparecer algo assim:
```bash
NAME      STATUS   ROLES    AGE   VERSION
vm1       Ready    <none>   10m   v1.27.0
vm2       Ready    <none>   5m    v1.27.0
vm3       Ready    <none>   5m    v1.27.0
```

# 🚀 Implantando uma API FastAPI em um Cluster HA MicroK8s

Este repositório fornece instruções para criar e implantar uma API FastAPI em um **Cluster Kubernetes HA** usando **MicroK8s**. O objetivo é garantir alta disponibilidade e escalabilidade.

---

## **1️⃣ Criar a API FastAPI**

1. **Crie um diretório para o projeto e entre nele:**
   ```bash
   mkdir fastapi-microk8s && cd fastapi-microk8s
   ```
2. **Crie um ambiente virtual e ative-o:** (Opcional, mas recomendado)
   ```bash
   python3 -m venv venv
   source venv/bin/activate
   ```
3. **Instale o FastAPI e o Uvicorn:**
   ```bash
   pip install fastapi uvicorn
   ```
4. **Crie o arquivo `app.py` com o seguinte código:**
   ```python
   from fastapi import FastAPI

   app = FastAPI()

   @app.get("/")
   def read_root():
       return {"message": "API FastAPI rodando no Kubernetes com MicroK8s!"}

   @app.get("/health")
   def health_check():
       return {"status": "ok"}
   ```
5. **Teste localmente:**
   ```bash
   uvicorn app:app --host 0.0.0.0 --port 8000
   ```
   Acesse: [http://localhost:8000](http://localhost:8000)

---

## **2️⃣ Criar a Imagem Docker**

1. **Crie o arquivo `Dockerfile`:**
   ```dockerfile
   FROM python:3.10

   WORKDIR /app

   COPY requirements.txt .

   RUN pip install --no-cache-dir -r requirements.txt

   COPY . .

   CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
   ```
2. **Crie o `requirements.txt`:**
   ```
   fastapi
   uvicorn
   ```
3. **Construa a imagem Docker:**
   ```bash
   docker build -t fastapi-microk8s .
   ```
4. **Teste a imagem localmente:**
   ```bash
   docker run -p 8000:8000 fastapi-microk8s
   ```

---

## **3️⃣ Implantar no Cluster HA MicroK8s**

### **3.1. Enviar a imagem para o MicroK8s**

1. **Ativar o registry do MicroK8s:**
   ```bash
   microk8s enable registry
   ```
2. **Taguear a imagem para o registry local:**
   ```bash
   docker tag fastapi-microk8s localhost:32000/fastapi-microk8s:v1
   ```
3. **Enviar a imagem para o registry:**
   ```bash
   docker push localhost:32000/fastapi-microk8s:v1
   ```

---

### **3.2. Criar os Manifests Kubernetes**

1. **Crie o arquivo `deployment.yaml`:**
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: fastapi-app
   spec:
     replicas: 3
     selector:
       matchLabels:
         app: fastapi
     template:
       metadata:
         labels:
           app: fastapi
       spec:
         containers:
           - name: fastapi
             image: localhost:32000/fastapi-microk8s:v1
             ports:
               - containerPort: 8000
   ```
2. **Crie o arquivo `service.yaml`:**
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: fastapi-service
   spec:
     selector:
       app: fastapi
     ports:
       - protocol: TCP
         port: 80
         targetPort: 8000
     type: LoadBalancer
   ```
3. **Implante no Kubernetes:**
   ```bash
   microk8s kubectl apply -f deployment.yaml
   microk8s kubectl apply -f service.yaml
   ```
4. **Verifique os pods:**
   ```bash
   microk8s kubectl get pods
   ```
5. **Obtenha o IP do serviço:**
   ```bash
   microk8s kubectl get service fastapi-service
   ```

---

## **4️⃣ Expondo a API Publicamente**

### **4.1. Usando MetalLB (Recomendado)**

1. **Ativar MetalLB:**
   ```bash
   microk8s enable metallb
   ```
2. **Fornecer um intervalo de IPs (exemplo):**
   ```
   192.168.1.200-192.168.1.220
   ```
3. **Verifique se o Service recebeu um IP externo:**
   ```bash
   microk8s kubectl get service fastapi-service
   ```
4. **Acesse no navegador:**
   ```
   http://192.168.1.201
   ```

### **4.2. Usando Ingress (Para Domínios Customizados)**

1. **Ativar Ingress no MicroK8s:**
   ```bash
   microk8s enable ingress
   ```
2. **Criar `ingress.yaml`:**
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: fastapi-ingress
   spec:
     rules:
       - host: fastapi.local
         http:
           paths:
             - path: /
               pathType: Prefix
               backend:
                 service:
                   name: fastapi-service
                   port:
                     number: 80
   ```
3. **Aplicar o Ingress:**
   ```bash
   microk8s kubectl apply -f ingress.yaml
   ```
4. **Adicionar ao `/etc/hosts` (Linux/Mac) ou `C:\Windows\System32\drivers\etc\hosts` (Windows):**
   ```
   192.168.1.201 fastapi.local
   ```
5. **Acesse no navegador:**
   ```
   http://fastapi.local
   ```

---

## **✅ Conclusão**

Agora sua API FastAPI está rodando em um **cluster Kubernetes HA** no **MicroK8s** com alta disponibilidade! 🎯

✅ **Alta Disponibilidade** (3 réplicas).  
✅ **Escalabilidade automática**.  
✅ **LoadBalancer via MetalLB**.  
✅ **Acesso via domínio usando Ingress**.  

Se precisar de ajustes, melhorias ou adicionar monitoramento (Prometheus/Grafana), é só avisar! 🚀
