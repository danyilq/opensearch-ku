#Cluster exportation notes
####Command to test if the cluster is configured properly
`curl -u 5mbg6d3afjt3:hellohello123 -XGET --insecure https://localhost:9200/5mbg6d3afjt3-wdehh`
###Apply all configurations
`./securityadmin.sh   -cacert ../../../config/root-ca.pem -cert ../../../config/kirk.pem -key ../../../config/kirk-key.pem -cd /usr/share/opensearch/plugins/opensearch-security/securityconfig`
###Command to create a snapshot
`curl -u admin:admin -XPUT https://localhost:9200/_snapshot/my-repository/<snapshot_id>`
###Command to obtain created snapshot from s3
`curl --insecure -u admin:admin https://<cluster_id>:9200/_snapshot/s3-snapshot-repo/<snapshot_id>/_restore -XPOST -H "content-tpe:application/json" -d '{"indices": "-.opendistro_security",}'`
###Command to create a backup
`./securityadmin.sh -backup mnt/backups \
  -icl \
  -nhnv \
  -cacert ../../../config/root-ca.pem \
  -cert ../../../config/kirk.pem \
  -key ../../../config/kirk-key.pem`
###Command to push backup into existing cluster
`./securityadmin.sh -h /a69a0b2d3d6f5487aadbe726ddccc1cc-e5103ff1aff7c993.elb.us-east-1.amazonaws.com,,,-p 9301 -cd /etc/backups/ -ts ... -tspass ... -ks ... -kspass ...`

#Cluster creation steps
####1. Create the s3 bucket to store the state data of the cluster, lets name it “opensearch-kube-state-store” for the example. In your case it should be another (all s3 storages must have unique names)
####2. Download the kops 
####3. Optional(create the route53 if you want to use the specific dns zone, we can use the gossip connection otherwise)
####4. Export name and state store of the cluster like so:
`export NAME=opensearch.k8s.local export KOPS_STATE_STORE="s3://opensearch-kube-state-store" `
#####k8s.local means the cluster will use the gossip connection
####5. Create the cluster using 
`kops create cluster --zones=us-east-1a,us-east-1d --cloud=aws --master-size=t2.medium --name=${NAME}`
####6. Get the configuration files Clone the git repository with configuration files using
`git clone https://github.com/danyilp/opensearch-ku.git`
######Open the git folder 
`cd opensearch-ku`
####7. After 5th step cluster is not yet created, to finish the creation of the cluster we must use 
`kops update cluster --name ${NAME} --yes --admin`
###But! before this we want to configure the instance groups for the nodes. 
#####First of all delete the default nodes instance-group since we’ll be creating the cluster in the us-east it should look like
`kops delete ig --name=${NAME} nodes-us-east-1a`
#####then create 2 new instance groups for master and data nodes
`kops create -f ig_master.yaml `

`kops create -f ig_data.yaml`

#####After it run the
`kops update cluster --name ${NAME} --yes --admin`
####8. Wait until cluster is created you can check it using `kops validate cluster --wait 10m`
####9. Install the Helm charts
######1. Get OpenSearch chart 
`helm repo add opensearch https://opensearch-project.github.io/helm-charts/`

`helm repo update`
######2. Install the helm charts
`helm install opensearch-master-us-east-1 opensearch/opensearch -f master.yaml`

`helm install opensearch-data-us-east-1 opensearch/opensearch -f data.yaml`
####10. Use the cluster
#####To get the cluster endpoint you can use
`kubectl get services opensearch-cluster-master-us-east-1`