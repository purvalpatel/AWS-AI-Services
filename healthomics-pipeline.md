https://aws.amazon.com/blogs/architecture/genomics-workflows-part-7-analyze-public-rna-sequencing-data-using-aws-healthomics/
**AWS Healthomics:** <br>
AWS HealthOmics is Amazon’s managed service for running biology data analysis without managing servers. <br>
Instead of: <br>
- Setting up big computers
- Installing complex bioinformatics software
- Managing storage and scaling
AWS does it for you.


# RNA sequencing using HealthOmics:

<img width="1199" height="583" alt="image" src="https://github.com/user-attachments/assets/050ea776-ce0f-4bd9-871a-971e8bc99d46" />

### Included services: <br>
- S3
- LAmbda
- DynamoDB
- HealthOmics
- EC2
- IAM
- Batch
- Ec2 Launch Template
- Step Functions

### Create Three S3 buckets:
Console -> S3 <br>
- `healthomics-raw-fastq`
- `healthomics-processed`
- `healthomics-metadata`

Set Trigger in heathomics-metadata Bucket.

### Create Role for Batch Service:
1. Role Name : `BatchServiceRole`

Below are the permissions:
<img width="1240" height="436" alt="image" src="https://github.com/user-attachments/assets/620af0a8-40a9-4687-9bda-bfd2dab3ecef" />

2. Role Name: `ecsInstanceRole`

Below are the permissions:
<img width="1110" height="352" alt="image" src="https://github.com/user-attachments/assets/e79f6b08-b8e2-4ce4-927b-9bc15b69ba4e" />

3. Role Name: `healthomics-metadata-upload-role-13aoiit0`

#### Attach Inline policy in this role: <br>
*Add Permissions* -> *Create Inline policy* <br>
Policy name : `healthomics-inlinepolicy-lambda` <br>
```
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			"Action": [
				"batch:SubmitJob",
				"batch:DescribeJobs"
			],
			"Resource": [
				"arn:aws:batch:ap-southeast-1:6812436656745:job-definition/rna-seq-job:*",
				"arn:aws:batch:ap-southeast-1:6812436656745:job-queue/healthomics-queue"
			]
		},
		{
			"Effect": "Allow",
			"Action": [
				"s3:GetObject",
				"s3:ListBucket"
			],
			"Resource": [
				"arn:aws:s3:::healthomics-raw-fastq",
				"arn:aws:s3:::healthomics-raw-fastq/*"
			]
		}
	]
}
```

Below are the permissions: <br>
<img width="1146" height="442" alt="image" src="https://github.com/user-attachments/assets/49e3e79a-3889-4cf6-b049-4140783dd00d" />


### Setup AWS Batch

#### Create Compute Environment

- console -> Batch -> Environments
- Create Environment -> Compute Environment
- Select Amazon Compute Cloud (EC2 )
- Orchestration type : Managed
- Name: `HealthOmics-batch-data-Preparation`
- Service Role : Select Created service role (`BatchServiceRole`)
- Instance role: Select Created Instance role (`ecsinstancerole`)
- Launch Template : Select Created Launch template. default will take 30 GB Storage only.
- Next
- Minium CPU : 0
- Maximum CPU : 32
- Allowed Instance Type : Select The instance type which you want
- Next
- Select VPC
- Select Subnets
- Select Security groups
- Next
- Click "*Create Compute Environment*"

#### Create Job Queue

- Console -> Batch -> Job Queue -> Create
- Orchestration Type : Amazon Elastic Compute Cloud (EC2)
- Job Name : `healthomics-queue`
- Priority : 1
- Select *Created Compute Environement*
- Create Job Queue

#### Create Job defination
- console -> Batch -> Job Defination -> Create
- Orchestration type : Amazon Elastic Compute Cloud (EC2)
- Name : `healthomics-rna-seq-job-def`
- Execution timeout : 60
- Next
- Image : `ncbi/sra-tools:latest`
- Command:
```BASH
["sh","-c","prefetch ${SRA_ID} && fasterq-dump ${SRA_ID} -O /data && gzip /data/*.fastq && aws s3 cp /data s3://healthomics-raw-fastq/${SRA_ID}/ --recursive"]
```
- vCPUS : 32
- memory : 5000
- Next
- Next Again
- Create Job Defination


### Create Lambda Function
Function name : `healthomics-metadata-upload` <br>

