Here are the complete, step-by-step manual instructions to set up the automated object reporting, Cloudflare Worker proxy, Oracle Cloud Infrastructure (OCI) Function, and SMTP notification system.
------------------------------
## Step 1: Create an OCI Object Storage Bucket and PAR

   1. Log in to the OCI Cloud Console.
   2. Open the navigation menu, go to Storage, and click Buckets.
   3. Select your Compartment and click Create Bucket. Name your bucket (e.g., report-bucket) and leave other settings as default. Click Create.
   
   5. Click on your newly created bucket name to open its details page.
   6. Under Resources (bottom left), click Pre-Authenticated Requests (PAR).
   7. Click Create Pre-Authenticated Request.
   8. Configure the PAR:
   * Name: worker-access-par
      * Bucket/Object: Select Bucket (this targets the entire bucket).
      * Permissions: Select Permit object reads and room listing (ensures reading and listing are enabled).
      *  Make sure to enable the toggle object listing other wise url wouldn't show any object from the respective bucket 
      * Expiration: Set a far-future expiration date according to your requirements.
   8. Click Create Pre-Authenticated Request.
   9. CRITICAL: Copy the generated Pre-Authenticated Request URL immediately. It will not be shown again. (Example format: https://objectstorage.<region>://<token>/n/<namespace>/b/<bucket>/o/)

------------------------------
## Step 2: Configure Bucket Lifecycle Policy (Auto-Delete after 23 Hours)
To ensure the bucket automatically empties objects 23 hours after creation:

   1. Inside your bucket details page under Resources, click Lifecycle Policy Rules.
   2. Click Create Rule.
   3. Configure the rule:
   * Name: delete-after-23-hours
      * Target Type: Objects
      * Action: Delete
      * Lifecycle Age: 23
      * Time Unit: Hours
   4. Click Create.

------------------------------
## Step 3: Create a Cloudflare Worker Proxy

   1. Log in to your [Cloudflare Dashboard](https://dash.cloudflare.com/).
   2. Navigate to Workers & Pages in the left sidebar and click Create.
   3. Select Create Worker, name it (e.g., oci-bucket-proxy), and click Deploy.
   4. Click Edit Code. Paste the following optimized code into your worker.js file:

```code


export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    const objectPath = url.pathname;

    // 1. ROOT GUARD: Stop bare URL access instantly
    if (objectPath === "/" || objectPath === "" || objectPath === "/index.html") {
      return new Response(
        `<!DOCTYPE html>
<html>
<head><title>404 Not Found</title></head>
<body style="font-family: sans-serif; text-align: center; padding: 50px;">
  <h1>404 Not Found</h1>
  <p>The requested URL was not found on this server.</p>
</body>
</html>`,
        {
          status: 404,
          headers: { "Content-Type": "text/html; charset=utf-8" }
        }
      );
    }

    // 2. READ SECURE ENVIRONMENT VARIABLE
    // env.OCI_PAR_BASE binds directly to the secret variable you created in Step 1
    const OCI_PAR_BASE = env.OCI_PAR_BASE;

    // Safety fallback: if you forgot to add the environment variable, drop a 500 error
    if (!OCI_PAR_BASE) {
      return new Response("Configuration Error: Missing Secret Key", { status: 500 });
    }

    try {
      // 3. SMART URL STITCHING: Combines PAR base with object name safely
      const ociTargetUrl = new URL(objectPath.substring(1), OCI_PAR_BASE).toString();

      // 4. Fetch the specific file from Oracle behind the scenes
      const ociResponse = await fetch(ociTargetUrl, {
        method: request.method,
        headers: request.headers,
        redirect: "follow"
      });

      // 5. If file doesn't exist in Oracle, surface clean 404
      if (!ociResponse.ok) {
        return new Response("Not Found", { status: 404 });
      }

      // 6. Return the file stream back to the browser seamlessly
      const responseHeaders = new Headers(ociResponse.headers);
      return new Response(ociResponse.body, {
        status: ociResponse.status,
        statusText: ociResponse.statusText,
        headers: responseHeaders
      });

    } catch (error) {
      return new Response("Internal Server Error", { status: 500 });
    }
  }
};



```

   1. Click Deploy (top right).
   2. Go back to your Worker's settings page by clicking the back arrow next to the project name.
   3. Go to the Settings tab → Variables.
   4. Under Environment Variables, click Add.
   5. Set the key Name as OCI_PAR_BASE and paste your copied Oracle Bucket PAR URL into the Value field.
   6. Click Save and deploy.
   7. Copy your Worker's Production URL (e.g., https://oci-bucket-proxy.<your-subdomain>.workers.dev).

------------------------------
## Step 4: Setup OCI Email Delivery (SMTP, SPF, and DKIM)## A. Configure SMTP Credentials

   1. In the OCI Console, open the navigation menu and go to Developer Services → Application Integration → Email Delivery.
   2. Click SMTP Credentials in the left menu.
   3. Click Create SMTP Credentials. Enter a description (e.g., function-email-sender) and click Create.
   4. Copy the Username and Password provided immediately.
   5. In the left menu under Email Delivery, click Configuration. Note down your SMTP Server (Host) and Ports (usually 25 or 587).

## B. Setup Approved Senders & SPF/DKIM

   1. In the Email Delivery menu, click Approved Senders.
   2. Click Create Approved Sender. Enter the email address you want to use as the SENDER_EMAIL and click Create.
   3. Navigate to Networking → DNS Management → Zones in OCI (or your custom domain provider like Cloudflare) to add your records:
   * SPF: Add a TXT record on your root domain enabling OCI delivery. Example: v=spf1 include:://oracleemaildelivery.com ~all
      * DKIM: In OCI Email Delivery, generate a DKIM key for your domain under DKIM Keys, then copy the generated TXT record name and value into your domain's DNS manager.
  ##Before use make sure to test the SMTP credentials on GMass or any SMTP tester so that it is working properly  
------------------------------
## Step 5: Gather All Environment Variables
Before proceeding to the function deployment, compile your configuration list:

* 
* WORKER_BASE_URL: The Cloudflare worker URL (e.g., https://workers.dev)
* 
* OCI_PAR_URL: The full URL string generated from Step 1.
* SMTP_HOST: The endpoint gathered from Email Delivery Configuration.
* 
* SMTP_USER: The generated OCID-based SMTP username.
* 
* SMTP_PASS: The generated SMTP password.
* 
* SMTP_PORT: 587 (Recommended for TLS authentication).
* 
* SENDER_EMAIL: The verified Approved Sender email.
* 
* RECEIVER_EMAIL: The destination address where notifications should land.
* WORKER_PAR_URL created in begging or this article 

------------------------------
## Step 6: Create, Configure, and Deploy the OCI Function

   1. In the OCI Console, go to Developer Services → Functions → Applications.
   2. Click Create Application. Name it, select your VCN/Subnets, and click Create.
   3. Open your Cloud Shell or local OCI CLI terminal and initialize a boilerplate python function project:
   
   fn init --runtime python oci-email-notifier
   cd oci-email-notifier
   
   4. Replace the contents of func.py with an implementation that connects to the bucket, fetches the newest object, and fires the email:

```code


cat << 'EOF' > func.py
import io
import json
import logging
import os
import urllib.request
import smtplib
from email.message import EmailMessage
from fdk import response

def handler(ctx, data: io.BytesIO = None):
    logging.getLogger().info("Oracle Function triggering custom SMTP execution plane...")
    
    # 1. RETRIEVE ENVIRONMENT SETTINGS
    WORKER_BASE_URL = os.environ.get("WORKER_BASE_URL")
    BUCKET_PAR_URL = os.environ.get("BUCKET_PAR_URL")
    
    # CUSTOM SMTP CONFIGURATION VARIABLES
    SMTP_HOST = os.environ.get("SMTP_HOST")       # e.g., ://yourserver.com
    SMTP_PORT = os.environ.get("SMTP_PORT", 587) # Default to standard TLS port 587
    SMTP_USER = os.environ.get("SMTP_USER")       # Mail server login username
    SMTP_PASS = os.environ.get("SMTP_PASS")       # Mail server login password
    SENDER_EMAIL = os.environ.get("SENDER_EMAIL")   # Authorized From: email address
    RECEIVER_EMAIL = os.environ.get("RECEIVER_EMAIL") # Destination inbox address
    
    if not all([WORKER_BASE_URL, BUCKET_PAR_URL, SMTP_HOST, SMTP_USER, SMTP_PASS, SENDER_EMAIL, RECEIVER_EMAIL]):
        return response.Response(ctx, response_data="Configuration Error: Missing environment variables.", status_code=500)
    
    try:
        # 2. SANITIZE BUCKET PAR URL PATH FORMAT
        if not BUCKET_PAR_URL.endswith('/o/') and not BUCKET_PAR_URL.endswith('/o'):
            if BUCKET_PAR_URL.endswith('/'):
                BUCKET_PAR_URL = BUCKET_PAR_URL + "o/"
            else:
                BUCKET_PAR_URL = BUCKET_PAR_URL + "/o/"
        elif BUCKET_PAR_URL.endswith('/o'):
            BUCKET_PAR_URL = BUCKET_PAR_URL + "/"

        # 3. HTTP GET REQUEST TO THE BUCKET PAR URL
        req_bucket = urllib.request.Request(BUCKET_PAR_URL, headers={'User-Agent': 'Mozilla/5.0'})
        with urllib.request.urlopen(req_bucket) as res:
            response_data = json.loads(res.read().decode('utf-8'))
            
        objects = response_data.get("objects", [])
        if not objects:
            return response.Response(ctx, response_data="Bucket is completely empty via PAR access.", status_code=200)
        
        # 4. CHRONOLOGICAL SORTING (NEWEST FIRST)
        objects.sort(key=lambda obj: obj.get("timeCreated", ""), reverse=True)
        latest_object = objects[0]
        object_name = latest_object.get("name")
        time_created = latest_object.get("timeCreated")
        
        # 5. STITCH CLEAN CLOUDFLARE WORKER LINK
        final_shareable_url = f"{WORKER_BASE_URL.rstrip('/')}/{object_name}"
        
        # 6. ASSEMBLE STANDARD SMTP MAIL CONTAINER
        msg = EmailMessage()
        msg['Subject'] = f"🚀 Custom SMTP Asset Alert: {object_name}"
        msg['From'] = SENDER_EMAIL
        msg['To'] = RECEIVER_EMAIL
        
        email_body = (
            f"The automated pipeline has scanned your bucket via your secure PAR URL configuration.\n\n"
            f"Latest File Name Identified: {object_name}\n"
            f"Time Created: {time_created}\n\n"
            f"Your Secure Cloudflare Worker Link:\n{final_shareable_url}"
        )
        msg.set_content(email_body)
        
        # 7. ESTABLISH TLS CONNECTION AND TRANSMIT
        port_int = int(SMTP_PORT)
        with smtplib.SMTP(SMTP_HOST, port_int) as server:
            server.ehlo()
            if port_int == 587:
                server.starttls() # Initiate secure handshake if utilizing standard submission port
                server.ehlo()
            server.login(SMTP_USER, SMTP_PASS)
            server.send_message(msg)
            
        return response.Response(ctx, response_data=f"Success! Custom SMTP email sent for: {object_name}", status_code=200)
            
    except Exception as ex:
        logging.getLogger().error(f"Function processing failure: {str(ex)}")
        return response.Response(ctx, response_data=f"Custom SMTP Execution error: {str(ex)}", status_code=500)
EOF



```






5.then create requirements.txt
```code

cat << 'EOF' > requirements.txt
fdk>=0.1.60
EOF



```


6. Then create func.yaml there

```code

cat << 'EOF' > func.yaml
schema_version: 20180708
name: daily_compartment_bill
version: 0.0.1
runtime: python
build_image: fnproject/python:3.11-dev
run_image: fnproject/python:3.11
entrypoint: func.handler
memory: 256
EOF



```


   1. Deploy the application using the CLI:
   
   fn -v deploy --app Your-OCI-Application-Name

   2. Set the values in the Oracle Application environment configuration via CLI or OCI console UI:
   
   fn config app Your-OCI-Application-Name WORKER_URL "https://workers.dev"
   fn config app Your-OCI-Application-Name OCI_PAR_URL "https://objectstorage..."
   fn config app Your-OCI-Application-Name SMTP_HOST "smtp.email..."
   fn config app Your-OCI-Application-Name SMTP_USER "your-smtp-ocid"
   fn config app Your-OCI-Application-Name SMTP_PASS "your-smtp-password"
   fn config app Your-OCI-Application-Name SMTP_PORT "587"
   fn config app Your-OCI-Application-Name SENDER_EMAIL "sender@domain.com"
   fn config app Your-OCI-Application-Name RECEIVER_EMAIL "receiver@domain.com"


   3. Then test the application once with command  fn invoke APP_NAME FUNCTION_NAME
   
------------------------------
## Step 7: Create Event Rules to Trigger the Function
To cause the function to execute immediately upon object creation:

   1. In the OCI Console, navigate to Observability & Management → Events Service → Rules.
   2. Click Create Rule.
   3. Fill out the rule logic:
   * Display Name: trigger-email-on-new-object
      * Condition: Event Type
      * Service Name: Object Storage
      * Event Type: Object - Create
   4. Under Actions:
   * Action Type: Functions
      * Function Compartment: Select your working compartment.
      * Function Application: Choose your application name.
      * Function: Select your deployed python function (oci-email-notifier).
   5. Click Create Rule.

------------------------------
## Step 8: Setup a Scheduled Report to Push Objects Every 24 Hours
Depending on whether you want an OCI native cost/usage report or a database dump, configure your schedule using OCI Scheduled Jobs or OCI Database Backups:

   1. Navigate to Governance & Administration → Usage → Scheduled Reports (or Compute → Management Agent Cloud Service → Scheduled Jobs).
   2. Click Create Scheduled Report / Job.
   3. Set the recurrence configuration profile:
   * Interval: Daily (Once every 24 Hours).
   4. Under Target Destination:
   * Select Object Storage.
      * Pick the Target Bucket name created in Step 1 (report-bucket).
   5. Save the configuration rule.

The schedule will automatically push a fresh object to your storage array once every 24 hours, which fires off the event rule trigger, triggers your function, retrieves the specific top object, formats the proxy link, sends the email, and wipes the storage container empty 23 hours later!
Would you like assistance with writing the exact dependencies (requirements.txt) for the Python function or debugging any OCI IAM permission policies required for the event rules?

