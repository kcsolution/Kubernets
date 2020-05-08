## Instalando e configurando um cluster Kubernetes no CentOS 7

### Configurações iniciais

Estas configurações deverão ser realizadas em todas os hosts que comporão o cluster, ou seja, tanto o nó master como os escravos.

#### Para iniciar, tome nota do hostname de cada máquina do cluster. Caso não saiba os nomes das máquinas você pode utilizar o seguinte comando para descobrir:

      $ hostname
  
#### Essas informações serão importante em toda configuração. No meu caso, tenho os seguintes hostnames:

      Master: kubernetes-master
      Node1: kubernetes-node-1
      Node2: kubernetes-node-2

    
#### Em cada um dos nós, edite o arquivo /etc/hosts e configure os ips de cada uns dos hosts.

      $ sudo vi /etc/hosts
      
      *** Como ficará o hosts
      
            [root@kubernetes-master kleber]# cat /etc/hosts
            127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
            ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
            192.168.207.200  kubernetes-master
            192.168.207.201  kubernetes-node-1
            192.168.207.202  kubernetes-node-2
      
      
#### Desabilite o SELinux do CentOS.

      $ sudo setenforce 0
      $ sudo sed -i follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
      
#### Execute o comando a seguir para corrigir possíveis falhas de roteamento.

      $ sudo bash -c 'cat <<EOF > /etc/sysctl.conf
      net.bridge.bridge-nf-call-ip6tables = 1
      net.bridge.bridge-nf-call-iptables = 1
      EOF'
  
  
#### Execute o comando abaixo para aplicar as configurações.

      $ sudo sysctl -p system

#### Desabilite o SWAP comentando (coloque # no início) a linha que começa com /dev/mapper/centos-swap do arquivo /etc/fstab.

      $ sudo vi /etc/fstab
      
      
#### Adicione o repositório do Kubernetes no gerenciador de pacotes Yum.

      $ sudo bash -c 'cat <<EOF > /etc/yum.repos.d/kubernetes.repo
      [kubernetes]
      name=Kubernetes
      baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
      enabled=1
      gpgcheck=1
      repo_gpgcheck=1
      gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
      EOF'
    
    
 #### Reinicie os hosts para aplicar as alterações, executando o seguinte comando:

     $ sudo init 6
     
 ## Instalando o Docker

      O Docker é um pré-requisito para a instalação e funcionamento do Kubernetes. Sendo assim, instale o Docker em cada um dos hosts.

#### Para iniciar a instalação do Docker, primeiramente instale o pacote yum-utils executando o seguinte comando:

      $ sudo yum install -y yum-utils \
        device-mapper-persistent-data \
        lvm2

      $ yum install pigz

#### Em seguida, adicione o repositório do Docker-CE de versões estáveis.

      $ sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
          
#### Execute o seguinte comando para instalar o docker:

      $ sudo yum install -y docker-ce

#### Habilite e inicie o Docker.

      $ sudo systemctl enable docker && sudo systemctl start docker
      
#### Adicione o seu usuário no grupo do Docker para garantir que possa usar os comandos Docker sem necessidade de sudo:

      $ sudo groupadd docker
      $ sudo usermod -aG docker kleber
      **Adicione o usuário que desejar, no meu caso é kleber.
      
      **Para que as configurações possam ter efeito, é necessário reiniciar a sessão.

#### Após logar novamente, cheque a versão a instalada com o seguinte comando:

      $ docker #### version

      **No meu caso, a versão instalada foi a Docker version 18.09.6, build 481bc77156

#### Para testar o pleno funcionamento do Docker execute o seguinte comando:

      $ docker run hello-world

     **Se tudo ocorrer bem, a imagem do hello-world deverá ser baixada e executada no host. Deverá mostrar resultado abaixo.
           [root@kubernetes-master kleber]# docker run hello-world

            Hello from Docker!
            This message shows that your installation appears to be working correctly.

            To generate this message, Docker took the following steps:
             1. The Docker client contacted the Docker daemon.
             2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
                (amd64)
             3. The Docker daemon created a new container from that image which runs the
                executable that produces the output you are currently reading.
             4. The Docker daemon streamed that output to the Docker client, which sent it
                to your terminal.

            To try something more ambitious, you can run an Ubuntu container with:
             $ docker run -it ubuntu bash

            Share images, automate workflows, and more with a free Docker ID:
             https://hub.docker.com/

            For more examples and ideas, visit:
             https://docs.docker.com/get-started/