`lambda_function.py`
```Python
import boto3
import json
import os 
import csv
from datetime import datetime

s3 = boto3.client("s3")
batch = boto3.client('batch')
dynamodb = boto3.resource('dynamodb')
TABLE_NAME = dynamodb.Table('rna-seq-data-store')  

def lambda_handler(event, context):
    # Print the incoming S3 event
    print("Event received:", json.dumps(event))
    
    # Get the uploaded file details
    record = event["Records"][0]
    bucket = record["s3"]["bucket"]["name"]
    s3_object = record["s3"]["object"]["key"]
    
    sra_id = os.path.splitext(os.path.basename(s3_object))[0]
    
    print(f"Processing file: {s3_object}")
    print(f"Bucket: {bucket}")
    print(f"SRA ID: {sra_id}")
    
    # Store initial status in DynamoDB
    try:
        TABLE_NAME.put_item(
            Item={
                "PK": f"SRA#{sra_id}",
                "SK": "STATUS#CURRENT",
                "sra_id": sra_id,
                "s3_key": s3_object,
                "bucket": bucket,
                "status": "BATCH_SUBMITTED",
                "submitted_at": datetime.utcnow().isoformat() + "Z"
            }
        )
        print(f"DynamoDB item created for {sra_id}")
    except Exception as e:
        print(f"Error writing to DynamoDB: {e}")
        raise e
    
    # STOP HERE FOR TESTING - Comment out the batch submission
    print("DynamoDB write for completed")
    # return {
    #     "statusCode": 200, 
    #     "body": json.dumps(f"Testing complete - data stored for {sra_id}")
    # }
    
    print("Event received:", json.dumps(event))
    
    # Get the uploaded file name
    s3_object = event['Records'][0]['s3']['object']['key']

    sra_id = os.path.splitext(os.path.basename(s3_object))[0]

    print("Processing file:", s3_object)
    
    # Submit job to AWS Batch
    try:
        response = batch.submit_job(
            jobName=f"rna-seq-{sra_id}",
            jobQueue="healthomics-queue",
            jobDefinition="healthomics-rna-seq-job-def",
            containerOverrides={
                'environment':[
                               {'name':'INPUT_FILE','value':s3_object},
                               {'name': 'SRA_ID', 'value': sra_id}      ## this is used in sending parameters to container
                            ]
                
            },
            parameters={
            "SRA_ID": sra_id
            }
        )
        print("Batch job submitted:", response)
    except Exception as e:
        print("Error submitting Batch job:", e)
        raise e
    
    return {"statusCode": 200, "body": json.dumps("Job submitted {sra_id}")}
```
🧠 **Note:** <br>
- This code will take sra_id as a file name.
- Provide Access of FullDynamoDB access policy to Lambda functione role.

**Phase 1 Testing:** <br>
- .csv uploaded into aws s3 bucket `heathomics-metadata`
- Lambda function triggerd
- Lambda function will call Batch Job and Store data of job into DynamoDB.
- Batch job will start the EC2 container and inside EC2 container sra-tools container will start the process.

**Requirement:** <br>
- sra-tools container should download the data from source: `prefetch SRR1186600`
- after downloading the data process should start: `fasterq-dump SRR1186600 -O /data`
- Once processed data should store into another s3 bucket. `healthomics-processed`

**Issue:** <br>
- we have not specified any commands in docker, due to this data processing process is not started.
- And after the process complete. data is not pushed to the s3. because aws cli is not configured into image.
- So for that we need to create own Docker image and push it into ECR.


- **Issue is resolved**:  .csv file name must be same as SRA_ID.

**What is SRA ID?** <br>
Sequence Read Archive <br>
- SRR1186600 → a single sequencing run
- SRX... → experiment
- GSE... → GEO study (contains many SRRs) 

Tools like `sra-tools` use SRA IDs to download raw sequencing data automatically from NCBI website.


### Create Custome Docker image:

`Dockerfile`:
```
FROM ncbi/sra-tools:latest

# Install AWS CLI
RUN apk update
RUN apk add aws-cli

ENV AWS_ACCESS_KEY_ID=xxxxxxxxxxxxxxxxxxx
ENV AWS_SECRET_ACCESS_KEY=5tWc8EUSN+xxxxxxxxxxxxxxxx/xxxxx
```
**Build and push Image on ECR:**
```
sudo docker build -t xxxxx-sra-aws:latest .
docker tag xxxxx-sra-aws:latest 44565475676876.dkr.ecr.ap-southeast-1.amazonaws.com/heathomics:latest
docker push 34546456578.dkr.ecr.ap-southeast-1.amazonaws.com/heathomics:latest
```

