1. set up cloud resource on AKS
1.1 run ndn-producer in pod of AKS
1.2 create ld_balancer service to expose TCP port 443 that maps to port 6363 for NFD
1.3 use ndn-producer API to inject resource into the NDN network with name /ndn-cloud-resource

2. set up on-prem resource on local subnet 
2.1 run ndn-producer in local VM
2.2 use ndn-procuder API to iject resource into NDN network with name /ndn-onprem-resource

2. set up ndnclient on local ( ideally need to be in the same subnet of Lab that has access to Azure)
2.1 install NFD
2.2 create face to ndn-producer on ASK
2.2.1 nfdc face create tcp://aks-ld-service-ip:443
2.2.2 nfdc route add /ndn-cloud-resource {faceID-from 2.2.1}
2.3 create face to ndn-producer on local VM
2.3.1 nfdc face create tcp://local-VM-ip:6363
2.3.2 nfdc route add /ndn-onprem-resource {faceID-from 2.3.1}

3. send curl command to ndnclient to access resource 
```
curl --request POST --data '{"hash":"4SDFSDJKLE", "mode": "async", "publicname":"/ndn-cloud-resource", "destination":"newfile.bin", "callback":"http://localhost:8000"}' http://ndnclient-ip:8080/interest
```

