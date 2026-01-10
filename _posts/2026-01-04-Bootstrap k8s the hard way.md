
1. 리눅스에 VirtualBox 설치하기
```bash
# 저장소 추가
sudo mkdir -p /etc/apt/keyrings
wget -O- https://www.virtualbox.org/download/oracle_vbox_2016.asc | sudo gpg --dearmor --yes --output /etc/apt/keyrings/oracle-virtualbox2016.gpg

echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/oracle-virtualbox2016.gpg] https://download.virtualbox.org/virtualbox/debian $(lsb_release -cs) contrib" | sudo tee /etc/apt/sources.list.d/virtualbox.list

# 업데이트 및 설치
sudo apt-get update
sudo apt-get install -y virtualbox-7.0
sudo apt-get install -y virtualbox-ext-pack

# 사용자 추가
sudo usermod -aG vboxusers $USER
```


- 트러블 슈팅
	- IP 추가 필요
```bash
sudo bash -c 'echo "* 192.168.10.0/24" >> /etc/vbox/networks.conf'
```
	
-  VirtualBox와 KVM 충돌


2. 리눅스에 Vagrant 설치하기
```bash
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install vagrant
```


![](obsidian://notepix/assets/20260110T042706107Z.png)

![](obsidian://notepix/assets/20260110T042615668Z.png)



3.  kubelet설치하기

kubelet 최신버전 다운로드하기 
https://www.downloadkubernetes.com/

```bash
wget https://dl.k8s.io/v1.34.2/bin/linux/amd64/kubelet

```


kubelet systemd 작성
```bash
curl -O https://storage.googleapis.com/configs.kuar.io/kubelet.service
```