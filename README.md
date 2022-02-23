# Introduction
I recently started using Kubernetes and I've really gotten into it. That's why I decided to try to build my own cluster at home, using a raspberry pi. My goal is to eventually use it as a little webserver to host some apps that I would like to play with or think are useful. To do this I also wanted to have wildcard ssl certificates using Let's Encrypt to be able to use HTTPS.

# Setup steps

## Hardware
The list of hardware I used:
- Raspberry Pi 4B 8GB
- Aluminium heatsink case
- 64GB A2 SD Card
- A fan case arround the heatsink casing

## Software
The software I used to make this work:
- Ubuntu 20.10 arm64
- Kubernetes 1.19.3
- Oh-my-zsh + WSL2 because i'm using Windows.

## Installation

### Initial Raspberry Pi configuration
I started with installing Ubuntu 20.10 on the SD Card using [Raspberry pi Imager](https://www.raspberrypi.org/software/).
After this was installed I booted up the rpi, with a screen connected to it and configured my wifi: `sudo vi /etc/netplan/50-cloud-init.yaml`
The result looked something like this, I only changed from `wifis:` and lower:
```yaml
network:
    ethernets:
        eth0:
            dhcp4: true
            optional: true
    version: 2
    wifis:
        wlan0:
            addresses: [192.168.2.7/24]
            gateway4: 192.168.2.254
            optional: false
            nameservers:
                addresses: [192.168.2.254,8.8.8.8]
            access-points:
                "ACCESS_POINT_NAME":
                    password: "WIFI_PASSWORD"
            dhcp4: true
```

After the wifi was working on, I disconnected the screen and did the rest by using ssh. I named my pi `serverpi` in my router DNS so I was able to ssh to it using `ssh ubuntu@serverpi.home`

### Installing Oh-My-Zsh and tools
Next I installed Oh-My-Zsh just for a nicer terminal experience:

```
sudo apt-get install vim
sudo snap install helm --classic
sudo apt install fonts-powerline
sudo apt install zsh
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
git clone https://github.com/zsh-users/zsh-completions ${ZSH_CUSTOM:=~/.oh-my-zsh/custom}/plugins/zsh-completions
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
git clone --depth 1 https://github.com/junegunn/fzf.git ~/.fzf && ~/.fzf/install
```

And I configured the .zshrc file: `sudo vim ~/.zshrc`

Here setup the plugins like this: `plugins=(git zsh-syntax-highlighting zsh-autosuggestions fzf)`

And set the theme to `wedisagree`

### Initial kubernetes cluster setup
The steps I took to setup the kubernetes cluster:
```
sudo apt install -y docker.io
sudo systemctl enable docker

sudo apt install apt-transport-https ca-certificates curl gnupg-agent software-properties-common

sudo swapoff -a
sudo vim /etc/fstab
Inside this file, comment out the /swapfile line by preceeding it with a # symbol, as seen below. Then, close this file and save the changes.
```

Enable the systemd cgroup driver for docker:
```
sudo touch /etc/docker/daemon.json
sudo vim /etc/docker/daemon.json
# add the following content:

{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
```

Append the cgroup and swap options to the kernal command line:
`sudo sed -i '$ s/$/ cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1 swapaccount=1/' /boot/firmware/cmdline.txt`

Enable net.bridge.bridge-nf-call-iptables and -iptables6:
```
sudo touch /etc/sysctl.d/k8s.conf
sudo vim /etc/sysctl.d/k8s.conf

# add the following content:
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
```
```
sudo sysctl --system
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```
Add the kubernetes repo
```
sudo touch /etc/apt/sources.list.d/kubernetes.list
sudo vim /etc/apt/sources.list.d/kubernetes.list

# add the following content:
deb https://apt.kubernetes.io/ kubernetes-xenial main
```

Install kubernetes tools, and put upgrading on hold
```
sudo apt update && sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Generate cluster token, and start initializing the cluster
```
TOKEN=$(sudo kubeadm token generate)
echo $TOKEN
sudo kubeadm init --token=$TOKEN --kubernetes-version=v1.19.3 --pod-network-cidr=10.244.0.0/16 --node-name master
```

Install Flannel for networking:
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/k8s-manifests/kube-flannel-rbac.yml
```
Create a file in "/etc/systemd/network" named "50-flannel.link"
With the following content:
```
[Match]
OriginalName=flannel*
[Link]
MACAddressPolicy=none
```

Edit /etc/sysctl.conf (this fixes forwarding network traffic to make sure communication between pods works correctly)
Add the line:
```
net.ipv4.ip_forward=1
```
reboot the pi

