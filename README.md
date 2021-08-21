# Setup Guide

## Prerequisites
- [Helm](https://helm.sh/docs/intro/install/) >= 3.2 (chocolaty install default helm version is less than 3.2, Please makesure to install required version)
- [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) you must use a version that is within one minor version difference of your cluster. For example, a v1.2 client should work with v1.1, v1.2, and v1.3 master.
- [Docker](https://docs.docker.com/desktop/) >= 18.0.0
---

1. Download the repository 
  
    ```git clone git@github.com:ifernando/k8s-demo.git```

2. First have a look on go-app-custom-metrics folder to get in touch with simple go-app. It consist of a Dockerfile. and the image also published in dockerhub.

   Inside the main.go file there are some implementations to expose prometheus metrices in the go-app. shown below is a some simple example with steps to build and push Docker image using a Dockerfile

        
        docker build -t roshaneishara/k8s-demo:1.0.1 .
        
        docker images

        docker run -d -p 8080:8080 roshaneishara/k8s-demo:1.0.1

        curl localhost:8080/metrics 

        docker login 

        docker push roshaneishara/k8s-demo:1.0.1
        

3. Provision a kubernetes cluster in Azure (AKS) using terraform

    ### Prerequisites
* Terraform version >= 0.14
    * cd terraform 

    * Export credentials for Azure

    ```
    export ARM_CLIENT_ID="XXXXX"
    export ARM_SUBSCRIPTION_ID="XXXXX"
    export ARM_TENANT_ID="XXXXX"
    export ARM_CLIENT_SECRET="XXXXX"
    ```
    * apply the below commands to create and destroy aks cluster. please make sure to keep the tfstate file with you! 

    ```
    terraform init

    terraform plan

    terraform apply --out=changes 
    
    terraform output

    ```

4. Login to azure portal and get the cluster config to local machine

    * Browse azure portal and navigate to specific resource group. Now you can see aks and network is created. (you can refer the connect option inside aks to connect to created cluster.example steps are given below)
    * check whether config is set to the current context 


    ```
    az login
    
    az account set --subscription xxxxxxxx

    az aks get-credentials --resource-group xxxxxxxx --name xxxxxxxx

    kubectl config current-context
    ```

5. Deploy our simple go-app in kubernetes 

    * cd kubernetes 
    * you can apply the below commands to deploy go-app & check whether your deployemnt, service, pod are  correctly created 

    ```
    kubectl apply -f deployment-goapp.yaml

    kubectl apply -f service-goapp.yaml

    kubectl get deployment

    kubectl get service

    kubectl get pods 
    
    kubectl port-forward svc/goapp-svc 8888:30001
    
    curl localhost:8888
    
    curl http://localhost:8888/metrics

    ```
6. Deploy prometheus operator into our kubernetes cluster using Helm 

    * prometheus operator -  https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack 
    * First add the helm repo 
    * You can install chart with fixed version number as well. examples are shown below

    ```
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

    helm repo ls 
    
    helm repo update

    helm install prometheus prometheus-community/kube-prometheus-stack

    # Optional step: To install a specific chart version
    helm install prometheus prometheus-community/kube-prometheus-stack --version "9.4.1"
    ```

    * After installing , you can refer all the resources created using below command (pod, service, daemonset, deployment, replicaset, statefulset)
    * We need to create a service monitor to our go-app . go inside kubernetes folder and apply the service-monitor-go-app.yaml.

    ```
    kubectl get all 

    kubectl apply -f service-monitor-go-app.yaml

    kubectl get servicemonitor
    
    cd ../grafana
    
    kubectl apply -f goapp-dashboard.yaml
    
    ```

    * Now we can port-forward prometheus, grafana instances (admin / prom-operator) and see whether the metrics of our simple-go-app and kubernetes cluster appears on dashboards

    ```
    kubectl port-forward prometheus-prometheus-kube-prometheus-prometheus-0 9090
    
    curl localhost:9090
    
    curl http://localhost:9090/targets#pool-serviceMonitor/default/goapp/0

    kubectl port-forward svc/prometheus-grafana 8888:80
    
    curl localhost:8888
    
    # You might need to kill the grafana pod if you want the goapp-dashboard.json to be mounted
    eg: kubectl.exe delete pod prometheus-grafana-7b8657ddc9-c4ll7
    
    
    kubectl.exe  exec -it  prometheus-grafana-xyz – sh
    
    df -h
    
    cd /tmp/dashboards
    
    grep -irl 'goapp' .
    
    cat goapp-dashboard.json

    ```

7. Logging 

    * Latest helm charts can be gathered from https://github.com/elastic/helm-charts 

    ```
    export HELM_IFS_MONITORING_VER='1.12.0'
    export RELEASE_NAME="ifs-monitoring"

   helm upgrade --install $RELEASE_NAME logging --version "$HELM_IFS_MONITORING_VER" --set global.fluentd.enabled=true --set global.elasticsearch.enabled=true --set global.kibana.enabled=true --set fluentd.aggregator.extraEnv[0].name=ELASTICSEARCH_HOST,fluentd.aggregator.extraEnv[0].value=elasticsearch-master --set-string fluentd.aggregator.extraEnv[1].name=ELASTICSEARCH_PORT,fluentd.aggregator.extraEnv[1].value=9200 --set fluentd.aggregator.extraEnv[2].name=ELASTICSEARCH_PATH,fluentd.aggregator.extraEnv[2].value=/ --set fluentd.aggregator.extraEnv[3].name=ELASTICSEARCH_SCHEME,fluentd.aggregator.extraEnv[3].value=http --set kibana.elasticsearchHosts=http://elasticsearch-master:9200  --set kibana.ingress.hosts[0]=ucsc-demo.ifs.com  --set elasticsearch.ingress.hosts[0]=ucsc-demo.ifs.com -n default -f ./logging/values.yaml
    ```

    * Go inside elastic-fluentd-kibana folder apply the counter.yaml to generate simple counter logs
    ```
    kubectl apply -f counter.yaml
    ```
    
     * Accessing Elasticsearch

    ```
    kubectl.exe  port-forward svc/elasticsearch-master 9200:9200
    
    curl http://localhost:9200/_cat/
    
    http://localhost:9200/_cat/indices
    
    http://localhost:9200/_cat/health

    ```
    
    * After following below steps, you can access kibana 

    ```
    kubectl.exe  port-forward svc/kibana 5601:5601
    
    curl http://localhost:5601/kibana
   
    ```

8. Deploy java big-memory-app to test auto-scaling 

   * Go inside kubernetes folder and deploy java-bigmemoryapp.yaml

   ```
   kubectl get nodes | wc -l

   kubectl apply -f java-bigmemoryapp.yaml

   kubectl describe pod <pod-id>
   
   kubectl get nodes | wc -l
   ```

   * If you see the following error then bump the aks-cluster.tf and run terraform apply
   ```

     Normal NotTriggerScaleUp  54s cluster-autoscaler  pod didn't trigger scale-up: 1 max node group size reached

     vi aks-cluster.tf

     max_count = 7

     az login
     
     terraform plan
     
     terraform apply


   ```

9. Finally destroy the cluster
 ```
  terraform destroy

 ```