### How to test:
- Upload CSV file into S3 bucket - xxxx-geo-accession-ids/upload <br>

**CSV Sample**:`SRR1186600.csv` <br>
🧠 **Note**: The file name must be same as SRA ID.<br>
```
geo_id,source_url,version
GSE123488,https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE123488,4
```

**CSV with low size data**:
`SRR12936759.csv`
```
geo_id,source_url,version
SRR12936759,https://sra-pub-run-odp.s3.amazonaws.com/sra/SRR12936759/SRR12936759,1
```

- This will trigger lambda function and store the data into DynamoDB.
- There must be SRA ID column in CSV

**You can check the logs of AWS Batch jobs:** <br>

<img width="1344" height="382" alt="image" src="https://github.com/user-attachments/assets/a80475d7-71a0-4253-9964-ca168bdd25c8" />



**Issue**: <br>
- Disk is getting full as default storage in created instance is 30GB.

**Solution**: <br>
- Edit the Existing Launch template and increase the size of storage in it. and make the version default.
- create Instance Template **Instance Template**.
- Name: "Enter Template Name"
- Source Template :  Select previously edited template.
- Click on "Create Launch Template"



Once job completed. it will looks like this, <br>
<img width="1613" height="478" alt="image" src="https://github.com/user-attachments/assets/fd077bb5-2b6f-4503-a65b-8bb4f5211692" />

<br>
Now .fastq files will be stored in `healthomics-raw-fastq` bucket. <br>

### Create Trigger on S3 Notification
- Bucket : healthomics-raw-fastq
- Event Name : Healthomics-samplesheet-csv-generate-trigger
- Prefix : fastq
- Suffix : _2.fastq.gz      [ Have to add _2 because, it will start execution even after 1 fastq.gz is generated. it should only start after 2nd file generates.]
- PUT
- Select Lambda function.


### Create Lambda function for create sample_sheet.csv
- Create Lambda function that will create sample_sheet.csv on s3.
- Name - `sample-Sheet-Generator`
- Lambda_function.py
```Python
import boto3
import csv
import io
import os
import re

s3 = boto3.client("s3")

CSV_BUCKET = "healthomics-raw-fastq"
CSV_KEY = "sample_sheet/sample_sheet.csv"

# Match filename only (NOT path)
FASTQ_REGEX = re.compile(r"(.+?)[\._]R?([12])\.fastq\.gz$", re.IGNORECASE)

def lambda_handler(event, context):
    # Get folder from the S3 event
    s3_record = event['Records'][0]['s3']
    bucket = s3_record['bucket']['name']
    key = s3_record['object']['key']

    # Extract folder path (everything before filename)
    folder_prefix = os.path.dirname(key) + "/"  # e.g., fastq/SRR1186601/
    print(f"Processing folder: {folder_prefix}")

    # Use paginator in case folder has many files
    paginator = s3.get_paginator('list_objects_v2')
    page_iterator = paginator.paginate(Bucket=bucket, Prefix=folder_prefix)

    samples = {}

    for page in page_iterator:
        if "Contents" not in page:
            continue
        for obj in page["Contents"]:
            obj_key = obj["Key"]

            if not obj_key.endswith(".fastq.gz"):
                continue

            filename = os.path.basename(obj_key)
            match = FASTQ_REGEX.match(filename)
            if not match:
                print(f"Skipping file {filename}, doesn't match pattern")
                continue

            sample_id, read = match.groups()
            s3_uri = f"s3://{bucket}/{obj_key}"

            samples.setdefault(sample_id, {})
            if read == "1":
                samples[sample_id]["read1_fastq"] = s3_uri
            elif read == "2":
                samples[sample_id]["read2_fastq"] = s3_uri

    # Generate CSV
    csv_buffer = io.StringIO()
    writer = csv.DictWriter(
        csv_buffer,
        fieldnames=["sample_id", "read1_fastq", "read2_fastq"]
    )
    writer.writeheader()

    valid = 0
    for sample_id, reads in samples.items():
        if "read1_fastq" in reads and "read2_fastq" in reads:
            writer.writerow({
                "sample_id": sample_id,
                "read1_fastq": reads["read1_fastq"],
                "read2_fastq": reads["read2_fastq"]
            })
            valid += 1
        else:
            print(f"Incomplete pair for {sample_id}, skipping")

    if valid == 0:
        print("No complete samples found, CSV not uploaded")
        return

    # Upload CSV to S3
    s3.put_object(
        Bucket=CSV_BUCKET,
        Key=CSV_KEY,
        Body=csv_buffer.getvalue(),
        ContentType="text/csv"
    )

    print(f"sample_sheet.csv uploaded with {valid} samples")
    print(f"s3://{CSV_BUCKET}/{CSV_KEY}")


```

