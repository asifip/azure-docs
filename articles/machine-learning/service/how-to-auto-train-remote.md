---
title: Set up remote compute targets for automated ML
titleSuffix: Azure Machine Learning service
description: This article explains how to build models using  automated machine learning on a Data Science Virtual machine (DSVM) remote compute target with Azure Machine Learning service
services: machine-learning
author: nacharya1
ms.author: nilesha
ms.reviewer: sgilley
ms.service: machine-learning
ms.component: core
ms.workload: data-services
ms.topic: conceptual
ms.date: 12/04/2018
ms.custom: seodec18

#Customer intent: As a professional data scientist, I can use automated machine learning (automated ML) functionality to build a model on a DSVM remote compute target.
---
# Train models with automated machine learning in the cloud

In Azure Machine Learning, you train your model on different types of compute resources that you manage. The compute target could be a local computer or a computer in the cloud.

You can easily scale up or scale out your machine learning experiment by adding additional compute targets. Compute target options include Ubuntu-based Data Science Virtual Machine (DSVM) or Azure Machine Learning Compute. The DSVM is a customized VM image on Microsoft’s Azure cloud built specifically for doing data science. It has many popular data science and other tools pre-installed and pre-configured.  

In this article, you learn how to build a model using automated ML on the DSVM.

## How does remote differ from local?

The tutorial "[Train a classification model with automated machine learning](tutorial-auto-train-models.md)" teaches you how to use a local computer to train model with automated ML.  The workflow when training locally also applies to  remote targets as well. However, with remote compute, automated ML experiment iterations are executed asynchronously. This functionality allows you to cancel a particular iteration, watch the status of the execution, or continue to work on other cells in the Jupyter notebook. To train remotely, you first create a remote compute target such as an Azure DSVM.  Then you configure the remote resource and submit your code there.

This article shows the extra steps needed to run an automated ML experiment on a remote DSVM.  The workspace object, `ws`, from the tutorial is used throughout the code here.

```python
ws = Workspace.from_config()
```

## Create resource

Create the DSVM in your workspace (`ws`) if it doesn't already exist. If the DSVM was previously created, this code skips the creation process and loads the existing resource detail into the `dsvm_compute` object.  

**Time estimate**: Creation of the VM takes approximately 5 minutes.

```python
from azureml.core.compute import DsvmCompute

dsvm_name = 'mydsvm' #Name your DSVM
try:
    dsvm_compute = DsvmCompute(ws, dsvm_name)
    print('found existing dsvm.')
except:
    print('creating new dsvm.')
    # Below is using a VM of SKU Standard_D2_v2 which is 2 core machine. You can check Azure virtual machines documentation for additional SKUs of VMs.
    dsvm_config = DsvmCompute.provisioning_configuration(vm_size = "Standard_D2_v2")
    dsvm_compute = DsvmCompute.create(ws, name = dsvm_name, provisioning_configuration = dsvm_config)
    dsvm_compute.wait_for_completion(show_output = True)
```

You can now use the `dsvm_compute` object as the remote compute target.

DSVM name restrictions include:
+ Must be shorter than 64 characters.  
+ Cannot include any of the following characters: 
  `\` ~ ! @ # $ % ^ & * ( ) = + _ [ ] { } \\\\ | ; : \' \\" , < > / ?.`

