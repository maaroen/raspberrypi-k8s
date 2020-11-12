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

`sudo apt-get install vim`

`sudo apt install zsh`

`sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"`

`git clone https://github.com/zsh-users/zsh-completions ${ZSH_CUSTOM:=~/.oh-my-zsh/custom}/plugins/zsh-completions`

`git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting`

`git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions`

`git clone --depth 1 https://github.com/junegunn/fzf.git ~/.fzf && ~/.fzf/install`

And I configured the .zshrc file: `sudo vim ~/.zshrc`

Here setup the plugins like this: `plugins=(git zsh-syntax-highlighting zsh-autosuggestions fzf)`

And set the theme to `wedisagree`
