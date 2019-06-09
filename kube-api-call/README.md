
# Create a namespace in your cluster
kubectl create namespace learning-ns
# Create a service account named api-explorer in your cluster
kubectl create serviceaccount api-explorer -n learning-ns
# Create a (Cluster)Role granting access to the necessary resources
kubectl apply -f role.yaml -n learning-ns
#Bind the ClusterRole to the ServiceAccount in the current namespace (eg. learning-ns).
kubectl create rolebinding api-explorer:log-reader --clusterrole log-reader --serviceaccount default:api-explorer -n learning-ns

SERVICE_ACCOUNT=api-explorer
# Get the ServiceAccount's token Secret's name
SECRET=$(kubectl get serviceaccount -n learning-ns ${SERVICE_ACCOUNT} -o json | jq -Mr '.secrets[].name | select(contains("token"))')
# Extract the Bearer token from the Secret and decode
TOKEN=$(kubectl get secret ${SECRET}  -n learning-ns  -o json | jq -Mr '.data.token' | base64 -d)
# Extract, decode and write the ca.crt to a temporary location
kubectl get secret ${SECRET} -n learning-ns  -o json | jq -Mr '.data["ca.crt"]' | base64 -d > ca.crt
# Get the API Server location
APISERVER=https://$(kubectl -n learning-ns get endpoints kubernetes --no-headers | awk '{ print $2 }')

# curl it to test
curl -s $APISERVER/openapi/v2  --header "Authorization: Bearer $TOKEN" --cacert ca.crt | less
curl -s $APISERVER/api/v1/namespaces/learning-ns/pods/ --header "Authorization: Bearer $TOKEN" --cacert ca.crt | jq -rM '.items[].metadata.name'
curl -s $APISERVER/api/v1/namespaces/learning-ns/pods/sa-web-app-666fc4b99-2qppz/log  --header "Authorization: Bearer $TOKEN" --cacert ca.crt
