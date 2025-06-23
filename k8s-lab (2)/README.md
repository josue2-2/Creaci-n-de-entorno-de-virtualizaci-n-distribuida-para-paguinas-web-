# Proyecto de Laboratorio Kubernetes On-Premise

Este repositorio documenta paso a paso la creación de un clúster Kubernetes en entorno de laboratorio on-premise utilizando Ubuntu Server 22.04, VMware y Containerd.

## Paso 1: Habilitar VMs

Instalar Ubuntu Server 22-04 en VMware en distintas máquinas virtuales

Las máquinas se llamaran:
1. k8s-master
2. k8s-worker-1
3. k8s-worker-2

Durante la creación de las máquinas:
* Escoger red NAT para las 3 máquinas
* Asignar 2gb de ram y 2 procesadores para el nodo master
* Asignar 1gb de ram y 1 procesador para los nodos workers

Durante la instalación del sistema operativo:
* Escoger la habilitación de OpenSSH
* No instalar ningun páquete adicional durante el proceso (microkubes, docker, phrometeus, etc)
* asignar usuario  "ubuntu" y password "ubuntu"
* asignar hostname de acuerdo al nombre de la máquina

## Paso 2: Configuración de las VMs

Desactivar cloud-init (servicio que sobreescribe la configuración de red)

```bash
sudo touch /etc/cloud/cloud-init.disabled
sudo apt purge cloud-init -y
sudo rm -rf /etc/cloud/ /var/lib/cloud/
sudo update-initramfs -u
```

```bash
sudo reboot
```

Ahora tendremos que asignar IPs fijas

Primero verificamos la IP del gateway nat con:

```bash
ip r
```

Dentro del archivo `/etc/netplan/50.....yml

Agregar las siguientes lineas despues de poner dhcp4 en false:

```bash
sudo nano /etc/netplan/50.....yml
```

```bash
dhcp4: false
addresses: [192.168.38.10/24] #Aqui asignamos las ips para cada máquina, en este caso 10,11,12
gateway4: 192.168.38.2 # Esta es la ip del gateway Nat obtenido 
nameservers:
	addresses: [8.8.8.8, 1.1.1.1]
```

```bash
sudo netplan apply
```

## Paso 3: Preparar la instalación de Kubernetes y Containerd

Deshabilitar Swap (necesario para correcto funcionamiento de Kubernetes)

```bash
sudo swapoff -a
sudo nano /etc/fstab # Comentar linea con swap img
sudo reboot
```

Eliminar rastros de docker (necesario en ubuntu server)

```bash
sudo systemctl stop docker docker.socket
sudo apt-get purge -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo rm -rf /var/lib/docker
sudo rm -rf /etc/docker
sudo rm -f /var/run/docker.sock
```

```bash
sudo apt-get autoremove -y
sudo apt-get autoclean
```

##  Paso 4: Instalar Kubernetes y Containerd

Importar los repositorios de k8s.io:

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Importar las claves gpg: 

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Actualizar e instalar

```bash
sudo apt update
sudo apt install -y kubelet kubeadm kubectl containerd
```

## Paso 5: Configurar containerd

Habilitar el servicio

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
```

Editar SystemdCgroup:

```bash
sudo nano /etc/containerd/config.toml
```

```bash
# Buscar la linea 
SystemdCgroup = false  # cambia a:
SystemdCgroup = true
```

FInalmente reiniciar

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

## Paso 6: Configurar Kubernetes

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

Cargar modulo para netfilter:

```bash
sudo modprobe br_netfilter
echo 'br_netfilter' | sudo tee /etc/modules-load.d/k8s.conf

```

```bash
sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```

```bash
sudo sysctl --system
```

### En el nodo master

Iniciar el cluster:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

Una vez termine verificar el archivo de configuración:

```bash
ls $HOME/.kube/config
```

Si no esta:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Instalar el plugin de red Flannel (intentar varias veces si es necesario):

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

Comprobar despues de 1 minuto:

```bash
kubectl get pods -n kube-system
```

