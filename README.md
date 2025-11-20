# Beyond the Region: Scaling Intelligent Features with Cloud Run and Gemini's Global Network 


Welcome to this self-paced lab guide\! In modern application development, **resilience** and **low latency** are non-negotiable, especially for intelligent features powered by generative AI. This workshop moves beyond single-region deployments and shows you how to leverage Google Cloud's **Global External Load Balancing (GELB)** to create a single, globally accessible, and fault-tolerant endpoint for a service powered by the Gemini API.

  * **Main Goal:** Deploy a single, intelligent feature using Gemini Flash across two distinct Google Cloud regions and expose it via a single, low-latency Global IP address.
  * **Target Audience:** Intermediate developers familiar with Python, Docker, and basic cloud concepts.
  * **Estimated Time:** 45–60 minutes (Self-Paced).

### Prerequisites

You must have the following to complete this lab:

  * **Google Colab Account**
  * **Gemini API Key:** (Get one from Google AI Studio).
  * **Google Cloud Project:** A project with **Billing Enabled**. (Cloud Run and Load Balancing incur minor, pay-per-use costs. Using Cloud Run's "scale to zero" feature ensures cost efficiency when idle.)
  * **Google Cloud Shell access**

-----

## Part 1: The Code (Local Test)

Before deploying to the cloud, the best practice is to isolate and test the core logic. Our core logic is a simple web handler that uses the Gemini API.

### Step 1: Confirming the AI Logic in Colab

We use **Google Colab** for this initial step because it provides a zero-setup environment to **isolate the AI logic** and confirm the functionality of the Python code *before* we containerize and deploy it. This separates AI-related bugs from deployment-related issues.

1.  Open the dedicated Colab environment: **`[[YOUR COLAB LINK](https://colab.research.google.com/)]`**
2.  Follow the instructions in the notebook to:
      * Install the `google-genai` SDK.
      * Set your Gemini API Key.
      * Run the sample code, which defines a simple web handler using the **`gemini-2.5-flash`** model to perform a non-compute-intensive task (e.g., a sentiment check or pre-query validation).

> **Cost-Saving Edition:** We are using **Gemini 2.5 Flash** because it is the optimal model for low-latency, cost-sensitive edge functions like this. Its speed and efficiency make it perfect for features that must scale rapidly.


## Part 2: Cloud Run Deployment (Initial Service)

Our intelligent feature will be hosted on **Cloud Run**, a serverless platform that automatically scales containers.

### Step 2: Configure and Deploy to Region A (`us-central1`)

Open your Google Cloud Shell (or a local environment with the gcloud CLI installed).

1.  **Set Your Project ID:** Replace `[YOUR_PROJECT_ID]` with your actual Google Cloud Project ID.

    ```bash
    gcloud config set project [YOUR_PROJECT_ID]
    ```

2.  **Enable Necessary APIs:**

    ```bash
    gcloud services enable run.googleapis.com \
        compute.googleapis.com
    ```

3.  **Deploy the Service (Region A):** We will use a simple, pre-built Cloud Run sample image for this lab to represent our containerized Python code from Part 1. The key takeaway is the deployment command.

    *We'll deploy to **Region A**: `us-central1`.*

    ```bash
    # Set the service name
    export SERVICE_NAME="gemini-service"
    export REGION_A="us-central1"

    # Deploy the container to Region A
    gcloud run deploy $SERVICE_NAME \
        --image us-docker.pkg.dev/cloudrun/container/hello \
        --region $REGION_A \
        --allow-unauthenticated \
        --platform managed \
        --ingress all
    ```

    Take note of the **Service URL** output, though we will soon be using the Global Load Balancer IP.



## Part 3: Global Resilience (Multi-Region Setup)

The "Beyond the Region" strategy is realized by deploying the same service to a second region and unifying them behind a single, intelligent entry point.

### Step 3: Deploy to Region B (`europe-west1`)

We deploy the exact same service, with the exact same container, to a second geographically distant region.

*We'll deploy to **Region B**: `europe-west1`.*

```bash
export REGION_B="europe-west1"

# Deploy the identical container to Region B
gcloud run deploy $SERVICE_NAME \
    --image us-docker.pkg.dev/cloudrun/container/hello \
    --region $REGION_B \
    --allow-unauthenticated \
    --platform managed \
    --ingress all
```

### Step 4: Configure Global External Load Balancing (GELB)

**Why GELB?**
The **Global External Load Balancer (GELB)** is critical here. It uses Google's global network to **terminate traffic near the user** and then routes that traffic to the **closest healthy Cloud Run service**. This provides a single, low-latency endpoint and fault tolerance—if one region fails, traffic seamlessly shifts to the other.

We must configure the GELB to point to our two Cloud Run services via **Serverless Network Endpoint Groups (NEGs)**.

1.  **Create Serverless NEG for Region A (`us-central1`):**

    ```bash
    export NEG_A_NAME="neg-us-central1"

    gcloud compute network-endpoint-groups create $NEG_A_NAME \
        --region $REGION_A \
        --network-endpoint-type serverless \
        --cloud-run-service $SERVICE_NAME
    ```

2.  **Create Serverless NEG for Region B (`europe-west1`):**

    ```bash
    export NEG_B_NAME="neg-europe-west1"

    gcloud compute network-endpoint-groups create $NEG_B_NAME \
        --region $REGION_B \
        --network-endpoint-type serverless \
        --cloud-run-service $SERVICE_NAME
    ```

3.  **Create Global Backend Service:** This service defines how traffic is distributed and associates the two NEGs.

    ```bash
    export BACKEND_NAME="global-backend"

    gcloud compute backend-services create $BACKEND_NAME \
        --global

    # Add Region A NEG to the Backend Service
    gcloud compute backend-services add-backend $BACKEND_NAME \
        --global \
        --network-endpoint-group $NEG_A_NAME \
        --network-endpoint-group-region $REGION_A

    # Add Region B NEG to the Backend Service
    gcloud compute backend-services add-backend $BACKEND_NAME \
        --global \
        --network-endpoint-group $NEG_B_NAME \
        --network-endpoint-group-region $REGION_B
    ```

4.  **Create URL Map, Target Proxy, and Global Forwarding Rule:** These steps create the actual global endpoint.

    ```bash
    export URL_MAP_NAME="gemini-url-map"
    export TARGET_HTTP_PROXY="gemini-target-proxy"
    export FORWARDING_RULE="gemini-global-ip"

    # 4a. Create URL Map
    gcloud compute url-maps create $URL_MAP_NAME \
        --default-service $BACKEND_NAME

    # 4b. Create Target HTTP Proxy
    gcloud compute target-http-proxies create $TARGET_HTTP_PROXY \
        --url-map $URL_MAP_NAME

    # 4c. Create Global Forwarding Rule (This creates your Global IP)
    gcloud compute forwarding-rules create $FORWARDING_RULE \
        --global \
        --target-http-proxy $TARGET_HTTP_PROXY \
        --ports 80
    ```


## Part 4: Verification & Deliverables

### Step 5: Test the Global IP and Verify Failover

1.  **Retrieve Your Global IP Address:** Wait a few moments for the load balancer to become active (it can take several minutes). Then, retrieve the IP address.

    ```bash
    gcloud compute forwarding-rules describe $FORWARDING_RULE --global --format="value(IPAddress)"
    ```

    Copy the IP address. This is your single, low-latency, fault-tolerant endpoint\!

2.  **Initial Verification Test:** Execute a `curl` command against your new Global IP. You should receive a healthy response from the Cloud Run service.

    ```bash
    # Replace [YOUR_GLOBAL_IP] with the IP you retrieved
    curl http://[YOUR_GLOBAL_IP]

    # Expected output (from the sample service): "Hello World!"
    ```

3.  **Verify Failover:** To confirm the **fault-tolerance**, let's simulate a regional outage by stopping the service in Region A (`us-central1`).

    ```bash
    # Set the service to use a placeholder image that immediately fails or serves no content
    gcloud run services update $SERVICE_NAME \
        --image gcr.io/cloudrun/service-broken \
        --region $REGION_A

    # Wait ~60 seconds for the load balancer's health check to fail for us-central1
    echo "Waiting for health check failure... (60s)"
    sleep 60

    # Re-test the Global IP
    curl http://[YOUR_GLOBAL_IP]
    ```

    The request will now be seamlessly routed to the only healthy backend, the service in **`europe-west1`** (Region B). Your Gemini-powered feature remains available without any change to the client's single Global IP address.

> **Cleanup Tip:** To return the service to its original state (for cleanup), re-deploy the healthy image to Region A:
> `gcloud run deploy $SERVICE_NAME --image us-docker.pkg.dev/cloudrun/container/hello --region $REGION_A --allow-unauthenticated --platform managed --ingress all`

### Deliverables Checklist

Upon successful completion, you will have built:

  * ☑️ A containerized, **Gemini Flash** powered service running on **Cloud Run**.
  * ☑️ Multi-region deployment across **`us-central1`** and **`europe-west1`**.
  * ☑️ A single, low-latency **Global IP** endpoint managed by **GELB**.
  * ☑️ Verified **fault-tolerance** through a manual failover test.

-----

## Next Steps

Congratulations on building a truly global and resilient intelligent feature\!

  * **Custom Domain:** Map a custom domain (e.g., `api.yourcompany.com`) to the Global IP address using Google Cloud DNS.
  * **Terraform/IaC:** Convert the `gcloud` commands into **Terraform** or another Infrastructure-as-Code tool to manage the infrastructure declaratively.
  * **Custom Containers:** Replace the sample container with the actual application code from your Colab test (Part 1).
