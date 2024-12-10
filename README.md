
# Workshop de Machine Learning com AWS SageMaker e EMR

Este guia detalhado descreve como configurar um ambiente AWS para criar um pipeline de Machine Learning usando Amazon SageMaker e Amazon EMR. O pipeline inclui as seguintes etapas:
1. Pré-processamento de dados com PySpark no EMR.
2. Treinamento de modelo com SageMaker.
3. Avaliação do modelo.
4. Registro do modelo no SageMaker Model Registry.

---

## **1. Pré-requisitos**

Antes de começar, certifique-se de que você tenha os seguintes itens configurados:
- Uma conta AWS ativa.
- AWS CLI instalado e configurado com as credenciais apropriadas.
- SageMaker Studio configurado, ou uma máquina local com Python 3.x e dependências instaladas.

---

## **2. Configuração dos Recursos AWS**

### **2.1 Criar Roles IAM**
O pipeline requer permissões específicas para interagir com os serviços AWS. Siga os passos abaixo:

#### **2.1.1 Role para SageMaker Execution**
1. Vá para o console AWS e acesse **IAM > Roles**.
2. Clique em **Create Role** e escolha **SageMaker**.
3. Anexe as políticas:
   - `AmazonSageMakerFullAccess`
   - `AmazonSageMakerPipelinesIntegrations`
   - `AmazonS3FullAccess` (ou uma política personalizada para o bucket específico).
4. Nomeie a role como `SageMakerExecutionRole`.

#### **2.1.2 Roles para EMR**
1. **Role de Serviço para EMR**:
   - Crie uma nova role e escolha **EMR**.
   - Anexe a política `AmazonElasticMapReduceRole`.
   - Salve como `EMR_DefaultRole_V2`.

2. **Role para EC2 no EMR**:
   - Crie outra role e escolha **EC2**.
   - Anexe as políticas:
     - `AmazonElasticMapReduceforEC2Role`
     - `AmazonS3FullAccess` (ou política personalizada).
   - Nomeie como `EMR_EC2_DefaultRole`.

---

### **2.2 Criar um Bucket no S3**
1. Crie um bucket para armazenar os dados do pipeline:
    ```bash
    aws s3 mb s3://<nome-do-seu-bucket>
    ```
2. Faça upload do dataset (Abalone.csv) para o bucket:
    ```bash
    aws s3 cp abalone.csv s3://<nome-do-seu-bucket>/datasets/abalone.csv
    ```

---

## **3. Configuração do Notebook**

### **3.1 Configurar Dependências**
1. Certifique-se de que as bibliotecas necessárias estão instaladas:
    ```bash
    pip install --upgrade sagemaker boto3
    ```

2. Crie um notebook no SageMaker Studio ou use Jupyter Notebook local.

### **3.2 Configurar o SageMaker**
1. Configure as sessões do SageMaker:
    ```python
    import sagemaker
    sagemaker_session = sagemaker.Session()
    role = sagemaker.get_execution_role()
    default_bucket = sagemaker_session.default_bucket()
    ```

2. Suba os scripts `preprocess.py` e `evaluate.py` para o bucket S3:
    ```bash
    aws s3 cp preprocess.py s3://<nome-do-seu-bucket>/code/
    aws s3 cp evaluate.py s3://<nome-do-seu-bucket>/code/
    ```

---

## **4. Configuração do Pipeline**

### **4.1 Configuração do EMR**
1. Configure os ARNs para as roles:
    ```python
    job_flow_role = "arn:aws:iam::<AWS_ACCOUNT_ID>:instance-profile/EMR_EC2_DefaultRole"
    service_role = "arn:aws:iam::<AWS_ACCOUNT_ID>:role/EMR_DefaultRole_V2"
    ```

2. Configure o EMR no pipeline:
    ```python
    from sagemaker.workflow.emr_step import EMRStep, EMRStepConfig

    emr_config = EMRStepConfig(
        jar="command-runner.jar",
        args=["spark-submit", "--deploy-mode", "cluster", script, "--input", input_data, "--output", output_path],
        cluster_config={
            "Applications": [{"Name": "Spark"}],
            "Instances": {
                "InstanceGroups": [
                    {"InstanceRole": "MASTER", "InstanceCount": 1, "InstanceType": "m5.2xlarge"},
                    {"InstanceRole": "CORE", "InstanceCount": 2, "InstanceType": "m5.2xlarge"},
                ]
            },
            "ReleaseLabel": "emr-6.6.0",
            "JobFlowRole": job_flow_role,
            "ServiceRole": service_role,
        },
    )
    ```

### **4.2 Treinamento do Modelo**
Configure o treinamento com o XGBoost:
```python
from sagemaker.estimator import Estimator

xgb_train = Estimator(
    image_uri="xgboost-image-uri",
    instance_type="ml.m5.xlarge",
    instance_count=1,
    output_path=f"s3://{default_bucket}/training_output",
    role=role,
)
xgb_train.set_hyperparameters(objective="reg:linear", num_round=50, max_depth=5, eta=0.2)
```

---

## **5. Executar o Pipeline**

1. Crie e suba o pipeline:
    ```python
    pipeline.upsert(role_arn=role)
    execution = pipeline.start()
    execution.wait()
    ```

2. Monitore a execução no console SageMaker.

---

## **6. Limpeza dos Recursos**
1. Exclua o pipeline:
    ```python
    sagemaker_client.delete_pipeline(PipelineName=pipeline_name)
    ```

2. Exclua o bucket:
    ```bash
    aws s3 rb s3://<nome-do-seu-bucket> --force
    ```

3. Revogue permissões extras no IAM.

---

## **Recursos Incluídos**
- `preprocess.py`: Script de pré-processamento de dados.
- `evaluate.py`: Script de avaliação do modelo.

Para mais detalhes, consulte a documentação oficial do [Amazon SageMaker](https://aws.amazon.com/sagemaker/) e [Amazon EMR](https://aws.amazon.com/emr/).

