# Hyperledger Fabric 2.3+ Setup 

Make sure your ubuntu version is `20.04 LTS or higher` and also you are in the `bash` shell. If you are using `fish` or `zsh`, just enter the following command to enter the bash shell. 

```bash 
sudo bash
``` 

## Setting up the Environment 

Let's first go ahead and setup docker and some prerequisites. 

```bash 
sudo apt install git curl jq 
sudo apt-get -y install docker-compose
docker --version 
sudo systemctl start docker
sudo systemctl enable docker 
echo $USER    # ensure your username shows here 
sudo usermod -a -G docker $USER 
``` 

Make sure you log out and log back in to ensure correct permissions are set for your user. Otherwise, docker gives errors. 

```bash 
docker ps 
``` 

If that works correctly i.e. does not throw any errors, you may proceed. Otherwise, first setup the permissions through `usermod` above. 


Let's now setup node. The latest supported version currently is 12. Future versions should be usable and will be mentioned on the Fabric site. 

```bash 
curl -fsSL https://deb.nodesource.com/setup_12.x | sudo -E bash - 
sudo apt update 
sudo apt-get install -y nodejs  
```

Ensure that correct versions of Node and NPM are installed: 

```bash 
npm  --version    # shouled be 6.x 
node --version    # should be 12.x 
```

## Downloading and Installing Fabric Binaries 

Setting up Fabric is now extremely straight-forward using the procided `test-net` scripts. Let's first create a fabric directory to ensure everything remains in one place. 

```bash 
mkdir ~/fabric 
cd fabric 
``` 

Let's download the Fabric binaries in this folder. 


```bash 
curl -sSL https://bit.ly/2ysbOFE | bash -s -- 2.3.1 
```

This will clone the repo and setup docker images using Docker Compose. I strongly recommend you read through all the outputs. 

## Setup the Test Network 

First, create a channel on which our chaincode (or smart contract) will be deployed: 

```bash 
cd fabric-samples/test-network 
./network.sh up createChannel -c channel1 -ca  
``` 

Verify that new containers are indeed up. 

```bash 
docker ps 
``` 

Now let's create a package for our chaincode. First, set up the dependencies and then create a package. 

```bash 
cd ../asset-transfer-basic/chaincode-javascript 
npm install  
``` 

Make sure everythingn installed correctly and without any errors. 

```bash 
cd ../../test-network 
export FABRIC_CFG_PATH=$PWD/../config/ 
echo $FABRIC_CFG_PATH 
```

Now, we must make sure that fabric commands are in the PATH. The following commands adds it to the current session but you should add it to your `~/.bashrc` file as well for future use. 

```
peer   # ensure it's there. If not, set the path 
export PATH=$PATH:/home/nam/fabric/fabric-samples/bin/ 
peer   # should work now 
```

Now, let's create a pacakge from our chaincode source. 


```bash {cmd=true}
peer lifecycle chaincode package basic.tar.gz \
      --path ../asset-transfer-basic/chaincode-javascript \
      --lang node \
      --label basic_1.0
```

Let's set the environment variables needed through the provided script. 

```bash 
source ./scripts/envVar.sh 
``` 

## Installing the Chaincode to Channel 

```bash 
setGlobals 1 
```

This selects the first organization. Now we can go ahead and install the chaincode on the channel. First let's save the certificate file locations for both organizations and the orderer. 

```bash 
# the $PWD is necessary as the commands need absolute paths, relatives don't work 

export ORDERER_CERTFILE=$PWD/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

export ORG1_PEER_CERTFILE=$PWD/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt

export ORG2_PEER_CERTFILE=$PWD/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
```

First, install the package. 

```bash 
peer lifecycle chaincode install basic.tar.gz 
``` 

Then, do the same for Organization 2. 

```bash 
setGlobals 2
peer lifecycle chaincode install basic.tar.gz 

setGlobals 1  # back to org1 
```

Take note of the Package ID given back. We need this to refer to the chaincode in later commands. 

```bash 
peer lifecycle chaincode queryinstalled \
      --peerAddresses localhost:7051 \
      --tlsRootCertFiles $ORDERER_CERTFILE 
```

