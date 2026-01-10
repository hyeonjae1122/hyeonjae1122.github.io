

# node-0/1에 PodCIDR과 통신을 위한 OS 커널에 (수동) 라우팅 설정

쿠버네티스에서 Pod은 각 노드 내에서만 IP를 갖는다. 다른 노드의 Pod과 통신하려면 OS 커널 수준에서 라우팅 테이블을 설정해야 한다. 보통 네트워크 플러그인(CNI)이 이를 자동으로 해주지만, 여기서는 수동으로 설정한다.

|항목|네트워크 대역 or IP|
|---|---|
|clusterCIDR|10.200.0.0/16|
|→ node-0 PodCIDR|**10.200.0.0/24**|
|→ node-1 PodCIDR|**10.200.1.0/24**|
|ServiceCIDR|10.32.0.0/24|
|→ api clusterIP|10.32.0.1|

- `vagrant ssh jumpbox`

```bash
SERVER_IP=$(grep server machines.txt | cut -d " " -f 1)
NODE_0_IP=$(grep node-0 machines.txt | cut -d " " -f 1)
NODE_0_SUBNET=$(grep node-0 machines.txt | cut -d " " -f 4)
NODE_1_IP=$(grep node-1 machines.txt | cut -d " " -f 1)
NODE_1_SUBNET=$(grep node-1 machines.txt | cut -d " " -f 4)
echo $SERVER_IP $NODE_0_IP $NODE_0_SUBNET $NODE_1_IP $NODE_1_SUBNET
192.168.10.100 192.168.10.101 10.200.0.0/24 192.168.10.102 10.200.1.0/24

ssh server ip -c route
ssh root@server <<EOF
  ip route add ${NODE_0_SUBNET} via ${NODE_0_IP}
  ip route add ${NODE_1_SUBNET} via ${NODE_1_IP}
EOF
ssh server ip -c route

ssh node-0 ip -c route
ssh root@node-0 <<EOF
  ip route add ${NODE_1_SUBNET} via ${NODE_1_IP}
EOF
ssh node-0 ip -c route

ssh node-1 ip -c route
ssh root@node-1 <<EOF
  ip route add ${NODE_0_SUBNET} via ${NODE_0_IP}
EOF
ssh node-1 ip -c route

```


