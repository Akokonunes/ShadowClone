
# ShadowClone Installation & Troubleshooting Guide

## **1. Prerequisites**

-   **AWS account** (with IAM permissions for Lambda, S3, ECR, etc.)
    
-   **Docker installed and running**  
    (If not, install with:  
    `curl -fsSL https://get.docker.com -o get-docker.sh && sh get-docker.sh`)
    
-   **Python 3.10**  
    (Recommended: use `python -m venv env` for a clean environment)
    

----------

## **2. Cloud Setup (AWS)**

-   **Create two S3 buckets** in your chosen AWS region  
    (e.g., `shadowclone-data-yourname` and `shadowclone-logs-yourname`)
    
-   **Create an IAM policy** with the following permissions:
    
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:*",
        "lambda:*",
        "ec2:*",
        "ecr:*",
        "sts:GetCallerIdentity"
      ],
      "Resource": "*"
    }
  ]
}
```
    
-   **Create an IAM role for Lambda** and attach this policy.

**Create a new IAM Role**
-   Go to **IAM > Roles > Create Role**   
-   **Trusted Entity:** AWS Service 
-   **Use Case:** Lambda 
-   Attach the policy you just created
-   Save the role and note the **Role ARN** (you’ll need this later)

-   **Copy the Role ARN** for use in your config.
    

----------

## **3. Local Setup**

-   **Clone the repo and install dependencies**
    
```
git clone https://github.com/Akokonunes/ShadowClone.git && cd ShadowClone
python -m venv env  source  env/bin/activate
pip install -r requirements.txt` 
```

----------

## **4. Lithops Configuration**

-   **Create a Lithops config at `~/.lithops/config`**:
    
```
lithops:
 backend: aws_lambda
 storage: aws_s3
 data_limit: False
 monitoring_interval: 2

aws:
 access_key_id: *********
 secret_access_key: *********

aws_lambda:
 execution_role: arn:*********/Roleshadowclone
 region_name: us-east-1
 runtime_memory: 512
 runtime_timeout: 900
 workers: 1000
  
aws_s3:
 storage_bucket: shadowclone-logs-roberto99-us-east-1
 region_name : us-east-1
```

## **5. Build the ShadowClone Runtime Docker Image**

**Common Gotchas and Solutions:**

-   If you hit **permission errors with tools like `eget`**, manually download and copy the binary into your Docker build context, set correct permissions (`chmod 755`).
    
-   Use **the correct and current `eget` release URL** if building from Dockerfile.
    
-   **Always match your local Python version and the one in Dockerfile.**
    

**Steps:**

bash

CopyEdit

`lithops runtime build sc-runtime -f Dockerfile_v0.2` 

_(Fix any Docker or permissions errors before continuing!)_

----------

## **6. Deploy the Runtime to Lambda (ECR)**

**If you change region, repeat this step!**

bash

CopyEdit

`lithops runtime deploy sc-runtime --memory 512 --timeout 300` 

-   If you see errors about missing ECR repository, **make sure your region matches everywhere, and re-run the build/deploy commands.**
    
-   If you see `pytest` errors, run `pip install pytest` in your environment.
    

----------

## **7. Update Project Config**

Edit `config.py` in the ShadowClone repo:

python

CopyEdit

`LITHOPS_RUNTIME="sc-runtime" STORAGE_BUCKET="shadowclone-data-yourname" PICKLE_DB="bucket-hash.db"` 

----------

## **8. Usage Examples**

**Basic:**

bash

CopyEdit

`python shadowclone.py -i subdomains.txt --split 625 -o output.txt -c "/usr/local/bin/httpx -l {INPUT}"` 

-   `--split 625` means 500,000 lines = 800 lambdas.
    
-   Use a larger split (e.g. 5000) for fewer lambdas, or a smaller split for more parallelism.
    

**See `Examples.md` for more sample usage:**

-   nuclei, ffuf, puredns, dnsx, etc.
    
-   For advanced (e.g. `--no-split`) see Examples.md.
    

----------

## **9. Troubleshooting Checklist**

**Problem**

**Solution**

`docker/podman not found`

Install Docker, ensure it's running.

`eget: Permission denied`

Manually copy and `chmod 755` the `eget` binary to `/usr/local/bin`.

Wrong `eget` URL

Use the correct version URL (see latest release page).

`Runtime ... not deployed to ECR`

Build and deploy runtime again, especially after region change.

`Error 429: Rate exceeded`

Increase split (fewer Lambdas at once), or lower batch size in scripts.

Lambda `TimeoutError`

Increase `runtime_timeout`, use bigger split, or optimize your tools for speed.

Exceeded S3 request free tier

Use bigger splits to reduce number of lambdas and S3 calls.

`pytest` not found

`pip install pytest`

IAM permissions errors

Double-check policies, role ARN, and region consistency.

Batch launcher file not found

Confirm all files exist, are correctly named and sorted.

----------

## **10. Pro Tips**

-   **Always monitor AWS Free Tier and billing dashboards.**
    
-   Use **larger splits** and **lowest memory that works** to keep Lambda GB-seconds cost low.
    
-   After changing region or config, **rebuild and redeploy runtime**.
    
-   **Set up AWS budgets and alerts** to catch runaway costs early!
    

----------

## **Happy Hacking!**
