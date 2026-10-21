# Machine Learning Operations Environment

[lumi-k-docs]: https://docs.lumi-supercomputer.eu/runjobs/lumi-k/getting-started/lumi_k_what_is/
[kubeflow-docs]: https://www.kubeflow.org/docs/
[mlflow-docs]: https://mlflow.org/docs/latest/ml/
[firecrest-docs]: https://docs.csc.fi/computing/firecrest/
[lumi-o-docs]: https://docs.lumi-supercomputer.eu/storage/lumio/
[kf-env]: https://kf.apps.lumi-k.eu/
[mlops-tutorial]: https://github.com/CSCfi/kubeflow-instructions-notebooks
[kf-trainer]: https://www.kubeflow.org/docs/components/trainer/

## WHAT??

LUMI AI Factory MLOps environment brings together tools and services for developing, training and deploying ML models. The environment provides a common platform for managing ML workflows throughout the development lifecycle, all in one user interface.

One of the main tools in the environment is **[Kubeflow][kubeflow-docs]**, which provides a platform for running and orchestrating machine learning workloads on Kubernetes. **[MLflow][mlflow-docs]** is used for tracking experiments, managing models, and monitoring the results of training runs.

At the heart of the environment is access to LUMI. Our custom Kubeflow Trainer `remote` runtime makes it possible to run jobs on LUMI directly from Kubeflow. The runtime uses **[FirecREST][firecrest-docs]** behind the scenes to transfer files, submit jobs, and monitor their execution on LUMI.

**[LUMI-O][lumi-o-docs]** provides object storage for project-related data and artifacts. It can be used to store datasets, trained models, scripts, and other files needed throughout the machine learning lifecycle, making them accessible to workloads running in the environment.

This allows users to take advantage of LUMI's HPC resources and project storage through the familiar Kubeflow workflow, without having to interact with FirecREST or manually manage jobs on LUMI.

The environment is built on **[LUMI-K][lumi-k-docs]**, providing the Kubernetes-based foundation for the platform.

## FOR WHOM??

The MLOps environment is for users who want to build and automate ML workflows and focus on their models rather than managing the infrastructure around them.

Whether you are building your first automated pipeline or running complex, resource-intensive workloads, the environment provides tools to simplify and automate the work around your models.

!!! success "No HPC expertise required"

    With Kubeflow and the `remote` runtime, you can access LUMI's HPC resources directly through the Kubeflow UI.

    You can submit and run jobs on LUMI without logging in to LUMI or managing Slurm, FirecREST, or the underlying infrastructure yourself.

## HOW??

Getting started with the MLOps environment only requires a few steps.

1. **Make sure** you have an active LUMI project with access to LUMI, LUMI-K, and LUMI-O.
2. **Log in** to CSC´s [Kubeflow][kf-env] and create a project for your work.
3. **Set up** your environment by following the instructions in the [MLOps tutorial][mlops-tutorial].
4. **Start working** using Kubeflow and the available MLOps tools.

Once the setup is finished, you can create and run your machine learning workloads through Kubeflow and use LUMI's HPC resources with the remote runtime.

**That's it.** You do not need to log in to LUMI separately or configure Slurm or FirecREST yourself.

### Using the custom Trainer runtime `remote`

!!! note "Using the environment effectively"

    The `remote` runtime, LUMI-O, and MLflow are designed to work together as part of your machine learning workflow. LUMI-O provides project storage for data and other artifacts, while MLflow tracks experiments and manages models, with its artifacts stored in LUMI-O.

    If you are not familiar with using LUMI-O or MLflow in your training scripts and Kubeflow pipelines, the [MLOps tutorial][mlops-tutorial] provides short, practical examples showing how these components can be integrated into your own work.

The remote runtime is used in the same way as other [Kubeflow Trainer][kf-trainer] runtimes. The main difference is that the remote runtime executes the job on LUMI instead of in the Kubeflow cluster.

The resource settings (`num_nodes`,`resources_per_node`) can be kept at the minimum values shown in the example below.

The remote runtime supports two ways of providing the files needed to run a job:

1. Slurm script from LUMI-O, training code as a Python function

    - The Slurm script is provided as an S3 URI using the slurm-uri annotation
    - The training code is provided as the func argument to CustomTrainer.

2. Both scripts from LUMI-O

    - Both the Slurm script and the training script are provided as S3 URIs using the slurm-uri and script-uri annotations
    - In this case, the func argument is required by the CustomTrainer API but is ignored by the remote runtime.

For example, when both scripts are stored in LUMI-O:
```python
from kubeflow.trainer import TrainerClient, CustomTrainer

client = TrainerClient()

# Update the path to point to your own LUMI-O bucket.
training_bucket = "<your-training-bucket-name>"

job_id = client.train(
    runtime=client.get_runtime("remote"),
    trainer=CustomTrainer(
        func=lambda *args, **kwargs: None, # Required dummy function when using script-uri
        num_nodes=1,
        resources_per_node={
            "cpu": 1,
            "memory": "1Gi"
        }
    ),
    options=[
        Annotations({
            "remote.trainer.kubeflow.org/slurm-uri":
                f"s3://{training_bucket}/slurm_script.sh",
            "remote.trainer.kubeflow.org/script-uri":
                f"s3://{training_bucket}/training_script.py"
        })
    ],
)

print(f"Submitted training job to LUMI: {job_id}")
print("Waiting for the job to complete...")

client.wait_for_job_status(job_id)

print(f"Training job completed successfully: {job_id}")
```

The `remote` runtime handles transferring the files to LUMI, submitting the Slurm job, and monitoring its execution.

You can check the pod logs after job is completed:
```python
print("\n".join(TrainerClient().get_job_logs(name=job_id, step="node-0")))
```

For a complete example, see the [MLOps tutorial][mlops-tutorial].