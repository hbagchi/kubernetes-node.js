# Kubernetes Node.js

How to containerize a **Node.js** application with **Docker** and **Kubernetes** on Google Cloud Platform (GCP).

# Introduction

This is a Catalog Service developed using Express on Node.js. It consists of 2 versions of the catalog service.

<u>V1</u> - 2 GET service endpoints ('/' and '/catalog') that returns hardcoded json

/ => Prints 'Catalog Service'

/catalog => Returns a hardcoded json

<u>V2</u> - 3 GET service endpoints ('/',  '/catalog' and '/intense') that returns hardcoded json

/intense => Generates CPU load to demo hpa auto scaling

Please set environment variable **PROJECT_ID** to your GCP project name

# Steps

1. Build the catalog docker images. Ensure your current directory is set to the correct path to build both images.

   ```
   docker build -t catalog-image .
   ```

2. Build catalog docker images with gcr tag:

   ```
   docker build -t gcr.io/$PROJECT_ID/catalog-image:v1 .
   docker build -t gcr.io/$PROJECT_ID/catalog-image:v2 .
   ```

3. Configure gcloud to use gcp docker service

   ```
   gcloud auth configure-docker
   ```

4. Push catalog images to gcr:

   ```
   docker -- push gcr.io/$PROJECT_ID/catalog-image:v1
   docker -- push gcr.io/$PROJECT_ID/catalog-image:v2
   ```

5. View the images in gcr:

   ```
   gcloud container images list --repository gcr.io/$PROJECT_ID
   gcloud container images list-tags gcr.io/$PROJECT_ID/catalog-image --repository gcr.io/$PROJECT_ID
   ```

6. Create a kubernetes cluster and connect to it from the command line. 

   ```
   gcloud container clusters create [cluster_name] --zone [zone] --machine-type g1-small
   gcloud container clusters get-credentials [cluster_name] --zone [zone]
   ```

   You can also create this cluster from the Google Cloud Console. I have used 'g1-small' machines. Other larger machine type is also fine.

7. Deploy the catalog image v1 version to GCP kubernetes cluster. Your current directory should be 'catalog-service/'. 
   Edit the image path in the yaml file to reflect your images.

   ```
   kubectl apply -f catalog_deployment.yml
   ```

8. Deploy the v1 version of the catalog kubernetes service:

   ```
   kubectl apply -f catalog_service.yml
   ```

9. Verify that 2 catalog pods are created. 

   ```
   kubectl get po
   kubectl describe deploy catalog-deploy
   ```

10. Verify that a catalog-service is created.

    ```
    kubectl get svc
    ```

    In the output, find the EXTERNAL_IP value for the catalog-service. If the field shows as  "&lt;pending&gt;" keep running the command till the value is populated. For LoadBalancer type services, a cloud load balancer is spawned and associated with the kubernetes service. So it takes a while for the IP address to display.

11. Curl for the catalog service using the LOAD_BALANCER ip address

    ```
    curl -s http://[LoadBalancer IP Address]:3000/catalog
    ```

12. Deploy catalog image v2:

    ```
    kubectl set image deployment/catalog-deploy catalog=gcr.io/$PROJECT_ID/catalog-image:v2 --record
    ```

13. Verify that new pods are being spawned and old pods are being terminated.

    ```
    kubectl get po -w
    ```

    Add the -w (watch) option to continuously auto refresh the display.

14. Curl for the catalog service '/catalog' endpoint using the LOAD_BALANCER ip address

    ```
    curl -s http://[LoadBalancer IP Address]:3000/catalog
    ```

    You will see a new field '*stock*' appear in the json output. 

15. Deploy the horizontal pod autoscaler:

    ```
    kubectl apply -f catalog_hpa.yml
    ```

16. View the hpa:

    ```
    kubectl get hpa
    ```

17. Curl the service endpoint:

    ```
    curl -s http://[LoadBalancer_IP Address]:3000/intense
    ```

18. View the pods:

    ```
    kubectl get po
    ```

    You will see 2 new pods spawned to address the extra load on the cpu generated by the '/intense' endpoint

