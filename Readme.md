##### Никакого minikube и всё в одном - хотелось максимально по взрослому
____

![Alt text](img/shema.png?raw=true "Title")


И так, у нас 4 сервака - 2 master ноды, 2 worker ноды

2 CPU, 2 Gb RAM, 20 GB HDD

Голая Ubuntu server 20 с предустановленным SSH

Я пытался работать с предустановленными Docker и etcd, но дистрибутивы оказались старые, по этой причине я предпочёл делать всё с ноля.
____
Начнем

Выполняем на всех машинах

```Bash
sudo apt update
sudo apt install net-tools
sudo ufw disable
sudo service ufw stop
sudo swapoff -a
```

Комментируем настройки swap в файле
```Bash
sudo vim /etc/fstab
```

Устанавливаем Docker согласно документации

https://docs.docker.com/engine/install/ubuntu/
```Bash
sudo apt-get remove docker docker-engine docker.io containerd runc
sudo apt-get install apt-transport-https ca-certificates curl gnupg-agent software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

добавляем себя в группу docker
```Bash
sudo usermod -aG docker alex
sudo usermod -aG sudo alex - опционально (можно не делать)
logout - для применения настроек безопасности
docker info - проверяем
```


Устанавливаем docker compose (опционально) согласно документации

https://docs.docker.com/compose/install/
```Bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.27.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version - проверяем
```

Устанавливаем kubeadm kubelet kubectl согласно документации

https://phoenixnap.com/kb/install-kubernetes-on-ubuntu
```Bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
sudo apt-get install kubeadm kubelet kubectl
sudo apt-mark hold kubeadm kubelet kubectl
kubeadm version - проверяем
```
____
Теперь работаем только с мастер нодами

Устанавливаем etcd кластер на обе мастер машины. Идея была в том, чтобы собрать etcd отказоустойчивый кластер. По нормальному, для таких целей etcd выносят на отдельные машины, но в данном стучае наш кластер etcd будет работать на master нодах, что и k8s
```Bash
sudo apt -y install wget
export RELEASE="3.4.0" - последняя на данный момент версия
wget https://github.com/etcd-io/etcd/releases/download/v${RELEASE}/etcd-v${RELEASE}-linux-amd64.tar.gz
tar xvf etcd-v${RELEASE}-linux-amd64.tar.gz
cd etcd-v${RELEASE}-linux-amd64
sudo mv etcd etcdctl /usr/local/bin 
cd ..
rm -r etcd-v${RELEASE}-linux-amd64
etcd --version
sudo mkdir -p /var/lib/etcd/
sudo mkdir /etc/etcd
sudo groupadd --system etcd
sudo useradd -s /sbin/nologin --system -g etcd etcd
sudo chown -R etcd:etcd /var/lib/etcd/
```

редактируем настройки etcd
```Bash
sudo vim /etc/systemd/system/etcd.service
```
содержание файла настроек etcd на master1
```Bash
[Unit]
Description=etcd key-value store
Documentation=https://github.com/etcd-io/etcd
After=network.target

[Service]
User=etcd
Type=notify
ExecStart=/usr/local/bin/etcd --name master1 --data-dir /var/lib/etcd --listen-client-urls http://0.0.0.0:2379 --advertise-client-urls http://192.168.79.128:2379 --listen-peer-urls http://0.0.0.0:2380  --initial-cluster master1=http://192.168.79.128:2380,master2=http://192.168.79.129:2380 --initial-cluster-token ETCD_TOKEN --initial-cluster-state new
Restart=always
RestartSec=10s
LimitNOFILE=40000

[Install]
WantedBy=multi-user.targe
```

содержание файла настроек etcd на master2
```Bash
[Unit]
Description=etcd key-value store
Documentation=https://github.com/etcd-io/etcd
After=network.target

