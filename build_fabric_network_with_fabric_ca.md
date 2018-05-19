# 基于fabric-ca手动搭建fabric网络

# 1 简介

在[一步一步搭建hyperledger环境](./build_fabric_network_step_by_step.md)和[kafka共识的orderer集群fabric网络环境搭建](./build_fabric_network_with_kafka_orders.md)两个例子中，我们都只使用`cryptogen`这个命令行工具来生成各个组织的证书和公私钥文件，在实际生产环境中，我们应当为每个组织建立单独的CA服务器，为这个组织提供PKI体系管理，如图所示：。

![1525833031472](build_fabric_network_with_fabric_ca.assets/1525833031472.png)

- enrollment certificate：用户身份证书，作为在网络中确认用户身份使用，长期证书。
- transaction certificate：交易证书(目前暂未实现)，每个交易生成的短期证书。
- tls certificate：tls通信使用的证书，节点之间建立https安全连接时使用。长期证书。

在网络拓扑中，`fabric ca`的工作方式如下图所示：

![1525424458286](build_fabric_network_with_fabric_ca.assets/1525424458286.png)

fabric ca client、fabric sdk通过REST API与`fabric ca`交互。大规模情况下，可以用ca形成集群，在集群中存在一个ca根服务器，和若干中间ca服务器，所有ca服务通过haproxy实现负载均衡，共享相同的存储数据库或者LDAP配置。而fabric ca client和fabric sdk都是通过haproxy来与ca集群交互的。

# 2 fabric ca的集成场景

> 集成验证环境使用1.1版本进行验证

## 2.1 使用fabric-ca管理所有证书

