# GSP301 — Configure a VM to Use a Remote Startup Script

A hands-on Google Cloud Skills Boost lab documenting how to store a startup script in Cloud Storage, attach it to a Compute Engine VM through instance metadata, and validate the installed Apache web server over HTTP.

## Lab objective

Deploy a Linux Compute Engine VM that downloads and executes a startup script stored in a Cloud Storage bucket. Configure network access for HTTP and verify that the web server is responding.

## Services and concepts

- Google Cloud Storage
- Compute Engine
- VM startup-script URL metadata
- VPC firewall rules and network tags
- Debian Linux and Apache HTTP Server

## Configuration used

| Component | Value |
|---|---|
| Lab | GSP301 — Configure VM to Use a Remote Startup Script |
| Project ID | Omitted (temporary Skills Boost project) |
| Cloud Storage bucket | Project ID used as bucket name |
| Startup script source | gs://spls/gsp301/install-web.sh |
| Uploaded script object | gs://<BUCKET_NAME>/install-web.sh |
| VM name | apache-vm |
| Zone used | us-west1-b |
| Machine type | e2-medium |
| Image family | debian-12 |
| Image project | debian-cloud |
| Network tag | http-server |
| Firewall rule | allow-http (TCP port 80) |
| Web server | Apache HTTP Server |

The zone shown is the zone used in this completed lab session. Use the zone provided by your own lab environment when repeating the exercise; organization location policies can restrict available zones.

## Implementation

### 1. Set project and bucket variables

~~~bash
export PROJECT_ID="$(gcloud config get-value project)"
export BUCKET_NAME="$PROJECT_ID"
~~~

Create the bucket if it has not already been created. Use a location allowed by the lab's organization policy and the lab instructions:

~~~bash
gcloud storage buckets create "gs://$BUCKET_NAME" --location=US
~~~

### 2. Copy the lab startup script to the bucket

The script supplied for this lab is stored at gs://spls/gsp301/install-web.sh.

~~~bash
gcloud storage cp gs://spls/gsp301/install-web.sh \
  "gs://$BUCKET_NAME/install-web.sh"

gcloud storage ls "gs://$BUCKET_NAME/"
~~~

### 3. Create the Compute Engine VM

~~~bash
export ZONE=us-west1-b

gcloud compute instances create apache-vm \
  --zone="$ZONE" \
  --machine-type=e2-medium \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --metadata=startup-script-url="gs://$BUCKET_NAME/install-web.sh" \
  --tags=http-server
~~~

The startup-script-url metadata value tells the VM to retrieve and run the remote startup script during boot.

### 4. Allow HTTP traffic

~~~bash
gcloud compute firewall-rules create allow-http \
  --allow=tcp:80 \
  --target-tags=http-server \
  --description="Allow HTTP traffic to Apache web server"
~~~

The firewall rule applies to VM instances carrying the http-server network tag.

### 5. Verify the VM and web server

Check the VM status:

~~~bash
gcloud compute instances list
~~~

Get the current external IP:

~~~bash
gcloud compute instances describe apache-vm \
  --zone="$ZONE" \
  --format="get(networkInterfaces[0].accessConfigs[0].natIP)"
~~~

Test the HTTP endpoint (replace the address with the current external IP):

~~~bash
curl -I --max-time 20 http://EXTERNAL_IP
~~~

## Observed result

The HTTP check against the lab VM returned:

~~~text
HTTP/1.1 200 OK
Server: Apache/2.4.68 (Debian)
Content-Type: text/html
~~~

This confirmed that the remote startup script had installed and started Apache and that HTTP traffic was reaching the VM.

## Verification checklist

- [x] Startup script copied into the lab Cloud Storage bucket.
- [x] apache-vm created and reached the RUNNING state.
- [x] VM configured with startup-script-url metadata.
- [x] allow-http firewall rule created for TCP port 80.
- [x] HTTP request returned 200 OK from Apache.

## Notes and safe reuse

- Bucket names are globally unique. Choose an available name if using a different project.
- The example bucket creation command uses the US multi-region; follow the lab's required location and applicable organization policy.
- Zone availability is organization-policy dependent. Use an allowed zone from your lab instructions.
- Do not publish temporary project IDs, VM external IPs, credentials, service-account keys, or other secrets.
- This repository records the lab workflow. It is not a production-ready web-server deployment.
