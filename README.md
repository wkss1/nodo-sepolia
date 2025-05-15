# nodo-sepolia
Guía sencilla para montar un nodo en Sepolia, testnet de ethereum con docker, utilizando los clientes geth (ejecución) y lighthouse (consenso).

Guía 1: Cómo correr un nodo Sepolia con Geth y Lighthouse (Ubuntu, Docker)
Requisitos de hardware
Componente	Recomendación
Sistema	Ubuntu 20.04 o superior
RAM	8 - 16 GB
CPU	4 - 6 núcleos
Disco	550 GB SSD (crece hasta 1 TB)

Paso 1: Instalar dependencias y Docker
bash
Copy
Edit
sudo apt-get update && sudo apt-get upgrade -y
sudo apt install curl iptables build-essential git wget lz4 jq make gcc nano automake autoconf tmux htop nvme-cli libgbm1 pkg-config libssl-dev libleveldb-dev tar clang bsdmainutils ncdu unzip -y
Instalar Docker:

bash
Copy
Edit
sudo apt update -y && sudo apt upgrade -y
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done

sudo apt-get update
sudo apt-get install ca-certificates curl gnupg -y
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update -y && sudo apt upgrade -y
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

# Verificar instalación Docker
sudo docker run hello-world

sudo systemctl enable docker
sudo systemctl restart docker
Paso 2: Crear directorios para datos y JWT
bash
Copy
Edit
mkdir -p /root/ethereum/execution
mkdir -p /root/ethereum/consensus
openssl rand -hex 32 > /root/ethereum/jwt.hex
cat /root/ethereum/jwt.hex  # Verificar contenido
Paso 3: Crear archivo docker-compose.yml
bash
Copy
Edit
cd /root/ethereum
nano docker-compose.yml
Pega el siguiente contenido (ajustado para Geth + Lighthouse):

yaml
Copy
Edit
version: "3.8"
services:
  geth:
    image: ethereum/client-go:stable
    container_name: geth
    restart: unless-stopped
    ports:
      - 30303:30303
      - 30303:30303/udp
      - 8545:8545
      - 8546:8546
      - 8551:8551
    volumes:
      - /root/ethereum/execution:/root/.ethereum
      - /root/ethereum/jwt.hex:/root/.ethereum/jwt.hex
    command:
      - --sepolia
      - --http
      - --http.api=eth,net,web3
      - --http.addr=0.0.0.0
      - --authrpc.addr=0.0.0.0
      - --authrpc.vhosts=*
      - --authrpc.jwtsecret=/root/.ethereum/jwt.hex
      - --authrpc.port=8551
      - --syncmode=snap
      - --datadir=/root/.ethereum

  lighthouse:
    image: sigp/lighthouse:stable
    container_name: lighthouse
    restart: unless-stopped
    volumes:
      - /root/ethereum/consensus:/root/.lighthouse
      - /root/ethereum/jwt.hex:/root/.lighthouse/jwt.hex
    depends_on:
      - geth
    ports:
      - 5052:5052   # HTTP API del beacon node
      - 9000:9000   # P2P puerto
    command:
      - beacon_node
      - --network=sepolia
      - --http
      - --http-address=0.0.0.0
      - --http-allow-origin=*
      - --execution-endpoint=http://geth:8551
      - --jwt-secret=/root/.lighthouse/jwt.hex
      - --metrics
      - --metrics-address=0.0.0.0
      - --metrics-port=5054
Paso 4: Comprobar puertos usados
bash
Copy
Edit
netstat -tuln | grep -E '30303|8545|8546|8551|5052|9000'
Si algún puerto está ocupado, cambia la asignación en docker-compose.yml.

Paso 5: Iniciar nodos
bash
Copy
Edit
docker compose up -d
Para ver logs:

bash
Copy
Edit
docker compose logs -f
Paso 6: Verificar sincronización
Nodo de ejecución (Geth):

bash
Copy
Edit
curl -X POST -H "Content-Type: application/json" --data '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}' http://localhost:8545
{"result":false} significa sincronizado.

Si devuelve un objeto con bloques, está sincronizando.

Nodo consenso (Lighthouse):

bash
Copy
Edit
curl http://localhost:5052/eth/v1/node/syncing
{"data":{"is_syncing":false}} indica sincronizado.

Paso 7: Configurar firewall
bash
Copy
Edit
sudo ufw allow 22
sudo ufw allow ssh
sudo ufw allow 8545/tcp    # RPC Geth
sudo ufw allow 5052/tcp    # HTTP Lighthouse
sudo ufw allow 30303/tcp   # P2P Geth
sudo ufw allow 30303/udp   # P2P Geth
sudo ufw allow 9000/tcp    # P2P Lighthouse
sudo ufw enable
Paso 8: Acceder a los endpoints RPC
Nodo	Endpoint local	Endpoint externo
Geth	http://localhost:8545	http://<IP_VPS>:8545
Lighthouse	http://localhost:5052	http://<IP_VPS>:5052

Paso 9: Monitorizar recursos
bash
Copy
Edit
htop
df -h
docker exec -it geth du -sh /root/.ethereum
docker exec -it lighthouse du -sh /root/.lighthouse