>[!Warning]
>If creation fails with a message about Marketplace purchase eligibility:
>    1. Go to the [Azure portal](https://portal.azure.com)
>    1. Start creating a DSVM 
>    1. Select "Want to create programmatically" to enable programmatic creation
>    1. Exit without actually creating the VM
>    1. Rerun the creation code

This code doesn't create a user name or password for the DSVM that is provisioned. If you want to connect directly to the VM, go to the [Azure portal](https://portal.azure.com) to create credentials.  

### Attach existing Linux DSVM

You can also attach an existing Linux DSVM as the compute target. This example utilizes an existing DSVM, but doesn't create a new resource.

> [!NOTE]
>
> The following code uses the `RemoteCompute` target class to attach an existing VM as your compute target.
> The `DsvmCompute` class will be deprecated in future releases in favor of this design pattern.

Run the following code to create the compute target from a pre-existing Linux DSVM.

```python
from azureml.core.compute import ComputeTarget, RemoteCompute 

attach_config = RemoteCompute.attach_configuration(username='<username>',
                                                   address='<ip_adress_or_fqdn>',
                                                   ssh_port=22,
                                                   private_key_file='./.ssh/id_rsa')
compute_target = ComputeTarget.attach(workspace=ws,
                                      name='attached_vm',
                                      attach_configuration=attach_config)

compute_target.wait_for_completion(show_output=True)
```

You can now use the `compute_target` object as the remote compute target.

## Access data using get_data file

Provide the remote resource access to your training data. For automated machine learning experiments running on remote compute, the data needs to be fetched using a `get_data()` function.  

To provide access, you must:
+ Create a get_data.py file containing a `get_data()` function 
* Place that file in a directory accessible as an absolute path 

You can encapsulate code to read data from a blob storage or local disk in the get_data.py file. In the following code sample, the data comes from the sklearn package.

>[!Warning]
>If you are using remote compute, then you must use `get_data()` where your data transformations are performed. If you need to install additional libraries for data transformations as part of get_data(), there are additional steps to be followed. Refer to the [auto-ml-dataprep sample notebook](https://aka.ms/aml-auto-ml-data-prep ) for details.


```python
# Create a project_folder if it doesn't exist
if not os.path.exists(project_folder):
    os.makedirs(project_folder)

#Write the get_data file.
%%writefile $project_folder/get_data.py

from sklearn import datasets
from scipy import sparse
import numpy as np

def get_data():
    
    digits = datasets.load_digits()
    X_digits = digits.data[10:,:]
    y_digits = digits.target[10:]

    return { "X" : X_digits, "y" : y_digits }
```

## Configure experiment

Specify the settings for `AutoMLConfig`.  (See a [full list of parameters](how-to-configure-auto-train.md#configure-experiment) and their possible values.)

In the settings, `run_configuration` is set to the `run_config` object, which contains the settings and configuration for the DSVM.  

```python
from azureml.train.automl import AutoMLConfig
import time
import logging

automl_settings = {
    "name": "AutoML_Demo_Experiment_{0}".format(time.time()),
    "iteration_timeout_minutes": 10,
    "iterations": 20,
    "n_cross_validations": 5,
    "primary_metric": 'AUC_weighted',
    "preprocess": False,
    "max_concurrent_iterations": 10,
    "verbosity": logging.INFO
}

automl_config = AutoMLConfig(task='classification',
                             debug_log='automl_errors.log',
                             path=project_folder,
                             compute_target = dsvm_compute,
                             data_script=project_folder + "/get_data.py",
                             **automl_settings,
                            )
```

### Enable model explanations

Set the optional `model_explainability` parameter in the `AutoMLConfig` constructor. Additionally, a validation dataframe object must be passed as a parameter `X_valid` to use the model explainability feature.

```python
automl_config = AutoMLConfig(task='classification',
                             debug_log='automl_errors.log',
                             path=project_folder,
                             compute_target = dsvm_compute,
                             data_script=project_folder + "/get_data.py",
                             **automl_settings,
                             model_explainability=True,
                             X_valid = X_test
                            )
```

## Submit training experiment

Now submit the configuration to automatically select the algorithm, hyper parameters, and train the model.

```python
from azureml.core.experiment import Experiment
experiment=Experiment(ws, 'automl_remote')
remote_run = experiment.submit(automl_config, show_output=True)
```

You will see output similar to the following example:

    Running on remote compute: mydsvmParent Run ID: AutoML_015ffe76-c331-406d-9bfd-0fd42d8ab7f6
    ***********************************************************************************************
    ITERATION: The iteration being evaluated.
    PIPELINE:  A summary description of the pipeline being evaluated.
    DURATION: Time taken for the current iteration.
    METRIC: The result of computing score on the fitted pipeline.
    BEST: The best observed score thus far.
    ***********************************************************************************************
    
     ITERATION     PIPELINE                               DURATION                METRIC      BEST
             2      Standardize SGD classifier            0:02:36                  0.954     0.954
             7      Normalizer DT                         0:02:22                  0.161     0.954
             0      Scale MaxAbs 1 extra trees            0:02:45                  0.936     0.954
             4      Robust Scaler SGD classifier          0:02:24                  0.867     0.954
             1      Normalizer kNN                        0:02:44                  0.984     0.984
             9      Normalizer extra trees                0:03:15                  0.834     0.984
             5      Robust Scaler DT                      0:02:18                  0.736     0.984
             8      Standardize kNN                       0:02:05                  0.981     0.984
             6      Standardize SVM                       0:02:18                  0.984     0.984
            10      Scale MaxAbs 1 DT                     0:02:18                  0.077     0.984
            11      Standardize SGD classifier            0:02:24                  0.863     0.984
             3      Standardize gradient boosting         0:03:03                  0.971     0.984
            12      Robust Scaler logistic regression     0:02:32                  0.955     0.984
            14      Scale MaxAbs 1 SVM                    0:02:15                  0.989     0.989
            13      Scale MaxAbs 1 gradient boosting      0:02:15                  0.971     0.989
            15      Robust Scaler kNN                     0:02:28                  0.904     0.989
            17      Standardize kNN                       0:02:22                  0.974     0.989
            16      Scale 0/1 gradient boosting           0:02:18                  0.968     0.989
            18      Scale 0/1 extra trees                 0:02:18                  0.828     0.989
            19      Robust Scaler kNN                     0:02:32                  0.983     0.989


## Explore results

You can use the same Jupyter widget as the one in [the training tutorial](tutorial-auto-train-models.md#explore-the-results) to see a graph and table of results.

```python
from azureml.widgets import RunDetails
RunDetails(remote_run).show()
```
Here is a static image of the widget.  In the notebook, you can click on any line in the table to see run properties and output logs for that run.   You can also use the dropdown above the graph to view a graph of each available metric for each iteration.

![widget table](./media/how-to-auto-train-remote/table.png)
![widget plot](./media/how-to-auto-train-remote/plot.png)

The widget displays a URL you can use to see and explore the individual run details.
 
### View logs

Find logs on the DSVM under `/tmp/azureml_run/{iterationid}/azureml-logs`.

## Best model explanation

Retrieving model explanation data allows you to see detailed information about the models to increase transparency into what's running on the back-end. In this example, you run model explanations only for the best fit model. If you run for all models in the pipeline, it will result in significant run time. Model explanation information includes:

* shape_values: The explanation information generated by shape lib
* expected_values: The expected value of the model applied to set of X_train data.
* overall_summary: The model level feature importance values sorted in descending order
* overall_imp: The feature names sorted in the same order as in overall_summary
* per_class_summary: The class level feature importance values sorted in descending order. Only available for the classification case
* per_class_imp: The feature names sorted in the same order as in per_class_summary. Only available for the classification case

Use the following code to select the best pipeline from your iterations. The `get_output` method returns the best run and the fitted model for the last fit invocation.

```python
best_run, fitted_model = remote_run.get_output()
```

Import the `retrieve_model_explanation` function and run on the best model.

```python
from azureml.train.automl.automlexplainer import retrieve_model_explanation

shape_values, expected_values, overall_summary, overall_imp, per_class_summary, per_class_imp = \
    retrieve_model_explanation(best_run)
```

Print results for the `best_run` explanation variables you want to view.

```python
print(overall_summary)
print(overall_imp)
print(per_class_summary)
print(per_class_imp)
```

Printing the `best_run` explanation summary variables results in the following output.

![Model explainability console output](./media/how-to-auto-train-remote/expl-print.png)

You can also visualize feature importance through the widget UI as well as the web UI on Azure portal inside your workspace.

![Model explainability UI](./media/how-to-auto-train-remote/model-exp.png)

## Example

The [how-to-use-azureml/automated-machine-learning/remote-execution/auto-ml-remote-execution.ipynb](https://github.com/Azure/MachineLearningNotebooks/blob/master/how-to-use-azureml/automated-machine-learning/remote-execution/auto-ml-remote-execution.ipynb) notebook demonstrates concepts in this article. 

[!INCLUDE [aml-clone-in-azure-notebook](../../../includes/aml-clone-for-examples.md)]

## Next steps

Learn [how to configure settings for automatic training](how-to-configure-auto-train.md).
