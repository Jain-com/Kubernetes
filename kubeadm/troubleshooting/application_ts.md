
git clone https://github.com/piyushsachdeva/example-voting-app.git
cd /example-voting-app/k8s-specifications
kubectl apply -f .
kubectl get netpol
kubectl get po -o wide

# To check the external IP
 kubectl get nodes -o wide
 kubectl get svc # it will list all service
 kubectl describe svc vote # to view particular service
 # To view vote yaml file
 kubectl get svc vote -o yaml





 
