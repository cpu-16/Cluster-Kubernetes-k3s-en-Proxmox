# Cluster-Kubernetes-k3s-en-Proxmox
# Cluster Kubernetes (k3s) en Proxmox con 1 master y 2 workers

Este documento describe paso a paso cómo monté un cluster **Kubernetes (k3s)** sobre **Proxmox**, con:

- 1 nodo **master** (`k8s-master`)
- 2 nodos **worker** (`k8s-worker1`, `k8s-worker2`)
- Primer despliegue: **Nginx** expuesto con **NodePort**

> Distribución usada en las VMs: Ubuntu Server / Debian (basado en `systemd` y `netplan`).

---

## 1. Topología del laboratorio

### 1.1. Infraestructura en Proxmox

- Hipervisor: **Proxmox VE**
- 3 Máquinas virtuales:
  - `k8s-master`
  - `k8s-worker1`
  - `k8s-worker2`
- Todas conectadas al mismo bridge de Proxmox (`vmbr0`), en la misma red `172.25.205.0/24`.

### 1.2. IPs y hostnames

| Rol     | Hostname     | IP              |
|--------|--------------|-----------------|
| Master | `k8s-master` | `172.25.205.116` |
| Worker | `k8s-worker1`| `172.25.205.117` |
| Worker | `k8s-worker2`| `172.25.205.118` |

> Ajustar las IPs según la red de tu laboratorio.

---

## 2. Requisitos mínimos de las VMs

Estos valores funcionaron bien para laboratorio con **k3s**:

### Master (`k8s-master`)

- vCPU: 2
- RAM: 2–4 GB
- Disco: 15–20 GB
- SO: Ubuntu Server / Debian
- Red: bridge a LAN (`vmbr0`)

### Workers (`k8s-worker1`, `k8s-worker2`)

- vCPU: 2
- RAM: 2 GB
- Disco: 10 GB
- SO: Ubuntu Server / Debian
- Red: bridge a LAN (`vmbr0`)

---

## 3. Configuración básica de cada nodo

Todos los pasos de esta sección se realizan **en cada VM** (master y workers), cambiando lo necesario en IPs y hostnames.

### 3.1. Crear usuario administrador

Ejemplo de usuario: `masteradmin` (puede ser cualquier nombre).

