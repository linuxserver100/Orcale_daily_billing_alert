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
    const objectName = url.pathname.slice(1); // Removes the leading '/'

    // If no specific object name is provided in the path, return 404
    if (!objectName || objectName === "") {
      return new Response("Not Found", { status: 404 });
    }

    // Read the base OCI PAR URL from environment variables
    let baseParUrl = env.OCI_PAR_URL;
    if (!baseParUrl.endsWith('/')) {
      baseParUrl += '/';
    }

    // Construct the direct URL to the requested object
    const targetUrl = `${baseParUrl}${objectName}`;

    // Fetch the file from Oracle Object Storage
    const ociResponse = await fetch(targetUrl);

    // If Oracle returns an error, forward a 404 or its original status
    if (!ociResponse.ok) {
      return new Response("Object Not Found", { status: 404 });
    }

    // Create a new response with the object data and force a download header
    const responseHeaders = new Headers(ociResponse.headers);
    responseHeaders.set("Content-Disposition", `attachment; filename="${objectName}"`);

    return new Response(ociResponse.body, {
      status: ociResponse.status,
      headers: responseHeaders
    });
  }
};
```

   1. Click Deploy (top right).
   2. Go back to your Worker's settings page by clicking the back arrow next to the project name.
   3. Go to the Settings tab → Variables.
   4. Under Environment Variables, click Add.
   5. Set the key Name as OCI_PAR_URL and paste your copied Oracle Bucket PAR URL into the Value field.
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
* WORKER_URL: The Cloudflare worker URL (e.g., https://workers.dev)
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
* 

------------------------------
## Step 6: Create, Configure, and Deploy the OCI Function

   1. In the OCI Console, go to Developer Services → Functions → Applications.
   2. Click Create Application. Name it, select your VCN/Subnets, and click Create.
   3. Open your Cloud Shell or local OCI CLI terminal and initialize a boilerplate python function project:
   
   fn init --runtime python oci-email-notifier
   cd oci-email-notifier
   
   4. Replace the contents of func.py with an implementation that connects to the bucket, fetches the newest object, and fires the email:

```code
import ioimport jsonimport loggingimport osimport smtplibfrom email.mime.text import MIMETextimport requests
def handler(ctx, data: io.BytesIO = None):
    logging.getLogger().info("OCI Function Triggered.")
    
    # Load Environment Configurations
    worker_url = os.environ.get("WORKER_URL")
    par_url = os.environ.get("OCI_PAR_URL")
    smtp_host = os.environ.get("SMTP_HOST")
    smtp_user = os.environ.get("SMTP_USER")
    smtp_pass = os.environ.get("SMTP_PASS")
    smtp_port = int(os.environ.get("SMTP_PORT", 587))
    sender = os.environ.get("SENDER_EMAIL")
    receiver = os.environ.get("RECEIVER_EMAIL")

    try:
        # 1. Fetch bucket objects using listing enabled PAR URL
        # OCI listing returns XML/JSON depending on target format. Cleanest way is parsing standard OCI listing
        response = requests.get(par_url)
        if response.status_code != 200:
            raise Exception("Failed to list bucket objects via PAR URL.")
        
        # Simple extraction assumes JSON response for bucket-level listing metadata
        bucket_data = response.json()
        objects = bucket_data.get("objects", [])
        
        if not objects:
            return response.Response(ctx, response_data="No objects found in bucket.", headers={"Content-Type": "text/plain"})
        
        # Sort objects by creation time to grab the absolute latest item
        latest_object = max(objects, key=lambda x: x.get("timeCreated"))
        latest_name = latest_object.get("name")
        
        # 2. Build worker attachment link
        download_link = f"{worker_url.rstrip('/')}/{latest_name}"
        
        # 3. Formulate and send the SMTP email
        msg = MIMEText(f"A new report object has been generated.\n\nYou can download the latest top object directly from this worker link:\n{download_link}")
        msg['Subject'] = f"New OCI Report Available: {latest_name}"
        msg['From'] = sender
        msg['To'] = receiver
        
        with smtplib.SMTP(smtp_host, smtp_port) as server:
            server.starttls()
            server.login(smtp_user, smtp_pass)
            server.sendmail(sender, [receiver], msg.as_string())
            
        logging.getLogger().info("Notification Email Sent successfully.")
        return response.Response(ctx, response_data="Success", headers={"Content-Type": "text/plain"})

    except Exception as ex:
        logging.getLogger().error(f"Error executing function: {str(ex)}")
        return response.Response(ctx, response_data=f"Error: {str(ex)}", headers={"Content-Type": "text/plain"})


```






5.then create requirements.txt



6. Then create func.yaml there


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

