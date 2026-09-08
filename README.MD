# Churn Prediction MLOps Demo

This guide shows how to generate data, train the churn model, upload it to S3 with DVC, and deploy it on EKS with KServe and Traefik.

## POC Screenshots

The screenshots below show the completed proof of concept:

### Local prediction API

The local FastAPI endpoint returns a churn prediction and probability.

![Local API prediction](Assests/Screenshot%202026-09-08%20182340.png)

### Model stored in Amazon S3

The trained model is available in the S3 bucket used by DVC and KServe.

![S3 model storage](Assests/Screenshot%202026-09-05%20021251.png)

### KServe deployment

The KServe application shows the ServiceAccount, InferenceService, Middleware, Ingress, service, deployment, and running pod.

![KServe deployment](Assests/Screenshot%202026-09-05%20023554.png)

### EKS cluster

The EKS cluster is active and ready to run the model workload.

![EKS cluster](Assests/Screenshot%202026-09-05%20023613.png)

### Kubernetes pods

The Kubernetes dashboard shows the running cluster workloads.

![Kubernetes pods](Assests/Screenshot%202026-09-02%20181537.png)

### Kubeflow Pipelines

Kubeflow Pipelines is available for managing machine learning workflows.

![Kubeflow Pipelines](Assests/Screenshot%202026-09-02%20181556.png)

### AWS IAM role

The IAM role provides S3 access for the KServe ServiceAccount.

![AWS IAM role](Assests/Screenshot%202026-09-05%20023648.png)

### Additional deployment view

![Deployment status](Assests/Screenshot%202026-09-05%20023626.png)

## 1. Install the tools

Install these tools before starting:

- Python 3
- AWS CLI
- `kubectl`
- `eksctl`
- Helm
- Access to an EKS cluster

## 2. Create a Python environment

Run these commands from the project directory:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
pip install dvc dvc-s3
```

## 3. Generate data and train the model

```bash
python generate_data.py
python train.py
```

The model is saved to:

```text
models/churn_model.pkl
```

## 4. Test the API locally

Start the API:

```bash
python api.py
```

In another terminal, test the health endpoint:

```bash
curl http://localhost:8000/health
```

Test a prediction:

```bash
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{
    "age": 45,
    "tenure_months": 24,
    "monthly_charges": 79.99,
    "total_charges": 1920.00,
    "num_support_calls": 3
  }'
```

Open the API documentation at <http://localhost:8000/docs>.

Stop the API with `Ctrl+C` when finished.

## 5. Store the model with DVC

Initialize DVC once:

```bash
dvc init
dvc remote add -d modelstore s3://YOUR_BUCKET/YOUR_FOLDER
```

Replace `YOUR_BUCKET` and `YOUR_FOLDER` with your S3 values. Configure AWS credentials first:

```bash
aws configure
```

Track the model and push it to S3:

```bash
dvc add models/churn_model.pkl
dvc push
```

Commit the DVC files to Git:

```bash
git add .dvc/config models/churn_model.pkl.dvc
git commit -m "Track churn model with DVC"
```

## 6. Set deployment variables

Use your own values in the commands below:

```bash
export CLUSTER_NAME=YOUR_EKS_CLUSTER
export AWS_REGION=ap-southeast-1
export AWS_ACCOUNT_ID=YOUR_AWS_ACCOUNT_ID
export S3_BUCKET=YOUR_BUCKET
export IAM_ROLE_NAME=kserve-s3-access-role
```

Create the namespace:

```bash
kubectl create namespace ml
```

## 7. Enable IRSA for EKS

Associate the IAM OIDC provider with the cluster:

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --approve
```

Get the OIDC provider ID:

```bash
aws eks describe-cluster \
  --name "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --query 'cluster.identity.oidc.issuer' \
  --output text
```

Create `trust-policy.json`. Replace `OIDC_PROVIDER_ID` with the ID from the previous command:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_AWS_ACCOUNT_ID:oidc-provider/oidc.eks.ap-southeast-1.amazonaws.com/id/OIDC_PROVIDER_ID"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.ap-southeast-1.amazonaws.com/id/OIDC_PROVIDER_ID:sub": "system:serviceaccount:ml:sa-s3-access",
          "oidc.eks.ap-southeast-1.amazonaws.com/id/OIDC_PROVIDER_ID:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

Create the IAM role and grant it read access to S3:

```bash
aws iam create-role \
  --role-name "$IAM_ROLE_NAME" \
  --assume-role-policy-document file://trust-policy.json

aws iam attach-role-policy \
  --role-name "$IAM_ROLE_NAME" \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

## 8. Update the Kubernetes files

Update these values before applying the manifests:

- S3 model path in `k8s/inference.yaml`
- IAM role ARN in `k8s/serviceaccount.yaml`
- S3 region in `k8s/serviceaccount.yaml`

The model path must contain the directory where DVC uploads the model, for example:

```text
s3://YOUR_BUCKET/YOUR_FOLDER/models/churn_model.pkl
```

Apply the ServiceAccount and KServe service:

```bash
kubectl apply -f k8s/serviceaccount.yaml
kubectl apply -f k8s/inference.yaml
```

Check the deployment:

```bash
kubectl get pods -n ml
kubectl get inferenceservice -n ml
```

Wait until the InferenceService shows `READY=True`.

## 9. Install Traefik

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update

helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace
```

Check the external address:

```bash
kubectl get service -n traefik
```

## 10. Expose the prediction endpoint

Apply the middleware and Ingress:

```bash
kubectl apply -f k8s/traefik.yaml
kubectl get ingress -n ml
```

Set the Traefik address:

```bash
export TRAEFIK_ADDRESS=YOUR_TRAEFIK_EXTERNAL_ADDRESS
```

Call the deployed model. The payload uses the KServe V1 format and keeps the same five feature order used during training:

```bash
curl -X POST "http://$TRAEFIK_ADDRESS/predict" \
  -H "Content-Type: application/json" \
  -d '{
    "instances": [
      [45, 24, 79.99, 1920.00, 3]
    ]
  }'
```

## Troubleshooting

### `InvalidIdentityTokenException`

The EKS OIDC provider is missing or the trust policy values are wrong.

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --approve
```

Check that the trust policy contains the exact namespace and ServiceAccount:

```text
system:serviceaccount:ml:sa-s3-access
```

### Pod is `Init:Error` or `CrashLoopBackOff`

Check the storage initializer logs:

```bash
kubectl get pods -n ml
kubectl logs POD_NAME -n ml -c storage-initializer
```

Confirm that the S3 path exists and that the IAM role can read it.

### InferenceService is not ready

```bash
kubectl describe inferenceservice churn-predictor -n ml
kubectl get pods -n ml
```

Check the model path, ServiceAccount, IAM role, and S3 region.

### `404 Not Found` from Traefik

Check that the middleware and Ingress are installed:

```bash
kubectl get middleware -n ml
kubectl describe ingress churn-predictor-ingress -n ml
```

The Ingress must route `/predict` and rewrite it to:

```text
/v1/models/churn-predictor:predict
```

### `X has N features` error

The request must contain exactly five values in this order:

```text
age, tenure_months, monthly_charges, total_charges, num_support_calls
```

### Traefik `EXTERNAL-IP` is `<pending>`

```bash
kubectl get service -n traefik
kubectl describe service traefik -n traefik
```

Wait for AWS load balancer provisioning and verify that the EKS subnets have the required load balancer tags.

### Useful logs

```bash
kubectl logs -l serving.kserve.io/inferenceservice=churn-predictor \
  -n ml -c kserve-container
```
