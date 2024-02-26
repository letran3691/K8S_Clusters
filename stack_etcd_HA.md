config All node


    swapoff -a
    sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

add hosts all nodes

    cat >> /etc/hosts << EOF
    10.84.4.120 master01
    10.84.4.121 master02
    10.84.4.122 master03
    10.84.4.120 worker01
    10.84.102.73 worker02
    10.84.4.123 vip-k8s
    EOF

Packages install all nodes

    yum install yum-utils -y
    yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
    dnf install containerd.io -y
    containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1

    tee /etc/modules-load.d/containerd.conf <<EOF
    overlay
    br_netfilter
    EOF

    sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
    systemctl restart containerd
    systemctl enable containerd
    systemctl status containerd

    cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
    name=Kubernetes
    baseurl=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/
    enabled=1
    gpgcheck=1
    gpgkey=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/repodata/repomd.xml.key
    exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
    EOF    

    yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
    
    SYSCTLDIRECTIVES='net.bridge.bridge-nf-call-iptables net.ipv4.conf.all.forwarding net.ipv4.conf.default.forwarding net.ipv4.ip_forward'
    
    for directive in ${SYSCTLDIRECTIVES}; do
      if cat /etc/sysctl.d/99-sysctl.conf | grep -q "${directive}"; then
        echo "Directive ${directive} is loaded"
      else
        echo "${directive}=1" >> /etc/sysctl.d/99-sysctl.conf
      fi
    done
    
    modprobe overlay
    modprobe br_netfilter

    sysctl -p /etc/sysctl.d/99-sysctl.conf
    
    systemctl enable containerd kubelet
    
    systemctl stop firewalld
    systemctl disable firewalld
    systemctl mask firewalld


## install on all master nodes

    yum install haproxy keepalived -y

## on master01
    vi /etc/keepalived/check_apiserver.sh
    
    #!/bin/sh
    APISERVER_VIP=10.84.4.123
    APISERVER_DEST_PORT=6443
    
    errorExit() {
        echo "*** $*" 1>&2
        exit 1
    }
    
    curl --silent --max-time 2 --insecure https://localhost:${APISERVER_DEST_PORT}/ -o /dev/null || errorExit "Error GET https://localhost:${APISERVER_DEST_PORT}/"
    if ip addr | grep -q ${APISERVER_VIP}; then
        curl --silent --max-time 2 --insecure https://${APISERVER_VIP}:${APISERVER_DEST_PORT}/ -o /dev/null || errorExit "Error GET https://${APISERVER_VIP}:${APISERVER_DEST_PORT}/"
    fi

    chmod +x /etc/keepalived/check_apiserver.sh

    cp /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf-org

        
    sh -c '> /etc/keepalived/keepalived.conf'
    vi /etc/keepalived/keepalived.conf
    
    ! /etc/keepalived/keepalived.conf
    ! Configuration File for keepalived
    global_defs {
        router_id LVS_DEVEL
    }
    vrrp_script check_apiserver {
      script "/etc/keepalived/check_apiserver.sh"
      interval 3
      weight -2
      fall 10
      rise 2
    }
    
    vrrp_instance VI_1 {
        state MASTER
        interface ens192
        virtual_router_id 151
        priority 255
        authentication {
            auth_type PASS
            auth_pass 8dasdsa8Aq@
        }
        virtual_ipaddress {
            10.84.4.123/24
        }
        track_script {
            check_apiserver
        }
    }


    cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg-org

### Note: remove config forntend, backend for example

    vi /etc/haproxy/haproxy.cfg
    #---------------------------------------------------------------------
    # apiserver frontend which proxys to the masters
    #---------------------------------------------------------------------
    frontend apiserver
        bind *:8443
        mode tcp
        option tcplog
        default_backend apiserver
    #---------------------------------------------------------------------
    # round robin balancing for apiserver
    #---------------------------------------------------------------------
    backend apiserver
        option httpchk GET /healthz
        http-check expect status 200
        mode tcp
        option ssl-hello-chk
        balance     roundrobin
            server master01 10.84.4.120:6443 check
            server master02 10.84.4.121:6443 check
            server master03 10.84.4.122:6443 check


#### copy config file haproxy and keepalive to master02 & 03 nodes
    for f in master02 master03; do scp /etc/keepalived/check_apiserver.sh /etc/keepalived/keepalived.conf root@$f:/etc/keepalived; scp /etc/haproxy/haproxy.cfg root@$f:/etc/haproxy; done

#### On master02 & 03

#### Note: Only two parameters of this file need to be changed for master-2 & 3 nodes. State will become SLAVE for master 2 and 3, priority will be 254 and 253 respectively.

    systemctl enable keepalived --now
    systemctl enable haproxy --now
    systemctl enable kubelet --now

#### init control plane on master01

    kubeadm init --control-plane-endpoint="vip-k8s:8443" --upload-certs --v=5

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

#### join control-plane nodes

    kubeadm join vip-k8s:8443 --token fxd5vd.kmtx7y9oeehg09gj \
            --discovery-token-ca-cert-hash sha256:d45bd1251af7bff8a036a7aa6b2c4d1a4a2bef59ec2346296d4f31fb9ac85b35 \
            --control-plane --certificate-key 70a858bc01c82e1cdb5380c0beec33d491499c741e1ec46e4d1a2b027b6a0b30
  
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

#### join worker nodes

    kubeadm join vip-k8s:8443 --token fxd5vd.kmtx7y9oeehg09gj \
            --discovery-token-ca-cert-hash sha256:d45bd1251af7bff8a036a7aa6b2c4d1a4a2bef59ec2346296d4f31fb9ac85b35