```bash
# Crear usuario
sudo adduser masteradmin

# Darle permisos de sudo
sudo usermod -aG sudo masteradmin
Iniciar sesión con ese usuario:

bash
Copiar código
su - masteradmin
3.2. Cambiar hostname
En cada VM:

bash
Copiar código
# En el master
sudo hostnamectl set-hostname k8s-master

# En el worker 1
sudo hostnamectl set-hostname k8s-worker1

# En el worker 2
sudo hostnamectl set-hostname k8s-worker2
Cerrar sesión y volver a entrar para ver el cambio en el prompt.

4. Configuración de red (IP estática + hosts)
4.1. IP estática con Netplan (ejemplo master)
Archivo: /etc/netplan/90-default.yaml

bash
Copiar código
sudo nano /etc/netplan/90-default.yaml
Contenido para el master (k8s-master):

yaml
Copiar código
network:
  version: 2
  renderer: networkd
  ethernets:
    ens18:
      dhcp4: no
      dhcp6: no
      addresses:
        - 172.25.205.116/24
      gateway4: 172.25.205.2
      nameservers:
        addresses: [1.1.1.1, 8.8.8.8]
Notas:

La interfaz puede ser ens18, enp0s18, etc. Ver con ip a.

gateway4 debe coincidir con la puerta de enlace de tu red (ip route | grep default).

Aplicar cambios:

bash
Copiar código
sudo netplan apply
ip a     # Verificar IP
Para los workers, se repite el proceso cambiando solo la IP:

k8s-worker1 → 172.25.205.117/24

k8s-worker2 → 172.25.205.118/24

4.2. Archivo /etc/hosts (en todos los nodos)
bash
Copiar código
sudo nano /etc/hosts
Contenido:

text
Copiar código
127.0.0.1   localhost
127.0.1.1   k8s-master   # En el master
# 127.0.1.1 k8s-worker1  # En worker1, adaptarlo
# 127.0.1.1 k8s-worker2  # En worker2, adaptarlo

172.25.205.116   k8s-master
172.25.205.117   k8s-worker1
172.25.205.118   k8s-worker2

::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
En cada nodo, ajustar la línea 127.0.1.1 al hostname local correspondiente.

5. Desactivar swap (requisito de Kubernetes)
En cada nodo:

bash
Copiar código
# Desactivar swap en la sesión actual
sudo swapoff -a

# Comentar la línea de swap en /etc/fstab
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Verificar
free -h   # La columna Swap debe estar en 0
6. Instalación de k3s
6.1. Instalar k3s en el master
En k8s-master:

bash
Copiar código
curl -sfL https://get.k3s.io | sh -
Verificar servicio:

bash
Copiar código
sudo systemctl status k3s
Ver nodos (de momento solo el master):

bash
Copiar código
sudo kubectl get nodes
6.2. Obtener el token del cluster
En el master:

bash
Copiar código
sudo cat /var/lib/rancher/k3s/server/node-token
Guardar ese token: se usará para unir los workers.
Ejemplo de formato (NO usar este literal):

text
Copiar código
K10a1b2c3d4e5f6g7h8i9j0k.lmnopqrstuvwxyz
6.3. Unir los workers al cluster
En cada worker (k8s-worker1 y k8s-worker2), ejecutar:

bash
Copiar código
curl -sfL https://get.k3s.io | \
  K3S_URL=https://172.25.205.116:6443 \
  K3S_TOKEN=K10a1b2c3d4e5f6g7h8i9j0k.lmnopqrstuvwxyz \
  sh -
Verificar que el agente está arriba:

bash
Copiar código
sudo systemctl status k3s-agent
6.4. Ver el cluster completo desde el master
En k8s-master:

bash
Copiar código
sudo kubectl get nodes
Salida esperada (similar a):

text
Copiar código
NAME          STATUS   ROLES                  AGE   VERSION
k8s-master    Ready    control-plane,master   XXm   v1.33.5+k3s1
k8s-worker1   Ready    <none>                 YYm   v1.33.5+k3s1
k8s-worker2   Ready    <none>                 ZZm   v1.33.5+k3s1
7. Configurar kubectl para el usuario normal
7.1. Copiar kubeconfig a ~/.kube/config
En el master, con el usuario masteradmin:

bash
Copiar código
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
7.2. Ajustar permisos del kubeconfig global (opcional pero útil)
bash
Copiar código
sudo chmod 644 /etc/rancher/k3s/k3s.yaml
7.3. Probar kubectl sin sudo
bash
Copiar código
kubectl get nodes
8. Etiquetar nodos (roles worker)
Por defecto, los workers aparecen sin rol.
Desde el master:

bash
Copiar código
kubectl label node k8s-worker1 node-role.kubernetes.io/worker=worker
kubectl label node k8s-worker2 node-role.kubernetes.io/worker=worker
Ver etiquetas:

bash
Copiar código
kubectl get nodes --show-labels
9. Primer namespace y primer Deployment
9.1. Crear namespace de pruebas
bash
Copiar código
kubectl create namespace demo-web
kubectl get namespaces
9.2. Crear Deployment de Nginx
bash
Copiar código
kubectl create deployment nginx-demo \
  --image=nginx \
  --replicas=3 \
  -n demo-web
Ver pods y en qué nodo corren:

bash
Copiar código
kubectl get pods -n demo-web -o wide
Salida ejemplo:

text
Copiar código
NAME                         READY   STATUS    RESTARTS   AGE   IP          NODE
nginx-demo-xxxxx-qqzxp       1/1     Running   0          2m    10.42.1.3   k8s-worker1
nginx-demo-xxxxx-jx8rj       1/1     Running   0          2m    10.42.2.3   k8s-worker2
nginx-demo-xxxxx-nfzwd       1/1     Running   0          2m    10.42.2.4   k8s-worker2
10. Exponer el Deployment con un Service NodePort
10.1. Crear el Service
bash
Copiar código
kubectl expose deployment nginx-demo \
  --type=NodePort \
  --port=80 \
  -n demo-web
Ver servicios:

bash
Copiar código
kubectl get svc -n demo-web
Ejemplo de salida:

text
Copiar código
NAME         TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx-demo   NodePort   10.43.29.57    <none>        80:31969/TCP   7m
80 → puerto interno del servicio dentro del cluster.

31969 → NodePort abierto en cada nodo.

10.2. Probar desde el navegador
Obtener IPs de los nodos:

bash
Copiar código
kubectl get nodes -o wide
Desde cualquier máquina en la misma red que las VMs (por ejemplo, el equipo físico):

text
Copiar código
http://172.25.205.117:31969   # k8s-worker1
http://172.25.205.118:31969   # k8s-worker2
Debería mostrarse la página por defecto de Nginx (“Welcome to nginx!”).

11. Operaciones básicas sobre el Deployment
11.1. Ver todos los recursos del namespace
bash
Copiar código
kubectl get all -n demo-web
11.2. Escalar el Deployment
Subir de 3 a 5 réplicas:

bash
Copiar código
kubectl scale deployment/nginx-demo --replicas=5 -n demo-web
kubectl get pods -n demo-web -o wide
Bajar a 2 réplicas:

bash
Copiar código
kubectl scale deployment/nginx-demo --replicas=2 -n demo-web
kubectl get pods -n demo-web -o wide
11.3. Eliminar recursos
Borrar solo el Service (NodePort):

bash
Copiar código
kubectl delete svc nginx-demo -n demo-web
Borrar el Deployment (y sus pods):

bash
Copiar código
kubectl delete deployment nginx-demo -n demo-web
Borrar todo el namespace de pruebas:

bash
Copiar código
kubectl delete namespace demo-web