Testing:
- Create Two files SRR1186601_R1.fastq.gz, SRR1186601_R2.fastq.gz
- And upload in S3 bucket.
- sample_sheet.csv file is uploading properly on s3 bucket. but sample_id is showing with fastq/ <br>

🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬🧬


## Phase 2:

1. Once .fastq files received in **S3 bucket** `healthomics-raw-fastq`
2. **Lambda function** should trigger the **Steps functions**. (Sample sheet.csv)
3. **Step function** will call **Lambda function**.
3. **Lambda Function** will call the  **Healthomics workflows** run with (Sequence store)
4. This will generate Output files and store into S3.(BAM, QC) Files.


Flow:
```
            +----------------------+
            |  S3 Upload           |
            |  sample_sheet.csv    |
            +----------+-----------+
                       |
                       v
            +----------------------+
            |  Lambda:             |
            |  healthomics-data-   |
            |  analysis            |
            |  (Parse CSV & create |
            |   samples list)      |
            +----------+-----------+
                       |
                       v
            +----------------------+
            |  Step Functions      |
            |  State Machine       |
            |  StartHealthOmics    |
            |  Workflow Task       |
            +----------+-----------+
                       |
                       v
            +----------------------+
            |  Lambda:             |
            |  start-healthomics-  |
            |  workflow            |
            |  (Call omics.start_run|
            |   per sample)        |
            +----------+-----------+
                       |
                       v
            +----------------------+
            |  HealthOmics Service |
            |  (RNA-Seq workflow)  |
            +----------+-----------+
                       |
                       v
            +----------------------+
            |  S3 Output Bucket    |
            |  (Processed data)    |
            +----------------------+

```

### Create ServiceRole
- Name: `HealthOmicsExecutionRole`
- Create Role -> Custom Trust Policy -> Select "json"
- Below are the Permissions
<img width="888" height="314" alt="image" src="https://github.com/user-attachments/assets/3f3499d4-fbdc-457d-a459-f3bc72039df9" />


### Create Workflow

#### Prerequisites before creating workflow:
- Require zip of workflow.zip
- Create directory `mkdir workflow; cd workflow`
- create `main.wdl`
```
version 1.0

workflow rnaseq_healthomics {

  input {
    File read1_fastq
    File read2_fastq
    File star_index
    String sample_id
  }

  call fastqc {
    input:
      read1 = read1_fastq,
      read2 = read2_fastq,
      sample_id = sample_id
  }

  call star_align {
    input:
      read1 = read1_fastq,
      read2 = read2_fastq,
      star_index = star_index,
      sample_id = sample_id
  }

  output {
    File aligned_bam = star_align.aligned_bam
#    File aligned_bam_index = star_align.aligned_bam_index
    File star_log = star_align.star_log
    Array[File] fastqc_reports = fastqc.reports
  }
}

task fastqc {

  input {
    File read1
    File read2
    String sample_id
  }

  command {
    fastqc ${read1} ${read2}
  }

  output {
    Array[File] reports = glob("*_fastqc.zip")
  }

  runtime {
    docker: "34545656778.dkr.ecr.ap-southeast-1.amazonaws.com/biocontainers:v0.11.9_cv8"
    cpu: 32
    memory: "64 GiB"
  }
}

task star_align {

  input {
    File read1
    File read2
    File star_index
    String sample_id
  }

  command {
    set -euxo pipefail 
    echo "STAR index zip path:" 
    echo ${star_index} 
    mkdir -p star_index 
    tar  -xvzf ${star_index} -C star_index
    echo "STAR index contents:" 
    ls -lh star_index 
    test -f star_index/genomeParameters.txt
    STAR \
      --genomeDir star_index\
      --readFilesIn ${read1} ${read2} \
      --readFilesCommand zcat \
      --runThreadN 4 \
      --outFileNamePrefix ${sample_id}. \
      --outSAMtype BAM SortedByCoordinate
    echo "-------------------- Completed ---------"

  }

  output {
    File aligned_bam = "${sample_id}.Aligned.sortedByCoord.out.bam"
    File star_log = "${sample_id}.Log.final.out"
  }

  runtime {
    docker: "75645668567889.dkr.ecr.ap-southeast-1.amazonaws.com/star:2.7.10b--h9ee0642_0"
    cpu: 32
    memory: "64 GiB"
  }
}

```
- Create Zip: `cd workflow; zip -r workflow.zip main.wdl`

