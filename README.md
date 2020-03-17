# Hyperledger Fabric meets Kubernetes
![Fabric Meets K8S](https://raft-fabric-kube.s3-eu-west-1.amazonaws.com/images/fabric_meets_k8s.png)

* [What is this?](#what-is-this)
* [Who made this?](#who-made-this)
* [License](#License)
* [Requirements](#requirements)
* [Network Architecture](#network-architecture)
* [Go over the samples](#go-over-samples)
  * [Launching the network](#launching-the-network)
  * [Creating channels](#creating-channels)
  * [Installing chaincodes](#installing-chaincodes)
  * [Scaled-up Kafka network](#scaled-up-kafka-network)
  * [Scaled-up Raft network with TLS](#scaled-up-raft-network-with-tls)
  * [Scaled-up Raft network without TLS](#scaled-up-raft-network-without-tls)
  * [Cross-cluster Raft network](#cross-cluster-raft-network)
  * [Adding new peer organizations](#adding-new-peer-organizations)
  * [Adding new peers to organizations](#adding-new-peers-to-organizations)
  * [Updating channel configuration](#updating-channel-configuration)
* [Configuration](#configuration)
* [TLS](#tls)
* [Backup-Restore](#backup-restore)
  * [Requirements](#backup-restore-requirements)
  * [Flow](#backup-restore-flow)
  * [Backup](#backup)
  * [Restore](#restore)
* [Limitations](#limitations)
* [FAQ and more](#faq-and-more)
* [Conclusion](#conclusion)

## [What is this?](#what-is-this)
This repository contains a couple of Helm charts to:
* Configure and launch the whole HL Fabric network or part of it, either:
  * A simple one, one peer per organization and Solo orderer
  * Or scaled up one, multiple peers per organization and Kafka or Raft orderer
* Populate the network declaratively:
  * Create the channels, join peers to channels, update channels for Anchor peers
  * Install/Instantiate all chaincodes, or some of them, or upgrade them to newer version
* Add new peer organizations to an already running network declaratively
* Make channel config updates declaratively
* Backup and restore the state of whole network

**IMPORTANT:** Declarative flows use our home built [CLI tools](https://hub.docker.com/u/raft) 
based on this [patch](https://github.com/hyperledger/fabric/pull/345), **use at your own risk!**
If you don't want this behaviour, you can use [release/0.7](https://github.com/APGGroeiFabriek/PIVT/tree/release/0.7) branch.

## [Who made this?](#who-made-this)
This work is a result of collaborative effort between [APG](https://www.apg.nl/en) and 
[Accenture NL](https://www.accenture.com/nl-en). 

We had implemented these Helm charts for our project's needs, and as the results looks very promising, 
decided to share the source code with the HL Fabric community. Hopefully it will fill a large gap!
Special thanks to APG for allowing opening the source code :)

We strongly encourage the HL Fabric community to take ownership of this repository, extend it for
further use cases, use it as a test bed and adapt it to the Fabric provided samples to get rid of endless 
Docker Compose files and Bash scripts. 

## [License](#License)
This work is licensed under the same license with HL Fabric; [Apache License 2.0](LICENSE).

## [Requirements](#requirements)
* A running Kubernetes cluster, Minikube should also work, but not tested
* [HL Fabric binaries](https://hyperledger-fabric.readthedocs.io/en/release-1.4/install.html)
* [Helm](https://github.com/helm/helm/releases/tag/v2.11.0), developed with 2.11, newer 2.xx versions should also work
* [jq](https://stedolan.github.io/jq/download/) 1.5+ and [yq](https://pypi.org/project/yq/) 2.6+
* [Argo](https://github.com/argoproj/argo), both CLI and Controller 2.4.0+
* [Minio](https://github.com/argoproj/argo/blob/master/docs/configure-artifact-repository.md), only required for backup/restore and new-peer-org flows
* Run all the commands in *fabric-kube* folder
* AWS EKS users please also apply this [fix](https://github.com/APGGroeiFabriek/PIVT/issues/1)

## [Network Architecture](#network-architecture)

### Simple Network Architecture

![Simple Network](https://raft-fabric-kube.s3-eu-west-1.amazonaws.com/images/HL_in_Kube_simple.png)

### Scaled Up Kafka Network Architecture

![Scaled Up Network](https://s3-eu-west-1.amazonaws.com/raft-fabric-kube/images/HL_in_Kube_scaled.png)

### Scaled Up Raft Network Architecture

![Scaled Up Raft Network](https://raft-fabric-kube.s3-eu-west-1.amazonaws.com/images/HL_in_Kube_raft.png)
**Note:** For transparent load balancing TLS should be disabled. This is only possible for Raft orderers since Fabric 1.4.5. See the [Scaled-up Raft network without TLS](#scaled-up-raft-network-without-tls) sample for details.

## [Go Over Samples](#go-over-samples)

### [Launching The Network](#launching-the-network)
First install chart dependencies, you need to do this only once:
```
helm repo add incubator http://storage.googleapis.com/kubernetes-charts-incubator
helm dependency update ./hlf-kube/
```
Then create necessary stuff:
```
./init.sh ./samples/simple/ ./samples/chaincode/
```
This script:
* Creates the `Genesis block` using `genesisProfile` defined in 
[network.yaml](fabric-kube/samples/simple/network.yaml) file in the project folder
* Creates crypto material using `cryptogen` based on 
[crypto-config.yaml](fabric-kube/samples/simple/crypto-config.yaml) file in the project folder
* Compresses chaincodes as `tar` archives via `prepare_chaincodes.sh` script
* Copies created stuff and configtx.yaml into main chart folder: `hlf-kube` 

Now, we are ready to launch the network:
```
helm install ./hlf-kube --name hlf-kube -f samples/simple/network.yaml -f samples/simple/crypto-config.yaml
```
This chart creates all the above mentioned secrets, pods, services, etc. cross configures them 
and launches the network in unpopulated state.

Wait for all pods are up and running:
```
kubectl get pod --watch
```
In a few seconds, pods will come up:
![Screenshot_pods](https://raft-fabric-kube.s3-eu-west-1.amazonaws.com/images/Screenshot_pods.png)
Congrulations you have a running HL Fabric network in Kubernetes!

### [Creating channels](#creating-channels)

Next lets create channels, join peers to channels and update channels for Anchor peers:
```
helm template channel-flow/ -f samples/simple/network.yaml -f samples/simple/crypto-config.yaml | argo submit - --watch
```
Wait for the flow to complete, finally you will see something like this:
![Screenshot_channel_flow](https://raft-fabric-kube.s3-eu-west-1.amazonaws.com/images/Screenshot_channel_flow_declarative.png)

Channel flow is declarative and idempotent. You can run it many times. It will create the channel only if it doesn't exist, join peers to channels only if they didn't join yet, etc.

### [Installing chaincodes](#installing-chaincodes)

Next lets install/instantiate/invoke chaincodes
```
helm template chaincode-flow/ -f samples/simple/network.yaml -f samples/simple/crypto-config.yaml | argo submit - --watch
```
Wait for the flow to complete, finally you will see something like this:
![Screenshot_chaincode_flow](https://raft-fabric-kube.s3-eu-west-1.amazonaws.com/images/Screenshot_chaincode_flow_declarative.png)

Install steps may fail even many times, nevermind about it, it's a known [Fabric bug](https://jira.hyperledger.org/browse/FAB-15026), 
the flow will retry it and eventually succeed.

Lets assume you had updated chaincodes and want to upgrade them in the Fabric network. Firt update chaincode `tar` archives:
```
./prepare_chaincodes.sh ./samples/simple/ ./samples/chaincode/
```
Then make sure chaincode ConfigMaps are updated with new chaincode tar archives:
```
helm upgrade hlf-kube ./hlf-kube -f samples/simple/network.yaml -f samples/simple/crypto-config.yaml  
```
Or alternatively you can update chaincode ConfigMaps directly:
```
helm template -f samples/simple/network.yaml -x templates/chaincode-configmap.yaml ./hlf-kube/ | kubectl apply -f -
```

Next invoke chaincode flow again:
```
helm template chaincode-flow/ -f samples/simple/network.yaml -f samples/simple/crypto-config.yaml --set chaincode.version=2.0 | argo submit - --watch
```
All chaincodes are upgraded to version 2.0!
![Screenshot_chaincode_upgade_all](https://raft-fabric-kube.s3-eu-west-1.amazonaws.com/images/Screenshot_chaincode_upgrade_all_declarative.png)

Lets upgrade only the chaincode named `very-simple` to version 3.0:
```
helm template chaincode-flow/ -f samples/simple/network.yaml -f samples/simple/crypto-config.yaml --set chaincode.version=3.0 --set flow.chaincode.include={very-simple} | argo submit - --watch
```
Chaincode `very-simple` is upgarded to version 3.0!
![Screenshot_chaincode_upgade_single](https://raft-fabric-kube.s3-eu-west-1.amazonaws.com/images/Screenshot_chaincode_upgrade_single_declarative.png)

Alternatively, you can also set chaincode versions individually via `network.chaincodes[].version`

Chaincode flow is declarative and idempotent. You can run it many times. It will install chaincodes only if not installed, instatiate them only if not instantiated yet, etc.

### [Scaled-up Kafka network](#scaled-up-kafka-network)
Now, lets launch a scaled up network backed by a Kafka cluster.

First tear down everything:
```
argo delete --all
helm delete hlf-kube --purge
```
Wait a bit until all pods are terminated:
```
kubectl  get pod --watch
```
Then create necessary stuff:
```
./init.sh ./samples/scaled-kafka/ ./samples/chaincode/
```
Lets launch our scaled up Fabric network:
```
helm install ./hlf-kube --name hlf-kube -f samples/scaled-kafka/network.yaml -f samples/scaled-kafka/crypto-config.yaml -f samples/scaled-kafka/values.yaml
```
Again lets wait for all pods are up and running:
```
kubectl get pod --watch
```
This time, in particular wait for 4 Kafka pods and 3 ZooKeeper pods are running and `ready` count is 1/1. 
Kafka pods may crash and restart a couple of times, this is normal as ZooKeeper pods are not ready yet, 
but eventually they will all come up.

![Screenshot_pods_kafka](https://s3-eu-west-1.amazonaws.com/raft-fabric-kube/images/Screenshot_pods_kafka.png)

Congrulations you have a running scaled up HL Fabric network in Kubernetes, with 3 Orderer nodes backed by a Kafka cluster 
and 2 peers per organization. Your application can use them without even noticing there are 3 Orderer nodes and 2 peers per organization.

Lets create the channels:
```
helm template channel-flow/ -f samples/scaled-kafka/network.yaml -f samples/scaled-kafka/crypto-config.yaml | argo submit - --watch
```
And install chaincodes:
```
helm template chaincode-flow/ -f samples/scaled-kafka/network.yaml -f samples/scaled-kafka/crypto-config.yaml | argo submit - --watch
```
### [Scaled-up Raft network with TLS](#scaled-up-raft-network-with-tls)
Now, lets launch a scaled up network based on three Raft orderer nodes spanning two Orderer organizations. This sample also demonstrates how to enable TLS and use actual domain names for peers and orderers instead of internal Kubernetes service names. Enabling TLS globally was mandatory until Fabric 1.4.5. See the [Scaled-up Raft network without TLS](#scaled-up-raft-network-without-tls) sample for running Raft orderers without globally enabling TLS.

Compare [scaled-raft-tls/configtx.yaml](fabric-kube/samples/scaled-raft-tls/configtx.yaml) with other samples, in particular it uses actual domain names like _peer0.atlantis.com_ instead of internal Kubernetes service names like _hlf-peer--atlantis--peer0_. This is necessary for enabling TLS since otherwise TLS certificates won't match service names.

Also in [network.yaml](fabric-kube/samples/scaled-raft-tls/network.yaml) file, there are two additional settings. As we pass this file to all Helm charts, it's convenient to put these settings into this file.
```
tlsEnabled: true
useActualDomains: true
```

First tear down everything:
```
argo delete --all
helm delete hlf-kube --purge
```
Wait a bit until all pods are terminated:
```
kubectl  get pod --watch
```
Then create necessary stuff:
```
./init.sh ./samples/scaled-raft-tls/ ./samples/chaincode/
```
Lets launch our Raft based Fabric network in _broken_ state:
```
helm install ./hlf-kube --name hlf-kube -f samples/scaled-raft-tls/network.yaml -f samples/scaled-raft-tls/crypto-config.yaml 
```
The pods will start but they cannot communicate to each other since domain names are unknown. You might also want to use the option `--set peer.launchPods=false --set orderer.launchPods=false` to make this process faster.

Run this command to collect the host aliases:
```
kubectl get svc -l addToHostAliases=true -o jsonpath='{"hostAliases:\n"}{range..items[*]}- ip: {.spec.clusterIP}{"\n"}  hostnames: [{.metadata.labels.fqdn}]{"\n"}{end}' > samples/scaled-raft-tls/hostAliases.yaml
```

Or this one, which is much convenient:
```
./collect_host_aliases.sh ./samples/scaled-raft-tls/ 
```

Let's check the created hostAliases.yaml file.
```
cat samples/scaled-raft-tls/hostAliases.yaml
```

The output will be something like:
```
hostAliases:
- ip: 10.0.110.93
  hostnames: [orderer0.groeifabriek.nl]
- ip: 10.0.32.65
  hostnames: [orderer1.groeifabriek.nl]
- ip: 10.0.13.191
  hostnames: [orderer0.pivt.nl]
- ip: 10.0.88.5
  hostnames: [peer0.atlantis.com]
- ip: 10.0.88.151
  hostnames: [peer1.atlantis.com]
- ip: 10.0.217.95
  hostnames: [peer10.aptalkarga.tr]
- ip: 10.0.252.19
  hostnames: [peer9.aptalkarga.tr]
- ip: 10.0.64.145
  hostnames: [peer0.nevergreen.nl]
- ip: 10.0.15.9
  hostnames: [peer1.nevergreen.nl]
```
The IPs are internal ClusterIPs of related services. Important point here is, as opposed to pod ClusterIPs, service ClusterIPs are stable, they won't change if service is not deleted and re-created.

Next, let's update the network with this host aliases information. These entries goes into pods' `/etc/hosts` file via Pod [hostAliases](https://kubernetes.io/docs/concepts/services-networking/add-entries-to-pod-etc-hosts-with-host-aliases/) spec.
```
helm upgrade hlf-kube ./hlf-kube -f samples/scaled-raft-tls/network.yaml -f samples/scaled-raft-tls/crypto-config.yaml -f samples/scaled-raft-tls/hostAliases.yaml  
```

Again lets wait for all pods are up and running:
```
kubectl get pod --watch
```
Congrulations you have a running scaled up HL Fabric network in Kubernetes, with 3 Raft orderer nodes spanning 2 Orderer organizations and 2 peers per organization. But unfortunately, due to TLS, your application cannot use them with transparent load balancing, you need to connect to relevant peer and orderer services separately.

Lets create the channels:
```
helm template channel-flow/ -f samples/scaled-raft-tls/network.yaml -f samples/scaled-raft-tls/crypto-config.yaml -f samples/scaled-raft-tls/hostAliases.yaml | argo submit - --watch
```
And install chaincodes:
```
helm template chaincode-flow/ -f samples/scaled-raft-tls/network.yaml -f samples/scaled-raft-tls/crypto-config.yaml -f samples/scaled-raft-tls/hostAliases.yaml | argo submit - --watch
```

### [Scaled-up Raft network without TLS](#scaled-up-raft-network-without-tls)

This sample has the same topology with scaled-up Raft network, but does not enable TLS globally. This is possible since Fabric 1.4.5. In the previous versions, enabling TLS globally was mandatory for Raft orderers.

You will notice the below entries in [network.yaml](fabric-kube/samples/scaled-raft-no-tls/network.yaml) file:
```
tlsEnabled: false
useActualDomains: true
```
TLS is disabled by default but still put in this file to be explicit. Using actual domain names is not mandatory but I believe is a good practice.

In [configtx.yaml](fabric-kube/samples/scaled-raft-no-tls/configtx.yaml#L270-L282) file, you will notice Raft consenters are using a different port:
```
- Host: orderer0.groeifabriek.nl
  Port: 7059
```
Raft orderers mutually authenticate each other so they always need TLS for inter-orderer communication. That's why disabling TLS globally requires Raft orderers communicate each other over another port instead of client-facing orderer port (7050). We enable this behavior by passing the argument `orderer.cluster.enabled=true` to hlf-kube chart. 

No other change is required. Any client of orderer, either application or Argo flows or whatever, will still use the client-facing port (7050)

Let's launch the Raft network without TLS. First tear down everything as usual:
```
argo delete --all
helm delete hlf-kube --purge
```
Wait a bit until all pods are terminated, then create necessary stuff:
```
./init.sh ./samples/scaled-raft-no-tls/ ./samples/chaincode/
```

Luanch the Raft based Fabric network in broken state (only because of `useActualDomains=true`)
```
helm install ./hlf-kube --name hlf-kube -f samples/scaled-raft-no-tls/network.yaml -f samples/scaled-raft-no-tls/crypto-config.yaml --set orderer.cluster.enabled=true --set peer.launchPods=false --set orderer.launchPods=false
```

Collect the host aliases:
```
./collect_host_aliases.sh ./samples/scaled-raft-no-tls/
```
 
Then update the network with host aliases:
``` 
helm upgrade hlf-kube ./hlf-kube -f samples/scaled-raft-no-tls/network.yaml -f samples/scaled-raft-no-tls/crypto-config.yaml -f samples/scaled-raft-no-tls/hostAliases.yaml --set orderer.cluster.enabled=true
```
Again lets wait for all pods are up and running:

```
kubectl get pod --watch
```

Congrulations you have a running scaled up HL Fabric network in Kubernetes, with 3 Raft orderer nodes spanning 2 Orderer organizations and 2 peers per organization. Since TLS is disabled, your application can use them without even noticing there are 3 Orderer nodes and 2 peers per organization.

Lets create the channels:
```
helm template channel-flow/ -f samples/scaled-raft-no-tls/network.yaml -f samples/scaled-raft-no-tls/crypto-config.yaml -f samples/scaled-raft-no-tls/hostAliases.yaml | argo submit - --watch
```

And install chaincodes:
```
helm template chaincode-flow/ -f samples/scaled-raft-no-tls/network.yaml -f samples/scaled-raft-no-tls/crypto-config.yaml -f samples/scaled-raft-no-tls/hostAliases.yaml | argo submit - --watch
```

### [Cross-cluster Raft network](#cross-cluster-raft-network)

#### Overview

This sample demonstrates how to spread `scaled-raft-tls` sample over three Kubernetes clusters. The same mechanism can be used
for any combination of hybrid networks, some parts running on premises as plain Docker containers, or on bare metal or whatever. 
In any case, these Helm charts are used for running and operating **the part of Fabric network** running in your cluster.

Basically there are two requirements to integrate your part of Fabric network with the rest:
* Peer and orderer nodes should be exposed to outer world (obviously)
* Any node should be accesible via the same address (host:port) either inside cluster or from outside of cluster

Exposing peer and orderer nodes is possible either via `Ingress` or via `LoadBalancer`. This sample demonstrates both.

The layout is as follows:

```
Cluster-One:
    OrdererOrgs:
    - Name: Groeifabriek
      NodeCount: 2
    PeerOrgs:
    - Name: Karga
      PeerCount: 2
    
Cluster-Two:
    OrdererOrgs:
    - Name: Pivt
      NodeCount: 1
    PeerOrgs:
    - Name: Atlantis
      PeerCount: 2
  
Cluster-Three:
    PeerOrgs:
    - Name: Nevergreen
      PeerCount: 2
```

Inspect the `crypto-config.yaml` and `network.yaml` files in `samples/cross-cluster-raft-tls/cluster-one/`, `cluster-two` and `cluster-three` folders respectively. You will notice each one is stripped down to relevant peer/orderer organizations that will run in that cluster. You will also notice `ExternalPeerOrgs`and `ExternalOrdererOrgs` in `crypto-config.yaml` files. We will come that in a bit why and when they are required. `configtx.yaml` files are the same for all three as they are part of the same Fabric network.

You can run this sample either on three separate Kubernetes clusters or on three different namespaces in the same cluster. The sample commands below uses different chart names and namespaces, so they can be used as they are on both setups. 

#### Preperation

`Cluster-One` and `Cluster-Three` is exposed via `Ingress` and `Cluster-Two` is exposed via `LoadBalancer`. So, for cluster one and three we need to install Ingress controllers:
```
helm install stable/nginx-ingress --name hlf-peer-ingress --namespace kube-system --set controller.service.type=LoadBalancer --set controller.ingressClass=hlf-peer --set controller.service.ports.https=7051 --set controller.service.enableHttp=false --set controller.extraArgs.enable-ssl-passthrough=''

helm install stable/nginx-ingress --name hlf-orderer-ingress  --namespace kube-system --set controller.service.type=LoadBalancer --set controller.ingressClass=hlf-orderer --set controller.service.ports.https=7050 --set controller.service.enableHttp=false --set controller.extraArgs.enable-ssl-passthrough=''
```
Notice we are installing one Ingress controller for peers and one for orderers. We are also enabling `ssl-passthrough` on these and using only the `https` port.

Wait for Ingress LoadBalancer services gets their external IP's (this can take a while):
```
kubectl -n kube-system get svc -l app=nginx-ingress,component=controller --watch
```
![Screenshot_peerorg_flow_declarative](https://raft-fabric-kube.s3-eu-west-1.amazonaws.com/images/Screenshot_waiting_for_ingress_loadbalancer_ip.png)

We need three copies of these Helm charts. So, first make two copies of `fabric-kube` folder inside `PIVT` folder and open each of them in a separete console.
```
/path/to/PIVT/
        ├── fabric-kube/
        ├── fabric-kube-two/
        └── fabric-kube-three/
```

#### Crypto material and genesis block

Lets create the crypto material for each of them:
```
# run in one
./init.sh samples/cross-cluster-raft-tls/cluster-one/ samples/chaincode/ false

# run in two
./init.sh samples/cross-cluster-raft-tls/cluster-two/ samples/chaincode/ false

# run in three
./init.sh samples/cross-cluster-raft-tls/cluster-three/ samples/chaincode/ false
```
The last optional argument `false` tells `init.sh` script not to create the `genesis` block. We will create it manually. But to do that we need:
* MSP certificates of other peer organization(s)
* MSP certificates of other orderer organization(s)
* TLS certificates of other Raft orderer node(s)

Lets copy them (the target empty placeholder directories are already created by `init.sh` script based on `ExternalPeerOrgs`and `ExternalOrdererOrgs`):
```
# run in one
cp -r ../fabric-kube-two/hlf-kube/crypto-config/ordererOrganizations/pivt.nl/msp/* hlf-kube/crypto-config/ordererOrganizations/pivt.nl/msp/
cp -r ../fabric-kube-two/hlf-kube/crypto-config/peerOrganizations/atlantis.com/msp/* hlf-kube/crypto-config/peerOrganizations/atlantis.com/msp/
cp -r ../fabric-kube-three/hlf-kube/crypto-config/peerOrganizations/nevergreen.nl/msp/* hlf-kube/crypto-config/peerOrganizations/nevergreen.nl/msp/
cp ../fabric-kube-two/hlf-kube/crypto-config/ordererOrganizations/pivt.nl/orderers/orderer0.pivt.nl/tls/server.crt hlf-kube/crypto-config/ordererOrganizations/pivt.nl/orderers/orderer0.pivt.nl/tls/
```
Now we can create the `genesis` block:
```
# run in one
cd hlf-kube/
configtxgen -profile OrdererGenesis -channelID testchainid -outputBlock ./channel-artifacts/genesis.block
cd ../
```
And copy the `genesis` block to other chart folders:
```
# run in one
cp hlf-kube/channel-artifacts/genesis.block ../fabric-kube-two/hlf-kube/channel-artifacts/
cp hlf-kube/channel-artifacts/genesis.block ../fabric-kube-three/hlf-kube/channel-artifacts/
```

#### Launch the network

The rest is more or less the same as launching the whole Raft network in the same cluster. We will first launch the network parts in broken state, collect host aliases and then update the network parts with host aliases.

But first we need to copy TLS CA certs of `Pivt` orderer organization to chart three. `cluster-three` doesn't run an `Orderer` organization, so it needs to connect to an external orderer. TLS certificates are required for that.
```
# run in three
cp ../fabric-kube-two/hlf-kube/crypto-config/ordererOrganizations/pivt.nl/msp/tlscacerts/* hlf-kube/crypto-config/ordererOrganizations/pivt.nl/msp/tlscacerts/
```

Good to go, lets launch network parts:
```
# run in one
helm install ./hlf-kube --name hlf-kube -f samples/cross-cluster-raft-tls/cluster-one/network.yaml -f samples/cross-cluster-raft-tls/cluster-one/crypto-config.yaml --set peer.launchPods=false --set orderer.launchPods=false

# run in two
helm install ./hlf-kube --name hlf-kube-two --namespace two -f samples/cross-cluster-raft-tls/cluster-two/network.yaml -f samples/cross-cluster-raft-tls/cluster-two/crypto-config.yaml --set peer.launchPods=false --set orderer.launchPods=false --set peer.externalService.enabled=true --set orderer.externalService.enabled=true

# run in three
helm install ./hlf-kube --name hlf-kube-three --namespace three -f samples/cross-cluster-raft-tls/cluster-three/network.yaml -f samples/cross-cluster-raft-tls/cluster-three/crypto-config.yaml --set peer.launchPods=false --set orderer.launchPods=false 
```

In `cluster-one` you will notice two external MSP secrets:
```
hlf-peer--atlantis--external-msp
hlf-peer--nevergreen--external-msp
```
These are created because of `ExternalPeerOrgs` in `crypto-config.yaml` and required for channel creation.

In `cluster-one` and `cluster-three` you will notice external orderer TLS CA secret:
```
hlf-orderer--pivt-external-tlsca
```
These are created because of `ExternalOrdererOrgs` in `crypto-config.yaml`. In `cluster-one` it's not used but in `cluster-three` required to communicate with external orderer.

#### Upgrade the networks with host aliases

Before continuing wait for LoadBalancer external IP's are retrieved in `cluster-two`:
```
# run in two
kubectl --namespace two get svc -l addToExternalHostAliases=true --watch
```

Then collect host aliases:
```
# run in one
./collect_host_aliases.sh samples/cross-cluster-raft-tls/cluster-one/
./collect_external_host_aliases.sh ingress samples/cross-cluster-raft-tls/cluster-one/ 

# run in two
./collect_host_aliases.sh samples/cross-cluster-raft-tls/cluster-two/ --namespace two
./collect_external_host_aliases.sh loadbalancer samples/cross-cluster-raft-tls/cluster-two/ --namespace two

# run in three
./collect_host_aliases.sh samples/cross-cluster-raft-tls/cluster-three/ --namespace three
./collect_external_host_aliases.sh ingress samples/cross-cluster-raft-tls/cluster-three/ --namespace three
```
`collect_external_host_aliases.sh` script is equivalent of `collect_host_aliases.sh` but its output is intended to be used by the rest of the Fabric network which will communicate with this part. Of course, if your domain names are registered to global DNS servers (security wise not a good idea I guess) or you are using other DNS tricks you don't need this.

**Important:** When peers/orderers are exposed via Ingress, `collect_external_host_aliases.sh` script assumes Ingress controllers are deployed to `kube-system` namespace and have `hlf-peer-ingress` and `hlf-orderer-ingress` release names respectively.

Now, lets merge host aliases together:
```
# run in one
cat ../fabric-kube-two/samples/cross-cluster-raft-tls/cluster-two/externalHostAliases.yaml >> ./samples/cross-cluster-raft-tls/cluster-one/hostAliases.yaml
cat ../fabric-kube-three/samples/cross-cluster-raft-tls/cluster-three/externalHostAliases.yaml >> ./samples/cross-cluster-raft-tls/cluster-one/hostAliases.yaml

# double check result:
cat ./samples/cross-cluster-raft-tls/cluster-one/hostAliases.yaml

# run in two
cat ../fabric-kube/samples/cross-cluster-raft-tls/cluster-one/externalHostAliases.yaml >> ./samples/cross-cluster-raft-tls/cluster-two/hostAliases.yaml
cat ../fabric-kube-three/samples/cross-cluster-raft-tls/cluster-three/externalHostAliases.yaml >> ./samples/cross-cluster-raft-tls/cluster-two/hostAliases.yaml

# double check result:
cat ./samples/cross-cluster-raft-tls/cluster-two/hostAliases.yaml

# run in three
cat ../fabric-kube/samples/cross-cluster-raft-tls/cluster-one/externalHostAliases.yaml >> ./samples/cross-cluster-raft-tls/cluster-three/hostAliases.yaml
cat ../fabric-kube-two/samples/cross-cluster-raft-tls/cluster-two/externalHostAliases.yaml >> ./samples/cross-cluster-raft-tls/cluster-three/hostAliases.yaml

# double check result:
cat ./samples/cross-cluster-raft-tls/cluster-three/hostAliases.yaml
```

Now we are ready to update the networks:
```
# run in one
helm upgrade hlf-kube ./hlf-kube -f samples/cross-cluster-raft-tls/cluster-one/network.yaml -f samples/cross-cluster-raft-tls/cluster-one/crypto-config.yaml -f samples/cross-cluster-raft-tls/cluster-one/hostAliases.yaml --set peer.ingress.enabled=true --set orderer.ingress.enabled=true

# run in two
helm upgrade hlf-kube-two ./hlf-kube -f samples/cross-cluster-raft-tls/cluster-two/network.yaml -f samples/cross-cluster-raft-tls/cluster-two/crypto-config.yaml -f samples/cross-cluster-raft-tls/cluster-two/hostAliases.yaml --set peer.externalService.enabled=true --set orderer.externalService.enabled=true

# run in three
helm upgrade hlf-kube-three ./hlf-kube -f samples/cross-cluster-raft-tls/cluster-three/network.yaml -f samples/cross-cluster-raft-tls/cluster-three/crypto-config.yaml -f samples/cross-cluster-raft-tls/cluster-three/hostAliases.yaml --set peer.ingress.enabled=true 
```
And wait for all pods are up and running as usual.

**Note:** When exposing peers/orderers via LoadBalancer, depending on your cloud provider, you might see contunious error logs at peer and orderer pods like below. For Azure AKS this is the case and caused by health probes done by Azure load balancer. In Azure AKS, as of Februaury 2020, there seems to be no way of disabling health probes. Check annotations specific to your cloud provider.
```
2020-02-19 10:44:47.609 UTC [core.comm] ServerHandshake -> ERRO 230 TLS handshake failed with error EOF server=Orderer remoteaddress=10.240.0.6:55843
```
#### Channels and Chaincodes

Everything is up and running, now we can create the channels. But before that, have a look at `network.yaml` at `cluster-three`. 
You will notice the `externalOrderer` section as below. Normally, channel and chaincode flows use the first orderer node of the first orderer organization. Since `cluster-three` does not run an orderer, it needs an external orderer. 
```
externalOrderer:
  enabled: true
  orgName: Pivt
  host: orderer0.pivt.nl
  port: "7050"
```

Lets create the channels:

**Note:** Don't run these flows in parallel. Wait for completion of each one before proceeding to next one. In particular `channels` can only be created by `cluster-one` as only it has all the MSP certificates.
```
# run in one
helm template channel-flow/ -f samples/cross-cluster-raft-tls/cluster-one/network.yaml -f samples/cross-cluster-raft-tls/cluster-one/crypto-config.yaml -f samples/cross-cluster-raft-tls/cluster-one/hostAliases.yaml | argo submit - --watch

# run in two
helm template channel-flow/ -f samples/cross-cluster-raft-tls/cluster-two/network.yaml -f samples/cross-cluster-raft-tls/cluster-two/crypto-config.yaml -f samples/cross-cluster-raft-tls/cluster-two/hostAliases.yaml | argo submit - --namespace two --watch

# run in three
helm template channel-flow/ -f samples/cross-cluster-raft-tls/cluster-three/network.yaml -f samples/cross-cluster-raft-tls/cluster-three/crypto-config.yaml -f samples/cross-cluster-raft-tls/cluster-three/hostAliases.yaml | argo submit - --namespace three --watch
```

And install/instantiate the chaincodes:
```
# run in one
helm template chaincode-flow/ -f samples/cross-cluster-raft-tls/cluster-one/network.yaml -f samples/cross-cluster-raft-tls/cluster-one/crypto-config.yaml -f samples/cross-cluster-raft-tls/cluster-one/hostAliases.yaml | argo submit - --watch

# run in two
helm template chaincode-flow/ -f samples/cross-cluster-raft-tls/cluster-two/network.yaml -f samples/cross-cluster-raft-tls/cluster-two/crypto-config.yaml -f samples/cross-cluster-raft-tls/cluster-two/hostAliases.yaml | argo submit - --namespace two --watch

# run in three
helm template chaincode-flow/ -f samples/cross-cluster-raft-tls/cluster-three/network.yaml -f samples/cross-cluster-raft-tls/cluster-three/crypto-config.yaml -f samples/cross-cluster-raft-tls/cluster-three/hostAliases.yaml | argo submit - --namespace three --watch
```

Congratulations! You now succesfully spread Fabric network over three Kubernetes clusters.

### [Adding new peer organizations](#adding-new-peer-organizations)

#### Simple network

First tear down and re-launch and populate the simple network as described in [launching the network](launching-the-network), [creating channels](creating-channels) and [installing chaincodes](installing-chaincodes).

At this point we can update the original configtx.yaml, crypto-config.yaml and network.yaml for the new organizations. First take backup of the originals:
```
rm -rf tmp && mkdir -p tmp && cp samples/simple/configtx.yaml samples/simple/crypto-config.yaml samples/simple/network.yaml tmp/
```
Then override with extended ones:
```
cp samples/simple/extended/* samples/simple/ && cp samples/simple/configtx.yaml hlf-kube/
```

Let's create the necessary stuff:
```
./extend.sh samples/simple
```
This script basically performs a `cryptogen extend` command to create missing crypto material.

Then update the network for new crypto material and configtx and launch the new peers:
```
helm upgrade hlf-kube ./hlf-kube -f samples/simple/network.yaml -f samples/simple/crypto-config.yaml
```

Then lets create new peer organizations:
```
helm template peer-org-flow/ -f samples/simple/network.yaml -f samples/simple/crypto-config.yaml -f samples/simple/configtx.yaml | argo submit - --watch
```
This flow:
* Parses consortiums from `configtx.yaml` using `genesisProfile` defined in `network.yaml`
* Adds missing organizations to consortiums
* Adds missing organizations to existing channels as defined in `network.yaml`
* Emits an error for non-existing consortiums
* Skips non-existing channels (they will be created by channel flow later)

When the flow completes the output will be something like this:
![Screenshot_peerorg_flow_declarative](https://raft-fabric-kube.s3-eu-west-1.amazonaws.com/images/Screenshot_peerorg_flow_declarative.png)

By default, peer org flow updates all existing channels and consortiums as necessary. You can limit this behaviour by setting `flow.channel.include` and `flow.consortium.include` variables respectively.

At this point make sure new peer pods are up and running. Then run the channel flow to create new channels and populate 
existing ones regarding the new organizations:
```
helm template channel-flow/ -f samples/simple/network.yaml -f samples/simple/crypto-config.yaml | argo submit - --watch
```

Finally run the chaincode flow to populate the chaincodes regarding new organizations:
```
helm template chaincode-flow/ -f samples/simple/network.yaml -f samples/simple/crypto-config.yaml --set chaincode.version=2.0 | argo submit - --watch
```
Please note, we increased the chaincode version. This is required to upgrade the chaincodes with new policies. Otherwise, new peers' endorsements will fail.

Peer org flow is declarative and idempotent. You can run it many times. It will add peer organizations to consortiums only if 
they are not already in consortiums, add peer organizations to channels only if not already in channels.

Restore the original files
```
cp tmp/configtx.yaml tmp/crypto-config.yaml tmp/network.yaml samples/simple/
```

#### Raft orderer network

Adding new peer organizations to a network which utilizes Raft orderer is similar. But there is one point to be aware of: After adding new organizations we need to update the rest of the network with new host aliases information. This means existing pods will be restarted and will lose all the data. That's why persistence should be enabled.

First tear down and re-launch and populate the Raft network as described in [scaled-up-raft-network](#scaled-up-raft-network-with-tls)(scaled-up-raft-network) but pass the following additional flag: `-f samples/scaled-raft-tls/persistence.yaml`

At this point we can update the original configtx.yaml, crypto-config.yaml and network.yaml for the new organizations. First take backup of the originals:
```
rm -rf tmp && mkdir -p tmp && cp samples/scaled-raft-tls/configtx.yaml samples/scaled-raft-tls/crypto-config.yaml samples/scaled-raft-tls/network.yaml tmp/
```

Then override with extended ones:
```
cp samples/scaled-raft-tls/extended/* samples/scaled-raft-tls/ && cp samples/scaled-raft-tls/configtx.yaml hlf-kube/
```

Create new crypto material:
```
./extend.sh samples/scaled-raft-tls
```

Update the network for new crypto material and configtx and launch new peers 
```
helm upgrade hlf-kube ./hlf-kube -f samples/scaled-raft-tls/network.yaml -f samples/scaled-raft-tls/crypto-config.yaml -f samples/scaled-raft-tls/persistence.yaml -f samples/scaled-raft-tls/hostAliases.yaml
```

Collect extended host aliases:
```
./collect_host_aliases.sh ./samples/scaled-raft-tls/ 
```

Upgrade host aliases in pods and wait for all pods are up and running:
```
helm upgrade hlf-kube ./hlf-kube -f samples/scaled-raft-tls/network.yaml -f samples/scaled-raft-tls/crypto-config.yaml -f samples/scaled-raft-tls/hostAliases.yaml -f samples/scaled-raft-tls/persistence.yaml
kubectl  get pod --watch
```

Let's create the new peer organizations:
```
helm template peer-org-flow/ -f samples/scaled-raft-tls/configtx.yaml -f samples/scaled-raft-tls/crypto-config.yaml -f samples/scaled-raft-tls/network.yaml -f samples/scaled-raft-tls/hostAliases.yaml | argo submit - --watch
```

Then run the channel flow to create new channels and populate existing ones regarding the new organizations:
```
helm template channel-flow/ -f samples/scaled-raft-tls/network.yaml -f samples/scaled-raft-tls/crypto-config.yaml -f samples/scaled-raft-tls/hostAliases.yaml | argo submit - --watch
```

Finally run the chaincode flow to populate the chaincodes regarding new organizations:
```
helm template chaincode-flow/ -f samples/scaled-raft-tls/network.yaml -f samples/scaled-raft-tls/crypto-config.yaml -f samples/scaled-raft-tls/hostAliases.yaml --set chaincode.version=2.0 | argo submit - --watch
```
Please note, we increased the chaincode version. This is required to upgrade the chaincodes with new policies. Otherwise, new peers' endorsements will fail.


Restore original files
```
cp tmp/configtx.yaml tmp/crypto-config.yaml tmp/network.yaml samples/scaled-raft-tls/
```

### [Adding new peers to organizations](#adding-new-peers-to-organizations)

Update the `Template.Count` value for relevant `PeerOrgs` in `crypto-config.yaml` and run the sequence 
in [adding new peer organizations](#adding-new-peer-organizations). 

No need to run `peer-org-flow` in this case as peer organizations didn't change. 
But running it won't hurt anyway, remember it's idempotent ;)

### [Updating channel configuration](#updating-channel-configuration)

The application capabilities version is already set to to _V1_4_2_ for Raft TLS sample, but lets assume it's at an older version and we want to update it:
```
helm template channel-update-flow/ -f samples/scaled-raft-tls/network.yaml -f samples/scaled-raft-tls/crypto-config.yaml -f samples/scaled-raft-tls/hostAliases.yaml --set update.scope=application --set update.application.capabilities.version='V1_4_2' | argo submit - --watch
```
The output will be something like:
![Screenshot_channel_update_flow](https://raft-fabric-kube.s3-eu-west-1.amazonaws.com/images/Screenshot_channel_update_flow.png)

By default any update at Application level requires _MAJORITY_ of organization admins, lets make it _ANY_ of the admins:
```
helm template channel-update-flow/ -f samples/scaled-raft-tls/network.yaml -f samples/scaled-raft-tls/crypto-config.yaml -f samples/scaled-raft-tls/hostAliases.yaml --set update.scope=application --set update.application.type=jsonPath --set update.application.jsonPath.key='.channel_group.groups.Application.policies.Admins.policy.value.rule' --set update.application.jsonPath.value="ANY" | argo submit - --watch
```
This way any arbitrary atomic config value can be updated. It's easy to extend the flow for more complex config updates.

_channel-update-flow_ is also declarative and idempotent, you can run it many times with the same settings.

If you are not running majority of organizations and the policy requires majority, sending config update will fail. In that case, you can run the flow with `flow.sendUpdate.enabled=false` flag, this will prevent the flow sending the config update and wait indefinitely. You can copy the signed config update from the pod `/work/signed_update.pb` and send to other organization admins by other means.

**Important:** Pay extreme attention when setting capability versions. If you set them to a non-existing value, like _V1_4_3_, orderers will accept the value but peers will crash immediately when they receive the update. To my knowledge, there is no way of [reverting it back](https://lists.hyperledger.org/g/fabric/message/7775) except restoring from a backup.

## [Configuration](#configuration)

There are basically 2 configuration files: [crypto-config.yaml](fabric-kube/samples/simple/crypto-config.yaml) 
and [network.yaml](fabric-kube/samples/simple/network.yaml). 


### crypto-config.yaml 
This is Fabric's native configuration for `cryptogen` tool. We use it to define the network architecture. We honour `OrdererOrgs`, 
`PeerOrgs`, `Template.Count` at PeerOrgs (peer count) and `Specs.Hostname[]` at OrdererOrgs.

```yaml
OrdererOrgs:
  - Name: Groeifabriek
    Domain: groeifabriek.nl
    Specs:
      - Hostname: orderer
PeerOrgs:
  - Name: Karga
    Domain: aptalkarga.tr
    EnableNodeOUs: true
    Template:
      Count: 1
    Users:
      Count: 1
  - Name: Nevergreen
    Domain: nevergreen.nl
    EnableNodeOUs: true
    Template:
      Count: 1
    Users:
      Count: 1
```
### network.yaml 
This file defines how network is populated regarding channels and chaincodes.

```yaml
network:
  # used by init script to create genesis block and by peer-org-flow to parse consortiums
  genesisProfile: OrdererGenesis
  # used by init script to create genesis block 
  systemChannelID: testchainid

  # defines which organizations will join to which channels
  channels:
    - name: common
      # all peers in these organizations will join the channel
      orgs: [Karga, Nevergreen, Atlantis]
    - name: private-karga-atlantis
      # all peers in these organizations will join the channel
      orgs: [Karga, Atlantis]

  # defines which chaincodes will be installed to which organizations
  chaincodes:
    - name: very-simple
      # if defined, this will override the global chaincode.version value
      version: # "2.0" 
      # chaincode will be installed to all peers in these organizations
      orgs: [Karga, Nevergreen, Atlantis]
      # at which channels are we instantiating/upgrading chaincode?
      channels:
      - name: common
        # chaincode will be instantiated/upgraded using the first peer in the first organization
        # chaincode will be invoked on all peers in these organizations
        orgs: [Karga, Nevergreen, Atlantis]
        policy: OR('KargaMSP.member','NevergreenMSP.member','AtlantisMSP.member')
        
    - name: even-simpler
      orgs: [Karga, Atlantis]
      channels:
      - name: private-karga-atlantis
        orgs: [Karga, Atlantis]
        policy: OR('KargaMSP.member','AtlantisMSP.member')
```

For chart specific configuration, please refer to the comments in the relevant [values.yaml](fabric-kube/hlf-kube/values.yaml) files.

## [TLS](#tls)
![TLS](https://raft-fabric-kube.s3-eu-west-1.amazonaws.com/images/HL_in_Kube_TLS.png)

Using TLS is a two step process. We first launch the network in broken state, then collect ClusterIPs of services and attach them to pods as DNS entries using pod [hostAliases](https://kubernetes.io/docs/concepts/services-networking/add-entries-to-pod-etc-hosts-with-host-aliases/) spec.

Important point here is, as opposed to pod ClusterIPs, service ClusterIPs are stable, they won't change if service is not deleted and re-created.

## [Backup-Restore](#backup-restore)

### [Backup Restore Requirements](#backup-restore-requirements)
* Persistence should be enabled in relevant components (Orderer, Peer, CouchDB)
* Configure Argo for some artifact repository. Easiest way is to install [Minio](https://github.com/argoproj/argo/blob/master/docs/configure-artifact-repository.md) 
* An Azure Blob Storage account with a container named `hlf-backup` (configurable). 
ATM, backups can only be stored at Azure Blob Storage but it's quite easy to extend backup/restore 
flows for other mediums, like AWS S3. See bottom of [backup-workflow.yaml](fabric-kube/backup-flow/templates/backup-workflow.yaml)

**IMPORTANT:** Backup flow does not backup contents of Kafka cluster, if you are using Kafka orderer you need to 
manually back it up. In particular, Kafka Orderer with some state cannot handle a fresh Kafka installation, see this 
[Jira ticket](https://jira.hyperledger.org/browse/FAB-15541), hopefully Fabric guys will fix this soon.

### [Backup Restore Flow](#backup-restore-flow)
![HL_backup_restore](https://raft-fabric-kube.s3-eu-west-1.amazonaws.com/images/HL_backup_restore.png)

First lets create a persistent network:
```
./init.sh ./samples/simple-persistent/ ./samples/chaincode/
helm install --name hlf-kube -f samples/simple-persistent/network.yaml -f samples/simple-persistent/crypto-config.yaml -f samples/simple-persistent/values.yaml ./hlf-kube
```
Again lets wait for all pods are up and running, this may take a bit longer due to provisioning of disks.
```
kubectl  get pod --watch
```
Then populate the network, you know how to do it :)

### Backup

Start backup procedure and wait for pods to be terminated and re-launched with `Rsync` containers.
```
helm upgrade hlf-kube --set backup.enabled=true -f samples/simple-persistent/network.yaml -f samples/simple-persistent/crypto-config.yaml -f samples/simple-persistent/values.yaml  ./hlf-kube
kubectl  get pod --watch
```
Then take backup:
```
helm template -f samples/simple-persistent/crypto-config.yaml --set backup.target.azureBlobStorage.accountName=<your account name> --set backup.target.azureBlobStorage.accessKey=<your access key> backup-flow/ | argo submit  -  --watch
```
![Screenshot_backup_flow](https://s3-eu-west-1.amazonaws.com/raft-fabric-kube/images/Screenshot_backup_flow.png)

This will create a folder with default `backup.key` (html formatted date `yyyy-mm-dd`), 
in Azure Blob Storage and hierarchically store backed up contents there.

Finally go back to normal operation:
```
helm upgrade hlf-kube -f samples/simple-persistent/network.yaml -f samples/simple-persistent/crypto-config.yaml -f samples/simple-persistent/values.yaml ./hlf-kube
kubectl  get pod --watch
```
### [Restore](#restore)

Start restore procedure and wait for pods to be terminated and re-launched with `Rsync` containers.
```
helm upgrade hlf-kube --set restore.enabled=true -f samples/simple-persistent/network.yaml -f samples/simple-persistent/crypto-config.yaml -f samples/simple-persistent/values.yaml ./hlf-kube
kubectl  get pod --watch
```

Then restore from backup:
```
helm template --set backup.key='<backup key>' -f samples/simple-persistent/crypto-config.yaml --set backup.target.azureBlobStorage.accountName=<your account name> --set backup.target.azureBlobStorage.accessKey=<your access key> restore-flow/  | argo submit  -  --watch
```
![Screenshot_restore_flow](https://s3-eu-west-1.amazonaws.com/raft-fabric-kube/images/Screenshot_restore_flow.png)

Finally go back to normal operation:
```
helm upgrade hlf-kube -f samples/simple-persistent/network.yaml -f samples/simple-persistent/crypto-config.yaml -f samples/simple-persistent/values.yaml ./hlf-kube
kubectl  get pod --watch
```

## [Limitations](#limitations)

### TLS

Transparent load balancing is not possible when TLS is globally enabled. So, instead of `Peer-Org`, `Orderer-Org` or `Orderer-LB` services, you need to connect to individual `Peer` and `Orderer` services.

Running Raft orderers without globally enabling TLS is possible since Fabric 1.4.5. See [Scaled-up Raft network without TLS](#scaled-up-raft-network-without-tls) sample for details.

### Multiple Fabric networks in the same Kubernetes cluster

This is possible but they should be run in different namespaces. We do not use Helm release name in names of components, 
so if multiple instances of Fabric network is running in the same namespace, names will conflict.

## [FAQ and more](#faq-and-more)

Please see [FAQ](FAQ.md) page for further details. Also this [post](https://accenture.github.io/blog/2019/06/25/hl-fabric-meets-kubernetes.html) at Accenture's open source blog provides some additional information like motivation, how it works, benefits regarding NFR's, etc.

## [Conclusion](#conclusion)

So happy BlockChaining in Kubernetes :)

And don't forget the first rule of BlockChain club:

**"Do not use BlockChain unless absolutely necessary!"**

*Hakan Eryargi (r a f t)*
