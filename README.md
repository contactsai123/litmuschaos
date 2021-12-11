# GTR Agile Alliance 2021 Lab Session 032 - The Know-Hows in Resilience & Reliability Testing for building an anti-fragile & highly scalable system 
## Handout for setting up and running Litmus Chaos experiments on Sock-Shop application
## Pre-Requisite:
1) Provision a AWS Linux EC2 (t3.small) and connect using SSH client like Putty
2) Configure the API key, default region etc., using AWS Configure command
```aws configure```
3) Install [kubectl 1.21](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)
4) Install [eksctl](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html). We use eksctl which is a CLI tool for creating clusters on Amazon’s Elastic Kubernetes Service (EKS) - a managed Kubernetes service for EC2. Update the region as per the requirement:

```eksctl create cluster --name sockshop-eks --version 1.21 --region us-east-2 --nodegroup-name standard-workers --node-type t3.medium --nodes 3 --nodes-min 1 --nodes-max 3```

4) Ensure all the required ports like 3000 (Grafana), 9091 (Prometheus), 9001 (Chaos front end portal), 9002 (Chaos portal service), 80 (sock-shop) are opened under AWS Security Groups - Inbound rules
5) Install git using the command ```sudo yum install git```
6) Ensure to clean up / decomission all the AWS resources once the experiments are completed (Refer Section 4), as it may incur signficant costs if left running

Estimated time to provision all the required machines, softwares and sucessfully run experiments: Approx 60 - 90 minutes.

## Section 1 - Sock shop installation:

Sock shop is a demo microservices e-commerce application. Sock Shop simulates the user-facing part of an e-commerce website that sells socks. It is intended to aid the demonstration and testing of microservice and cloud native technologies.

1)	Clone the github repo and go into the deploy/kubernetes folder.
```git clone https://github.com/microservices-demo/microservices-demo```
2)	Create a new namespace
```kubectl create namespace sock-shop```
3)	Update NodePort to LoadBalancer in complete-demo.yaml for front-end service
```kubectl apply -f complete-demo.yaml```
Switch region to us-west-2 through aws configure
4)	Get the pods in the sock-shop namespace:
```kubectl get pods -n sock-shop```
5)	Get services through LoadBalancer. ```kubectl get svc -n sock-shop```. Open the front-end service URL in browser.


## Section 2 – Litmus installation for Chaos experiments:

Litmus is an open source Chaos Engineering platform that enables teams to identify weaknesses & potential outages in infrastructures by inducing chaos tests in a controlled way.

1)	Download the litmus yaml file in a new directory called litmuschaos:
```wget https://litmuschaos.github.io/litmus/2.3.0/litmus-2.3.0.yaml```
2)	Create a new namespace called litmus:
```kubectl create namespace litmus```
3)	Update both the frontend-service and litmusportal-server-service to LoadBalancer
```kubectl apply -f limus-2.3.0.yaml```
4)	List the litmus pods and services:
```kubectl get pods -n litmus```
```kubectl get svc -n litmus```
Open the litmus front end URL listed in browser along with the port number

5)	Once you login to front-end portal (admin/litmus) and click on ChaosAgents tab..additional pods likechaos-operator will get installed and run. By default self-service agent will be in non-active status.  To enable to Active status, run the below command:
```kubectl edit configmap agent-config -n litmus```
in the SERVER_ADDR update the litmus-portal-service URL embedded with double quotes in the URL and save it. For e.g. "http://a33a1d3df22d74be4b4b604aadf08f2a-1542679795.us-east-2.elb.amazonaws.com:9002/query"
Please note the above URL will be different based on the litmus-portal-server service URL
6)	restart the kubectl services:
```kubectl apply -f litmus-2.3.0.yaml```
7)	Running the Chaos Workflows:
Once the self-agent is active:
Create Workflow:
a) Choose Agent -> Self-service agent displayed
b) Choose a Workflow  -> Create a new workflow using the experiments from ChaosHubs
c) Workflow Settings -> Provide Workflow name, description
d) Tune Workflow -> Add a new experiment. In this we are invoking network loss for Catalogue pod in Sock-shop application
	generic/pod-network-loss
	Click Edit -> General-> Target Application -> appns: sock-shop appkind: deployment applabel: name=catalogue jobcleanup: retain
	Define the steady state for this application -> leave blank
	Tune Experiment -> leave default

8)	To confirm whether the container run time is docker:
```kubectl describe pod -n sock-shop```
9)	Cleanup chaos resource: False
10)	Click Finish and schedule now. During the chaos experiment you will see the Catalogue will not be loaded for the chaos experiment period in Sock-shop front end URL.


## Section 3 – Observability dashboards using Grafana and Prometheus

1)	Create a new namespace called monitoring:
```kubectl create ns monitoring```
2)	In the same litmuschaos directory created in the last section, Go to monitoring directory in limus/monitoring folder:

3)	Run the following commands:
For Prometheus set up, run the below command:
```kubectl -n monitoring apply -f utils/prometheus/prometheus-scrape-configuration/```
https://github.com/litmuschaos/litmus/tree/master/monitoring#setup-prometheus-tsdb

4) For Grafana set up, run the below command:
```kubectl -n monitoring apply -f utils/grafana/```

5) Incase you need to change any default Prometheus / Grafanaconfigurations:
```kubectl get configmaps -n monitoring```
```kubectl get configmaps prometheus-configmap -n monitoring -o yaml | less```

6) List the pods and services:
```kubectl get pods -n monitoring```
```kubectl get svc -n monitoring```

7) Open the Grafana URL with port number 3000. Login with default credentials in Grafana: admin/admin. By default, choose the default Sock-shop dashboard that is pre-available in grafana.
In Grafana -> click settings on top right -> Annotation -> LitmusChaos_Metrics:
Replace the search expression: litmuschaos_cluster_scoped_awaited_experiments{app="chaos-exporter", job="litmus/chaos-exporter"}

As a final step, when you run the chaos tests in the chaos center portal, you can monitor the corresponding impact in the Grafana dashboards.

## Section 4 – Cleanup

Clean up / Decomission all the AWS resources
```eksctl delete cluster --name=sockshop-eks --region=us-east-2```

Goto AWS console once and verify if there are no active CloudFormation / EC2 / LoadBalancers running and delete if any