After successful test run:
<img width="1605" height="250" alt="image" src="https://github.com/user-attachments/assets/f8d89d6a-d43d-4fc5-85c5-6be9d9b68e64" />


#### Create image and push it on ECR
🧠 **Note**: Healthomics workflow can not download Public images, you need to create own image and push it on ECR.

```
## create for fastq image
docker pull biocontainers/fastqc:v0.11.9_cv8
## Create repo in ECR if not exists
docker tag biocontainers/fastqc:v0.11.9_cv8 6812436656745.dkr.ecr.ap-southeast-1.amazonaws.com/biocontainers:v0.11.9_cv8

## push it
docker push 6812436656745.dkr.ecr.ap-southeast-1.amazonaws.com/biocontainers:v0.11.9_cv8

## do the same for star image.
docker pull quay.io/biocontainers/star:2.7.10b--h9ee0642_0
## Create repo on ECR if not exists.
docker tag quay.io/biocontainers/star:2.7.10b--h9ee0642_0 6812436656745.dkr.ecr.ap-southeast-1.amazonaws.com/star:2.7.10b--h9ee0642_0
docker push 6812436656745.dkr.ecr.ap-southeast-1.amazonaws.com/star:2.7.10b--h9ee0642_0
```

##### Create Repository policy: <br>
```BASH
aws ecr set-repository-policy \
  --repository-name biocontainers \
  --region ap-southeast-1 \
  --policy-text '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "AllowOmicsPull",
        "Effect": "Allow",
        "Principal": {
          "Service": "omics.amazonaws.com"
        },
        "Action": [
          "ecr:BatchGetImage",
          "ecr:GetDownloadUrlForLayer"
        ]
      }
    ]
  }'
  aws ecr set-repository-policy \
  --repository-name star \
  --region ap-southeast-1 \
  --policy-text '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "AllowOmicsPull",
        "Effect": "Allow",
        "Principal": {
          "Service": "omics.amazonaws.com"
        },
        "Action": [
          "ecr:BatchGetImage",
          "ecr:GetDownloadUrlForLayer"
        ]
      }
    ]
  }'
```
#### upload start_index on s3 by creating zip.
Currently i have uploaded into `https://healthomics-raw-fastq.s3.ap-southeast-1.amazonaws.com/references/star-hg38.zip`

##### **Now  Start Setting up workflow**. <br>
- Login console -> HealthOmics -> Private Workflow
- Create workflow
- Workflow name : Any name
- Workflow definition - Workflow language - WDL
- Workflow definition source - Select definition folder from a local source ( Upload created zip of main.wdl)
- Next
- Next
- Next
- Create Workflow


##### 📃 **Start Run** <br> [This is for manual testing] Optional

- Click **Start-Run**
- Workflow ID : Select workflow ID
- Run name : Any name
- Run Priority : Keep default
- Select S3 output destination : Select S3 bucket where you wanted to save files
- Service role name : Select Role
- Next
- Required Parameter values:
*read2_fast1*: s3://healthomics-raw-fastq/fastq/SRR1186600.fastq.gz <br>
*read1_fastq*: s3://healthomics-raw-fastq/fastq/SRR1234567.fastq.gz <br>
*star_index*: s3://healthomics-raw-fastq/references/star-hg38.tar.gz <br>
*sample_id*: Sample_01 <br>
- Next
- Next
- Start run


You can check the job status from here: <br>

<img width="1596" height="176" alt="image" src="https://github.com/user-attachments/assets/58c909a8-1b9e-4625-9af9-e23ea5043e3b" />


- Output files are stored into S3 bucket : `healthomics-processed/out` Folder

#### 📃 Test on local: [For Testing purpose only.] Optional
- We can run the workflow on local which we are running on AWS. which is for saving cost of AWS. once everything is working on local then we can check the workflow on aws.