Instalando o Kubernetes

#### UFA! Agora vamos de fato iniciar a instalação do Kubernetes. Este procedimento deve ainda ser executado em cada um dos nós do cluster. Vamos lá? Para começar, então, execute o seguinte comando:

            $ sudo yum install -y kubelet kubeadm kubectl

Agora este próximo passo, caso não seja feito, costuma causar graves problemas na instalação. Então muita atenção para ele!

#### Precisamos configurar o Cgroup Driver do Kubernetes para ser exatamente igual ao do Docker. Para checar o CGroup Driver do Docker execute o seguinte comando:

            $ docker info | grep -i cgroup
            
            No meu caso o Cgroup Driver mostrado foi o cgroupfs, conforme mostra a figura a seguir:
            
            [root@kubernetes-master kleber]# docker info | grep -i cgroup
            Cgroup Driver: systemd

#### Agora precisamos editar o arquivo 10-kubeadm.conf para indicar o nome do Cgroup Driver. Para isso execute o seguinte comando, tendo cuidado para alterar, caso necessário, o texto cgroupfs pelo Cgroup Driver apresentado pelo Docker:

             $ sudo sed -i "s/cgroup-driver=systemd/cgroup-driver=cgroupfs/g" /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf

              PS: Nas versões mais antigas do kuberntes, o arquivo fica localizado no seguinte endereço: /etc/systemd/system/kubelet.service.d/10-kubeadm.conf     
              
              
 #### Recarregue as configurações do kubelet:

              $ sudo systemctl daemon-reload

#### Habilite e inicie o serviço do kubelet.

              $ sudo systemctl enable kubelet && sudo systemctl start kubelet
              
  Configurando o master

Agora vamos realizar a configuração do nó master do Kubernetes. Este é a única seção onde o procedimento deve ser executado apenas no nó master.

#### Para iniciar, vamos precisar desabilitar o serviço do firewall do nó master:

                $ sudo systemctl disable firewalld
                $ sudo systemctl stop firewalld


Inicie o Master com o seguinte comando.

$  kubeadm init #### pod-network-cidr=192.168.0.0/16 #### ignore-preflight-errors=NumCPU 