Using the output of the above command, set the package ID env variable. (It's ok if the queryinstalled one gives you an error. We can proceed without it.)

```bash 
# make sure you use the actual package ID and not copy/paste the one below 
 
export PKGID=basic_1.0-94c84bb2cc404bab2d945d62dc8d8d3837e2074966495d037b27ecfdf7fe171a
```


Once we have that, we need to approve the chaincode definition for organizations. 

```bash 
peer lifecycle chaincode approveformyorg \
      -o localhost:7050 \
      --ordererTLSHostnameOverride  orderer.example.com \
      --tls --cafile $ORDERER_CERTFILE  \
      --channelID channel1 \
      --name basic \
      --version 1 \
      --package-id $PKGID \
      --sequence 1  
```

And do the same for the second organization: 

```bash 
setGlobals 2 

peer lifecycle chaincode approveformyorg \
      -o localhost:7050 \
      --ordererTLSHostnameOverride  orderer.example.com \
      --tls --cafile $ORDERER_CERTFILE  \
      --channelID channel1 \
      --name basic \
      --version 1 \
      --package-id $PKGID \
      --sequence 1  
```

Let's commit the chnages made to the chaincode. We have to specify peers here explicitly. When we later use the chaincode through our code, we will not have this issue. 

```bash 
peer lifecycle chaincode commit \
      -o localhost:7050 \
      --ordererTLSHostnameOverride orderer.example.com \
      --tls --cafile $ORDERER_CERTFILE \
      --channelID channel1 --name basic \
      --peerAddresses localhost:7051 --tlsRootCertFiles $ORG1_PEER_CERTFILE \
      --peerAddresses localhost:9051 --tlsRootCertFiles $ORG2_PEER_CERTFILE \
      --version 1 --sequence 1  
```

And confirm that it went through. 

```bash 
peer lifecycle chaincode querycommitted \
      --channelID channel1 --name basic \
      --cafile  $ORDERER_CERTFILE
```

Finally, see `docker ps` to see chaincode containers running. 
```bash 
docker ps 
```

## Working with the Chaincode through CLI 

Let's first initialize our ledger. For this, we will call the `InitLedger` function that we defined in `chaincode-javascript` project. This can be done using the `invoke` subcommand. 

```bash 
peer chaincode invoke \
      -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com \
      --tls --cafile $ORDERER_CERTFILE  \
      -C channel1 -n basic \
      --peerAddresses localhost:7051 --tlsRootCertFiles $ORG1_PEER_CERTFILE \
      --peerAddresses localhost:9051 --tlsRootCertFiles $ORG2_PEER_CERTFILE \
      -c '{"function": "InitLedger", "Args": []}'
```

The last line is the most important one and is pretty straight-forward. If we just want to read the contents, it's much simpler. 

```bash 
peer chaincode query -C channel1 -n basic -c '{"Args":["GetAllAssets"]}' | jq 
```

And we can also query a particular asset using the `ReadAsset` function, passing it the target asset name. 

```bash 
peer chaincode query -C channel1 -n basic -c '{"Args":["ReadAsset", "asset6"]}' | jq 
```


While you're doing this, you can open the docker logs in another console to see how stuff is going. 

```bash 
# Watch the past logs 
docker-compose -f docker/docker-compose-test-net.yaml logs  | more 

# or put it on follow as a running log 
docker-compose -f docker/docker-compose-test-net.yaml logs -f 
```

Of course, this doesn''t work in another console as we don't have the proper environment variables set. Let's see what needs to be done for that. 

```bash 
bash   # make sure it's bash 

# head over to the proper working folder 
cd ~/fabric/fabric-samples/test-network 

# set the environment variables 
export PATH=$PATH:~/fabric/fabric-samples/bin/
export FABRIC_CFG_PATH=$PWD/../config/

export ORDERER_CERTFILE=$PWD/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

export ORG1_PEER_CERTFILE=$PWD/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt

export ORG2_PEER_CERTFILE=$PWD/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt


source ./scripts/envVar.sh
# Select which organization you're emulating 
setGlobals  1   
```