参考官方[fabric-ca集成例子](https://github.com/hyperledger/fabric-samples/tree/release-1.1/fabric-ca)，将所有用户证书、tls证书生成的部分采用手动方式，加深具体操作的理解。

> - 环境有三个组织：ordererOrg、peerOrg1、peerOrg2，分别为orderer和peer的示例组织。
>
> ​       每个peer组织中有两个peer。
>
> - 相关yaml和参考脚本参见链接。

使用命令`docker-compose -f docker-compose-root-ca.yaml up -d`文件，启动各个组织的root ca服务。对应yaml：

```yaml
version: '2'

services:

  rca-ordererOrg:
    container_name: rca-ordererOrg
    image: hyperledger/fabric-ca
    command: /bin/bash -c '/scripts/start-root-ca.sh 2>&1'
    environment:
      - FABRIC_CA_SERVER_HOME=/etc/hyperledger/fabric-ca
      - FABRIC_CA_SERVER_TLS_ENABLED=true
      - FABRIC_CA_SERVER_CSR_CN=rca-ordererOrg
      - FABRIC_CA_SERVER_CSR_HOSTS=rca-ordererOrg
      - FABRIC_CA_SERVER_DEBUG=true
      - TARGET_CERTFILE=/data/ordererOrg-ca-cert.pem
    volumes:
      - ./scripts:/scripts
      - ./data:/data

  rca-peerOrg1:
    container_name: rca-peerOrg1
    image: hyperledger/fabric-ca
    command: /bin/bash -c '/scripts/start-root-ca.sh 2>&1'
    environment:
      - FABRIC_CA_SERVER_HOME=/etc/hyperledger/fabric-ca
      - FABRIC_CA_SERVER_TLS_ENABLED=true
      - FABRIC_CA_SERVER_CSR_CN=rca-peerOrg1
      - FABRIC_CA_SERVER_CSR_HOSTS=rca-peerOrg1
      - FABRIC_CA_SERVER_DEBUG=true
      - TARGET_CERTFILE=/data/peerOrg1-ca-cert.pem
    volumes:
      - ./scripts:/scripts
      - ./data:/data

  rca-peerOrg2:
    container_name: rca-peerOrg2
    image: hyperledger/fabric-ca
    command: /bin/bash -c '/scripts/start-root-ca.sh 2>&1'
    environment:
      - FABRIC_CA_SERVER_HOME=/etc/hyperledger/fabric-ca
      - FABRIC_CA_SERVER_TLS_ENABLED=true
      - FABRIC_CA_SERVER_CSR_CN=rca-peerOrg2
      - FABRIC_CA_SERVER_CSR_HOSTS=rca-peerOrg2
      - FABRIC_CA_SERVER_DEBUG=true
      - TARGET_CERTFILE=/data/peerOrg2-ca-cert.pem
    volumes:
      - ./scripts:/scripts
      - ./data:/data
```

待root ca服务运行正常后，使用命令`docker-compose -f docker-compose-inter-ca.yaml up -d`来启动各组织的中间ca和生成证书用的容器。对应yaml：

```yaml
version: '2'

services:

  ica-ordererOrg:
    container_name: ica-ordererOrg
    image: hyperledger/fabric-ca
    command: /bin/bash -c '/scripts/start-intermediate-ca.sh ordererOrg 2>&1'
    environment:
      - FABRIC_CA_SERVER_HOME=/etc/hyperledger/fabric-ca
      - FABRIC_CA_SERVER_CA_NAME=ica-ordererOrg
      - FABRIC_CA_SERVER_INTERMEDIATE_TLS_CERTFILES=/data/ordererOrg-ca-cert.pem
      - FABRIC_CA_SERVER_CSR_HOSTS=ica-ordererOrg
      - FABRIC_CA_SERVER_TLS_ENABLED=true
      - FABRIC_CA_SERVER_DEBUG=true
      - TARGET_CHAINFILE=/data/ordererOrg-ca-chain.pem
      - PARENT_URL=https://admin:adminpw@rca-ordererOrg:7054
    volumes:
      - ./scripts:/scripts
      - ./data:/data

  ica-peerOrg1:
    container_name: ica-peerOrg1
    image: hyperledger/fabric-ca
    command: /bin/bash -c '/scripts/start-intermediate-ca.sh peerOrg1 2>&1'
    environment:
      - FABRIC_CA_SERVER_HOME=/etc/hyperledger/fabric-ca
      - FABRIC_CA_SERVER_CA_NAME=ica-peerOrg1
      - FABRIC_CA_SERVER_INTERMEDIATE_TLS_CERTFILES=/data/peerOrg1-ca-cert.pem
      - FABRIC_CA_SERVER_CSR_HOSTS=ica-peerOrg1
      - FABRIC_CA_SERVER_TLS_ENABLED=true
      - FABRIC_CA_SERVER_DEBUG=true
      - TARGET_CHAINFILE=/data/peerOrg1-ca-chain.pem
      - PARENT_URL=https://admin:adminpw@rca-peerOrg1:7054
    volumes:
      - ./scripts:/scripts
      - ./data:/data

  ica-peerOrg2:
    container_name: ica-peerOrg2
    image: hyperledger/fabric-ca
    command: /bin/bash -c '/scripts/start-intermediate-ca.sh peerOrg2 2>&1'
    environment:
      - FABRIC_CA_SERVER_HOME=/etc/hyperledger/fabric-ca
      - FABRIC_CA_SERVER_CA_NAME=ica-peerOrg2
      - FABRIC_CA_SERVER_INTERMEDIATE_TLS_CERTFILES=/data/peerOrg2-ca-cert.pem
      - FABRIC_CA_SERVER_CSR_HOSTS=ica-peerOrg2
      - FABRIC_CA_SERVER_TLS_ENABLED=true
      - FABRIC_CA_SERVER_DEBUG=true
      - TARGET_CHAINFILE=/data/peerOrg2-ca-chain.pem
      - PARENT_URL=https://admin:adminpw@rca-peerOrg2:7054
    volumes:
      - ./scripts:/scripts
      - ./data:/data

  setup:
    container_name: setup
    image: hyperledger/fabric-ca-tools
    command: /bin/bash -c '/scripts/setup.sh;sleep 99999'
    environment:
      - FABRIC_CFG_PATH=/etc/hyperledger/fabric
    volumes:
      - ./scripts:/scripts
      - ./data:/data
    depends_on:
      - ica-ordererOrg
      - ica-peerOrg1
      - ica-peerOrg2
```

> 由于使用两个文件，docker-compose会产生警告，可忽略。
>
> `WARNING: Found orphan containers (rca-peerOrg2, rca-ordererOrg, rca-peerOrg1) for this project. If you removed or renamed this service in your compose file, you can run this command with the --remove-orphans flag to clean it up`

在`setup`容器中，通过`fabric-ca-client`与中间ca交互，生成各组织的msp证书和tls证书。

###  2.1.1 生成orderer组织公私钥

在`setup`容器中，执行下列命令，以管理员身份登录到ordererOrg的中间ca服务器。

> 使用fabric-ca中间根服务器的根证书链作为tls通信证书。

```bash
export FABRIC_CA_CLIENT_HOME=$HOME/cas/ica-ordererOrg
export FABRIC_CA_CLIENT_TLS_CERTFILES=/data/ordererOrg-ca-chain.pem
fabric-ca-client enroll -d -u https://admin:adminpw@ica-ordererOrg:7054
```

产生如下输出，表示生成成功。

![1526268834325](build_fabric_network_with_fabric_ca.assets/1526268834325.png)

会在目录`$HOME/cas/ica-ordererOrg`生成下列文件，代表管理员的身份信息。

![1526268934509](build_fabric_network_with_fabric_ca.assets/1526268934509.png)

- cacerts：根ca服务器的根证书
- intermediatecerts：中间根服务器`ica-ordererOrg`的根证书
- keystone：签发给登录用户的证书私钥
- signcerts：签发给登录用户的证书公钥

1）注册ordererOrg的节点用户和管理员用户。

使用如下命令，注册orderer节点用户，**用户名与将要启动的容器名、tls公私钥请求的主机名保持一致**

`fabric-ca-client register -d --id.name orderer --id.secret password --id.type orderer`

产生如下输出表示用户注册成功，如果不指定`--id.secret`参数，则fabric ca会生成随机密码。

![1526269478773](build_fabric_network_with_fabric_ca.assets/1526269478773.png)

使用如下命令，注册ordererOrg的管理员用户。

`fabric-ca-client register -d --id.name admin-ordererOrg --id.secret password --id.attrs "admin=true:ecert"`

2）生成ordererOrg的普通用户和管理员用户的证书

使用如下命令，将根证书和中间根证书下载到ordererOrg组织的msp目录

```shell
export ORG_MSP_DIR=/data/orgs/ordererOrg/msp
export FABRIC_CA_CLIENT_TLS_CERTFILES=/data/ordererOrg-ca-chain.pem
fabric-ca-client getcacert -d -u https://ica-ordererOrg:7054 -M $ORG_MSP_DIR
```

使用如下命令，为组织准备tls证书和tls中间根证书，用作tls通信使用。

> tls所需证书无法自动生成，需要手动复制创建。后面的所有用户操作与此类似。

```shell
cp -r /data/orgs/ordererOrg/msp/cacerts/ /data/orgs/ordererOrg/msp/tlscacerts
cp -r /data/orgs/ordererOrg/msp/intermediatecerts /data/orgs/ordererOrg/msp/tlsintermediatecerts 
```

使用如下命令，切换到ordererOrg的管理员用户，并生成对应证书到msp目录。

> admincerts目录需要手动生成，复制到组织和admin用户的admincerts目录。后面的所有用户操作与此类似

```shell
export FABRIC_CA_CLIENT_HOME=/data/orgs/ordererOrg/admin
export FABRIC_CA_CLIENT_TLS_CERTFILES=/data/ordererOrg-ca-chain.pem
fabric-ca-client enroll -d -u https://admin-ordererOrg:password@ica-ordererOrg:7054
cp -r /data/orgs/ordererOrg/admin/msp/signcerts /data/orgs/ordererOrg/msp/admincerts
cp -r /data/orgs/ordererOrg/admin/msp/signcerts /data/orgs/ordererOrg/admin/msp/admincerts
```

3）生成orderer节点启动需要的tls公私钥

使用如下命令，生成orderer节点的tls公私钥

> 注意登录命令，切换为了orderer节点用户。csr.hosts注意填写为orderer容器的容器名称。

```shell
rm -rf /tmp/tls
export FABRIC_CA_CLIENT_TLS_CERTFILES=/data/ordererOrg-ca-chain.pem

fabric-ca-client enroll -d --enrollment.profile tls -u https://orderer:password@ica-ordererOrg:7054 -M /tmp/tls --csr.hosts orderer
mkdir -p /data/orgs/ordererOrg/orderer/tls

cp /tmp/tls/keystore/* /data/orgs/ordererOrg/orderer/tls/server.key
cp /tmp/tls/signcerts/* /data/orgs/ordererOrg/orderer/tls/server.crt
```

生成orderer节点用户的msp公私钥，用于启动orderer进程，命令如下所示。

```bash
export FABRIC_CA_CLIENT_HOME=/data/orgs/ordererOrg/orderer
fabric-ca-client enroll -d -u https://orderer:password@ica-ordererOrg:7054

cp -r /data/orgs/ordererOrg/orderer/msp/cacerts/ /data/orgs/ordererOrg/orderer/msp/tlscacerts 
cp -r /data/orgs/ordererOrg/orderer/msp/intermediatecerts /data/orgs/ordererOrg/orderer/msp/tlsintermediatecerts

cp -r /data/orgs/ordererOrg/msp/admincerts /data/orgs/ordererOrg/orderer/msp
```



最终，生成的ordererOrg的公私钥目录结构如下图所示：

![1526352826684](build_fabric_network_with_fabric_ca.assets/1526352826684.png)

### 2.1.2 生成peer组织公私钥

#### 生成peerOrg1组织公私钥

在`setup`容器中，执行下列命令，以管理员身份登录到peerOrg1的中间ca服务器。

```bash
export FABRIC_CA_CLIENT_HOME=$HOME/cas/ica-peerOrg1
export FABRIC_CA_CLIENT_TLS_CERTFILES=/data/peerOrg1-ca-chain.pem
fabric-ca-client enroll -d -u https://admin:adminpw@ica-peerOrg1:7054
```

1）注册peerOrg1组织的节点用户、普通用户和管理员用户

使用下列命令注册两个节点用户，普通用户和组织的管理员用户。

> peer节点用户的名称最好与将要启动的容器名称保持一致，便于识别处理。

```shell
fabric-ca-client register -d --id.name peer0-peerOrg1 --id.secret password --id.type peer
fabric-ca-client register -d --id.name peer1-peerOrg1 --id.secret password --id.type peer
fabric-ca-client register -d --id.name admin-peerOrg1 --id.secret password --id.attrs "hf.Registrar.Roles=client,hf.Registrar.Attributes=*,hf.Revoker=true,hf.GenCRL=true,admin=true:ecert,abac.init=true:ecert"
fabric-ca-client register -d --id.name user-peerOrg1 --id.secret password
```

2）生成peerOrg1组织的相应公私钥证书和msp目录

使用如下命令，将根证书和中间根证书下载到peerOrg1的msp目录。

```shell
export ORG_MSP_DIR=/data/orgs/peerOrg1/msp
export FABRIC_CA_CLIENT_TLS_CERTFILES=/data/peerOrg1-ca-chain.pem
fabric-ca-client getcacert -d -u https://ica-peerOrg1:7054 -M $ORG_MSP_DIR
```

使用如下命令，为peerOrg1组织准备tls证书和tls中间根证书，用作tls通信使用。

```shell
cp /data/orgs/peerOrg1/msp/cacerts/ /data/orgs/peerOrg1/msp/tlscacerts -r 
cp /data/orgs/peerOrg1/msp/intermediatecerts /data/orgs/peerOrg1/msp/tlsintermediatecerts -r
```

使用如下命令，切换到peerOrg1的管理员用户，并生成对应证书到msp目录。

```shell
export FABRIC_CA_CLIENT_HOME=/data/orgs/peerOrg1/admin
export FABRIC_CA_CLIENT_TLS_CERTFILES=/data/peerOrg1-ca-chain.pem
fabric-ca-client enroll -d -u https://admin-peerOrg1:password@ica-peerOrg1:7054
cp -r /data/orgs/peerOrg1/admin/msp/signcerts /data/orgs/peerOrg1/msp/admincerts
cp -r /data/orgs/peerOrg1/admin/msp/signcerts /data/orgs/peerOrg1/admin/msp/admincerts
```

3）生成peerOrg1的节点的公私钥和tls公私钥

启动peer节点时，需要生成对应的MSP公私钥和tls证书，生成公私钥过程中的节点命令需要与启动的容器名称保持一致。约定peerOrg1的两个节点为peer0-peerOrg1、peer1-peerOrg1。

使用下列命令，生成peer0-peerOrg1、peer1-peerOrg1节点的tls公私钥和msp目录

```shell
export FABRIC_CA_CLIENT_TLS_CERTFILES=/data/peerOrg1-ca-chain.pem

#使用节点用户peer0-peerOrg1，准备节点的tls服务公私钥
rm -rf /tmp/tls
mkdir -p /data/orgs/peerOrg1/peer0/tls
fabric-ca-client enroll -d --enrollment.profile tls -u https://peer0-peerOrg1:password@ica-peerOrg1:7054 -M /tmp/tls --csr.hosts peer0-peerOrg1
cp /tmp/tls/keystore/* /data/orgs/peerOrg1/peer0/tls/server.key
cp /tmp/tls/signcerts/* /data/orgs/peerOrg1/peer0/tls/server.crt
#节点用户peer0-peerOrg1的msp目录准备
export FABRIC_CA_CLIENT_HOME=/data/orgs/peerOrg1/peer0
fabric-ca-client enroll -d -u https://peer0-peerOrg1:password@ica-peerOrg1:7054
cp /data/orgs/peerOrg1/peer0/msp/cacerts/ /data/orgs/peerOrg1/peer0/msp/tlscacerts -r 
cp /data/orgs/peerOrg1/peer0/msp/intermediatecerts /data/orgs/peerOrg1/peer0/msp/tlsintermediatecerts -r
#peer节点使用管理员用户的证书
cp -r /data/orgs/peerOrg1/msp/admincerts /data/orgs/peerOrg1/peer0/msp

#使用节点用户peer1-peerOrg1，准备节点的tls服务公私钥
rm -rf /tmp/tls
mkdir -p /data/orgs/peerOrg1/peer1/tls
fabric-ca-client enroll -d --enrollment.profile tls -u https://peer1-peerOrg1:password@ica-peerOrg1:7054 -M /tmp/tls --csr.hosts peer1-peerOrg1
cp /tmp/tls/keystore/* /data/orgs/peerOrg1/peer1/tls/server.key
cp /tmp/tls/signcerts/* /data/orgs/peerOrg1/peer1/tls/server.crt
#节点用户peer0-peerOrg1的msp目录准备
export FABRIC_CA_CLIENT_HOME=/data/orgs/peerOrg1/peer1
fabric-ca-client enroll -d -u https://peer1-peerOrg1:password@ica-peerOrg1:7054
cp /data/orgs/peerOrg1/peer1/msp/cacerts/ /data/orgs/peerOrg1/peer1/msp/tlscacerts -r 
cp /data/orgs/peerOrg1/peer1/msp/intermediatecerts /data/orgs/peerOrg1/peer1/msp/tlsintermediatecerts -r
#peer节点使用管理员用户的证书
cp -r /data/orgs/peerOrg1/msp/admincerts /data/orgs/peerOrg1/peer1/msp
```

***可选）***如果peer节点开启了客户端请求的tls验证，使用如下命令生成客户端验证证书和命令行调用的证书。否则可忽略。

> 两份证书可以使用相同的文件，也可以使用不同的，只要是同一个ca服务签发的就可以

```bash
export FABRIC_CA_CLIENT_TLS_CERTFILES=/data/peerOrg1-ca-chain.pem
mkdir /data/tls -p

#生成客户端校验证书
rm -rf /tmp/tls
fabric-ca-client enroll -d --enrollment.profile tls -u https://peer0-peerOrg1:password@ica-peerOrg1:7054 -M /tmp/tls --csr.hosts peer0-peerOrg1
cp /tmp/tls/keystore/* /data/tls/peer0-peerOrg1-clientauth.key
cp /tmp/tls/signcerts/* /data//tls/peer0-peerOrg1-clientauth.crt

#生成命令行使用的证书
rm -rf /tmp/tls
fabric-ca-client enroll -d --enrollment.profile tls -u https://peer0-peerOrg1:password@ica-peerOrg1:7054 -M /tmp/tls --csr.hosts peer0-peerOrg1
cp /tmp/tls/keystore/* /data/tls/peer0-peerOrg1-cli.key
cp /tmp/tls/signcerts/* /data//tls/peer0-peerOrg1-cli.crt

rm -rf /tmp/tls
fabric-ca-client enroll -d --enrollment.profile tls -u https://peer1-peerOrg1:password@ica-peerOrg1:7054 -M /tmp/tls --csr.hosts peer1-peerOrg1
cp /tmp/tls/keystore/* /data/tls/peer1-peerOrg1-clientauth.key
cp /tmp/tls/signcerts/* /data//tls/peer1-peerOrg1-clientauth.crt

rm -rf /tmp/tls
fabric-ca-client enroll -d --enrollment.profile tls -u https://peer1-peerOrg1:password@ica-peerOrg1:7054 -M /tmp/tls --csr.hosts peer1-peerOrg1
cp /tmp/tls/keystore/* /data/tls/peer1-peerOrg1-cli.key
cp /tmp/tls/signcerts/* /data//tls/peer1-peerOrg1-cli.crt
```

4）生成普通用户的公私钥和tls公私钥

使用如下命令，生成peerOrg1组织中用户user-peerOrg1的公私钥

> 普通用户使用自己的证书作为管理使用

```shell
export FABRIC_CA_CLIENT_TLS_CERTFILES=/data/peerOrg1-ca-chain.pem
export FABRIC_CA_CLIENT_HOME=/data/orgs/peerOrg1/user-peerOrg1
fabric-ca-client enroll -d -u https://user-peerOrg1:password@ica-peerOrg1:7054
cp /data/orgs/peerOrg1/user-peerOrg1/msp/cacerts/ /data/orgs/peerOrg1/user-peerOrg1/msp/tlscacerts -r 
cp /data/orgs/peerOrg1/user-peerOrg1/msp/intermediatecerts /data/orgs/peerOrg1/user-peerOrg1/msp/tlsintermediatecerts -r
cp -r /data/orgs/peerOrg1/user-peerOrg1/msp/signcerts /data/orgs/peerOrg1/user-peerOrg1/msp/admincerts
```

最终peerOrg1生成的公私钥目录结构如下图所示：

![1526287832731](build_fabric_network_with_fabric_ca.assets/1526287832731.png)

#### 生成peerOrg2组织公私钥

peerOrg2的相关操作与之类似，相应命令更改为对应组织名称即可。

登录中间ca管理员的命令如下：

```bash
export FABRIC_CA_CLIENT_HOME=$HOME/cas/ica-peerOrg2
export FABRIC_CA_CLIENT_TLS_CERTFILES=/data/peerOrg2-ca-chain.pem
fabric-ca-client enroll -d -u https://admin:adminpw@ica-peerOrg2:7054
```

1）注册peerOrg2组织的节点用户、普通用户和管理员用户的命令

```bash
fabric-ca-client register -d --id.name peer0-peerOrg2 --id.secret password --id.type peer
fabric-ca-client register -d --id.name peer1-peerOrg2 --id.secret password --id.type peer
fabric-ca-client register -d --id.name admin-peerOrg2 --id.secret password --id.attrs "hf.Registrar.Roles=client,hf.Registrar.Attributes=*,hf.Revoker=true,hf.GenCRL=true,admin=true:ecert,abac.init=true:ecert"
fabric-ca-client register -d --id.name user-peerOrg2 --id.secret password
```

2）准备peerOrg2组织公私钥

使用如下命令，将根证书和中间根证书下载到peerOrg1的msp目录。

```shell
export ORG_MSP_DIR=/data/orgs/peerOrg2/msp
export FABRIC_CA_CLIENT_TLS_CERTFILES=/data/peerOrg2-ca-chain.pem
fabric-ca-client getcacert -d -u https://ica-peerOrg2:7054 -M $ORG_MSP_DIR
```

使用如下命令，为peerOrg2组织准备tls证书和tls中间根证书，用作tls通信使用。

```shell
cp /data/orgs/peerOrg2/msp/cacerts/ /data/orgs/peerOrg2/msp/tlscacerts -r 
cp /data/orgs/peerOrg2/msp/intermediatecerts /data/orgs/peerOrg2/msp/tlsintermediatecerts -r
```

使用如下命令，切换到peerOrg2的管理员用户，并生成对应证书到msp目录。

```shell
export FABRIC_CA_CLIENT_HOME=/data/orgs/peerOrg2/admin
export FABRIC_CA_CLIENT_TLS_CERTFILES=/data/peerOrg2-ca-chain.pem
fabric-ca-client enroll -d -u https://admin-peerOrg2:password@ica-peerOrg2:7054
cp -r /data/orgs/peerOrg2/admin/msp/signcerts /data/orgs/peerOrg2/msp/admincerts
cp -r /data/orgs/peerOrg2/admin/msp/signcerts /data/orgs/peerOrg2/admin/msp/admincerts
```



3）生成peerOrg2各节点的公私钥命令

```shell
export FABRIC_CA_CLIENT_TLS_CERTFILES=/data/peerOrg2-ca-chain.pem

#使用节点用户peer0-peerOrg2
rm -rf /tmp/tls
mkdir -p /data/orgs/peerOrg2/peer0/tls
fabric-ca-client enroll -d --enrollment.profile tls -u https://peer0-peerOrg2:password@ica-peerOrg2:7054 -M /tmp/tls --csr.hosts peer0-peerOrg2
cp /tmp/tls/keystore/* /data/orgs/peerOrg2/peer0/tls/server.key
cp /tmp/tls/signcerts/* /data/orgs/peerOrg2/peer0/tls/server.crt
#节点用户peer0-peerOrg2的msp目录准备
export FABRIC_CA_CLIENT_HOME=/data/orgs/peerOrg2/peer0
fabric-ca-client enroll -d -u https://peer0-peerOrg2:password@ica-peerOrg2:7054
cp /data/orgs/peerOrg2/peer0/msp/cacerts/ /data/orgs/peerOrg2/peer0/msp/tlscacerts -r 
cp /data/orgs/peerOrg2/peer0/msp/intermediatecerts /data/orgs/peerOrg2/peer0/msp/tlsintermediatecerts -r
#peer节点使用管理员用户的证书
cp -r /data/orgs/peerOrg2/msp/admincerts /data/orgs/peerOrg2/peer0/msp

#使用节点用户peer1-peerOrg2
rm -rf /tmp/tls
mkdir -p /data/orgs/peerOrg2/peer1/tls
fabric-ca-client enroll -d --enrollment.profile tls -u https://peer1-peerOrg2:password@ica-peerOrg2:7054 -M /tmp/tls --csr.hosts peer1-peerOrg2
cp /tmp/tls/keystore/* /data/orgs/peerOrg2/peer1/tls/server.key
cp /tmp/tls/signcerts/* /data/orgs/peerOrg2/peer1/tls/server.crt
#节点用户peer0-peerOrg2的msp目录准备
export FABRIC_CA_CLIENT_HOME=/data/orgs/peerOrg2/peer1
fabric-ca-client enroll -d -u https://peer1-peerOrg2:password@ica-peerOrg2:7054
cp /data/orgs/peerOrg2/peer1/msp/cacerts/ /data/orgs/peerOrg2/peer1/msp/tlscacerts -r 
cp /data/orgs/peerOrg2/peer1/msp/intermediatecerts /data/orgs/peerOrg2/peer1/msp/tlsintermediatecerts -r
#peer节点使用管理员用户的证书
cp -r /data/orgs/peerOrg2/msp/admincerts /data/orgs/peerOrg2/peer1/msp
```

生成peer节点的客户端校验证书和命令行证书命令。

```shell
export FABRIC_CA_CLIENT_TLS_CERTFILES=/data/peerOrg2-ca-chain.pem
mkdir /data/tls -p

#peer链接orderer使用的tls公私钥
rm -rf /tmp/tls
fabric-ca-client enroll -d --enrollment.profile tls -u https://peer0-peerOrg2:password@ica-peerOrg2:7054 -M /tmp/tls --csr.hosts peer0-peerOrg2
cp /tmp/tls/keystore/* /data/tls/peer0-peerOrg2-clientauth.key
cp /tmp/tls/signcerts/* /data//tls/peer0-peerOrg2-clientauth.crt

#命令行连接orderer使用tls公私钥
rm -rf /tmp/tls
fabric-ca-client enroll -d --enrollment.profile tls -u https://peer0-peerOrg2:password@ica-peerOrg2:7054 -M /tmp/tls --csr.hosts peer0-peerOrg2
cp /tmp/tls/keystore/* /data/tls/peer0-peerOrg2-cli.key
cp /tmp/tls/signcerts/* /data//tls/peer0-peerOrg2-cli.crt

rm -rf /tmp/tls
fabric-ca-client enroll -d --enrollment.profile tls -u https://peer1-peerOrg2:password@ica-peerOrg2:7054 -M /tmp/tls --csr.hosts peer1-peerOrg2
cp /tmp/tls/keystore/* /data/tls/peer1-peerOrg2-clientauth.key
cp /tmp/tls/signcerts/* /data//tls/peer1-peerOrg2-clientauth.crt

rm -rf /tmp/tls
fabric-ca-client enroll -d --enrollment.profile tls -u https://peer1-peerOrg2:password@ica-peerOrg2:7054 -M /tmp/tls --csr.hosts peer1-peerOrg2
cp /tmp/tls/keystore/* /data/tls/peer1-peerOrg2-cli.key
cp /tmp/tls/signcerts/* /data//tls/peer1-peerOrg2-cli.crt
```



4）生成peerOrg2普通用户的公私钥命令

> 普通用户使用自己的证书作为管理使用

```shell
export FABRIC_CA_CLIENT_TLS_CERTFILES=/data/peerOrg2-ca-chain.pem
export FABRIC_CA_CLIENT_HOME=/data/orgs/peerOrg2/user-peerOrg2
fabric-ca-client enroll -d -u https://user-peerOrg2:password@ica-peerOrg2:7054
cp /data/orgs/peerOrg2/user-peerOrg2/msp/cacerts/ /data/orgs/peerOrg2/user-peerOrg2/msp/tlscacerts -r 
cp /data/orgs/peerOrg2/user-peerOrg2/msp/intermediatecerts /data/orgs/peerOrg2/user-peerOrg2/msp/tlsintermediatecerts -r
cp -r /data/orgs/peerOrg2/user-peerOrg2/msp/signcerts /data/orgs/peerOrg2/user-peerOrg2/msp/admincerts
```

### 2.1.3 生成创世块和channel配置

完整所有组织和用户的公私钥准备后，就可以按照正常搭建fabric网络环境那样继续进行了。

使用如下`configtx.yaml`，创建创世块

```yaml
Profiles:

  OrgsOrdererGenesis:
  
    Orderer:
      OrdererType: solo
      
      Addresses:
        - orderer:7050

      BatchTimeout: 2s

      BatchSize:

        MaxMessageCount: 10

        AbsoluteMaxBytes: 99 MB

        PreferredMaxBytes: 512 KB

      Kafka:
        Brokers:
          - 127.0.0.1:9092

      Organizations:
        - *ordererOrg

    Consortiums:

      SampleConsortium:

        Organizations:
          - *peerOrg1
          - *peerOrg2

  OrgsChannel:
    Consortium: SampleConsortium
    Application:
      <<: *ApplicationDefaults
      Organizations:
        - *peerOrg1
        - *peerOrg2

Organizations:

  - &ordererOrg

    Name: ordererOrg

    ID: ordererOrgMSP

    MSPDir: /data/orgs/ordererOrg/msp

  - &peerOrg1

    Name: peerOrg1

    ID: peerOrg1MSP

    MSPDir: /data/orgs/peerOrg1/msp

    AnchorPeers:
       - Host: peer0-peerOrg1
         Port: 7051

  - &peerOrg2

    Name: peerOrg2

    ID: peerOrg2MSP

    MSPDir: /data/orgs/peerOrg2/msp

    AnchorPeers:
       - Host: peer0-peerOrg2
         Port: 7051

Application: &ApplicationDefaults
    Organizations:
```

在`setup`容器中执行命令，来生成创世块。

> 默认执行命令时，先从环境变量FABRIC_CFG_PATH指定目录查询，没有时在找当前目录的configtx.yaml文件

```shell
configtxgen -profile OrgsOrdererGenesis -outputBlock /data/genesis.block
```

生成channel的配置

`configtxgen -profile OrgsChannel -outputCreateChannelTx /data/channel.tx -channelID mychannel`

生成组织锚节点配置

```shell
configtxgen -profile OrgsChannel -outputAnchorPeersUpdate /data/peerOrg1Anchor.tx -channelID mychannel -asOrg peerOrg1
configtxgen -profile OrgsChannel -outputAnchorPeersUpdate /data/peerOrg2Anchor.tx -channelID mychannel -asOrg peerOrg2
```



### 2.1.4 启动fabric网络并进行基本测试

使用`docker-compose-cli.yaml`启动fabric网络，内容如下：

> 由于使用了多个yaml文件，docker-compose会产生如下告警，可忽略：
>
> `WARNING: Found orphan containers (setup, ica-peerOrg1, ica-ordererOrg, ica-peerOrg2, rca-ordererOrg, rca-peerOrg1, rca-peerOrg2) for this project. If you removed or renamed this service in your compose file, you can run this command with the --remove-orphans flag to clean it up.`

```yaml
version: '2'

services:

  orderer:
    container_name: orderer
    image: hyperledger/fabric-ca-orderer
    environment:
      - ORDERER_GENERAL_LISTENADDRESS=0.0.0.0
      - ORDERER_GENERAL_GENESISMETHOD=file
      - ORDERER_GENERAL_GENESISFILE=/data/genesis.block
      - ORDERER_GENERAL_LOCALMSPID=ordererOrgMSP
      - ORDERER_GENERAL_LOCALMSPDIR=/data/orgs/ordererOrg/orderer/msp
      - ORDERER_GENERAL_TLS_ENABLED=true
      - ORDERER_GENERAL_TLS_PRIVATEKEY=/data/orgs/ordererOrg/orderer/tls/server.key
      - ORDERER_GENERAL_TLS_CERTIFICATE=/data/orgs/ordererOrg/orderer/tls/server.crt
      - ORDERER_GENERAL_TLS_ROOTCAS=[/data/ordererOrg-ca-chain.pem]
      # 客户端发送请求需要tls证书验证
      #- ORDERER_GENERAL_TLS_CLIENTAUTHREQUIRED=true
      #- ORDERER_GENERAL_TLS_CLIENTROOTCAS=[/data/ordererOrg-ca-chain.pem]
      - ORDERER_GENERAL_LOGLEVEL=debug
      - ORDERER_DEBUG_BROADCASTTRACEDIR=data/logs
    command: orderer
    volumes:
      - ./data:/data

  peer0-peerOrg1:
    container_name: peer0-peerOrg1
    image: hyperledger/fabric-ca-peer
    environment:
      - CORE_PEER_ID=peer0-peerOrg1
      - CORE_PEER_ADDRESS=peer0-peerOrg1:7051
      - CORE_PEER_LOCALMSPID=peerOrg1MSP
      - CORE_PEER_MSPCONFIGPATH=/data/orgs/peerOrg1/peer0/msp
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      # 启动chaincode容器的网络
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=ca_default
      - CORE_LOGGING_LEVEL=DEBUG
      - CORE_PEER_TLS_ENABLED=true
      # peer服务自身启动使用的tls证书
      - CORE_PEER_TLS_CERT_FILE=/data/orgs/peerOrg1/peer0/tls/server.crt
      - CORE_PEER_TLS_KEY_FILE=/data/orgs/peerOrg1/peer0/tls/server.key
      - CORE_PEER_TLS_ROOTCERT_FILE=/data/peerOrg1-ca-chain.pem
      # 客户端请求的tls验证证书
      #- CORE_PEER_TLS_CLIENTAUTHREQUIRED=true
      #- CORE_PEER_TLS_CLIENTROOTCAS_FILES=/data/peerOrg1-ca-chain.pem
      #- CORE_PEER_TLS_CLIENTCERT_FILE=/data/tls/peer0-peerOrg1-clientauth.crt
      #- CORE_PEER_TLS_CLIENTKEY_FILE=/data/tls/peer0-peerOrg1-clientauth.key
      - CORE_PEER_GOSSIP_USELEADERELECTION=true
      - CORE_PEER_GOSSIP_ORGLEADER=false
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0-peerOrg1:7051
      - CORE_PEER_GOSSIP_SKIPHANDSHAKE=true
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: peer node start
    volumes:
      - ./data:/data
      - /var/run:/host/var/run

  peer1-peerOrg1:
    container_name: peer1-peerOrg1
    image: hyperledger/fabric-ca-peer
    environment:
      - CORE_PEER_ID=peer1-peerOrg1
      - CORE_PEER_ADDRESS=peer1-peerOrg1:7051
      - CORE_PEER_LOCALMSPID=peerOrg1MSP
      - CORE_PEER_MSPCONFIGPATH=/data/orgs/peerOrg1/peer1/msp
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=ca_default
      - CORE_LOGGING_LEVEL=DEBUG
      - CORE_PEER_TLS_ENABLED=true
      - CORE_PEER_TLS_CERT_FILE=/data/orgs/peerOrg1/peer1/tls/server.crt
      - CORE_PEER_TLS_KEY_FILE=/data/orgs/peerOrg1/peer1/tls/server.key
      - CORE_PEER_TLS_ROOTCERT_FILE=/data/peerOrg1-ca-chain.pem
      #- CORE_PEER_TLS_CLIENTAUTHREQUIRED=true
      #- CORE_PEER_TLS_CLIENTROOTCAS_FILES=/data/peerOrg1-ca-chain.pem
      #- CORE_PEER_TLS_CLIENTCERT_FILE=/data/tls/peer1-peerOrg1-clientauth.crt
      #- CORE_PEER_TLS_CLIENTKEY_FILE=/data/tls/peer1-peerOrg1-clientauth.key
      - CORE_PEER_GOSSIP_USELEADERELECTION=true
      - CORE_PEER_GOSSIP_ORGLEADER=false
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer1-peerOrg1:7051
      - CORE_PEER_GOSSIP_SKIPHANDSHAKE=true
      # 其他节点先与锚节点建立gossip通道
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer0-peerOrg1:7051
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: peer node start
    #command: sleep 99999
    volumes:
      - ./data:/data
      - /var/run:/host/var/run

  peer0-peerOrg2:
    container_name: peer0-peerOrg2
    image: hyperledger/fabric-ca-peer
    environment:
      - CORE_PEER_ID=peer0-peerOrg2
      - CORE_PEER_ADDRESS=peer0-peerOrg2:7051
      - CORE_PEER_LOCALMSPID=peerOrg2MSP
      - CORE_PEER_MSPCONFIGPATH=/data/orgs/peerOrg2/peer0/msp
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=ca_default
      - CORE_LOGGING_LEVEL=DEBUG
      - CORE_PEER_TLS_ENABLED=true
      - CORE_PEER_TLS_CERT_FILE=/data/orgs/peerOrg2/peer0/tls/server.crt
      - CORE_PEER_TLS_KEY_FILE=/data/orgs/peerOrg2/peer0/tls/server.key
      - CORE_PEER_TLS_ROOTCERT_FILE=/data/peerOrg2-ca-chain.pem
      #- CORE_PEER_TLS_CLIENTAUTHREQUIRED=true
      #- CORE_PEER_TLS_CLIENTROOTCAS_FILES=/data/peerOrg2-ca-chain.pem
      #- CORE_PEER_TLS_CLIENTCERT_FILE=/data/tls/peer0-peerOrg2-clientauth.crt
      #- CORE_PEER_TLS_CLIENTKEY_FILE=/data/tls/peer0-peerOrg2-clientauth.key
      - CORE_PEER_GOSSIP_USELEADERELECTION=true
      - CORE_PEER_GOSSIP_ORGLEADER=false
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0-peerOrg2:7051
      - CORE_PEER_GOSSIP_SKIPHANDSHAKE=true
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: peer node start
    volumes:
      - ./data:/data
      - /var/run:/host/var/run

  peer1-peerOrg2:
    container_name: peer1-peerOrg2
    image: hyperledger/fabric-ca-peer
    environment:
      - CORE_PEER_ID=peer1-peerOrg2
      - CORE_PEER_ADDRESS=peer1-peerOrg2:7051
      - CORE_PEER_LOCALMSPID=peerOrg2MSP
      - CORE_PEER_MSPCONFIGPATH=/data/orgs/peerOrg2/peer1/msp
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=ca_default
      - CORE_LOGGING_LEVEL=DEBUG
      - CORE_PEER_TLS_ENABLED=true
      - CORE_PEER_TLS_CERT_FILE=/data/orgs/peerOrg2/peer1/tls/server.crt
      - CORE_PEER_TLS_KEY_FILE=/data/orgs/peerOrg2/peer1/tls/server.key
      - CORE_PEER_TLS_ROOTCERT_FILE=/data/peerOrg2-ca-chain.pem
      #- CORE_PEER_TLS_CLIENTAUTHREQUIRED=true
      #- CORE_PEER_TLS_CLIENTROOTCAS_FILES=/data/peerOrg2-ca-chain.pem
      #- CORE_PEER_TLS_CLIENTCERT_FILE=/data/tls/peer1-peerOrg2-clientauth.crt
      #- CORE_PEER_TLS_CLIENTKEY_FILE=/data/tls/peer1-peerOrg2-clientauth.key
      - CORE_PEER_GOSSIP_USELEADERELECTION=true
      - CORE_PEER_GOSSIP_ORGLEADER=false
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer1-peerOrg2:7051
      - CORE_PEER_GOSSIP_SKIPHANDSHAKE=true
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer0-peerOrg2:7051
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: peer node start
    volumes:
      - ./scripts:/scripts
      - ./data:/data
      - /var/run:/host/var/run

  cli:
    container_name: cli
    image: hyperledger/fabric-ca-tools
    environment:
      - GOPATH=/opt/gopath
    command: /bin/bash -c '/scripts/prepare.sh;sleep 99999'
    volumes:
      - ./scripts:/scripts
      - ./data:/data
      - ./examples:/opt/gopath/src/github.com/
    depends_on:
      - orderer
      - peer0-peerOrg1
      - peer1-peerOrg1
      - peer0-peerOrg2
      - peer1-peerOrg2
```

#### 创建channel并加入peer

使用下面的命令，创建channel，并将所有peer加入到channel中。

```shell
#使用peerOrg1或者peerOrg2任意两个组织的管理员身份操作
export CORE_PEER_LOCALMSPID=peerOrg1MSP
export CORE_PEER_MSPCONFIGPATH=/data/orgs/peerOrg1/admin/msp
peer channel create --logging-level=DEBUG -c mychannel -f /data/channel.tx -o orderer:7050 --cafile /data/ordererOrg-ca-chain.pem --tls 

#设置tls通信根证书和要操作的peer节点
export CORE_PEER_ADDRESS=peer0-peerOrg1:7051
export CORE_PEER_TLS_ROOTCERT_FILE=/data/peerOrg1-ca-chain.pem
export CORE_PEER_TLS_ENABLED=true
peer channel join -b mychannel.block

#设置节点
export CORE_PEER_ADDRESS=peer1-peerOrg1:7051
peer channel join -b mychannel.block

#更新组织锚节点
peer channel update -c mychannel -f /data/peerOrg1Anchor.tx -o orderer:7050 --tls --cafile /data/ordererOrg-ca-chain.pem

#切换到peerOrg2管理员来操作
export CORE_PEER_LOCALMSPID=peerOrg2MSP
export CORE_PEER_MSPCONFIGPATH=/data/orgs/peerOrg2/admin/msp
export CORE_PEER_ADDRESS=peer0-peerOrg2:7051
export CORE_PEER_TLS_ROOTCERT_FILE=/data/peerOrg2-ca-chain.pem
export CORE_PEER_TLS_ENABLED=true
peer channel join -b mychannel.block

#设置节点
export CORE_PEER_ADDRESS=peer1-peerOrg2:7051
peer channel join -b mychannel.block

#更新组织锚节点
peer channel update -c mychannel -f /data/peerOrg2Anchor.tx -o orderer:7050 --tls --cafile /data/ordererOrg-ca-chain.pem
```

>如果要启用客户端tls验证，在所有命令上增加参数：
>
>--clientauth --keyfile /data/tls/peer0-peerOrg1-cli.key --certfile /data/tls/peer0-peerOrg1-cli.crt
>
>cafile：请求orderer使用的中间根证书
>
>keyfile：peer命令行使用的私钥
>
>certifle：peer命令行使用的公钥

#### 测试chaincode基本功能

使用如下命令，在peerOrg1和peerOrg2的第一个peer上上安装和实例化chaincode。

```shell
#在peer0-peerOrg1上安装和实例化chaincode
export CORE_PEER_LOCALMSPID=peerOrg1MSP
export CORE_PEER_MSPCONFIGPATH=/data/orgs/peerOrg1/admin/msp
export CORE_PEER_ADDRESS=peer0-peerOrg1:7051
export CORE_PEER_TLS_ROOTCERT_FILE=/data/peerOrg1-ca-chain.pem
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_ADDRESS=peer0-peerOrg1:7051
#安装chaincode
peer chaincode install -n mycc -v 1.0 -p github.com/chaincode/chaincode_example02/go/
#实例化
peer chaincode instantiate -o orderer:7050 --tls true --cafile /data/ordererOrg-ca-chain.pem -C mychannel -n mycc -v 1.0 -c '{"Args":["init","a","100","b","200"]}'

#转账
peer chaincode invoke -o orderer:7050 --tls true --cafile /data/ordererOrg-ca-chain.pem -C mychannel -n mycc -c '{"Args":["invoke","b","a","10"]}'

#分别再查询
peer chaincode query -C mychannel -n mycc -c '{"Args":["query","a"]}'
peer chaincode query -C mychannel -n mycc -c '{"Args":["query","b"]}'

#在peer0-peerOrg2上安装
export CORE_PEER_LOCALMSPID=peerOrg2MSP
export CORE_PEER_MSPCONFIGPATH=/data/orgs/peerOrg2/admin/msp
export CORE_PEER_ADDRESS=peer0-peerOrg2:7051
export CORE_PEER_TLS_ROOTCERT_FILE=/data/peerOrg2-ca-chain.pem
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_ADDRESS=peer0-peerOrg2:7051
#每个channel中，同名channel只能安装和实例化一次，无须再执行
#通过peer0-peerOrg2执行chaincode
peer chaincode invoke -o orderer:7050 --tls true --cafile /data/ordererOrg-ca-chain.pem -C mychannel -n mycc -c '{"Args":["invoke","b","a","10"]}'
#分别再查询
peer chaincode query -C mychannel -n mycc -c '{"Args":["query","a"]}'
peer chaincode query -C mychannel -n mycc -c '{"Args":["query","b"]}'
```



以上yaml可根据参考资料中的相关链接，使用yaml和相应脚本启动环境。手动生成组织公私钥的过程都已经通过脚本自动化了。

## 2.2 使用fabric-ca结合用户提供根证书

**目前，根据fabric-ca的官方文档，仅支持ECDSA算法的用户证书。**在这种场景下，fabric-ca作为中间ca服务为各组织提供PKI相关能力。用户需要提供下列文件：

- 颁发给fabric-ca的中间根证书、私钥
- 办法给fabric-ca服务的根证书链
- （可选），tls证书

指定的证书目前测试可以有两种方式生成：

1. 搭建一个fabric-ca服务器，使用fabric-ca-client的命令行，为三个组织生成中间ca服务器的证书，将证书分发给对应的中间ca服务器，进行启动。使用的示例命令如下：

   ```shell
   export FABRIC_CA_CLIENT_HOME=/root/ica-org0
   fabric-ca-client enroll -u https://rca-org0-admin:rca-org0-adminpw@rca-org0:7054  --tls.certfiles /etc/hyperledger/fabric-ca/tls-cert.pem --enrollment.profile ca
   ```

2. 借助fabric-ca中的[pki](https://github.com/hyperledger/fabric-ca/blob/release-1.1/scripts/fvt/utils/pki)工具脚本，生成中间ca服务器的证书，然后在进行启动，响应的示例命令如下：

   ```shell
   #生成ca根证书
   ./pki -f newca -a ca -t ec -l 256 -d sha256 \
                           -n "/C=US/ST=NC/L=RTP/O=IBM/O=Hyperledger/OU=FVT/CN=ca/" \
                           -K "digitalSignature,nonRepudiation,keyEncipherment,dataEncipherment,keyAgreement,keyCertSign,cRLSign" \
                           -E "serverAuth,clientAuth,codeSigning,emailProtection,timeStamping" \
                           -e 20370101000000Z -s 20160101000000Z -p ca-
   
   #生成org0组织的中间根证书
   ./pki -f newsub -b ica-org0 -a ca -t ec -l 256 -d sha256 \
                           -n "/C=US/ST=NC/L=RTP/O=IBM/O=Hyperledger/OU=FVT/CN=ica-org0/"  \
                           -K "digitalSignature,nonRepudiation,keyEncipherment,dataEncipherment,keyAgreement,keyCertSign,cRLSign" \
                           -E "serverAuth,clientAuth,codeSigning,emailProtection,timeStamping" \
                           -e 20370101000000Z -s 20160101000000Z -p ica-org0-
   
   
   ./pki -f newsub -b ica-org1 -a ca -t ec -l 256 -d sha256 \
                           -n "/C=US/ST=NC/L=RTP/O=IBM/O=Hyperledger/OU=FVT/CN=ica-org1/" \
                           -K "digitalSignature,nonRepudiation,keyEncipherment,dataEncipherment,keyAgreement,keyCertSign,cRLSign" \
                           -E "serverAuth,clientAuth,codeSigning,emailProtection,timeStamping" \
                           -e 20370101000000Z -s 20160101000000Z -p ica-org1-
   
   
   ./pki -f newsub -b ica-org2 -a ca -t ec -l 256 -d sha256 \
                           -n "/C=US/ST=NC/L=RTP/O=IBM/O=Hyperledger/OU=FVT/CN=ica-org2/" \
                           -K "digitalSignature,nonRepudiation,keyEncipherment,dataEncipherment,keyAgreement,keyCertSign,cRLSign" \
                           -E "serverAuth,clientAuth,codeSigning,emailProtection,timeStamping" \
                           -e 20370101000000Z -s 20160101000000Z -p ica-org2-
   ```

   然后根据中间根证书的目录结构，创建对应的文件，以生成org0中间ca服务器目录为例：

   ```shell
   mkdir -p ica-org0/msp/{cacerts,intermediatecerts,keystore,signcerts}
   cp $HOME/ica-org0-cert.pem ica-org0/ca-cert.pem
   cp $HOME/ica-org0-key.pem ica-org0/msp/keystore
   cp $HOME/ca-cert.pem ica-org0/msp/cacerts
   cp $HOME/ca-cert.pem ica-org0/msp/intermediate/certs
   cp $HOME/ica-org0-cert.pem ica-org0/msp/signcerts
   cat ica-org0/ca-certs.pem ica-org0/msp/cacerts/ca-certs.pem > ica-org0/ca-chaim.pem
   ```

   按照相同的方式，构造org1、org2的中间ca服务器目录。

测试使用fabric-ca，为三个组织的中间ca服务器生成公私钥和tls证书，然后修改[社区例子](https://github.com/swordboy/fabric-samples/tree/release-1.1/fabric-ca)，可以正常运行。

## 2.3 使用cryptogen生成的根证书

可以使用crypotgen预先生成的各组织的根证书和对应私钥作为启动条件，代表这个组织来管理用户，可参考网络资料[Fabric CA环境的集成](http://www.cnblogs.com/studyzy/p/7482451.html)进行操作验证。

# 3 fabric-ca功能测试

参考2.1或者2.2完成环境集成后，开始测试。

## 3.1 增加用户

在简单功能测试的基础上，以peerOrg1组织的管理员身份注册用户

```shell
export FABRIC_CA_CLIENT_HOME=$HOME/cas/ica-peerOrg1
export FABRIC_CA_CLIENT_TLS_CERTFILES=/data/peerOrg1-ca-chain.pem
fabric-ca-client enroll -d -u https://admin:adminpw@ica-peerOrg1:7054
fabric-ca-client register -d --id.name user2-peerOrg1 --id.secret password
```

切换至新创建的用户，生成msp目录。

```shell
rm -rf /tmp/user2
export FABRIC_CA_CLIENT_HOME=/tmp/user2
fabric-ca-client enroll -d -u https://user2-peerOrg1:password@ica-peerOrg1:7054
cp -r /tmp/user2/msp/cacerts /tmp/user2/msp/cacerts 
cp -r /tmp/user2/msp/cacerts/ /tmp/user2/msp/tlscacerts
cp -r /tmp/user2/msp/intermediatecerts /tmp/user2/msp/tlsintermediatecerts
cp -r /tmp/user2/msp/signcerts /tmp/user2/msp/admincerts
```

切换至新增用户，测试调用chaincode。

```shell
export CORE_PEER_LOCALMSPID=peerOrg1MSP
export CORE_PEER_MSPCONFIGPATH=/tmp/user2/msp
export CORE_PEER_ADDRESS=peer0-peerOrg1:7051
export CORE_PEER_TLS_ROOTCERT_FILE=/data/peerOrg1-ca-chain.pem
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_ADDRESS=peer0-peerOrg1:7051

#转账
peer chaincode invoke -o orderer:7050 --tls true --cafile /data/ordererOrg-ca-chain.pem -C mychannel -n mycc -c '{"Args":["invoke","b","a","10"]}'

#分别再查询
peer chaincode query -C mychannel -n mycc -c '{"Args":["query","a"]}'
peer chaincode query -C mychannel -n mycc -c '{"Args":["query","b"]}'
```

可以正常执行，说明新创建的用户身份有效。

## 3.2 ABAC验证

使用修改过的社区[ABAC](https://github.com/swordboy/fabric-samples/tree/release-1.1/chaincode/abac/go)例子。涉及ABAC权限的相关代码如下图，检查用户证书必须带有`abac.init=true:ecert`属性。

即前面给各个peer组织创建的管理员用户所带的自定义属性。

![1526451004951](build_fabric_network_with_fabric_ca.assets/1526451004951.png)

以peerOrg1组织的管理员`admin-peerOrg1`为例，其证书通过`openssl`查询，可见增加的扩展属性

![1526433100254](build_fabric_network_with_fabric_ca.assets/1526433100254.png)

使用如下的命令测试chaincode的abac校验，只允许管理员身份来实例化chaincode。

```shell
#在peer0-peerOrg1上安装和实例化chaincode
export CORE_PEER_LOCALMSPID=peerOrg1MSP
export CORE_PEER_MSPCONFIGPATH=/data/orgs/peerOrg1/admin/msp
export CORE_PEER_ADDRESS=peer0-peerOrg1:7051
export CORE_PEER_TLS_ROOTCERT_FILE=/data/peerOrg1-ca-chain.pem
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_ADDRESS=peer0-peerOrg1:7051
#安装chaincode，无须设置，系统限制只能管理员来安装chaincode
peer chaincode install -n abac -v 1.0 -p github.com/chaincode/abac/go/

#实例化，增加背书策略
peer chaincode instantiate -o orderer:7050 --tls true --cafile /data/ordererOrg-ca-chain.pem -C mychannel -n abac -v 1.0 -c '{"Args":["init","a","100","b","200"]}' -P 'OR('\''peerOrg1MSP.member'\'','\''peerOrg2MSP.member'\'')' 

#转账
peer chaincode invoke -o orderer:7050 --tls true --cafile /data/ordererOrg-ca-chain.pem -C mychannel -n abac -c '{"Args":["invoke","b","a","100"]}'

#分别再查询
peer chaincode query -C mychannel -n abac -c '{"Args":["query","a"]}'
#切换至用户身份，再转账
export CORE_PEER_MSPCONFIGPATH=/data/orgs/peerOrg1/user-peerOrg1/msp
#会因为用户证书不存在属性而失败Attribute 'abac.init' was not found
peer chaincode invoke -o orderer:7050 --tls true --cafile /data/ordererOrg-ca-chain.pem -C mychannel -n abac -c '{"Args":["invoke","b","a","1"]}'
#用户可以正常查询信息
peer chaincode query -C mychannel -n abac -c '{"Args":["query","b"]}'

#在peer0-peerOrg2上安装
export CORE_PEER_LOCALMSPID=peerOrg2MSP
export CORE_PEER_MSPCONFIGPATH=/data/orgs/peerOrg2/admin/msp
export CORE_PEER_ADDRESS=peer0-peerOrg2:7051
export CORE_PEER_TLS_ROOTCERT_FILE=/data/peerOrg2-ca-chain.pem
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_ADDRESS=peer0-peerOrg2:7051
#安装chaincode
peer chaincode install -n abac -v 1.0 -p github.com/chaincode/abac/go/
#每个channel中，同名chaincode实例化一次，无须再执行
#通过peer0-peerOrg2执行chaincode
peer chaincode invoke -o orderer:7050 --tls true --cafile /data/ordererOrg-ca-chain.pem -C mychannel -n abac -c '{"Args":["invoke","b","a","10"]}'
#分别再查询
peer chaincode query -C mychannel -n abac -c '{"Args":["query","a"]}'
#切换至用户身份再查询
export CORE_PEER_MSPCONFIGPATH=/data/orgs/peerOrg2/user-peerOrg2/msp
peer chaincode query -C mychannel -n abac -c '{"Args":["query","b"]}'
```



##3.3 吊销用户

在abac测试流程的基础上，增加吊销用户证书的流程。

吊销用户证书和动态增加组织一样，都需要通过configtxlator工具来辅助，同事需要使用中间ca服务器来生成CRL(CertificateRevocationList)，分发到各个组织。

吊销一个用户，需要经过以下流程。

1）生成CRL

```shell
#切换至管理员身份，生成CRL
export CORE_PEER_LOCALMSPID=peerOrg2MSP
export CORE_PEER_MSPCONFIGPATH=/data/orgs/peerOrg2/admin/msp
export CORE_PEER_ADDRESS=peer0-peerOrg2:7051
export CORE_PEER_TLS_ROOTCERT_FILE=/data/peerOrg2-ca-chain.pem
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_ADDRESS=peer0-peerOrg2:7051
#只想到管理员的msp目录
export FABRIC_CA_CLIENT_HOME=/data/orgs/peerOrg2/admin
export FABRIC_CA_CLIENT_TLS_CERTFILES=/data/peerOrg2-ca-chain.pem
#吊销用户，生成CRL，会在msp下生成crls/crl*.pem文件
fabric-ca-client revoke -d --revoke.name user-peerOrg2 --gencrl
```

命令执行完成后，在`$/msp/crls`目录下生成一个crl.pem文件。

![1526452053966](build_fabric_network_with_fabric_ca.assets/1526452053966.png)

2）修改channel配置

````shell
cd /data
#1.获取channel配置，生成一个protobuff格式的文件
peer channel fetch config mychannel_block.pb -c mychannel -o orderer:7050 --tls --cafile /data/ordererOrg-ca-chain.pem

#2.启动configtxlator，解析pb文件。会产生一个7059的监听服务
configtxlator start &
configtxlator_pid=$!

#3.转换channel配置的protobuff格式为json
curl -X POST --data-binary @mychannel_block.pb http://127.0.0.1:7059/protolator/decode/common.Block > mychannel_block.json

#4.修改配置json，增加吊销证书信息
jq .data.data[0].payload.data.config mychannel_block.json > config.json
crl=$(cat $CORE_PEER_MSPCONFIGPATH/crls/crl*.pem | base64 | tr -d '\n')
cat config.json | jq '.channel_group.groups.Application.groups.peerOrg2.values.MSP.value.config.revocation_list = ["'"${crl}"'"]' > updated_config.json

#5.生成配置块的更新diff protobuff文件
curl -X POST --data-binary @config.json http://127.0.0.1:7059/protolator/encode/common.Config > config.pb
curl -X POST --data-binary @updated_config.json http://127.0.0.1:7059/protolator/encode/common.Config > updated_config.pb
curl -X POST -F original=@config.pb -F updated=@updated_config.pb http://127.0.0.1:7059/configtxlator/compute/update-from-configs -F channel=mychannel > channel_update.pb

#6.转换配置块更新protobuff为json
curl -X POST --data-binary @channel_update.pb http://127.0.0.1:7059/protolator/decode/common.ConfigUpdate >channel_update.json

#7.生成peer channel update命令所需要的envelope protobuff文件
echo '{"payload":{"header":{"channel_header":{"channel_id":"mychannel", "type":2}},"data":{"config_update":'$(cat channel_update.json)'}}}' > channel_update_envelope.json
curl -X POST --data-binary @channel_update_envelope.json http://127.0.0.1:7059/protolator/encode/common.Envelope > channel_update_envelope.pb

kill -9 $configtxlator_pid

#8.更新channel配置
peer channel update -f channel_update_envelope.pb -c mychannel -o orderer:7050 --tls --cafile /data/ordererOrg-ca-chain.pem
````

3）验证用户已吊销

切换到用户身份，执行chaincode的查询命令。

```shell
export CORE_PEER_MSPCONFIGPATH=/data/orgs/peerOrg2/user-peerOrg2/msp
peer chaincode query -C mychannel -n abac -c '{"Args":["query","b"]}'
```

执行失败产生类似如图的异常

![1526452493919](build_fabric_network_with_fabric_ca.assets/1526452493919.png)

同时在peer0-peerOrg2上可见日志，说明证书已经吊销成功。

![1526452543934](build_fabric_network_with_fabric_ca.assets/1526452543934.png)

以用户身份，向ca中间服务器请求重新生成证书。

```shell
export FABRIC_CA_CLIENT_HOME=/tmp/user
fabric-ca-client enroll -d -u https://user-peerOrg2:password@ica-peerOrg2:7054
```

从ca中间服务器上可见日志，用户已吊销，无法重新生成证书。

![1526454067561](build_fabric_network_with_fabric_ca.assets/1526454067561.png)

#附录

## 证书目录结构

tls目录用于节点启动时，自身所使用的公私钥。

msp目录用于表示节点的身份。主要包括以下几个子目录

- admincerts：这个身份的管理员证书。对peer节点来说，admincerts应当是组织的管理员证书。其它case一般与signcerts是一样的。
- cacerts：组织的根证书
- keystore：这个身份的私钥，用来签名数据。
- （可选）intermediatecerts：组织的中间根证书
- signcerts：这个身份的公钥
- tlscacerts：组织tls通信的根证书
- （可选）tlsintermediatecerts：组织tls通信的中间根证书

`cryptogen`程序生成各个证书目录及其作用如下图所示：

![证书目录结构](build_fabric_network_with_fabric_ca.assets/msp_structure.png)

## 参考资料

1）在cryptogen生成的根证书和私钥，启动组织的fabric-ca服务，管理此组织的用户。

http://hyperledger-fabric-ca.readthedocs.io/en/latest/users-guide.html



2）社区的fabric-ca例子

https://github.com/hyperledger/fabric-samples/tree/release-1.1/fabric-ca

特点：

- 每个组织都有一个根ca和中间ca。

  原则上来说，从脚本执行来看，可以让所有组织都使用同一个ca来进行管理。

- 示例脚本自带验证用户的注册、登录、吊销证书等测试case。

3）修改过的社区fabric-ca例子

https://github.com/swordboy/fabric-samples/tree/release-1.1/fabric-ca

- 使用自定义fabric-ca服务器生成中间根证书、tls证书等文件。手动生成中间根证书的命令。

  `fabric-ca-server init -b admin:adminpw -u $PARENT_URL`

  即可在`FABRIC_CA_CLIENT_HOME`环境变量的目录得到准备好的中间根证书。

- 没有根ca服务器，直接使用中间ca服务器管理组织

4）手动搭建环境的yaml和配置

https://github.com/swordboy/fabric_examples/tree/master/ca

- fabric-ca的功能测试case

- 添加用户
- 吊销用户
- abac检查