[Service]
User=etcd
Type=notify
ExecStart=/usr/local/bin/etcd --name master2 --data-dir /var/lib/etcd --listen-client-urls http://0.0.0.0:2379 --advertise-client-urls http://192.168.79.129:2379 --listen-peer-urls http://0.0.0.0:2380  --initial-cluster master1=http://192.168.79.128:2380,master2=http://192.168.79.129:2380 --initial-cluster-token ETCD_TOKEN --initial-cluster-state new
Restart=always
RestartSec=10s
LimitNOFILE=40000

[Install]
WantedBy=multi-user.targe
```

Далее перезагружаем и тестируем наш etcd кластер
```Bash
sudo systemctl  daemon-reload
sudo systemctl  start etcd.service - вот тут его надо запустить на обеих машинах как можно одновременно т.к. они будут пытаться подключиться друг к другу
sudo systemctl  status etcd.service
etcdctl member list - проверяем (мы должны увидеть обе mster1 & master2 машины в класстере)
```

Далее готовим наш k8s к первому запуску - создаём файлы настроек для запуска k8s
```Bash
sudo vim kubeadm-init.yaml
```

содержание файла настроек k8s для master1
```Bash
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: jmej22.ix28qrxj6ojt5bac
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.79.128
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: master1
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  certSANs:
  - 127.0.0.1
  - 192.168.79.128
  - 192.168.79.129
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: 192.168.79.128:6443
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  external:
    caFile: ""
    certFile: ""
    endpoints:
    - http://192.168.79.128:2379
    - http://192.168.79.129:2379
    keyFile: ""
imageRepository: k8s.gcr.io
kind: ClusterConfiguration
kubernetesVersion: v1.19.2
networking:
  dnsDomain: cluster.local
  podSubnet: 192.168.0.0/16
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```

содержание файла настроек k8s для master2
```Bash
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: jmej22.ix28qrxj6ojt5bac
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.79.129
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: master2
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  certSANs:
  - 127.0.0.1
  - 192.168.79.128
  - 192.168.79.129
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: 192.168.79.129:6443
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  external:
    caFile: ""
    certFile: ""
    endpoints:
    - http://192.168.79.128:2379
    - http://192.168.79.129:2379
    keyFile: ""
imageRepository: k8s.gcr.io
kind: ClusterConfiguration
kubernetesVersion: v1.19.2
networking:
  dnsDomain: cluster.local
  podSubnet: 192.168.0.0/16
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```

Работаем на master1 ноде
```Bash
sudo kubeadm init  --config=kubeadm-init.yaml
```
если всё ОК, то получим примерно такой вывод
```Bash
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join 192.168.79.128:6443 --token jmej22.ix28qrxj6ojt5bac \
    --discovery-token-ca-cert-hash sha256:893b61baf05c040b0fc717fc1ec1628e34c82399dc0d15b4db25fed27da6f37f \
    --control-plane

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.79.128:6443 --token jmej22.ix28qrxj6ojt5bac \
    --discovery-token-ca-cert-hash sha256:893b61baf05c040b0fc717fc1ec1628e34c82399dc0d15b4db25fed27da6f37f
```	
далее выполняем команды
```Bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
проверяем
```Bash
kubectl get pods -A
или
watch -n1 kubectl get pods -A
```

Далее для связи наших подов - надо установит CNI. Я буду использовать Flannel

Ставим согласно документации