```
python3 -m venv miniwdl
source miniwdl/bin/activate
pip install miniwdl
```
Create `inputs.json`
```
{
  "rnaseq_healthomics.read1_fastq": "s3://healthomics-raw-fastq/fastq/SRR1186600.fastq.gz",
  "rnaseq_healthomics.read2_fastq": "s3://healthomics-raw-fastq/fastq/SRR1234567.fastq.gz",
  "rnaseq_healthomics.star_index": "/home/siddharth/reference-genome/star-hg38.tar.gz",
  "rnaseq_healthomics.sample_id": "sample_0"
}
```
create `main.wdl`
```
version 1.0

workflow rnaseq_healthomics {

  input {
    File read1_fastq
    File read2_fastq
    File star_index
    String sample_id
  }

  call fastqc {
    input:
      read1 = read1_fastq,
      read2 = read2_fastq,
      sample_id = sample_id
  }

  call star_align {
    input:
      read1 = read1_fastq,
      read2 = read2_fastq,
      star_index = star_index,
      sample_id = sample_id
  }

  output {
    File aligned_bam = star_align.aligned_bam
#    File aligned_bam_index = star_align.aligned_bam_index
    File star_log = star_align.star_log
    Array[File] fastqc_reports = fastqc.reports
  }
}

task fastqc {

  input {
    File read1
    File read2
    String sample_id
  }

  command {
    fastqc ${read1} ${read2}
  }

  output {
    Array[File] reports = glob("*_fastqc.zip")
  }

  runtime {
    docker: "6812436656745.dkr.ecr.ap-southeast-1.amazonaws.com/biocontainers:v0.11.9_cv8"
    cpu: 32
    memory: "64 GiB"
  }
}

task star_align {

  input {
    File read1
    File read2
    File star_index
    String sample_id
  }

  command {
    set -euxo pipefail 
    echo "STAR index zip path:" 
    echo ${star_index} 
    mkdir -p star_index 
    tar  -xvzf ${star_index} -C star_index
    echo "STAR index contents:" 
    ls -lh star_index 
    test -f star_index/genomeParameters.txt
    STAR \
      --genomeDir star_index\
      --readFilesIn ${read1} ${read2} \
      --readFilesCommand zcat \
      --runThreadN 4 \
      --outFileNamePrefix ${sample_id}. \
      --outSAMtype BAM SortedByCoordinate
    echo "-------------------- Completed ---------"

  }

  output {
    File aligned_bam = "${sample_id}.Aligned.sortedByCoord.out.bam"
    File star_log = "${sample_id}.Log.final.out"
  }

  runtime {
    docker: "6812436656745.dkr.ecr.ap-southeast-1.amazonaws.com/star:2.7.10b--h9ee0642_0"
    cpu: 32
    memory: "64 GiB"
  }
}

```
run:
```
miniwdl run main.wdl -i inputs.json
# miniwdl run main.wdl -i inputs.json   --env AWS_ACCESS_KEY_ID=xxxxxxxx  --env AWS_SECRET_ACCESS_KEY=xxxx+xxxxxx/xxxxx   --env AWS_DEFAULT_REGION=ap-southeast-1
```

🧠 **Note:**
- Use .tar.gz instead of .zip because sometimes .zip is getting corrupted and it shows an error.


#### Create Trigger on S3 bucket:
- Name - `healthomics-raw-fastq`
- **Create Event notification**
- Name: `healthomics-data-analysis-trigger`
- Put
- Prefix - sample_sheet
- Suffix : .csv
- Lambda function : `healthomics-data-analysis `

#### Create Role:
- Role name : `run-seq-step-function-policy`
- Select Policies: `AmazonS3ReadOnlyAccess, AWSLambdaRole, AWSStepFunctionsFullAccess`
- Create Inline policy: 
- Name : `rna-step-inlinepolicy` <br>
```JSON
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			"Action": [
				"omics:StartWorkflowRun",
				"omics:GetWorkflowRun"
			],
			"Resource": "*"
		}
	]
}
```
#### Create Lambda 
- Name : `healthomics-data-analysis`
- Code: `lambda_function.py`

