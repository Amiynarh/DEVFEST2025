# Beyond The Region: Scaling Intelligent Features with Gemini & Cloud Run

**Welcome to DevFest Abuja 2025\!** 👋

If you are reading this, you are ready to get your hands dirty. Today, we are not just going to be deploying a container; we are building a **Global AI Architecture**.

### The Goal

We are going to build a Generative AI service that:

1.  **Lives almost everywhere:** Deployed to 3 continents simultaneously.
2.  **Knows where you are:** Automatically detects user location (without GPS permissions).
3.  **Feels instant:** Uses Google's global fiber network to route users to the nearest server.

-----

## Prerequisites

To follow along, you need:

1.  **A Google Cloud Project** (with billing enabled).
2.  **Google Cloud Shell** (Recommended\! No local setup required).
      * *Click the icon in the top right of the GCP Console that looks like a terminal prompt `>_`.*

-----

## Phase 1: The "Location-Aware" Code

We aren't building a generic chatbot. We are building a "Local Guide" that changes its personality based on where the request comes from.

### 1\. Enable the APIs

Wake up the services we need. Run this in your terminal:

```bash
gcloud services enable \
  run.googleapis.com \
  artifactregistry.googleapis.com \
  compute.googleapis.com \
  aiplatform.googleapis.com \
  cloudbuild.googleapis.com
```

### 2\. Create `main.py`

This is the brain. It looks for a specific header (`X-Client-Geo-Location`) that the Load Balancer will inject later.

```python
import os
import logging
from flask import Flask, request, jsonify
import vertexai
from vertexai.generative_models import GenerativeModel

app = Flask(__name__)

# Initialize Vertex AI
PROJECT_ID = os.environ.get("GOOGLE_CLOUD_PROJECT")
vertexai.init(project=PROJECT_ID)

@app.route("/", methods=["GET", "POST"])
def generate():
    # 1. Identify where the code is physically running (We set this ENV var later)
    service_region = os.environ.get("SERVICE_REGION", "unknown-region")
    
    # 2. Identify where the user is (Header comes from Global Load Balancer)
    # Format typically: "City,State,Country"
    user_location = request.headers.get("X-Client-Geo-Location", "Unknown Location")
    
    model = GenerativeModel("gemini-2.5-flash")
    
    # 3. Construct a location-aware prompt
    prompt = (
        f"You are a helpful local guide. The user is currently in {user_location}. "
        "Greet them warmly mentioning their city, and suggest one "
        "hidden gem activity to do nearby right now. Keep it under 50 words."
    )

    try:
        response = model.generate_content(prompt)
        return jsonify({
            "ai_response": response.text,
            "meta": {
                "served_from_region": service_region,
                "user_detected_location": user_location
            }
        })
    except Exception as e:
        return jsonify({"error": str(e)}), 500

if __name__ == "__main__":
    app.run(debug=True, host="0.0.0.0", port=int(os.environ.get("PORT", 8080)))
```

### 3\. Create the `Dockerfile`

Tell Cloud Run how to build our Python app.

```dockerfile
FROM python:3.9-slim

WORKDIR /app
COPY main.py .

# Install Flask and Vertex AI SDK
RUN pip install flask google-cloud-aiplatform

CMD ["python", "main.py"]
```

-----

## Phase 2: Build & Push

Let's package this up. We will build the container image once and store it in Google's Artifact Registry.

```bash
# 1. Set your Project ID variable
export PROJECT_ID=$(gcloud config get-value project)

# 2. Create the repository
gcloud artifacts repositories create gemini-global-repo \
    --repository-format=docker \
    --location=us-central1 \
    --description="Repo for Global Gemini App"

# 3. Prepare the Build Environment (Crucial Step! 💡). To ensure the build process only includes our necessary code and avoids including temporary files from Cloud Shell's home directory 
mkdir gemini-app
cd gemini-app
⚠️ IMPORTANT: Ensure that your main.py and Dockerfile are now located inside the gemini-app/ folder.

# 4. Build the image (This takes about 2 minutes)
gcloud builds submit --tag us-central1-docker.pkg.dev/$PROJECT_ID/gemini-global-repo/region-ai:v1
```

-----

## Phase 3: The "Triangle" Deployment

We are deploying the **exact same image** to three corners of the world. This ensures that whether a user is in Lagos, London, or Tokyo, they are close to a server.

```bash
# Define our image URL
export IMAGE_URL=us-central1-docker.pkg.dev/$PROJECT_ID/gemini-global-repo/region-ai:v1

# 1. Deploy to USA (New York)
gcloud run deploy gemini-service \
    --image $IMAGE_URL \
    --region us-east4 \
    --set-env-vars SERVICE_REGION=us-east4 \
    --allow-unauthenticated

# 2. Deploy to Europe (Belgium)
gcloud run deploy gemini-service \
    --image $IMAGE_URL \
    --region europe-west1 \
    --set-env-vars SERVICE_REGION=europe-west1 \
    --allow-unauthenticated

# 3. Deploy to Asia (Tokyo)
gcloud run deploy gemini-service \
    --image $IMAGE_URL \
    --region asia-northeast1 \
    --set-env-vars SERVICE_REGION=asia-northeast1 \
    --allow-unauthenticated
```