https://github.com/coreos/flannel
```Bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
проверяем
```Bash
kubectl get pods -A
```

Переходим на master2, у нас уже есть файл настроек для запуска нашего k8s, указанный выше

Первое, что нам надо сделать - это скопировать сертификаты master1 на master2

Тут можно разрешить root через SSH, но я временно поменяю владельца папки с сертификатами на master2 ноде для возможности скопировать его с master1
```Bash
sudo chown -R alex:alex /etc/kubernetes/
```
Копирум сертификаты с master1
```Bash
scp -r /etc/kubernetes/pki alex@192.168.79.129:/etc/kubernetes/
```
Меняем владельца на master2 ноде обратно
```Bash
sudo chown -R root:root /etc/kubernetes/
```
Запускаем наш master2
```Bash
sudo kubeadm init  --config=kubeadm-init.yaml
```
Если всё ОК, то мы получим тотже вывод что и на master1 только IP будет разные
```Bash
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join 192.168.79.129:6443 --token jmej22.ix28qrxj6ojt5bac \
    --discovery-token-ca-cert-hash sha256:893b61baf05c040b0fc717fc1ec1628e34c82399dc0d15b4db25fed27da6f37f \
    --control-plane

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.79.129:6443 --token jmej22.ix28qrxj6ojt5bac \
    --discovery-token-ca-cert-hash sha256:893b61baf05c040b0fc717fc1ec1628e34c82399dc0d15b4db25fed27da6f37f
```
```Bash
kubectl get nodes - выведет наши 2 master - а
```
____
Переходим на наши worker ноды. Для того, чтобы worker ноды не были привязаны к определённому master ноде, заставим их работать через, в моём случае, HAPROXY.

Ставим на все worker ноды HAPROXY

http://cbonte.github.io/haproxy-dconv/2.2/intro.html - документация пипец какая страшная
```Bash
sudo add-apt-repository ppa:vbernat/haproxy-2.2
sudo apt update
sudo apt install -y haproxy
haproxy -v - проверяем
```
Далее, редактирум настройка нашего балансировщика
```Bash
sudo vim /etc/haproxy/haproxy.cfg
```
Содержание файла настроек на worker1
```Bash
global
	log /dev/log	local0
	log /dev/log	local1 notice
	chroot /var/lib/haproxy
	stats socket /run/haproxy/admin.sock mode 660 level admin
	stats timeout 30s
	user haproxy
	group haproxy
	daemon

	# Default SSL material locations
	ca-base /etc/ssl/certs
	crt-base /etc/ssl/private

	# Default ciphers to use on SSL-enabled listening sockets.
	# For more information, see ciphers(1SSL). This list is from:
	#  https://hynek.me/articles/hardening-your-web-servers-ssl-ciphers/
	# An alternative list with additional directives can be obtained from
	#  https://mozilla.github.io/server-side-tls/ssl-config-generator/?server=haproxy
	ssl-default-bind-ciphers ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:RSA+AESGCM:RSA+AES:!aNULL:!MD5:!DSS
	ssl-default-bind-options no-sslv3
defaults
	log	global
	mode	http
	option	httplog
	option	dontlognull
  timeout connect 5000
  timeout client  50000
  timeout server  50000
	errorfile 400 /etc/haproxy/errors/400.http
	errorfile 403 /etc/haproxy/errors/403.http
	errorfile 408 /etc/haproxy/errors/408.http
	errorfile 500 /etc/haproxy/errors/500.http
	errorfile 502 /etc/haproxy/errors/502.http
	errorfile 503 /etc/haproxy/errors/503.http
	errorfile 504 /etc/haproxy/errors/504.http

frontend k8s-api
	bind 192.168.79.132:5002
	bind 127.0.0.1:5002
	mode tcp
	option tcplog
	default_backend k8s-api

backend k8s-api
	mode tcp
	option tcplog
	option tcp-check
	balance roundrobin
	default-server port 6443 inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
        server master1 192.168.79.128:6443 check
        server master2 192.168.79.129:6443 check
```	
		
Содержание файла настроек на worker2
```Bash
global
	log /dev/log	local0
	log /dev/log	local1 notice
	chroot /var/lib/haproxy
	stats socket /run/haproxy/admin.sock mode 660 level admin
	stats timeout 30s
	user haproxy
	group haproxy
	daemon

	# Default SSL material locations
	ca-base /etc/ssl/certs
	crt-base /etc/ssl/private

	# Default ciphers to use on SSL-enabled listening sockets.
	# For more information, see ciphers(1SSL). This list is from:
	#  https://hynek.me/articles/hardening-your-web-servers-ssl-ciphers/
	# An alternative list with additional directives can be obtained from
	#  https://mozilla.github.io/server-side-tls/ssl-config-generator/?server=haproxy
	ssl-default-bind-ciphers ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:RSA+AESGCM:RSA+AES:!aNULL:!MD5:!DSS
	ssl-default-bind-options no-sslv3
