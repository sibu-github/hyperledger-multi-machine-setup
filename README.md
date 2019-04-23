# Hyperledger Fabric multi machine setup

This setup guide describes the process of deploying hyperledger fabric blockchain on multiple machines. Note: In this setup we have used "solo" as the orderer service and we have assumed both machines are under same Local Area Network.

## Prerequisites

Following prerequisites are required for this guide:

- curl
- node js
- npm
- go
- docker
- docker-compose
- git

## Steps:

Please follow below steps. The step description will describe whether that step to be executed from machine 1 or machine 2.

### Step1 (Machine 1):

Clone `fabric-samples` repository in machine 1.

```
git clone https://github.com/mahoney1/fabric-samples.git
```

### Step 2 (Machine 1):

Change directory into f`abric-samples`.

```
cd fabric-samples
```

### Step 3 (Machine 1):

Download platform binaries including `cryptogen`, `configtxgen`.

```
curl -sSL http://bit.ly/2ysbOFE | bash -s 1.2.1 1.2.1 0.4.10
```

### Step 4 (Machine 1):

Checkout `multi-org` branch.

### Step 5 (Machine 1):

Change directory into `first-network`.

```
cd first-network
```

### Step 6 (Machine 1):

Copy the `chaincode` folder from this repo into the `first-network` directory.

### Step 7 (Machine 1):

Create a blank `ccp` folder.

```
mkdir ccp
```

### Step 8 (Machine 1):

Copy `configtx.yaml` from this repo into the `first-network` directory.

### Step 9 (Machine 1):

Copy `crypto-config.yaml` from this repo into the `first-network` directory.

### Step 10 (Machine 1):

Copy `docker-compose-org1.yaml` from this repo into `first-network` directory.

### Step 11 (Machine 1):

Create a blank `channel-artifacts` directory.

```
rm -rf channel-artifacts
mkdir channel-artifacts
```

### Step 12 (Machine 1):

Create a blank `crypto-config` directory.

```
rm -rf crypto-config
mkdir crypto-config
```

### Step 13 (Machine 1):

Add cryptogen and configtxgen binaries in PATH.

```
export PATH=${PWD}/../bin:${PWD}:$PATH

which cryptogen
which configtxgen
```

### Step 14 (Machine 1):

Generate crypto files.

```
cryptogen generate --config=./crypto-config.yaml
```

### Step 15 (Machine 1):

Generate channel artifacts.

```
// orderer block
configtxgen -profile TwoOrgsOrdererGenesis -outputBlock ./channel-artifacts/genesis.block

// channel.tx
configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID mychannel

// Org1MSP
configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org1MSPanchors.tx -channelID mychannel -asOrg Org1MSP

// Org2MSP
configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org2MSPanchors.tx -channelID mychannel -asOrg Org2MSP
```

### Step 16 (Machine 1):

Replace the CA Private Key in `docker-compose-org1.yaml` file.

```
CURRENT_DIR=$PWD
cd crypto-config/peerOrganizations/org1.example.com/ca/
PRIV_KEY=$(ls *_sk)
cd "$CURRENT_DIR"
sed -i "s/CA1_PRIVATE_KEY/${PRIV_KEY}/g" docker-compose-org1.yaml
```

> NOTE: On mac run the below command to replace the ceritificate file name.

```
sed -it "s/CA1_PRIVATE_KEY/${PRIV_KEY}/g" docker-compose-org1.yaml
```

### Step 17 (Machine 1):

Replace SECOND_MACHINE_IP in `docker-compose-org1.yaml`.
Get the second machine IP and run the below command in first machine.

```
sed -i "s/SECOND_MACHINE_IP/<PUT_ACTUAL_IP_HERE>/g" docker-compose-org1.yaml
```

### Step 18 (Machine 2):

Repeat steps 1 - 7 in second machine.

### Step 19 (Machine 2):

Copy `crypto-config` and `channel-artifacts` directory from machine 1 to machine 2.

### Step 20 (Machine 2):

Copy `docker-compose-org2.yaml` from this repo into `first-network` directory.

### Step 21 (Machine 2):

Replace CA Private Key in `docker-compose-org2.yaml`.

```
CURRENT_DIR=$PWD
cd crypto-config/peerOrganizations/org2.example.com/ca/
PRIV_KEY=$(ls *_sk)
cd "$CURRENT_DIR"
sed -i "s/CA2_PRIVATE_KEY/${PRIV_KEY}/g" docker-compose-org2.yaml
```

> NOTE: On mac run the below command to replace the ceritificate file name.

