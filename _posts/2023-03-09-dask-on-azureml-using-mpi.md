---
title: "Data preparation at scale on Azure Machine Learning using Dask"
layout: post
---

In a recent project, we worked with our customer on a sustainability project whose goal was to leverage geospatial and climate data to build a platform to perform machine learning model training and inference. The project was of the "machine learning operationalization" type, where we take a codebase that was written by data scientists and machine learning engineers, in the form of experimental Python notebooks, and we port this code to a platform where the processes of data preparation and model training can be automated using workflows and pipelines, and performed at scale.

One of the challenges of the project was to find a cost-effective way to process hundreds of gigabytes of geospatial data at scale, to create the machine learning features required to train the models. In this post we will introduce the general technical context of the project, and we will then detail how we leveraged Azure Machine Learning Compute Clusters to perform these large-scale data preparation tasks using the [Dask](https://www.dask.org/) framework.

## What is Dask?

[Dask](https://www.dask.org/) is a flexible library for parallel computing in Python. It can scale Python libraries such as [NumPy](http://www.numpy.org/), [Pandas](https://pandas.pydata.org/), [Scikit-Learn](https://scikit-learn.org/stable/), and more, to multi-core machines and [distributed clusters](https://distributed.dask.org/en/latest/) when datasets exceed memory. Dask has a familiar Python API that integrates natively with Python code to ensure consistency and minimize friction.

Dask is very popular with data scientists because it enables parallel computing with these familiar Python APIs, which makes it easy to use and integrate with existing code.

Dask works by providing parallelism to the existing Python stack. It does this by splitting large data structures, like arrays and dataframes, into smaller chunks that can be processed independently, and then combined. Dask also implements dynamic task scheduling that can optimize computation based on dependencies, resources, and priorities.

Dask is particularly popular in the world of [geoscience](https://stories.dask.org/en/latest/pangeo.html) and [geospatial data processing](https://www.youtube.com/watch?v=ZpA9jgSqAkk). In our project, Dask has been used to create a significant amount of code; the codebase represented about *5000 lines of Python code and 40 Jupyter notebooks*. Re-using this codebase was essential to accelerate the implementation of the project, since we had a starting point that might only need some refactoring in order to be executed in a batch environment. All we needed to do was to determine how to run this code at scale on the customer's platform!

## Running Dask jobs on Azure Machine Learning

The platform we were building was designed to leverage Azure Machine Learning from the start. In a few words, [Azure Machine Learning](https://azure.microsoft.com/en-us/products/machine-learning/) is a cloud-based platform that empowers data scientists and developers to build, deploy, and manage high-quality models faster and with confidence. It [supports several features](https://learn.microsoft.com/en-us/azure/machine-learning/overview-what-is-azure-machine-learning) such as compute options, datastores, notebooks, a designer GUI, and automated ML. It also enables industry-leading machine learning operations (MLOps) with tools such as CI/CD integration, MLFlow integration, and pipeline scheduling.

One of our ideas was to leverage the [Azure ML Compute Clusters](https://learn.microsoft.com/en-us/azure/machine-learning/concept-compute-target#azure-machine-learning-compute-managed) to run the Dask-based data preparation and feature engineering tasks at scale, without any code modification. This would be a huge optimization when compared to the cost of deploying a separate data processing component like Spark, and possibly having to rewrite the code to leverage this new infrastructure.

When we looked at Dask in detail, we found it can use a number of back-end platforms to distribute computing tasks, like [Kubernetes](https://docs.dask.org/en/stable/deploying-kubernetes.html), [Virtual Machines](https://cloudprovider.dask.org/en/latest/azure.html), etc.

One of the supported backends is the venerable [MPI](http://mpi.dask.org/en/latest/), a.k.a. Message Passing Interface, which is also [supported by Azure ML compute clusters](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-train-distributed-gpu#mpi). This means that the code written in the Planetary Computer Hub using Jupyter notebooks could be easily executed on Azure ML clusters in order to automate and scale the data preparation jobs.

In other words, since Azure ML Compute Clusters natively support MPI workloads, all that was needed to run Dask jobs at scale was to assemble the right environment and configuration.

### Create the environment

In Azure ML, we can [use Conda environment files](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-manage-environments-v2?tabs=cli#create-an-environment-from-a-conda-specification), that list all the required dependencies to run data preparation scripts on a cluster node. Here is a minimal example with the required libraries required to run Dask via MPI:

```yaml
# conda.yml
dependencies:
  - python=3.8
  - pip:
    - dask[complete]
    - dask_mpi
    - mpi4py
name: dask-mpi
```

To create the environment in Azure ML, we can use the [Azure command line with the Azure ML extension](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-configure-cli?tabs=public). This will upload our Conda environment definition, and trigger the build of the [Docker image](https://learn.microsoft.com/en-us/azure/machine-learning/concept-environments#building-environments-as-docker-images) that our cluster nodes will use:

```sh
az ml environment create \
-g $GROUP \
-w $WORKSPACE \
--name dask-mpi \
--conda-file conda.yml \
--image mcr.microsoft.com/azureml/openmpi4.1.0-ubuntu20.04
```

This will create an Azure ML environment called `dask-mpi`.

The base Docker image specified in the environment creation already contains all the necessary [OpenMPI](https://www.open-mpi.org/) libraries. You can see a full list of available base images in the GitHub repo [AzureML-Containers](https://github.com/Azure/AzureML-Containers).

### Create the compute cluster

We need an Azure ML Compute Cluster to run our script. The command below will create one with the following settings:

- VM size Standard_D8_v3, which is 8 vCPU and 32 GiB RAM. See [Supported VM series and sizes](https://learn.microsoft.com/en-us/azure/machine-learning/concept-compute-target#supported-vm-series-and-sizes) for a list of possible options.
- Maximum of 6 instances.
- Use your current SSH key so you can connect to the nodes.

```sh
az ml compute create \
-g $GROUP \
-w $WORKSPACE \
--type AmlCompute \
--name dask-cluster \
--size Standard_D8_v3 \
--max-instances 6 \
--admin-username azureuser \
--ssh-key-value "$(cat ~/.ssh/id_rsa.pub)"
```

### Run the script

To run the script on an Azure ML cluster, we will need a job definition file. Here is an example that we could use to execute the script `prep_nyctaxi.py`.

```yaml
# job.yml
$schema: https://azuremlschemas.azureedge.net/latest/commandJob.schema.json

display_name: dask-job
experiment_name: azureml-dask
description: Dask data preparation job

environment: azureml:dask-mpi@latest

compute: azureml:dask-cluster

inputs:
  nyc_taxi_dataset:
    path: wasbs://datasets@azuremlexamples.blob.core.windows.net/nyctaxi/
    mode: ro_mount

outputs:
  output_folder:
    type: uri_folder

distribution:
  type: mpi
  process_count_per_instance: 8
resources:
  instance_count: 4

code: src

command: >-
  python prep_nyctaxi.py --nyc_taxi_dataset ${{inputs.nyc_taxi_dataset}} --output_folder ${{outputs.output_folder}}
```

The important part of that file, regarding Dask, is the following section:

```yaml
distribution:
  type: mpi
  process_count_per_instance: 8
resources:
  instance_count: 4
```

This is where we request to run the script using an MPI cluster of 4 instances (`instance_count`) and 8 processes per instance (`process_count_per_instance`). You should adjust these numbers according to the configuration of your cluster.

The job also defines inputs and outputs, both mounted directly from Blob Storage to the compute nodes. This means the inputs and outputs will appear on all the nodes as local folders.

Also note that the job definition requests to use the `dask-mpi` environment that we created above.

The job execution can be triggered using the following command:

```sh
az ml job create -g $GROUP -w $WORKSPACE --file job.yml
```

You can then track the execution of the job in the Azure ML Studio. In the screen capture below, you can see the job using all four nodes to run a data preparation job.

![Dask job in Azure ML](/images/dask-job-cpu.png)

## Full source code

You can find a full working example in this GitHub repository: [dask-on-azureml-sample](https://github.com/tomconte/dask-on-azureml-sample). It includes a sample script, plus all the necessary Azure ML configuration.

## Conclusion

Using Dask-MPI and the native Azure ML MPI support, we were able to run our Dask-based data preparation at scale on Azure ML Compute Clusters, with minimal effort, no additional custom dependencies, and no code changes.
