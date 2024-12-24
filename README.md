# TPC-DS AWS EMR Benchmark Using

This is a fork of the two EMR Benchmark Repositories Provided By AWS Samples which can be found at the given links but lack proper documentation, this repository is a more beginner friendly & easy to understand.

EMR On EKS Benchmark: [https://github.com/aws-samples/emr-on-eks-benchmark](https://github.com/aws-samples/emr-on-eks-benchmark)

EMR Spark Benchmark: [https://github.com/aws-samples/emr-spark-benchmark](https://github.com/aws-samples/emr-spark-benchmark).

This repository provides a general tool to benchmark Spark performance. For more details on the benchmark and results, visit the following [http://ashharshah.com/blogs/blogslist/be43e55a-4a4f-465a-9217-d2d783a90fb2](http://ashharshah.com/blogs/blogslist/be43e55a-4a4f-465a-9217-d2d783a90fb2).

---

## Prerequisites

### Install `eksctl` (>= 0.143.0)

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv -v /tmp/eksctl /usr/local/bin
eksctl version
```

### Update AWS CLI (>= 2.11.23) on macOS

Refer to the [AWS CLI installation guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) for Linux or Windows.

```bash
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg ./AWSCLIV2.pkg -target /
aws --version
rm AWSCLIV2.pkg
```

### Install `kubectl` (>= 1.26.4)

Refer to the [installation guide](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/) for Linux or Windows.

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --short --client
```

### Install Helm CLI (>= 3.2.1)

```bash
curl -sSL https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
helm version --short
```

### Install Docker on macOS

Refer to [Docker installation](https://docs.docker.com/desktop/mac/install/) for other OS options.

```bash
brew install --cask docker
docker --version
```

---

## Generate 10TB TPC-DS Data

This project uses Amazon EKS to create a 10TB TPC-DS dataset in a specified S3 bucket. Ensure the AWS service quota for the R-type on-demand instance is set to **384 vCPUs**.

### 1. Set up the test environment

Run the `provision.sh` script to create a new EKS cluster, enable EMR on EKS, and build a private ECR for the `eks-spark-benchmark` utility Docker image.

```bash
export EKSCLUSTER_NAME=eks-nvme
export AWS_REGION=us-east-1
./provision.sh
```

### 2. Generate the TPC-DS dataset

Replace `<S3_BUCKET_HAS_TPCDS_DATASET>` with your S3 bucket name created via the provision script. The benchmark file contains a configmap that dynamically map your S3 bucket to the environment variable **codeBucket** in EKS.

```bash
app_code_bucket=<S3_BUCKET_HAS_TPCDS_DATASET>
kubectl create -n oss configmap special-config --from-literal=codeBucket=$app_code_bucket

kubectl apply -f ./tpcds-data-generation.yaml
```

Monitor data generation status using the Spark UI:

```bash
kubectl port-forward tpcds-data-generation-3t-driver 4040:4040 -n oss
```

### 3. Terminate the EKS cluster

Once the dataset generation is complete, deprovision the EKS cluster using `deprovision.sh`.

To delete the S3 bucket and its contents, uncomment the following lines in `deprovision.sh`:

```bash
aws s3 rm s3://$S3TEST_BUCKET --recursive
aws s3api delete-bucket --bucket $S3TEST_BUCKET
```

---

## Run Benchmark on the Generated Dataset

### 1. Create an EMR Cluster

#### Name & Applications

- EMR Version: **7.5.0**
- Installed Applications:
  - Hadoop 3.4.0
  - Hive 3.1.3
  - JupyterEnterpriseGateway 2.6.0
  - Livy 0.8.0
  - Spark 3.5.2

#### Cluster Configurations

- **Instance Groups**: One primary and one core node, both `c5.9xlarge` (36 vCPUs, 72 GiB memory, 600 GB EBS-only storage).
- **Auto-scaling**: Enabled, with a minimum of 2 and a maximum of 10 nodes (subject to quota restrictions).
- **Region**: Match the S3 bucket's region.
- **Networking**: Default VPC and public subnet.
- **Logs**: Store logs in your S3 bucket (e.g., `logs/` folder).
- **IAM Roles**:
  - Amazon EMR Service Role: Default VPC, public subnet, and security groups.
  - EC2 Instance Profile: Read/write access to all S3 buckets.

#### Security Configurations

- **Key Pair**: `emr-key.pem` for SSH access.

### 2. Submit the EMR Step

#### Upload the JAR Application

Upload `spark-benchmark-assembly-3.3.0.jar` to a `/jar/` folder in your S3 bucket.

#### Submit the Spark Job

Define the cluster ID and S3 bucket name:

```bash
export YOUR_CLUSTERID=<Cluster ID>
export YOUR_S3Bucket=<S3 Bucket Name>
```

Submit the job:

```bash
aws emr add-steps --cluster-id $YOUR_CLUSTERID --steps Type=Spark,Name="TPCDS Benchmark Job",Args=[--deploy-mode,cluster,--class,com.amazonaws.eks.tpcds.BenchmarkSQL,--conf,spark.driver.cores=4,--conf,spark.driver.memory=5g,--conf,spark.executor.cores=4,--conf,spark.executor.memory=6g,--conf,spark.executor.instances=47,--conf,spark.network.timeout=2000,--conf,spark.executor.heartbeatInterval=300s,--conf,spark.executor.memoryOverhead=2G,--conf,spark.driver.memoryOverhead=1000,--conf,spark.dynamicAllocation.enabled=false,--conf,spark.shuffle.service.enabled=false,s3://$YOUR_S3Bucket/jar/spark-benchmark-assembly-3.3.0.jar,s3://$YOUR_S3Bucket/BLOG_TPCDS-TEST-10T-partitioned/,s3://$YOUR_S3Bucket/BLOG_TPCDS-TEST-100G-RESULT,/opt/tpcds-kit/tools,parquet,3000,3,false,'q1-v2.4,q2-v2.4,...']
```
