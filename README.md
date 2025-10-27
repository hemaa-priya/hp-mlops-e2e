# MLOps E2E on Databricks

An end-to-end MLOps reference on Databricks showcasing a simple Iris classification workflow across data ingestion, model training, evaluation, approval, deployment, and batch inference using Unity Catalog and MLflow.

## Repository Structure

```
/ hp-mlops-e2e
├── artifacts/
│   ├── 01_data_ingestion/
│   │   └── adls_to_uc_volume.ipynb
│   ├── 02_model_training/
│   │   └── pickle_to_mlflow_uc.ipynb
│   ├── 03_model_deployment/
│   │   ├── model_approval.ipynb
│   │   ├── model_deployment.ipynb
│   │   └── model_evaluation.ipynb
│   └── 04_model_inference/
│       └── batch_inference.ipynb
├── jobs/
│   ├── batch-inference-job.yml
│   ├── data-ingestion-job.yml
│   ├── e2e-job.yml
│   ├── model-deployment-job.yml
│   └── model-training-job.yml
└── databricks.yml
```

- **artifacts**: Databricks notebooks for each pipeline stage.
- **jobs**: Databricks bundle job definitions for each stage and a full end-to-end pipeline.
- **databricks.yml**: Databricks bundle configuration (targets, variables, permissions, and job includes).

## Key Concepts

- **MLflow**: Used for experiment tracking and model registry. Models are logged and versioned, then evaluated and promoted.
- **Databricks Jobs**: Orchestrate notebook tasks for each stage, with parameters and dependencies.
- **Volumes**: UC volumes are used to land artifacts like the demo `iris_model.pkl` and to persist checkpoints.

## Configurations

Configured in `databricks.yml` and referenced by `jobs/*.yml`.

- **Targets**:
  - `dev`
  - `prod`
- **Variables**:
  - `catalog_name`: default `hp_az_catalog`
  - `schema_name`: default `hp_iris_ml`
  - `environment`: `dev` or `prod`

Update these as needed for your workspace. Job parameters can be overridden at run time.

## Pipeline Overview

Stages in order (see `jobs/e2e-job.yml`):

1. **Data Ingestion** (`artifacts/01_data_ingestion/adls_to_uc_volume.ipynb`)
   - Reads from ADLS path and writes to UC Volume
   - Parameters: `source_path`, `volume_path`, `checkpoint_path`
2. **Model Training** (`artifacts/02_model_training/pickle_to_mlflow_uc.ipynb`)
   - Loads a pickle model or trains/logs model to MLflow and registers
   - Parameters: `pkl_path`, `experiment_path`, `model_name`
3. **Model Evaluation** (`artifacts/03_model_deployment/model_evaluation.ipynb`)
   - Evaluates a chosen `model_name` and `model_version`
4. **Model Approval** (`artifacts/03_model_deployment/model_approval.ipynb`)
   - Applies approval logic (e.g., thresholds) and sets state/alias
5. **Model Deployment** (`artifacts/03_model_deployment/model_deployment.ipynb`)
   - Promotes/serves the approved model
6. **Batch Inference** (`artifacts/04_model_inference/batch_inference.ipynb`)
   - Runs batch predictions and writes results to UC tables

## Sample Architecture

```
        +-------------------------+        +-----------------------+
        |  ADLS (source data)     |        |   Unity Catalog       |
        +------------+------------+        |  (Catalog/Schema)     |
                     |                      +----------+------------+
                     v                                 |
        +-------------------------+                    |
        | Data Ingestion Notebook |--writes--> UC Volume (files, checkpoints)
        +------------+------------+                    |
                     |                                 v
                     v                        +---------------------+
        +-------------------------+           |   MLflow Registry   |
        |  Model Training         |--logs-->  |  Experiments/Models |
        +------------+------------+           +----------+----------+
                     |                                   |
                     v                                   v
        +-------------------------+           +---------------------+
        |  Model Evaluation       |---------> |   Model Approval    |
        +------------+------------+           +----------+----------+
                     |                                   |
                     v                                   v
        +-------------------------+           +---------------------+
        |  Model Deployment       |---------> |  Batch Inference    |
        +-------------------------+           +---------------------+
```

## Prerequisites

- Databricks workspace access with Databricks CLI v0.205+ (bundles).
- Unity Catalog enabled and a catalog/schema you can write to.
- Access to the ADLS container and path configured in jobs.
- Permissions to run and manage Jobs in the selected environment.

## Setup

1. Install/upgrade Databricks CLI and configure your profile:
   ```bash
   pip install -U databricks-cli
   databricks configure --profile hp-mlops --host https://<your-workspace-host> --token <PAT>
   ```
2. Authenticate and set default profile (or pass `--profile` on each command).
3. Review and update `databricks.yml` variables and workspace host.
4. If needed, update job parameters in `jobs/*.yml` (e.g., ADLS `source_path`, UC `catalog_name`/`schema_name`, `experiment_path`).

## Deploy (Bundle)

- Validate the bundle:
  ```bash
  databricks bundle validate --profile hp-mlops --target dev
  ```
- Deploy resources (jobs) to the workspace:
  ```bash
  databricks bundle deploy --profile hp-mlops --target dev
  ```

## Run Jobs

You can run individual stage jobs or the end-to-end pipeline.

- Run Data Ingestion job:
  ```bash
  databricks bundle run data_ingestion_job --profile hp-mlops --target dev \
    --json '{"source_path":"abfss://<container>@<storage>.dfs.core.windows.net/<path>","volume_path":"/Volumes/<catalog>/<schema>/hp_ml_vol","checkpoint_path":"/Volumes/<catalog>/<schema>/hp_ml_vol/checkpoints"}'
  ```

- Run Model Training job:
  ```bash
  databricks bundle run model_training_job --profile hp-mlops --target dev \
    --json '{"pkl_path":"/Volumes/<catalog>/<schema>/hp_ml_vol/iris_model.pkl","experiment_path":"/Workspace/Users/<you>/hp_mlops_e2e/models/hp_iris_ml"}'
  ```

- Run Model Deployment job (evaluation → approval → deployment):
  ```bash
  databricks bundle run model_deployment_job --profile hp-mlops --target dev \
    --json '{"model_name":"<catalog>.<schema>.irisclassifierdemo","model_version":"5"}'
  ```

- Run Batch Inference job:
  ```bash
  databricks bundle run batch_inference_job --profile hp-mlops --target dev \
    --json '{"catalog_name":"<catalog>","schema_name":"<schema>"}'
  ```

- Run the End-to-End pipeline:
  ```bash
  databricks bundle run pipeline_job --profile hp-mlops --target dev
  ```

Notes:
- Swap `--target dev` with `--target prod` for production after updating permissions and variables.
- You can also trigger jobs in the Databricks UI after deployment.

## Troubleshooting

- Validate and deploy with `-v` for more logs:
  ```bash
  databricks -v bundle validate && databricks -v bundle deploy
  ```
- Check job run output in the Databricks UI (Runs tab) for notebook error details.
- Ensure your user/service principal has UC and ADLS permissions.
