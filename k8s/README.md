## Installation and Testing Instructions

### Prerequisites

* A running Kubernetes cluster (e.g., Minikube, a cloud-based cluster).
* `kubectl` configured to communicate with your cluster.
* Knative Serving and Kourier installed on your cluster.

### Deployment

1.  **Deploy MongoDB:**
    ```bash
    kubectl apply -f k8s/mongo-deployment.yaml
    ```

2.  **Deploy the microservices:**
    ```bash
    kubectl apply -f k8s/api-get-books.yaml
    kubectl apply -f k8s/api-post-books.yaml
    kubectl apply -f k8s/api-put-books.yaml
    kubectl apply -f k8s/api-delete-books.yaml
    kubectl apply -f k8s/frontend-renderer.yaml
    ```

### Testing

1.  **Get the URL of the frontend service:**
    ```bash
    kubectl get ksvc frontend-renderer
    ```
    The output will show the URL of the service.

2.  **Access the application:**
    Open the URL in your browser. You should see the book store application.

3.  **Test the API endpoints:**
    You can use `curl` or a tool like Postman to test the API endpoints. The URLs for the API services can be retrieved using `kubectl get ksvc`. For example, to get the URL for the `api-get-books` service:
    ```bash
    kubectl get ksvc api-get-books
    ```
