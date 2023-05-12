k3s on CentOS 7

Given the popularity of kubernetes these days, there are a number of really good tutorials on how to install various kubernetes distros on multiple platforms including one of my favorites, the Raspberry Pi.  Being that I used other non-Debian based Linux distros during the day, I looked for something on CentOS 7 but did not find anything that took me start to finish on Intel-based hardware.

First, I bought a small experimental [Lenovo ThinkCentre](https://www.amazon.com/dp/B07G4LVZQZ/ref=cfb_at_prodpg) from Amazon.

Starting with a [CentOS 7 minimal install](http://mirror.centos.iad1.serverforge.org/7.8.2003/isos/x86_64/CentOS-7-x86_64-Minimal-2003.iso), the following steps were used to install the [k3s](https://k3s.io/) version of Kubernetes along with some additional optional steps.

These instructions currently outline a single node cluster but k3s makes it easy to add nodes if/as needed.

Some adjustments are made to the firewall so that the server can be accessed remotely using kubectl (on port 6443)  and via services which utilize http and https protocols (on ports 80 and 443 respectively).
There is also the need for masquerading so that a working ingress can be configured.
```
sudo firewall-cmd --zone=public --permanent --add-port=6443/tcp
sudo firewall-cmd --zone=public --permanent --add-port=443/tcp
sudo firewall-cmd --zone=public --permanent --add-port=8443/tcp
sudo firewall-cmd --zone=public --permanent --add-port=8472/udp
sudo firewall-cmd --zone=public --permanent --add-port=10250/tcp
sudo firewall-cmd --zone=public --add-masquerade --permanent
sudo firewall-cmd --reload
```

The next command should download the script which installs k3s.  A parameter is included which will prevent the installation of Traefik, the default ingress, because the nginx ingress is installed in a later step.
```
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server --disable traefik" sh
```

```
sudo groupadd k3s
sudo usermod -aG k3s $USER
sudo chown root:k3s /etc/rancher/k3s/k3s.yaml
sudo chmod 740 /etc/rancher/k3s/k3s.yaml
```

Assuming the above command succeeded, the k3s cluster should be up and running.

The following command is an opportunity to look at the settings which are needed for the kubectl config file.  This can be added to ~/.kube/config on another computer to access the k3s cluster.
```
cat /etc/rancher/k3s/k3s.yaml
```

Next, helm is installed which will be used to install the nginx ingresss.
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod +x get_helm.sh 
./get_helm.sh
```

In order to install the nginx ingress, the repo needs to be added.
```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

Below a namespace is created and then helm is used to install the ingress.
```
kubectl create namespace ingress-nginx 
helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx
```

The following steps are not required but nice to have, at least in my opinion.

First off, kubectx is a nice tool to switch between contexts.  It is used to switch between the k3s cluster and the nginx ingress.  Along with it, kubens is used to switch between namespaces.  And finally, fzf allows easier command line selection of options.
```
sudo git clone https://github.com/ahmetb/kubectx /opt/kubectx
sudo ln -s /opt/kubectx/kubectx /usr/local/bin/kubectx
sudo ln -s /opt/kubectx/kubens /usr/local/bin/kubens
git clone --depth 1 https://github.com/junegunn/fzf.git ~/.fzf
~/.fzf/install
```

Next, prometheus is installed so that we can monitor the k3s cluster.  Prometheus seems to be a bit of a moving target lately, at least if you are trying to use a Helm chart.
```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install my-release bitnami/kube-prometheus --namespace=monitoring --create-namespace
```