```Python
import json
import boto3
import csv
import io
import os

s3 = boto3.client('s3')
stepfunctions = boto3.client('stepfunctions')

def lambda_handler(event, context):
    """
    Triggered by S3 upload of sample_sheet.csv
    Reads CSV, prepares data, starts Step Functions
    """
    
    # Get bucket and key from S3 event
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']
    
    print(f"Processing file: s3://{bucket}/{key}")
    
    # Read CSV from S3
    try:
        response = s3.get_object(Bucket=bucket, Key=key)
        csv_content = response['Body'].read().decode('utf-8')
    except Exception as e:
        print(f"Error reading S3 file: {e}")
        raise
    
    # Parse CSV
    samples = []
    csv_reader = csv.DictReader(io.StringIO(csv_content))
    
    for row in csv_reader:
        sample = {
            'sample_id': row['sample_id'],
            'read1_fastq': row['read1_fastq'],
            'read2_fastq': row['read2_fastq']
        }
        samples.append(sample)
    
    print(f"Found {len(samples)} samples")
    
    # Prepare input for Step Functions
    step_functions_input = {
        'samples': samples,
        'bucket': bucket,
        'workflow_id': os.environ['WORKFLOW_ID'],
        'star_index': os.environ['STAR_INDEX'],
        'output_uri': os.environ['OUTPUT_URI'],
        'omics_role_arn': os.environ['OMICS_ROLE_ARN']
    }
    
    # Start Step Functions execution
    state_machine_arn = os.environ['STATE_MACHINE_ARN']
    
    try:
        response = stepfunctions.start_execution(
            stateMachineArn=state_machine_arn,
            input=json.dumps(step_functions_input)
        )
        
        print(f"Started Step Functions execution: {response['executionArn']}")
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': 'Pipeline started successfully',
                'execution_arn': response['executionArn'],
                'samples_count': len(samples)
            })
        }
    except Exception as e:
        print(f"Error starting Step Functions: {e}")
        raise

```
Set Environment variables:
```
OMICS_ROLE_ARN - arn:aws:iam::6812436656745:role/HealthomicsExecutionRoloe
OUTPUT_BUCKET -  s3://healthomics-processed/
OUTPUT_URI - s3://healthomics-processed/
STAR_INDEX - s3://healthomics-raw-fastq/references/star-hg38.tar.gz
STATE_MACHINE_ARN  - arn:aws:states:ap-southeast-1:6812436656745:stateMachine:Healthomics-state-machine
WORKFLOW_ID  - 2350233
```


#### Create Step function:
- Console -> Step Functions -> State Machines
- Create State machines
- Create From blank
- State machine name : healthomics-state-machine
- Standard
- Continue
- Code
```JSON
{
  "Comment": "Run HealthOmics RNA-Seq workflow",
  "StartAt": "StartHealthOmicsWorkflow",
  "States": {
    "StartHealthOmicsWorkflow": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "start-healthomics-workflow",
        "Payload.$": "$"
      },
      "End": true
    }
  }
}
```

#### Create Lambda Function 2
- Name : `start-healthomics-workflow`
- code : `lambda_function.py`
```python
import json
import boto3
from datetime import datetime

# Initialize AWS Omics client
omics = boto3.client('omics')

def lambda_handler(event, context):
    # Log the incoming event
    print(f"Received event: {json.dumps(event)}")
    
    # Extract inputs
    samples = event.get('samples', [])
    workflow_id = event.get('workflow_id')
    star_index = event.get('star_index')
    output_uri = event.get('output_uri')
    omics_role_arn = event.get('omics_role_arn')
    
    # Validate inputs
    if not samples:
        print("No samples found in input. Exiting.")
        return {
            'statusCode': 400,
            'body': json.dumps({'message': 'No samples to process'})
        }
    
    missing_vars = []
    if not workflow_id: missing_vars.append('workflow_id')
    if not star_index: missing_vars.append('star_index')
    if not output_uri: missing_vars.append('output_uri')
    if not omics_role_arn: missing_vars.append('omics_role_arn')
    
    if missing_vars:
        print(f"Missing required environment variables or event fields: {missing_vars}")
        return {
            'statusCode': 400,
            'body': json.dumps({'message': f"Missing fields: {missing_vars}"})
        }

    results = []

    # Process each sample
    for sample in samples:
        sample_id = sample.get('sample_id')
        read1_fastq = sample.get('read1_fastq')
        read2_fastq = sample.get('read2_fastq')

        if not sample_id or not read1_fastq or not read2_fastq:
            print(f"Skipping malformed sample: {sample}")
            results.append({
                'sample': sample,
                'error': 'Missing sample_id or FASTQ paths'
            })
            continue

        print(f"Processing sample: {sample_id}")

        parameters = {
            'sample_id': sample_id,
            'read1_fastq': read1_fastq,
            'read2_fastq': read2_fastq,
            'star_index': star_index
        }

        print(f"Parameters for start_run: {json.dumps(parameters)}")

        try:
            response = omics.start_run(
                workflowId=workflow_id,
                roleArn=omics_role_arn,
                name=f"rnaseq-{sample_id}-{datetime.now().strftime('%Y%m%d-%H%M%S')}",
                parameters=parameters,
                outputUri=f"{output_uri}{sample_id}/",
                priority=5
            )
            
            print(f"HealthOmics start_run response: {json.dumps(response)}")

            results.append({
                'sample_id': sample_id,
                'run_id': response.get('id'),
                'arn': response.get('arn'),
                'status': response.get('status')
            })

        except Exception as e:
            print(f"Error starting workflow for {sample_id}: {e}")
            results.append({
                'sample_id': sample_id,
                'error': str(e)
            })

    return {
        'statusCode': 200,
        'processed_samples': len(results),
        'results': results
    }

```
- Deploy all Lambda functions.
- After this Lambda function is created, new Role is automatically created. Provide policy `AmazonOmicsFullAccess` Access to that role.


