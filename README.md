web_voting_app_with_K8s_on_EC2

This app allows the user to vote for their preffered programming language out of six choices: Python, C#, JS, GO, Java, and NodeJS.

-> Frontend: App is built using React and JS, which provides a responsive and user-friendly interface for voting.

-> Backend and API: Backend of this App is powered by Golang. It is used as the API handling voting requests. MongoDB is used as the backend database, configured with a replica set for data redundancy and high availability.

Process:
  
  Step 1: Create EKS Cluster with NodeGroup(2 nodes of t2.medium instance type)
  Set IAM role for EC2:
   
    Command:

      {
              "Version": "2012-10-17",
              "Statement": [{
                      "Effect": "Allow",
                      "Action": [
                              "eks:DescribeCluster",
                              "eks:ListClusters",
                              "eks:DescribeNodegroup",
                              "eks:ListNodegroups",
                              "eks:ListUpdates",
                              "eks:AccessKubernetesApi"
                        ],
                        "Resource": "*"
              }]
        }

Installing Kubectl:

    curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.24.11/2023-03-17/bin/linux/amd64/kubectl
    chmod +x ./kubectl
    sudo cp ./kubectl /usr/local/bin
    export PATH=/usr/local/bin:$PATH

Installing AWScli:

    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
    unzip awscliv2.zip
    sudo ./aws/install

Make sure the cluster is ready, run the command:

    aws eks update-kubeconfig --name EKS_CLUSTER_NAME --region us-west-2

To check the nodes in your cluster, run the command:
      
      kubectl get nodes

Clone github repo

    https://github.com/rajatmall05/cloud_native_web_voting_app_with_K8s.git

Crete Namespace

    kubectl create ns <ns>
    kubectl config set-context --current --namespace <ns>

MongoDB Setup:

To create Mongo statefulset with Persistent volumes, run the command in manifests folder:

    kubectl apply -f mongo-statefulset.yaml

Mongo Service:

    kubectl apply -f mongo-service.yaml

Create a temporary network utils pods. Enter into bash session, and run the command in terminal:
    
    kubectl run --rm utils -it --image praqma/network-multitool -- bash

Within the new utils pod shell, execute DNS queries:

    for i in {0..2}; do nslookup mongo-$i.mongo; done

Exit the utils container

    exit

On the mongo-0 pod, initialise the MongoDB Replica set, and run the command in terminal:

    cat << EOF | kubectl exec -it mongo-0 -- mongo
    rs.initiate();
    sleep(2000);
    rs.add("mongo-1.mongo:27017");
    sleep(2000);
    rs.add("mongo-2.mongo:27017");
    sleep(2000);
    cfg = rs.conf();
    cfg.members[0].host = "mongo-0.mongo:27017";
    rs.reconfig(cfg, {force: true});
    sleep(5000);
    EOF

To confirm run this in the terminal:

    kubectl exec -it mongo-0 -- mongo --eval "rs.status()" | grep "PRIMARY\|SECONDARY"

Load the data in the DB by running the command:

    cat << EOF | kubectl exec -it mongo-0 -- mongo
    use langdb;
    db.languages.insert({"name" : "csharp", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 5, "compiled" : false, "homepage" : "https://dotnet.microsoft.com/learn/csharp", "download" : "https://dotnet.microsoft.com/download/", "votes" : 0}});
    db.languages.insert({"name" : "python", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 3, "script" : false, "homepage" : "https://www.python.org/", "download" : "https://www.python.org/downloads/", "votes" : 0}});
    db.languages.insert({"name" : "javascript", "codedetail" : { "usecase" : "web, client-side", "rank" : 7, "script" : false, "homepage" : "https://en.wikipedia.org/wiki/JavaScript", "download" : "n/a", "votes" : 0}});
    db.languages.insert({"name" : "go", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 12, "compiled" : true, "homepage" : "https://golang.org", "download" : "https://golang.org/dl/", "votes" : 0}});
    db.languages.insert({"name" : "java", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 1, "compiled" : true, "homepage" : "https://www.java.com/en/", "download" : "https://www.java.com/en/download/", "votes" : 0}});
    db.languages.insert({"name" : "nodejs", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 20, "script" : false, "homepage" : "https://nodejs.org/en/", "download" : "https://nodejs.org/en/download/", "votes" : 0}});
    
    db.languages.find().pretty();
    EOF

Create Mongo secret:

    kubectl apply -f mongo-secret.yaml

API Setup:
Create GO API deployment by running the command:

    kubectl apply -f api-deployment.yaml

Expose API deployment through the service using the command:

    kubectl expose deploy api \
     --name=api \
     --type=LoadBalancer \
     --port=80 \
     --target-port=8080

Next set the environment variable:

    {
    API_ELB_PUBLIC_FQDN=$(kubectl get svc api -ojsonpath="{.status.loadBalancer.ingress[0].hostname}")
    until nslookup $API_ELB_PUBLIC_FQDN >/dev/null 2>&1; do sleep 2 && echo waiting for DNS to propagate...; done
    curl $API_ELB_PUBLIC_FQDN/ok
    echo
    }
    
Test and confirm that the API route URL /languages, and /languages/{name} endpoints can be called successfully. In the terminal run any of the following commands:

    curl -s $API_ELB_PUBLIC_FQDN/languages | jq .
    curl -s $API_ELB_PUBLIC_FQDN/languages/go | jq .
    curl -s $API_ELB_PUBLIC_FQDN/languages/java | jq .
    curl -s $API_ELB_PUBLIC_FQDN/languages/nodejs | jq .
    
If everything works fine, go ahead with Frontend setup.

    {
    API_ELB_PUBLIC_FQDN=$(kubectl get svc api -ojsonpath="{.status.loadBalancer.ingress[0].hostname}")
    echo API_ELB_PUBLIC_FQDN=$API_ELB_PUBLIC_FQDN
    }

Frontend setup

Create the Frontend Deployment resource. In the terminal run the following command:

    kubectl apply -f frontend-deployment.yaml

Create a new Service resource of LoadBalancer type. In the terminal run the following command:

    kubectl expose deploy frontend \
     --name=frontend \
     --type=LoadBalancer \
     --port=80 \
     --target-port=8080

Confirm that the Frontend ELB is ready to recieve HTTP traffic. In the terminal run the following command:

    {
    FRONTEND_ELB_PUBLIC_FQDN=$(kubectl get svc frontend -ojsonpath="{.status.loadBalancer.ingress[0].hostname}")
    until nslookup $FRONTEND_ELB_PUBLIC_FQDN >/dev/null 2>&1; do sleep 2 && echo waiting for DNS to propagate...; done
    curl -I $FRONTEND_ELB_PUBLIC_FQDN
    }

Generate the Frontend URL for browsing. In the terminal run the following command:

    echo http://$FRONTEND_ELB_PUBLIC_FQDN

Copy the URL created from after hitting the above command and paste in browser and test the full end-to-end cloud native app

When the app loaded successfully, vote by clicking on "+1" buttons, this will generate AJAX traffic which will be sent back to the API via the API's assigned ELB.

Query the MongoDB database directly to observe the updated vote data and run the command in the terminal:

    kubectl exec -it mongo-0 -- mongo langdb --eval "db.languages.find().pretty()"

Summary:

So in this project, we learnt how to deploy a cloud native app into EKS. Once deployed up and running, we used our local workstation's browser to test out the app. Then later confirmed that the activity within the app generated data which was captured and recorded successfully within the MongoDB ReplicaSet back end within the cluster.