-----

##  Phase 4: The Global Network (The Glue)

You are now ready to execute the steps to create the Global External HTTP Load Balancer infrastructure. This is the "magic" that will give you the single, global IP address and automatically inject the user's location. We need to stitch these three services together behind a single Anycast IP Address.

### 1\. The Global IP

Create the single door for all traffic.

```bash
gcloud compute addresses create gemini-global-ip \
    --global \
    --ip-version IPV4
```

### 2\. The Network Endpoint Groups (NEGs)

These map your Cloud Run services to the Load Balancer's backend service. Create a "group" for each region so the Load Balancer knows where they are.

```bash
# USA NEG
gcloud compute network-endpoint-groups create neg-us \
    --region=us-east4 \
    --network-endpoint-type=serverless  \
    --cloud-run-service=gemini-service

# Europe NEG
gcloud compute network-endpoint-groups create neg-eu \
    --region=europe-west1 \
    --network-endpoint-type=serverless \
    --cloud-run-service=gemini-service

# Asia NEG
gcloud compute network-endpoint-groups create neg-asia \
    --region=asia-northeast1 \
    --network-endpoint-type=serverless \
    --cloud-run-service=gemini-service
```

### 3\. The Backend Service & Routing

This is the load balancer's core, distributing traffic across your regions. Connect the NEGs to a global backend.

```bash
# Create the backend service
gcloud compute backend-services create gemini-backend-global \
    --global \
    --protocol=HTTP

# Add the 3 regions to the backend
gcloud compute backend-services add-backend gemini-backend-global \
    --global --network-endpoint-group=neg-us --network-endpoint-group-region=us-east4
gcloud compute backend-services add-backend gemini-backend-global \
    --global --network-endpoint-group=neg-eu --network-endpoint-group-region=europe-west1
gcloud compute backend-services add-backend gemini-backend-global \
    --global --network-endpoint-group=neg-asia --network-endpoint-group-region=asia-northeast1
```

### 4\. The URL Map & Frontend

Finalize the connection.

```bash
# Create URL Map (Maps incoming requests to the backend service)
gcloud compute url-maps create gemini-url-map \
    --default-service gemini-backend-global

# Create HTTP Proxy (The component that inspects the request headers)
gcloud compute target-http-proxies create gemini-http-proxy \
    --url-map gemini-url-map

# Get your IP Address variable
export VIP=$(gcloud compute addresses describe gemini-global-ip --global --format="value(address)")

# Create Forwarding Rule (Open port 80)
gcloud compute forwarding-rules create gemini-forwarding-rule \
    --address=$VIP \
    --global \
    --target-http-proxy=gemini-http-proxy \
    --ports=80
```

-----

##  Phase 5: Testing (Teleportation Time)

Global Load Balancers take about **5-7 minutes** to propagate worldwide. Take a breather. Drink some water. 🥤

This is where you verify that the Global Load Balancer is working correctly:
- Using the single VIP (Virtual IP) address.
- Routing traffic to the nearest server.
- Injecting the X-Client-Geo-Location header to tell your code where the user is.

### 1\. Get your Global IP

```bash
echo "http://$VIP/"
```

### 2\. Test "Teleportation"

We will use `curl` to spoof our location headers. This simulates what the Load Balancer does automatically for real users.

**Simulate Paris:**

```bash
curl -s -H "X-Client-Geo-Location: Paris,France" http://$VIP/ | jq .
```

*Expected Output:* Gemini should say "Bonjour" and mention Paris. The `served_from_region` should be `europe-west1`.

**Simulate Tokyo:**

```bash
curl -s -H "X-Client-Geo-Location: Tokyo,Japan" http://$VIP/ | jq .
```

*Expected Output:* Gemini should mention Tokyo. The `served_from_region` should be `asia-northeast1`.

**Simulate Abuja:**
```bash
curl -s -H "X-Client-Geo-Location: Abuja,Nigeria" http://$VIP/ | jq .
```

Note: The | jq . part is optional, but highly recommended as it formats the JSON output, making it much easier to read the served_from_region and ai_response details. If jq isn't available, you can just run curl ... without it.

-----

##  Cleanup

Don't leave the meter running\!

```bash
gcloud run services delete gemini-service --region us-east4 --quiet
gcloud run services delete gemini-service --region europe-west1 --quiet
gcloud run services delete gemini-service --region asia-northeast1 --quiet
gcloud compute forwarding-rules delete gemini-forwarding-rule --global --quiet
gcloud compute addresses delete gemini-global-ip --global --quiet
```

-----

###  Congratulations\!

You just built a global, latency-optimized, context-aware AI architecture. You are no longer limited by a single region.

**Enjoy the rest of DevFest Abuja\!** 🇳🇬
