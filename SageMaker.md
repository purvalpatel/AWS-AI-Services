Build,Train,Manage your own AI models.

- BedRock : Use existing AI models easily.
- SageMaker : Build/Train/Manage your own AI models.


### Without SageMaker:
```
EC2 GPU Instances
      ↓
Jupyter Notebook
      ↓
Training Scripts
      ↓
Model Storage (S3)
      ↓
Inference Server
      ↓
Monitoring
```
You manage everything.

### With SageMaker:
```
DataSet -> SageMaker Notebook -> Training Job -> Trained Model -> Endpoint deployment -> Inference API
```

## AWS Manages the infrastructure.

- AWs SageMaker deployment actully run on EC2 instances.
- When you create: training jobs, notebook instances, Inference endpoints - SageMaker Automatically launches EC2 instances behind scenes.


## Deployment:
- Create Bucket S3
    - Upload Data into bucket .CSV
- Create SageMaker Notebook.
    - Upload Data ( from s3 )
    - Start training ( EC2 -> Training -> S3 model artificats )
    - Deploy Model ( Provides Endpoints )
