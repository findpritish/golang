
# Create a namespace in your cluster
kubectl create namespace learning-ns
# Create a service account named api-explorer in your cluster
kubectl create serviceaccount api-explorer
# Create a (Cluster)Role granting access to the necessary resources
kubectl apply -f role.yaml
#Bind the ClusterRole to the ServiceAccount in the current namespace (eg. learning-ns).
kubectl create rolebinding api-explorer:log-reader --clusterrole log-reader --serviceaccount default:api-explorer -n learning-ns

SERVICE_ACCOUNT=api-explorer
# Get the ServiceAccount's token Secret's name
SECRET=$(kubectl get serviceaccount ${SERVICE_ACCOUNT} -o json | jq -Mr '.secrets[].name | select(contains("token"))')
# Extract the Bearer token from the Secret and decode
TOKEN=$(kubectl get secret ${SECRET}  -o json | jq -Mr '.data.token' | base64 -D)
# Extract, decode and write the ca.crt to a temporary location
kubectl get secret ${SECRET} -o json | jq -Mr '.data["ca.crt"]' | base64 -D > ca.crt
# Get the API Server location
APISERVER=https://$(kubectl get endpoints kubernetes --no-headers | awk '{ print $2 }')

# curl it to test
curl -s $APISERVER/openapi/v2  --header "Authorization: Bearer $TOKEN" --cacert ca.crt | less

#create a pod named nginx 
kubectl -n learning-ns run nginx --image=nginx --restart=Never 

# curl it to get pod details which is created
curl -s $APISERVER/api/v1/namespaces/default/pods/ --header "Authorization: Bearer $TOKEN" --cacert ca.crt | jq -rM '.items[].metadata.name'

# Get IP of the nginx pod
NGINX_IP=$(kubectl -n learning-ns get pod nginx -o jsonpath='{.status.podIP}')

# create a temp busybox pod
kubectl -n learning-ns run busybox --image=busybox --env="NGINX_IP=$NGINX_IP" --rm -it --restart=Never -- sh

# run wget on specified IP:Port to generete log for nginxpod which was created
wget -O- $NGINX_IP:80
exit

# curl it to get the log from pod
curl -s $APISERVER/api/v1/namespaces/learning-ns/pods/nginx/log  --header "Authorization: Bearer $TOKEN" --cacert ca.crt