##### how to test ?
- Upload sample_sheet.csv file
- Lambda Function `healthomics-data-analysis` should trigger. you can check the logs.
- State machine of state function should execute.
![alt text](state-image.png)
- Another lambda Function `start-healthomics-workflow` should trigger. you can check the logs.
- Healthomics workflow should trigger. you can check the logs.
```
aws s3 cp sample_sheet.csv s3://healthomics-raw-fastq/sample_sheet/
```


⛔ **Pending things:** <br>
- Data store into DynamoDB
- Sample_sheet.csv should store into sample_sheet directory on s3.




# Complete flow:
```
┌─────────────────────────────────────────────────────────────────────┐
│                        AWS RNA-Seq Analysis Pipeline                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Initial Trigger:                                                │
│     ┌─────────────────┐                                             │
│     │  S3: healthomics- │  SRR1186600.csv uploaded                 │
│     │    metadata      │ ───────────────────────────────────────┐  │
│     └─────────────────┘                                          │  │
│                                                                  │  │
│  2. Metadata Processing:                                         │  │
│     ┌────────────────────────────┐                               │  │
│     │ Lambda: healthomics-       │ ←─────────────────────────────┘  │
│     │   metadata-upload          │                                  │
│     └─────────────┬──────────────┘                                  │
│                   │                                                  │
│  3. Raw Data Generation:                                            │
│     ┌─────────────▼──────────────┐                                  │
│     │ AWS Batch Service:         │                                  │
│     │  healthomics-rna-seq-job-def │                                │
│     └─────────────┬──────────────┘                                  │
│                   │                                                  │
│     ┌─────────────▼──────────────┐                                  │
│     │ EC2 Instance Started       │                                  │
│     │ (RNA-Seq Processing)       │                                  │
│     └─────────────┬──────────────┘                                  │
│                   │                                                  │
│     ┌─────────────▼──────────────┐                                  │
│     │ S3: healthomics-raw-fastq  │                                  │
│     │   .fastq.gz files created  │                                  │
│     └─────────────┬──────────────┘                                  │
│                   │                                                  │
│  4. Sample Sheet Creation:                                          │
│     ┌─────────────▼──────────────┐                                  │
│     │ Lambda: Sample-Sheet-      │ ←──────────────────────────────┐ │
│     │   Generator                │                                │ │
│     └─────────────┬──────────────┘                                │ │
│                   │                                                │ │
│     ┌─────────────▼──────────────┐                                │ │
│     │ S3: .../sample_sheet/      │                                │ │
│     │   sample_sheet.csv created │                                │ │
│     └─────────────┬──────────────┘                                │ │
│                   │                                                │ │
│  5. Analysis Orchestration:                                        │ │
│     ┌─────────────▼──────────────┐                                │ │
│     │ Lambda: healthomics-data-  │ ←──────────────────────────────┘ │
│     │   analysis                 │                                  │
│     └─────────────┬──────────────┘                                  │
│                   │                                                  │
│     ┌─────────────▼──────────────┐                                  │
│     │ Step Functions Workflow    │                                  │
│     └─────────────┬──────────────┘                                  │
│                   │                                                  │
│     ┌─────────────▼──────────────┐                                  │
│     │ Lambda: start-healthomics- │                                  │
│     │   workflow                 │                                  │
│     └─────────────┬──────────────┘                                  │
│                   │                                                  │
│  6. Main Analysis Pipeline:                                        │
│     ┌─────────────▼──────────────┐                                  │
│     │ Healthomics Workflow       │                                  │
│     │ (RNA-Seq Analysis)         │                                  │
│     └─────────────┬──────────────┘                                  │
│                   │                                                  │
│  7. Final Output:                                                  │
│     ┌─────────────▼──────────────┐                                  │
│     │ S3: healthomics-processes  │                                  │
│     │   BAM files stored         │                                  │
│     └────────────────────────────┘                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

