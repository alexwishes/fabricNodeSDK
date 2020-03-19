# 1. Node环境准备
使用nvm工具可以很方便的安装nodejs的任意版本，网络没问题的同学就直接去官方安装(文档编辑的时候是0.35.3版，可以访问`https://github.com/nvm-sh/nvm`查看最新版本以及使用方式，本文最后也会提供下载好的安装脚本)：
```
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.35.3/install.sh | bash
```
完成后刷新一下环境变量:
```
source ~/.bashrc
```
然后就能够使用nvm命令安装node和npm了：
```
nvm install --lts
```

# 2. Fabric环境准备
本文例子使用的是2台主机5个orderer节点进行的raft共识，组织数量为2个，底层网络采用docker swarm，具体请参考链接：[fabric使用raft共识多机部署方式](http://osa.foresealife.com/topic/1182/fabric%E4%BD%BF%E7%94%A8raft%E5%85%B1%E8%AF%86%E5%A4%9A%E6%9C%BA%E9%83%A8%E7%BD%B2%E6%96%B9%E5%BC%8F)
# 3. SDK细节详解
## 3.1 准备工作
### 3.1.1 生成CCP文件
CCP文件全称Common Connection Profile，这个文件里面描述了网络的结构，我们通过SDK读取这个文件就可以很方便的与区块链网络进行连接．在官方的fabric-samples中，我们可以找到ccp文件的例子(fabric-samples/first-network/connection-org1.json等等)，同时，官方也提供了生成ccp文件的脚本和模板(fabric-samples/first-network/ccp-generate.sh，ccp-template.json，ccp-template.yaml)，这里我稍微做了一点更改：

ccp-generate.sh:
```
#!/bin/bash

function one_line_pem {
    echo "`awk 'NF {sub(/\\n/, ""); printf "%s\\\\\\\n",$0;}' $1`"
}

#－－－－－－需要把Orderer端口和OrdererTLS证书变量传进函数－－－－－－
function json_ccp {
    local PP=$(one_line_pem $5)
    local CP=$(one_line_pem $6)
    local OP=$(one_line_pem $8)
    sed -e "s/\${ORG}/$1/" \
        -e "s/\${P0PORT}/$2/" \
        -e "s/\${P1PORT}/$3/" \
        -e "s/\${CAPORT}/$4/" \
        -e "s#\${PEERPEM}#$PP#" \
        -e "s#\${CAPEM}#$CP#" \
        -e "s/\${O0PORT}/$7/" \
        -e "s#\${ORDERERPEM}#$OP#" \
        ccp-template.json 
}

#－－－－－－需要把Orderer端口和OrdererTLS证书变量传进函数－－－－－－
function yaml_ccp {
    local PP=$(one_line_pem $5)
    local CP=$(one_line_pem $6)
    local OP=$(one_line_pem $8)
    sed -e "s/\${ORG}/$1/" \
        -e "s/\${P0PORT}/$2/" \
        -e "s/\${P1PORT}/$3/" \
        -e "s/\${CAPORT}/$4/" \
        -e "s#\${PEERPEM}#$PP#" \
        -e "s#\${CAPEM}#$CP#" \
        -e "s/\${O0PORT}/$7/" \
        -e "s#\${ORDERERPEM}#$OP#" \
        ccp-template.yaml | sed -e $'s/\\\\n/\\\n        /g'
}

#－－－－－－新增Orderer的Port以及Orderer的TLS证书文件－－－－－－
#Orderer端口
O0PORT=7050
#Orderer的TLS证书
ORDERERPEM=/home/vagrant/1.4.6/raft-2-machines/crypto-config/ordererOrganizations/example.com/tlsca/tlsca.example.com-cert.pem

#－－－－－－相应的变量请调整为自己的网络情况－－－－－－
ORG=1
P0PORT=7051
P1PORT=8051
CAPORT=7054
PEERPEM=/home/vagrant/1.4.6/raft-2-machines/crypto-config/peerOrganizations/example.com/tlsca/tlsca.example.com-cert.pem
CAPEM=/home/vagrant/1.4.6/raft-2-machines/crypto-config/peerOrganizations/example.com/ca/ca.example.com-cert.pem

#－－－－－－需要把Orderer端口和OrdererTLS证书变量传进函数－－－－－－
echo "$(json_ccp $ORG $P0PORT $P1PORT $CAPORT $PEERPEM $CAPEM $O0PORT $ORDERERPEM)" > connection-org1.json
echo "$(yaml_ccp $ORG $P0PORT $P1PORT $CAPORT $PEERPEM $CAPEM $O0PORT $ORDERERPEM)" > connection-org1.yaml

#－－－－－－相应的变量请调整为自己的网络情况－－－－－－
ORG=2
P0PORT=7051
P1PORT=8051
CAPORT=7054
PEERPEM=/home/vagrant/1.4.6/raft-2-machines/crypto-config/peerOrganizations/example.com/tlsca/tlsca.example.com-cert.pem
CAPEM=/home/vagrant/1.4.6/raft-2-machines/crypto-config/peerOrganizations/example.com/ca/ca.example.com-cert.pem

#－－－－－－如果有更多组织，按照例子新增－－－－－－
#ORG=3
#P0PORT=...
#P1PORT=...
#CAPORT=...
#PEERPEM=...
#CAPEM=...

#－－－－－－需要把Orderer端口和OrdererTLS证书变量传进函数－－－－－－
echo "$(json_ccp $ORG $P0PORT $P1PORT $CAPORT $PEERPEM $CAPEM)" > connection-org2.json
echo "$(yaml_ccp $ORG $P0PORT $P1PORT $CAPORT $PEERPEM $CAPEM)" > connection-org2.yaml
```
ccp-template.json（新增了orderer和channel的配置信息，按需更新）:
```
{
    "name": "first-network-org${ORG}",
    "version": "1.0.0",
    "client": {
        "organization": "Org${ORG}",
        "connection": {
            "timeout": {
                "peer": {
                    "endorser": "300"
                }
            }
        }
    },
    "organizations": {
        "Org${ORG}": {
            "mspid": "Org${ORG}MSP",
            "peers": [
                "peer0.org${ORG}.example.com",
                "peer1.org${ORG}.example.com"
            ],
            "certificateAuthorities": [
                "ca.org${ORG}.example.com"
            ]
        }
    },
    "channels": {
        "mychannel": {
          "orderers": "orderer.org${ORG}.example.com",
          "peers": {
            "peer0.org${ORG}.example.com": {
              "endorsingPeer": true,
              "chaincodeQuery": true,
              "ledgerQuery": true,
              "eventSource": true,
              "discover": true
            },
            "peer0.baonengmotor.com": {
              "endorsingPeer": true,
              "chaincodeQuery": true,
              "ledgerQuery": true,
              "eventSource": true,
              "discover": true
            }
          }
        }
    },
    "orderers": {
	"orderer.org${ORG}.example.com": {
	    "url": "grpcs://localhost:${O0PORT}",
	    "tlsCACerts": {
		"pem": "${ORDERERPEM}"
	    },
	    "grpcOptions": {
                "ssl-target-name-override": "orderer.org${ORG}.example.com",
                "hostnameOverride": "orderer.org${ORG}.example.com"
	    }
	}
    },
    "peers": {
        "peer0.org${ORG}.example.com": {
            "url": "grpcs://localhost:${P0PORT}",
            "tlsCACerts": {
                "pem": "${PEERPEM}"
            },
            "grpcOptions": {
                "ssl-target-name-override": "peer0.org${ORG}.example.com",
                "hostnameOverride": "peer0.org${ORG}.example.com"
            }
        },
        "peer1.org${ORG}.example.com": {
            "url": "grpcs://localhost:${P1PORT}",
            "tlsCACerts": {
                "pem": "${PEERPEM}"
            },
            "grpcOptions": {
                "ssl-target-name-override": "peer1.org${ORG}.example.com",
                "hostnameOverride": "peer1.org${ORG}.example.com"
            }
        }
    },
    "certificateAuthorities": {
        "ca.org${ORG}.example.com": {
            "url": "https://localhost:${CAPORT}",
            "caName": "ca-org${ORG}",
            "tlsCACerts": {
                "pem": "${CAPEM}"
            },
            "httpOptions": {
                "verify": false
            }
        }
    }
}
```
ccp-template.yaml（新增了orderer和channel的配置信息，按需更新）:
```
---
name: first-network-org${ORG}
version: 1.0.0
client:
  organization: Org${ORG}
  connection:
    timeout:
      peer:
        endorser: '300'
organizations:
  Org${ORG}:
    mspid: Org${ORG}MSP
    peers:
    - peer0.org${ORG}.example.com
    - peer1.org${ORG}.example.com
    certificateAuthorities:
    - ca.org${ORG}.example.com
channels:
  mychannel:
    orderers:
      - orderer.org${ORG}.example.com
    peers:
      peer0.org${ORG}.example.com:
        endorsingPeer: true
        chaincodeQuery: true
        ledgerQuery: true
        eventSource: true
        discover: true
      peer0.baonengmotor.com:
        endorsingPeer: true
        chaincodeQuery: true
        ledgerQuery: true
        eventSource: true
        discover: true
orderers:
  orderer.org${ORG}.example.com:
    url: grpcs://localhost:${O0PORT}
    tlsCACerts:
      pem: |
        ${ORDERERPEM}
    grpcOptions:
      ssl-target-name-override: orderer.org${ORG}.example.com
      hostnameOverride: orderer.org${ORG}.example.com
peers:
  peer0.org${ORG}.example.com:
    url: grpcs://localhost:${P0PORT}
    tlsCACerts:
      pem: |
        ${PEERPEM}
    grpcOptions:
      ssl-target-name-override: peer0.org${ORG}.example.com
      hostnameOverride: peer0.org${ORG}.example.com
  peer1.org${ORG}.example.com:
    url: grpcs://localhost:${P1PORT}
    tlsCACerts:
      pem: |
        ${PEERPEM}
    grpcOptions:
      ssl-target-name-override: peer1.org${ORG}.example.com
      hostnameOverride: peer1.org${ORG}.example.com
certificateAuthorities:
  ca.org${ORG}.example.com:
    url: https://localhost:${CAPORT}
    caName: ca-org${ORG}
    tlsCACerts:
      pem: |
        ${CAPEM}
    httpOptions:
      verify: false
```
文件配置更改好之后，直接运行ccp-generate.sh就行在当前目录下生成connection-org1.json, connection-org1.yaml, connection-org2.json, connection-org2.yaml文件，这些就是我们需要用来连接网络的文件．
### 3.1.2 用户注册
Fabric网络是一个授权的区块链网络，访问网络是需要用户的，在这里我们使用的用户类型有2种：
1. 管理员，负责创建普通用户，创建channel，安装链码等操作
2. 普通用户，负责查询链码，调用链码等一般操作

在本文中，由于没有采用Fabric-CA服务，我们是直接使用cryptogen生成的用户证书文件注册用户的(使用Fabric-CA的注册方式可以查看fabric-sampels/fabcar/javascript/enrollAdmin.js还有registerUser.js)，这里通过SDK完成用户的注册操作：

registerUser.js:
```
/*
 * SPDX-License-Identifier: Apache-2.0
 */

'use strict';

const { FileSystemWallet, Gateway, X509WalletMixin } = require('fabric-network');
const path = require('path');

const fs = require('fs');

async function main() {
    try {

        // Create a new file system based wallet for managing identities.
        const walletPath = path.join(process.cwd(), 'wallet');
        const wallet = new FileSystemWallet(walletPath);
        console.log(`Wallet path: ${walletPath}`);

        // Check to see if we've already enrolled the user.
        const userExists = await wallet.exists('org1_admin');
        if (userExists) {
            console.log('An identity for the user "org1_admin" already exists in the wallet');
            return;
        }

        //导入生成好的cert文件与key文件，需要导入管理员就将路径指向管理员，否则就指向需要的用户
	var cert = fs.readFileSync('/home/vagrant/1.4.6/raft-2-machines/crypto-config/peerOrganizations/example.com/users/Admin@example.com/msp/signcerts/Admin@example.com-cert.pem');
	var key = fs.readFileSync('/home/vagrant/1.4.6/raft-2-machines/crypto-config/peerOrganizations/example.com/users/Admin@example.com/msp/keystore/0d9c402080acfdf11c0c1dadadb9936eefcf583109a617bf398cb4add02a7af0_sk');

        //通过SDK提供的X509WalletMixin方法将cert和key生成SDK可以调用的identity
        const userIdentity = X509WalletMixin.createIdentity('Org1MSP', cert, key);
        await wallet.import('org1_admin', userIdentity);
        console.log('Successfully registered and enrolled admin user "org1_admin" and imported it into the wallet');

        //使用Fabric-CA的调用方式(获取管理员信息)：
        //const enrollment = await ca.enroll({ enrollmentID: 'admin', enrollmentSecret: 'adminpw' });
        //const identity = X509WalletMixin.createIdentity('Org1MSP', enrollment.certificate, enrollment.key.toBytes());
        //await wallet.import('admin', identity);
        //console.log('Successfully enrolled admin user "admin" and imported it into the wallet');

        //使用Fabric-CA的调用方式(使用管理员注册用户)：
        //const secret = await ca.register({ affiliation: 'org1.department1', enrollmentID: 'user4', enrollmentSecret: 'user4pw', role: 'client' }, adminIdentity);
        //const enrollment = await ca.enroll({ enrollmentID: 'user4', enrollmentSecret: 'user4pw' });
        //const userIdentity = X509WalletMixin.createIdentity('Org1MSP', enrollment.certificate, enrollment.key.toBytes());
        //await wallet.import('user4', userIdentity);
        //console.log('Successfully registered and enrolled admin user "user4" and imported it into the wallet');


    } catch (error) {
        console.error(`Failed to register user "org1_admin": ${error}`);
        process.exit(1);
    }
}

main();
```
执行上述代码后，会在当前文件夹生成wallet目录，里面包含了注册的用户信息，后续调用链上操作都是基于这个目录中的用户identity来做的．

## 3.2 Channel操作
### 3.2.1 创建Channel
以下操作的前提是正常启动Fabric网络，并已经生成网络所需要的genesis.block，channel.tx等文件
通过SDK来创建网络，createChannel.js:
```
/*
 * SPDX-License-Identifier: Apache-2.0
 */

'use strict';

const { FileSystemWallet, Gateway } = require('fabric-network');
const path = require('path');
const fs = require('fs');

//设置ccp文件地址，这里用的是json格式的
const ccpPath = path.resolve(__dirname,'connection-org1.json');

async function main() {
    try {

        //设置用户identity位置
        // Create a new file system based wallet for managing identities.
        const walletPath = path.join(process.cwd(), 'wallet');
        const wallet = new FileSystemWallet(walletPath);

        //检查管理员用户是否存在
        // Check to see if we've already enrolled the user.
        const userExists = await wallet.exists('org1_admin');
        if (!userExists) {
            console.log('An identity for the user "org1_admin" does not exist in the wallet');
            console.log('Run the registerUser.js application before retrying');
            return;
        }

        // Create a new gateway for connecting to our peer node.
        const gateway = new Gateway();
        await gateway.connect(ccpPath, { wallet, identity: 'org1_admin', discovery: { enabled: true, asLocalhost: true } });
	
	var fabric_client = await gateway.getClient();
	var orderer = fabric_client.getOrderer("orderer.org1.example.com");
	var channelName = 'mychannel';
        //设置通道配置文件channel.tx
	const envelope = fs.readFileSync('/home/vagrant/1.4.6/raft-2-machines/channel-artifacts/channel.tx');
	var channelConfig = fabric_client.extractChannelConfig(envelope);
	var signature = fabric_client.signChannelConfig(channelConfig);
	const request = {
	        name: channelName,
	        orderer: orderer,
	        config: channelConfig,
	        signatures: [signature],
	        txId: fabric_client.newTransactionID(true)
	};

	fabric_client.createChannel(request);

    } catch (error) {
        console.error(`Failed to create channel: ${error}`);
        process.exit(1);
    }
}

main();
```
执行上面代码没有报错，可以在orderer日志中看到创建channel的相关日志信息．
# 3.2.2 加入Channel
joinChannel.js:
```
/*
 * SPDX-License-Identifier: Apache-2.0
 */

'use strict';

const { FileSystemWallet, Gateway } = require('fabric-network');
const path = require('path');
const fs = require('fs');

const ccpPath = path.resolve(__dirname, 'connection-org1.json');

async function main() {
    try {

        // Create a new file system based wallet for managing identities.
        const walletPath = path.join(process.cwd(), 'wallet');
        const wallet = new FileSystemWallet(walletPath);

        // Check to see if we've already enrolled the user.
        const userExists = await wallet.exists('org1_admin');
        if (!userExists) {
            console.log('An identity for the user "org1_admin" does not exist in the wallet');
            console.log('Run the registerUser.js application before retrying');
            return;
        }

        // Create a new gateway for connecting to our peer node.
        const gateway = new Gateway();
        await gateway.connect(ccpPath, { wallet, identity: 'org1_admin', discovery: { enabled: true, asLocalhost: true } });
	
	var fabric_client = await gateway.getClient();
	var orderer = fabric_client.getOrderer("orderer.org1.example.com");
	var channelName = 'mychannel';

	const channel = fabric_client.getChannel(channelName);
        //在创世区块中包含了channel信息，通过创世区块加入channel
	const myblock = await channel.getGenesisBlock({"orderer": orderer});
	await channel.joinChannel({'block':myblock, 'txId':fabric_client.newTransactionID(true)});

    } catch (error) {
        console.error(`Failed to create channel: ${error}`);
        process.exit(1);
    }
}

main();
```
## 3.3 Chaincode操作
### 3.3.1 Chaincode安装
installCC.js:
```
/*
 * SPDX-License-Identifier: Apache-2.0
 */

'use strict';

const { FileSystemWallet, Gateway } = require('fabric-network');
const path = require('path');
const fs = require('fs');

const ccpPath = path.resolve(__dirname,'connection-org1.json');

async function main() {
    try {

        // Create a new file system based wallet for managing identities.
        const walletPath = path.join(process.cwd(), 'wallet');
        const wallet = new FileSystemWallet(walletPath);

        // Check to see if we've already enrolled the user.
        const userExists = await wallet.exists('org1_admin');
        if (!userExists) {
            console.log('An identity for the user "org1_admin" does not exist in the wallet');
            console.log('Run the registerUser.js application before retrying');
            return;
        }

        // Create a new gateway for connecting to our peer node.
        const gateway = new Gateway();
        await gateway.connect(ccpPath, { wallet, identity: 'org1_admin', discovery: { enabled: true, asLocalhost: true } });
	
	var fabric_client = await gateway.getClient();
	var orderer = fabric_client.getOrderer("orderer.org1.example.com");
	var peer = fabric_client.getPeer("peer0.org1.example.com");
	var channelName = 'mychannel';
	const channel = fabric_client.getChannel(channelName);

        //golang类型的chaincodePath填写方式与node的不一样，不能写绝对路径，需要填写$GOPATH/src下剩余的路径，同时调用命令的时候传入$GOPATH
	var rst = await fabric_client.installChaincode({
		targets: [peer],
		chaincodePath: "github.com/chaincode/chaincode_example02/go/",
		chaincodeId: "mycc",
		chaincodeVersion: "1",
		chaincodeType: "golang",
	});

	console.log(rst);

    } catch (error) {
        console.log(`Failed to install chaincode: ${error}`);
        process.exit(1);
    }
}

main();
```
如果chaincode是使用golang编写的，这部分代码的启动命令有一点不一样，需要在node前面增加GOPATH的地址：`GOPATH=/home/vagrant/go node installCC.js`否则会报找不到chaincode的问题．
### 3.3.2 chaincode实例化
实例化chaincode分为2个步骤，缺少第2步的情况下，在peer节点会启动容器，但是无法进行任何的操作：
1. 发送proposal获取背书
2. 将背书与原始proposal信息发送到orderer执行上链交易

instantiateCC.js:
```
/*
 * SPDX-License-Identifier: Apache-2.0
 */

'use strict';

const { FileSystemWallet, Gateway } = require('fabric-network');
const path = require('path');
const fs = require('fs');

const ccpPath = path.resolve(__dirname, 'connection-org1.json');

async function main() {
    try {

        // Create a new file system based wallet for managing identities.
        const walletPath = path.join(process.cwd(), 'wallet');
        const wallet = new FileSystemWallet(walletPath);

        // Check to see if we've already enrolled the user.
        const userExists = await wallet.exists('org1_admin');
        if (!userExists) {
            console.log('An identity for the user "org1_admin" does not exist in the wallet');
            console.log('Run the registerUser.js application before retrying');
            return;
        }

        // Create a new gateway for connecting to our peer node.
        const gateway = new Gateway();
        await gateway.connect(ccpPath, { wallet, identity: 'org1_admin', discovery: { enabled: true, asLocalhost: true } });
	
	var channelName = 'mychannel';
	const network = await gateway.getNetwork(channelName);
	var channel = await network.getChannel();

	var fabric_client = await gateway.getClient();

        //实例化chaincode分为2个步骤：
        //1. 发送proposal获取背书
        //2. 将背书与原始proposal信息发送到orderer执行上链交易

        //定义发送proposal的对象
	const proposalObj = {
		chaincodeId: "mycc",
		chaincodeVersion: "1",
		chaincodeType: "golang",
		txId: fabric_client.newTransactionID(true),
		args: ["a","100","b","200"],
		endorsementPolicy: {
			identities: [
			    { role: { name: "member", mspId: "Org1MSP" }},
			    { role: { name: "member", mspId: "Org2MSP" }}
			  ],
			  policy: {
			    "1-of": [{ "signed-by": 0 }, { "signed-by": 1 }]
			  }
		}
	};
        
	const proposalRes = await channel.sendInstantiateProposal(proposalObj);

        //发送Proposal后，背书节点会返回response，结构如下：
        // response[0]: 包含背书信息或者错误信息
        // response[1]: 交易需要的原始proposal信息
        //直接将这个response数组的2个元素传给sendTransaction方法
	var rst = await channel.sendTransaction({
		proposalResponses: proposalRes[0],
		proposal: proposalRes[1]
	});

	console.log(rst);

    } catch (error) {
        console.error(`Failed to create channel: ${error}`);
        process.exit(1);
    }
}

main();
```
关键的步骤在代码中已经注释说明了，执行上述代码即可完成chaincode的实例化，在orderer节点和安装chaincode的peer节点上都能看到相应的log信息．
### 3.3.3 chaincode查询
query.js:
```
/*
 * SPDX-License-Identifier: Apache-2.0
 */

'use strict';

const { FileSystemWallet, Gateway } = require('fabric-network');
const path = require('path');

const ccpPath = path.resolve(__dirname, 'connection-org1.json');

async function main() {
    try {

        // Create a new file system based wallet for managing identities.
        const walletPath = path.join(process.cwd(), 'wallet');
        const wallet = new FileSystemWallet(walletPath);
        console.log(`Wallet path: ${walletPath}`);

        // Check to see if we've already enrolled the user.
        const userExists = await wallet.exists('org1_user1');
        if (!userExists) {
            console.log('An identity for the user "org1_user1" does not exist in the wallet');
            console.log('Run the registerUser.js application before retrying');
            return;
        }

        // Create a new gateway for connecting to our peer node.
        const gateway = new Gateway();
	console.log(await wallet.list());
        await gateway.connect(ccpPath, { wallet, identity: 'org1_user1', discovery: { enabled: true, asLocalhost: true } });

        //通过channel名字获取指定的Channel
        // Get the network (channel) our contract is deployed to.
        const network = await gateway.getNetwork('mychannel');

        //通过合约名字获取指定的chaincode
        // Get the contract from the network.
        const contract = network.getContract('mycc');

        //调用合约的Query方法，注意这里使用的调用方法名是evaluateTransaction
        // Evaluate the specified transaction.
        const result = await contract.evaluateTransaction('query','a');
        console.log(`Transaction has been evaluated, result is: ${result.toString()}`);

    } catch (error) {
        console.error(`Failed to evaluate transaction: ${error}`);
        process.exit(1);
    }
}

main();
```
上述代码比较简单，直接通过channel和chaincode名字找到对应的chaincode，执行chaincode的query方法，查询a的值，这里需要注意的点是调用合约的方法名有几种：
1. evaluateTransaction: 调用查询的时候使用这个方法
2. submitTransaction: 需要发起交易的时候调用这个方法
### 3.3.4 chaincode交易
invoke.js:
```
/*
 * SPDX-License-Identifier: Apache-2.0
 */

'use strict';

const { FileSystemWallet, Gateway } = require('fabric-network');
const path = require('path');

const ccpPath = path.resolve(__dirname, 'connection-org1.json');

async function main() {
    try {

        // Create a new file system based wallet for managing identities.
        const walletPath = path.join(process.cwd(), 'wallet');
        const wallet = new FileSystemWallet(walletPath);
        console.log(`Wallet path: ${walletPath}`);

        // Check to see if we've already enrolled the user.
        const userExists = await wallet.exists('org1_user1');
        if (!userExists) {
            console.log('An identity for the user "org1_user1" does not exist in the wallet');
            console.log('Run the registerUser.js application before retrying');
            return;
        }

        // Create a new gateway for connecting to our peer node.
        const gateway = new Gateway();
        await gateway.connect(ccpPath, { wallet, identity: 'org1_user1', discovery: { enabled: true, asLocalhost: true } });

        // Get the network (channel) our contract is deployed to.
        const network = await gateway.getNetwork('mychannel');

        // Get the contract from the network.
        const contract = network.getContract('mycc');

        // Submit the specified transaction.
        await contract.submitTransaction('invoke', 'a', 'b', '1');
        console.log('Transaction has been submitted');

        // Disconnect from the gateway.
        await gateway.disconnect();

    } catch (error) {
        console.error(`Failed to submit transaction: ${error}`);
        process.exit(1);
    }
}

main();
```
与查询代码基本一致，只是将evaluateTransaction部分改为submitTransaction，调用chaincode内的invoke方法，实现从a往b转账1的操作，执行完invoke.js后，再次调用query.js可以发现数值已经变更．
# 4. 结语
上面就是完整的Fabric Node SDK调用过程实例，从创建channel开始一直到最后的chaincode查询调用完整的过程，网上的资源一般都比较零散，整理的这波文档希望能帮助到大家．

参考链接：
[Fabric Node SDK官方文档](https://hyperledger.github.io/fabric-sdk-node/release-1.4/)
[Fabric Samples工程](https://github.com/hyperledger/fabric-samples)
