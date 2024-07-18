# portainers
**docker installs:**
1.  docker
2.  nginx proxy
3.  speedtest
4.  grafana
5.  influxdb
6.  portainers

#README.md docker_installs

This script will help install any, or all, of Docker-CE, Docker-Compose, NGinX Proxy Manager, and Portainer-CE.

#Run: Download và chạy code cài docker Gõ lệnh trên thông qua màn PuTTy đã kết nối thành công tới ipv4 của Ubuntu 20.04:

wget https://raw.githubusercontent.com/PhDLeToanThang/k8s-k3s/main/dockers_portainers/install-docker-nginx.sh && sudo bash install-docker-nginx.sh

Future Work:

Make it work for Raspberry Pi
Make it work for Arch
Make it work for OpenSuse
Maybe add a few other default containers to pull down and start running
Prompt for Credentials to use in NGinX Proxy Manager db settings vs. using the defaults.

Contributing:

If you find issues, please let me know. I'm always open to new contributors helping me add Distro support, more software packages, etc. Just clone the project and make a pull request with any changes you add.

Licensing:

My script is offered without warranty against defect, and is free for you to use any way / time you want. You may modify it in any way you see fit. Please see the individual project pages of the software packages for their licensing.


Troubleshootings:

sudo docker restart portainer