```
sed -it "s/CA2_PRIVATE_KEY/${PRIV_KEY}/g" docker-compose-org2.yaml
```

### Step 22 (Machine 2):

Replace FIRST_MACHINE_IP in `docker-compose-org2.yaml`.
Get the first machine IP and run the below command in second machine.

```
sed -i "s/FIRST_MACHINE_IP/<PUT_ACTUAL_IP_HERE>/g" docker-compose-org2.yaml
```

### Step 23 (Machine 1):

Start up the docker containers in machine 1.

```
docker-compose -f docker-compose-org1.yaml up -d
```

### Step 24 (Machine 2):

Start up the docker containers in machine 2.

```
docker-compose -f docker-compose-org2.yaml up -d
```

### Step 25 (Machine 1):

Create channel from peer0.org1

```
docker exec -it cli bash

CORE_PEER_LOCALMSPID="Org1MSP"

CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt

CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp

CORE_PEER_ADDRESS=peer0.org1.example.com:7051

peer channel create -o orderer.example.com:7050 -c mychannel -f ./channel-artifacts/channel.tx --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

```

### Step 26 (Machine 1):

Join channel for peer0.org1

```
peer channel join -b mychannel.block
```

### Step 27 (Machine 1):

Join channel for peer1.org1

```
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer1.org1.example.com/tls/ca.crt

CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp

CORE_PEER_ADDRESS=peer1.org1.example.com:7051

peer channel join -b mychannel.block
```

### Step 28 (Machine 2):

Fetch channel config and join channel for peer0.org2

```
docker exec -it cli bash

CORE_PEER_LOCALMSPID="Org2MSP"

CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt

CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp

CORE_PEER_ADDRESS=peer0.org2.example.com:7051

peer channel fetch config -c mychannel -o orderer.example.com:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

peer channel join -b mychannel_config.block
```

### Step 29 (Machine 2):

Join channel for peer1.org2

```
CORE_PEER_LOCALMSPID="Org2MSP"

CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer1.org2.example.com/tls/ca.crt

CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp

CORE_PEER_ADDRESS=peer1.org2.example.com:7051

peer channel join -b mychannel_config.block
```

### Step 30 (Machine 1):

Update anchor peer for org1.

```
CORE_PEER_LOCALMSPID="Org1MSP"

CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt

CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp

CORE_PEER_ADDRESS=peer0.org1.example.com:7051

peer channel update -o orderer.example.com:7050 -c mychannel -f ./channel-artifacts/Org1MSPanchors.tx --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

### Step 31 (Machine 2):

Update anchor peer for org2.

```
CORE_PEER_LOCALMSPID="Org2MSP"

CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt

CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp

CORE_PEER_ADDRESS=peer0.org2.example.com:7051

peer channel update -o orderer.example.com:7050 -c mychannel -f ./channel-artifacts/Org2MSPanchors.tx --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

### Step 32 (Machine 2):

Package and build chaincode.

```
peer chaincode package ccpack.out -n mycc -l golang -p github.com/chaincode/chaincode_example02/go/ -v 1.0

mv ccpack.out ccp/

exit
```

### Step 33:

Move the generated `ccpack.out` file from machine 2 to machine 1. `ccpack.out` file can be found in `ccp` folder under `first-network` directory. Copy this file from machine 2 to machine 1 and put under `ccp` folder.

### Step 34 (Machine 2):

Install chaincode.

```
docker exec -it cli bash
peer chaincode install ccpack.out
peer chaincode list --installed
```

### Step 35 (Machine 1):

Install chaincode.

```
docker exec -it cli bash
peer chaincode install ccpack.out
peer chaincode list --installed
```

### Step 36 (Machine 1):

Instantiate chaincode.

```
docker exec -it cli bash

peer chaincode instantiate -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n mycc -l golang -v 1.0 -c '{"Args":["init","a","100","b","200"]}' -P "AND ('Org1MSP.peer','Org2MSP.peer')"
```

### Step 37 (Machine 2):

Query chaincode.

```
docker exec -it cli bash
peer chaincode query -C mychannel -n mycc -c '{"Args":["query","a"]}'
```

### Step 38 (Machine 1):

Invoke chaincode.

```
peer chaincode invoke -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n mycc --peerAddresses peer0.org2.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt -c '{"Args":["invoke","a","b","10"]}'
```

### Step 39 (Machine 2):

Query chaincode.

```
docker exec -it cli bash
peer chaincode query -C mychannel -n mycc -c '{"Args":["query","a"]}'
```
