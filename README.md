# azodha

from fastapi import FastAPI
from fastapi.responses import JSONResponse

app = FastAPI()

@app.get("/health")
def health():
    return {"status": "ok"}

@app.get("/predict")
def predict():
    return JSONResponse(content={"score": 0.75})
fastapi
uvicorn

# ---------- Build stage ----------
FROM python:3.11-slim AS builder

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# ---------- Runtime stage ----------
FROM python:3.11-slim

RUN addgroup --system appgroup && adduser --system appuser --ingroup appgroup

WORKDIR /app
COPY --from=builder /usr/local/lib/python3.11/site-packages \
     /usr/local/lib/python3.11/site-packages
COPY app ./app

USER appuser

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=3s \
  CMD python -c "import requests; requests.get('http://localhost:8000/health')" || exit 1

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]


apiVersion: apps/v1
kind: Deployment
metadata:
  name: predictor
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
  selector:
    matchLabels:
      app: predictor
  template:
    metadata:
      labels:
        app: predictor
    spec:
      containers:
        - name: predictor
          image: <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/predictor:latest
          ports:
            - containerPort: 8000
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          livenessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 10
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 10


apiVersion: v1
kind: Service
metadata:
  name: predictor
spec:
  selector:
    app: predictor
  ports:
    - port: 80
      targetPort: 8000
  type: ClusterIP

name: CI-CD

on:
  push:
    branches: [ "main" ]

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: predictor

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run tests
        run: |
          python -m py_compile app/main.py

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/github-actions-ecr
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push image
        run: |
          docker build -t $ECR_REPOSITORY .
          docker tag $ECR_REPOSITORY:latest \
            <ACCOUNT_ID>.dkr.ecr.${AWS_REGION}.amazonaws.com/$ECR_REPOSITORY:latest
          docker push <ACCOUNT_ID>.dkr.ecr.${AWS_REGION}.amazonaws.com/$ECR_REPOSITORY:latest

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/github-actions-eks
          aws-region: ${{ env.AWS_REGION }}

      - name: Deploy to EKS
        run: |
          aws eks update-kubeconfig --name prod-cluster
          kubectl apply -f k8s/