```bash
kubectl get pods -n kube-flannel
```

```bash
kubectl get nodes
```
### En los nodos workers

Copiar esta salida en el nodo amster

```bash
kubeadm token create --print-join-command
```

y pegarla en el nodo worker

```bash
kubeadm join 192.168.38.10:6443 --token 61lip9.oj9h8d31vh2bnwif \
	--discovery-token-ca-cert-hash sha256:32e025cc0b4ee23cc6ac211874f1ddcfcf47d8eb2d9ad28568fd1cef5f8310f4 
```


## Verificar el levantamiento del entorno

En el nodo master

```Bash
kubectl get pods -n kube-system -o wide
```

Deberiamos ver los nodos workers

## Desplegar una aplicacion 

Dentro del nodo de prueba creamos un archivo .yaml para el deployment

```
nano hello.yaml
```

pegamos

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
      - name: hello-container
        image: nginxdemos/hello
        ports:
        - containerPort: 80

---

apiVersion: v1
kind: Service
metadata:
  name: hello-service
spec:
  type: NodePort
  selector:
    app: hello
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30081  # Puedes cambiarlo si ya está en uso

```

Este archivo de deployment y servicio hace:
* Crea 2 replicas del contenedor nginxdemos/hello dentro de los nodos workers
* Asigna el puerto 30081 a cada una de las IPs de los nodos workers para acceder a la aplicación

Desplegamos la app:

```bash
kubectl apply -f hello.yaml
```

accedemos en nuestro nacegador desde un cliente:

```
http://192.168.38.11:30081
```

## En caso de error

resetiar todo

```
sudo kubeadm reset -f
sudo systemctl stop kubelet
sudo rm -rf /etc/cni/net.d /var/lib/cni /var/lib/kubelet /etc/kubernetes
sudo systemctl start kubelet
```

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```




Reinicar el cluster, Paso 7: sección en el nodo master 

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

## Solucion nodos worker

Crea o edita el archivo de configuración para `crictl`:

bash
```
sudo nano /etc/crictl.yaml
```

Y dentro, pega esto:
```
runtime-endpoint: unix:///run/containerd/containerd.sock 
image-endpoint: unix:///run/containerd/containerd.sock
```

```
HABILITAR KUBELET
```

## HABILITAR KUBELET
## Reiniciar de manera limpia

### Cluster

Reset kubernetes

```
sudo kubeadm reset -f
```

Limpieza de archivos residuales

```
sudo rm -rf /etc/cni /var/lib/cni /var/lib/kubelet /etc/kubernetes
sudo rm -rf $HOME/.kube
```

Reiniciar servicios

```
sudo systemctl restart containerd
sudo systemctl restart kubelet
```

Reiniciar cluster

```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

Acceso a kubectl

```
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Plugin de red

```
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

Obtener join

```
kubeadm token create --print-join-command
```

## Workers

Resetiar kubernetes

```
sudo kubeadm reset -f
```

Eliminar residuales

```
sudo rm -rf /etc/kubernetes /var/lib/kubelet /etc/cni /var/lib/cni
sudo rm -rf $HOME/.kube
```

Reiniciar servicios

```
sudo systemctl restart containerd
sudo systemctl restart kubelet
```

Volver a unir el cluster

```
kubeadm token create --print-join-command
```

```
sudo kubeadm join 192.168.254.10:6443 --token pd4318.rexoclwsu12higm9 \
--discovery-token-ca-cert-hash sha256:61f311995669b7b11f4bf2c1e41d403e58ea2badc1ddddb139f802356fa76039
```
## Comprobaciones

Ver pods

```
kubectl get pods
```

borrar un pod

```bash
kubectl delete pod <nombre-del-pod>
```

## Borrar un despliegue

Por .yaml

```
kubectl delete -f archivo.yaml
```

Por nombre

```
kubectl delete deployment nombre-del-deployment
kubectl delete service nombre-del-service
```

## Escalar un deployment

```
kubectl scale deployment hello-deploy --replicas=6
```

```
## Links:

* 

202506111430

