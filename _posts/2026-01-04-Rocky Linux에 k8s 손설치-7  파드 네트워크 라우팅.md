

# node-0/1에 PodCIDR과 통신을 위한 OS 커널에 (수동) 라우팅 설정

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