******Se tudo ocorrer bem você receberá uma resposta conforme a tela a seguir.*******

                  [root@kubernetes-master kleber]#  kubeadm init #### pod-network-cidr=192.168.0.0/16 #### ignore-preflight-              errors=NumCPU
                  [init] Using Kubernetes version: v1.15.0
                  [preflight] Running pre-flight checks
                          [WARNING NumCPU]: the number of available CPUs 1 is less than the required 2
                  [preflight] Pulling images required for setting up a Kubernetes cluster
                  [preflight] This might take a minute or two, depending on the speed of your internet connection
                  [preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
                  [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
                  [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
                  [kubelet-start] Activating the kubelet service
                  [certs] Using certificateDir folder "/etc/kubernetes/pki"
                  [certs] Generating "front-proxy-ca" certificate and key
                  [certs] Generating "front-proxy-client" certificate and key
                  [certs] Generating "etcd/ca" certificate and key
                  [certs] Generating "etcd/server" certificate and key
                  [certs] etcd/server serving cert is signed for DNS names [kubernetes-master localhost] and IPs [192.168.207.200 127.0.0.1 ::1]
                  [certs] Generating "etcd/healthcheck-client" certificate and key
                  [certs] Generating "etcd/peer" certificate and key
                  [certs] etcd/peer serving cert is signed for DNS names [kubernetes-master localhost] and IPs [192.168.207.200 127.0.0.1 ::1]
                  [certs] Generating "apiserver-etcd-client" certificate and key
                  [certs] Generating "ca" certificate and key
                  [certs] Generating "apiserver-kubelet-client" certificate and key
                  [certs] Generating "apiserver" certificate and key
                  [certs] apiserver serving cert is signed for DNS names [kubernetes-master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.207.200]
                  [certs] Generating "sa" key and public key
                  [kubeconfig] Using kubeconfig folder "/etc/kubernetes"
                  [kubeconfig] Writing "admin.conf" kubeconfig file
                  [kubeconfig] Writing "kubelet.conf" kubeconfig file
                  [kubeconfig] Writing "controller-manager.conf" kubeconfig file
                  [kubeconfig] Writing "scheduler.conf" kubeconfig file
                  [control-plane] Using manifest folder "/etc/kubernetes/manifests"
                  [control-plane] Creating static Pod manifest for "kube-apiserver"
                  [control-plane] Creating static Pod manifest for "kube-controller-manager"
                  [control-plane] Creating static Pod manifest for "kube-scheduler"
                  [etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
                  [wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
                  [kubelet-check] Initial timeout of 40s passed.
                  [apiclient] All control plane components are healthy after 55.506496 seconds
                  [upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
                  [kubelet] Creating a ConfigMap "kubelet-config-1.15" in namespace kube-system with the configuration for the kubelets in the cluster
                  [upload-certs] Skipping phase. Please see #### upload-certs
                  [mark-control-plane] Marking the node kubernetes-master as control-plane by adding the label "node-role.kubernetes.io/master=''"
                  [mark-control-plane] Marking the node kubernetes-master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
                  [bootstrap-token] Using token: wo692u.nq96x93oskm97nsn
                  [bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
                  [bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
                  [bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
                  [bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
                  [bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
                  [addons] Applied essential addon: CoreDNS
                  [addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

        mkdir -p $HOME/.kube
        sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
        sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

******Anote o seu token abaixo, iremos usar mais para frente, no meu caso apareceu esse, mas o seu aparecerá outro*******
kubeadm join 192.168.207.200:6443 #### token wo692u.nq96x93oskm97nsn \
    #### discovery-token-ca-cert-hash sha256:69b666799686469f344b430621da92a95bae15f5aff3c5cb5bd58be65032135e


No meu caso aconteceram alguns erros na hora de subir o cluster, caso tenham problemas como esse:
____________________________________________________________________________________________________________________________________

                  [root@kubernetes-master kleber]# kubeadm init
                  I0620 17:48:55.805409   27046 version.go:240] remote version is much newer: v1.15.0; falling back to: stable-1.14
                  [init] Using Kubernetes version: v1.14.3
                  [preflight] Running pre-flight checks
                          [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
                  error execution phase preflight: [preflight] Some fatal errors occurred:
                          [ERROR NumCPU]: the number of available CPUs 1 is less than the required 2
                  [preflight] If you know what you are doing, you can make a check non-fatal with `#### ignore-preflight-errors=...`


            **Erro de versão, repare que estou usando a versão 1.14.3 e necessita da v1.15:0
                    #### O comando para ajustar isso é:
                    **Verificar as versões existentes
                     $ yum list #### showduplicates kubeadm #### disableexcludes=kubernetes
                    **Após a verificação, instalei a última versão, conforme o erro acima, em cada um dos nós 
                     $ yum install -y kubeadm-1.15.0-0 #### disableexcludes=kubernetes

 **No erro acima, tem um outro problema que é: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd", 
 para corrigir isso eu executei os procedimentos do site abaixo, relaciona a minha versão que é o CentOS.
 
        https://kubernetes.io/docs/setup/production-environment/container-runtimes/
_____________________________________________________________________________________________________________________________________
#### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### -
 ***Em um dos nós tive probrema da base Yum estar corrompida, aí precisamos fazer alguns passos para restabelecer essa base, conforme abaixo:
 #### #### -Erro
 Yum Update: DB_RUNRECOVERY Fatal error, run database recovery
 
         # yum update
            ...
            rpmdb: page 18816: illegal page type or format
            rpmdb: PANIC: Invalid argument
            rpmdb: Packages: pgin failed for page 18816
            error: db4 error(-30974) from dbcursor->c_get: DB_RUNRECOVERY: Fatal error, run database recovery
            rpmdb: PANIC: fatal region error detected; run recovery

#### remove the RPM database, rebuild it and let yum download all the mirror's file lists.

        $ mv /var/lib/rpm/__db* /tmp/
        $ rpm #### rebuilddb
        $ yum clean all
        
#### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### #### 
 Após ajustar esses problemas, voltamos para os nós, vamos usar o Token que foi mostrado após a execução do comando:
 
             $ sudo kubeadm init .......
 
####  O próximo passo deve ser executado no seu usuário comum. Perceba que em todo procedimento utilizamos sudo para elevar as permissões. Se você utilizou o atalho de logar diretamente com o root, agora é a hora de voltar ao seu usuário. Feito isso, execute os seguintes comandos:

            $ mkdir -p $HOME/.kube
            $ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
            $ sudo chown $(id -u):$(id -g) $HOME/.kube/config

#### Podemos executar o seguinte comando para verificar o pods em execução:

            $ kubectl get pods #### all-namespaces
            
            **Neste momento, o resultado deve ser como mostra a figura abaixo. Perceba que o pod do DNS permanece pendente. Isso ocorre por que ainda não configuramos a rede. Faremos isso na sequência.
 
      ** O resultado esperado é:
            [root@kubernetes-master kleber]# docker info | grep -i cgroup
            Cgroup Driver: systemd
            [root@kubernetes-master kleber]# kubectl get pods #### all-namespaces
            NAMESPACE     NAME                                        READY   STATUS    RESTARTS   AGE
            kube-system   coredns-5c98db65d4-5pdnz                    1/1     Running   0          3d
            kube-system   coredns-5c98db65d4-hgbt4                    1/1     Running   0          3d
            kube-system   etcd-kubernetes-master                      1/1     Running   0          3d
            kube-system   kube-apiserver-kubernetes-master            1/1     Running   0          3d
            kube-system   kube-controller-manager-kubernetes-master   1/1     Running   1          3d
            kube-system   kube-flannel-ds-amd64-8c2jv                 1/1     Running   2          2d23h
            kube-system   kube-flannel-ds-amd64-8qf94                 1/1     Running   0          2d23h
            kube-system   kube-flannel-ds-amd64-rrlv5                 1/1     Running   0          2d23h
            kube-system   kube-proxy-4bs5r                            1/1     Running   0          2d23h
            kube-system   kube-proxy-8ms2n                            1/1     Running   0          2d23h
            kube-system   kube-proxy-ftqh5                            1/1     Running   0          3d
            kube-system   kube-scheduler-kubernetes-master            1/1     Running   1          3d
            [root@kubernetes-master kleber]#

#### Agora vamos instalar nossa rede que será responsável pela comunicação entre os nós. Existem vários tipos de redes suportadas pelo Kubernetes. Neste tutorial, utilizaremos a rede Flannel. Para iniciar, execute os seguintes comandos:

            $ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
            $ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/k8s-manifests/kube-flannel-rbac.yml
            
#### Pode levar algum tempo até todos os pods estarem rodando. Espere até que os pods fiquem todos no estado de Running. Para verificar o estado execute o seguinte comando:

            $ kubectl get pods #### all-namespaces


Configurando os nós escravos

Em cada nós escravo é necessário executar o comando kubeadm join para entrar no cluster. O comando exato é mostrado logo após o comando kubeadm init no nó master. Recupere o comando mostrado e execute em cada nó escravo.

Como exemplo, o meu comando foi o seguinte:

      $ sudo kubeadm join 192.168.207.200:6443 #### token wo692u.nq96x93oskm97nsn \
            #### discovery-token-ca-cert-hash sha256:69b666799686469f344b430621da92a95bae15f5aff3c5cb5bd58be65032135e
    
    
#### Para ter certeza que o nó entrou no cluster volte ao nó master e execute o seguinte comando:

      $ kubectl get nodes

Este comando mostrará os nós disponíveis no cluster. Ao final, o resultado esperado é o seguinte:

            [root@kubernetes-master kleber]# kubectl get nodes
            NAME                STATUS   ROLES    AGE     VERSION
            kubernetes-master   Ready    master   3d      v1.15.0
            kubernetes-node-1   Ready    <none>   2d23h   v1.15.0
            kubernetes-node-2   Ready    <none>   2d23h   v1.15.0


      $ kubeadm token create #### print-join-command

Bem gente, é isso! Espero que consigam configurar o cluster de vocês.
É um pouco trabalhoso, mas se seguir direitinho todos os passos dará certo. 

Abraços e até a próxima!

Kleber Vilarim


# Kubernetes course
This repository contains the course files for my Kubernetes course on Udemy: https://www.udemy.com/learn-devops-the-complete-kubernetes-course/?couponCode=KUBERNETES_GITHUB

