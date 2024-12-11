+++
date = '2024-12-10T22:24:48-05:00'
draft = true
title = 'Packetbeat'
description = ''
series = []
tags = []
categories = []
+++

headline
<!--more-->

I am writing this series of posts to keep track of everything I learn while adding a new protocol to Packetbeat. I am very new to Go and pretty new to development in general and especially new to contributing to open source software - so I expect to learn a lot. 

# Why am I doing this?

Short answer: 
- Working on a project integrating a SIEM into a nuclear power plant simulator
- Need to support OPCUA traffic
- ElasticSearch and Kibana is the SIEM solution
- Current solution is slow, inefficient, and flimsy

Long answer:

I am working on a project with the [IAEA]() and their Asherah Nuclear Power Plant simulator, which is used for security training exercises. My task is to integrate a SIEM into the simulator's stack, and I chose ElasticSearch/Kibana - we don't really have a use for Logstash for our purposes. A few months ago I hacked together a Python script that uses the PyShark package (which uses tshark) to capture packets and then send them to ElasticSearch via API requests. This works, but extending Packetbeat instead will have multiple benefits:
- Everything will be faster simply because it's Go vs. Python
- Ingestion will be more reliable due to leveraging existing Beats mechansims
- I will likely learn numerous design improvements along the way from the existing codebase, since I'm a complete novice

# Getting started

## Development environment

Before diving right into the codebase, there are multiple things to set up. For now I'll be setting up one development VM and one Elastic VM. Technically these could be the same VM but I like to keep things separated. 

I typically just develop on my day-to-day machine, but I felt like having an isolated environment that I can easily spin up again if the need arises. Initially I was actually going to use this as an excuse to check out NixOS, but they're missing the required version of Go so I just went back to Ubuntu Server. 

There isn't really too much to set up the environment. We install Git, Make, GCC, libpcap, tmux, Python venv, Docker, Go, and Mage. We also clone the Beats repository and set our GOPATH environment variable.
    
```bash
sudo apt update && sudo apt upgrade -y
sudo apt-get install -y git gcc make libpcap-dev tmux python3-venv

curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh ./get-docker.sh

wget https://go.dev/dl/go1.22.9.linux-amd64.tar.gz
tar -xvzf go1.22.9.linux-amd64.tar.gz

sudo bash -c 'echo "export PATH=\$PATH:/home/hayden/go/bin" >> /home/hayden/.bashrc && echo "export GOPATH=/home/hayden/" >> /home/hayden/.bashrc'5106
mkdir -p ${GOPATH}/src/github.com/elastic
git clone https://github.com/elastic/beats ${GOPATH}/src/github.com/elastic/beats
go install github.com/magefile/mage@latest
mage -init

# lazygit install
wget https://github.com/jesseduffield/lazygit/releases/download/v0.44.1/lazygit_0.44.1_Linux_x86_64.tar.gz
tar -xvzf lazygit*
rm LICENSE
rm README.md
sudo mv lazygit /usr/bin
```

Following is the Neovim setup which obviously is only required if you use Neovim. 

```bash
cd /tmp
wget https://github.com/neovim/neovim/releases/latest/download/nvim-linux64.tar.gz
mv nvim-linux64 /usr/bin/nvim

sudo bash -c 'echo "export PATH=\$PATH:/usr/bin/nvim/bin/" >> /home/hayden/.bashrc'

# add "vim" alias for nvim in .bashrc
alias vim="nvim"
```

After copying over my Neovim config files, the dev environment is finished being set up. Test that it works by running `make update` (takes a while to complete).

For our ELK machine we'll use a barebones Ubuntu Server setup and run ELK using [docker-elk](https://github.com/deviantony/docker-elk).

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y tmux vim
curl -fsSL https://get.docker.com -o docker.sh
sudo sh ./docker.sh

git clone https://github.com/deviantony/docker-elk
cd docker-elk
sudo docker compose up setup
sudo docker compose up -d
```