defaults
	log	global
	mode	http
	option	httplog
	option	dontlognull
  timeout connect 5000
  timeout client  50000
  timeout server  50000
	errorfile 400 /etc/haproxy/errors/400.http
	errorfile 403 /etc/haproxy/errors/403.http
	errorfile 408 /etc/haproxy/errors/408.http
	errorfile 500 /etc/haproxy/errors/500.http
	errorfile 502 /etc/haproxy/errors/502.http
	errorfile 503 /etc/haproxy/errors/503.http
	errorfile 504 /etc/haproxy/errors/504.http

frontend k8s-api
	bind 192.168.79.133:5002
	bind 127.0.0.1:5002
	mode tcp
	option tcplog
	default_backend k8s-api

backend k8s-api
	mode tcp
	option tcplog
	option tcp-check
	balance roundrobin
	default-server port 6443 inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
        server master1 192.168.79.128:6443 check
        server master2 192.168.79.129:6443 check
```		
		
Перезапускаем наш HAPROXY
```Bash
sudo systemctl restart haproxy
```

Далее можно запускать наши workers. Вы можете попробовать подключиться на наш, только что созданный прокси, я так не делал, или подключиться к одному из наших master нод и потом переконфигурировать наш worker для работы с прокси.

Кто решиться попробовать через прокси - просто меняем адрес IP
```Bash
kubeadm join 127.0.0.1:5002 --token jmej22.ix28qrxj6ojt5bac \
    --discovery-token-ca-cert-hash sha256:893b61baf05c040b0fc717fc1ec1628e34c82399dc0d15b4db25fed27da6f37f
```
Для отчаянных
```Bash
kubeadm join 192.168.79.129:6443 --token jmej22.ix28qrxj6ojt5bac \
    --discovery-token-ca-cert-hash sha256:893b61baf05c040b0fc717fc1ec1628e34c82399dc0d15b4db25fed27da6f37f
```

Если все Ок - проверяем нашу ноду в кластере
```Bash
kubectl get nodes
```

Для тех кто не использовал прокси - теперь переконфигурируем наш worker для работы с локальной проксёй
```Bash
sudo vim  /etc/kubernetes/kubelet.conf
```
Меняем адрес сервера

server 127.0.0.1:5002

Перезапускаем и ждём пока появится наш worker1
```Bash
sudo systemctl restart kubelet && systemctl restart docker
watch -n1 kubectl get nodes
```


Повторяем всё на worker2 (настройка прокси, запуск прокси, инициализация и настройка k8s worker2). 
____
Далее, мы можем немного поиграться с нашим кластером - поочерёдно отключать наши master ноды.

Сначала на первом мастере
```Bash
sudo systemctl stop kubelet && systemctl stop docker
```
И посмотреть что наши workers не отвалились
```Bash
kubectl get nodes
```
Тем самым мы проверим работу наших HAPROXY на worker нодах.

Не забываем ключить и проверить
```Bash
sudo systemctl start kubelet && systemctl start docker
```
Процедуру можно повторить на master2 ноде
____
##### В продолжении - я попытаюсь, на данном кластере, развернуть мой простенький nginx сервер из данного git репозитория(папка src).

##### P.S. Я думал будет проще, что-то похожее на docker swarm, но из-за всей своей сложности и почитав https://rtfm.co.ua/kubernetes-znakomstvo-chast-1-arxitektura-i-osnovnye-komponenty-obzor/, https://kubernetes.io/ru/docs/home/, я только сейчас понимаю какие архитектуры можно строить в k8s.