# Low latency item-to-item recommendation system

This directory contains code samples that demonstrate how to implement a low latency item-to-item recommendation solution. The foundation of the solution are [BigQuery](https://cloud.google.com/bigquery) and [ScaNN](https://github.com/google-research/google-research/tree/master/scann) - an open source library for efficient vector similarity search at scale.

There are two variants of the solution:
1. The first utilizes generally available releases of BigQuery and AI Platform together with open source components including [ScaNN](https://github.com/google-research/google-research/tree/master/scann) and [Kubeflow Pipelines](https://www.kubeflow.org/docs/pipelines/overview/pipelines-overview/)
2. The second one is a fully managed solution that leverages the experimental releases of AI Platform Pipelines and AI Platform ANN service.


## ScaNN and OSS Kubeflow Pipelines based solution 

The system utilizes [BigQuery ML Matrix Factorization](https://cloud.google.com/bigquery-ml/docs/reference/standard-sql/bigqueryml-syntax-create-matrix-factorization)
model to train the embeddings, and the open-source [ScaNN framework](https://ai.googleblog.com/2020/07/announcing-scann-efficient-vector.html) to build and
approximate nearest neighbour index.

1. Compute pointwise mutual information (PMI) between items based on their co-occurrences.
2. Train item embeddings using BigQuery ML Matrix Factorization, with item PMI as implicit feedback.
3. Export and post-process the embeddings from BigQuery ML model to Cloud Storage as CSV files using Cloud Dataflow.
4. Implement an embedding lookup model using Keras and deploy it to AI Platform Prediction.
5. Serve the embedding as an approximate nearest neighbor index using ScaNN on AI Platform Prediction for real-time similar items matching.

![Workflow](figures/diagram.png)

### Example Dataset

We use the public `bigquery-samples.playlists` BigQuery dataset to demonstrate
the solutions. We use the playlist data to learn embeddings for songs based on their co-occurrences
in different playlists. The learnt embeddings can be used to match and recommend relevant songs to a given song or playlist.

### Before you begin

Complete the following steps to set up your GCP environment:

1. In the [Cloud Console, on the project selector page](https://console.cloud.google.com/projectselector2/home/dashboard), select or create a Cloud project.
2. Make sure that [billing is enabled](https://cloud.google.com/billing/docs/how-to/modify-project) for your Google Cloud project. 
3. [Enable the APIs](https://console.cloud.google.com/apis/library)
 required for the solution: Compute Engine, Dataflow, Datastore, AI Platform, Artifact Registry, Identity and Access Management, Cloud Build, and BigQuery.
4. Use BigQuery [flat-rate or reservations](https://cloud.google.com/bigquery/docs/reservations-intro) to run BigQuery ML matrix factorization.
5. Create or have access to an existing [Cloud Storage bucket](https://cloud.google.com/storage/docs/creating-buckets).
6. Create a [Datastore database instance](https://cloud.google.com/datastore/docs/quickstart)  with Firestore in Datastore Mode.
7. [Create an AI Notebook Instance](https://cloud.google.com/ai-platform/notebooks/docs/create-new)  with TensorFlow 2.3 runtime.
8. [Create AI Platform Pipelines](https://cloud.google.com/ai-platform/pipelines/docs/setting-up) to run the TFX pipeline.

To go through the tasks for running the solution, you need to open the JupyterLab environment in the AI Notebook and clone the repository:
1. In the AI Platform Notebook list, click Open Jupyterlab. This opens the JupyterLab environment in your browser.
2. To launch a terminal tab, click the Terminal icon from the Launcher menu.
3. In the terminal, clone the `analytics-componentized-patterns` repository:

    ```git clone https://github.com/GoogleCloudPlatform/analytics-componentized-patterns.git```
   
4. Install the required libraries:

    ```
    cd analytics-componentized-patterns/retail/recommendation-system/bqml-scann
    pip install -r requirements.txt
    ```

When the command finishes, navigate to the `analytics-componentized-patterns/retail/recommendation-system/bqml-scann` directory in the file browser.

### Using the Notebooks to Run the Solution

We provide the following notebooks to prepare the BigQuery run environment 
and the steps of the solution:

**Preparing the BigQuery environment**

1. [00_prep_bq_and_datastore.ipynb](00_prep_bq_and_datastore.ipynb) - 
This is a prerequisite notebook that covers:
   1. Copying the `bigquery-samples.playlists.playlist` table to your BigQuery dataset.
   2. Exporting the songs information to Datastore so that you can lookup the information of a given song in real-time.
2. [00_prep_bq_procedures.ipynb](00_prep_bq_procedures.ipynb) - This is a prerequisite notebook that covers creating the BigQuery 
stored procedures executed by the solution.

**Running the solution**

1. [01_train_bqml_mf_pmi.ipynb](01_train_bqml_mf_pmi.ipynb) - This notebook covers computing pairwise item co-occurrences
to train the the BigQuery ML Matrix Factorization model, and generate embeddings for the items.
2. [02_export_bqml_mf_embeddings.ipynb](02_export_bqml_mf_embeddings.ipynb) - 
This notebook covers exporting the trained embeddings from the Matrix Factorization BigQuery ML Model to Cloud Storage,
as CSV files, using Apache Beam and Cloud Dataflow.
3. [03_create_embedding_lookup_model.ipynb](03_create_embedding_lookup_model.ipynb) - 
This notebook covers wrapping the item embeddings in a Keras model and exporting it
as a SavedModel, to act as an item-embedding lookup.
4. [04_build_embeddings_scann.ipynb](04_build_embeddings_scann.ipynb) - 
This notebook covers building an approximate nearest neighbor index for the embeddings 
using ScaNN and AI Platform Training. The built ScaNN index then is stored in Cloud Storage.

### Running the Solution using TFX on AI Platform Pipelines

In addition to manual steps outline above, we provide a [TFX pipeline](tfx_pipeline) that automates the process of building and deploying the solution:
1. Compute PMI using a [Custom Python function](https://www.tensorflow.org/tfx/guide/custom_function_component) component.
2. Train BigQuery ML matrix factorization model using a [Custom Python function](https://www.tensorflow.org/tfx/guide/custom_function_component) component.
3. Extract the Embeddings from the BigQuery ML model to a BigQuery table using a [Custom Python function](https://www.tensorflow.org/tfx/guide/custom_function_component) component.
4. Export the embeddings as TFRecords using the standard [BigQueryExampleGen](https://www.tensorflow.org/tfx/api_docs/python/tfx/extensions/google_cloud_big_query/example_gen/component/BigQueryExampleGen) component.
5. Import the schema for the embeddings using the standard [ImporterNode](https://www.tensorflow.org/tfx/api_docs/python/tfx/components/ImporterNode) component.
6. Validate the embeddings against the imported schema using the standard [StatisticsGen](https://www.tensorflow.org/tfx/guide/statsgen) and [ExampleValidator](https://www.tensorflow.org/tfx/guide/exampleval) components. 
7. Create an embedding lookup SavedModel using the standard [Trainer](https://www.tensorflow.org/tfx/api_docs/python/tfx/components/Trainer) component.
8. Push the embedding lookup model to a model registry directory using the standard [Pusher](https://www.tensorflow.org/tfx/guide/pusher) component.
9. Build the ScaNN index using the standard [Trainer](https://www.tensorflow.org/tfx/api_docs/python/tfx/components/Trainer) component.
10. Evaluate and validate the ScaNN index latency and recall by implementing a [TFX Custom Component](https://www.tensorflow.org/tfx/guide/custom_component).
11. Push the ScaNN index to a model registry directory using standard [Pusher](https://www.tensorflow.org/tfx/guide/pusher) component.

![tfx](figures/tfx.png)

The implementation of the pipeline is in the [tfx_pipeline](tfx_pipeline) directory. 
We provide the following notebooks to facilitate running the TFX pipeline:
1. [tfx01_interactive](tfx01_interactive.ipynb) - This notebook covers interactive execution of the 
TFX pipeline components.
2. [tfx02_deploy_run](tfx02_deploy_run.ipynb) - This notebook covers building the Docker container image required by
the TFX pipeline and the AI Platform Training job, compiling the TFX pipeline, and deploying the pipeline to 
AI Platform Pipelines.

### Deploying the Embedding Lookup and ScaNN index to AI Platform

After running the solution, an embedding lookup SaveModel and a ScaNN index will be produced.
To deploy these artifacts to AI Platform as prediction service, you can use the 
[05_deploy_lookup_and_scann_caip.ipynb](05_deploy_lookup_and_scann_caip.ipynb) notebook, which covers:
1. Deploying the Embedding Lookup SavedModel to AI Platform Prediction. 
2. Deploying the ScaNN index to AI Platform Prediction, using a Custom Container, for real-time similar item matching. 

The ScaNN matching service works as follows:
1. Accepts a query item Id.
2. Looks up the embedding of the query item Id from Embedding Lookup Model in AI Platform Prediction.
3. Uses the ScaNN index to find similar item Ids for the given query item embedding.
4. Returns a list of the similar item Ids to the query item Id.



## AI Platform ANN service and AI Platform (Unified) Pipelines based solution 

This is a fully managed variant of the solution that utilizes the new AI Platform  and AI Platform (Unified) Pipelines services. Note that both services are currently in the Experimental stage and that the provided examples may have to be updated when the services move to the Preview and eventually to the General Availability.

![Workflow Ann](figures/ann-flow.png)

The first three tasks:

1. Computing item co-occurrences,
2. Training item embeddings,
3. Post-processing and exporting the embeddings from BigQuery ML Matrix Factorization Model,

are the same as in the variant that utilizes the open-source ScaNN library.

After the item embeddings have been trained and exported to a BigQuery table the process diverges:

4. The item embeddings are exported from the BigQuery table to a set of JSONL formatted files
7. The ANN service is used to create an approximate nearest search index from the exported files
8. The index is deployed to the ANN service endpoint.

Similarly to the ScaNN library based solution, in addition to a manual step by step walk-through, we provide an example TFX pipeline that automates the process of training the embeddings and deploying the index.

The pipeline is designed to run on AI Platform (Unified) Pipelines and relies on features introduced in v0.25 of TFX. Each step of the pipeline is implemented as a [TFX Custom Python function component](https://www.tensorflow.org/tfx/guide/custom_function_component).

![TFX Ann](figures/ann-tfx.png)

1. The first step of the pipeline is to compute item co-occurences using BigQuery 
2. Next, the BQML Matrix Factorization model is created using the item co-occurance data created by the previous step. 
3. Item embeddings are extracted from the trained model weights and stored in a BigQuery table. 
4. The embeddings are exported in the JSONL format to the GCS location using the BigQuery extract job.
5. The embeddings in the JSONL format are used to create an ANN index.
6. Finally, the ANN index is deployed to an ANN endpoint.

All steps and their inputs and outputs are tracked in the AI Platform (Unified) ML Metadata service. 


### Setting up the environment

To use the Experimental releases of AI Platform Pipelines and ANN services you need to allow-list you project and user account.

Please contact your Google representative for more information and support.

## Using Jupyter notebooks

The solution implementation steps have been detailed in two Jupyter notebooks:

1. [ann-01-create-index.ipynb](ann-01-create-index.ipynb) -
This notebook walks you through creating an ANN index, creating an ANN endpoint, and deploying the index to the endpoint. It also shows how to call the interfaces exposed by the deployed index.

2. [ann-02-create-tfx-pipeline.ipynb](ann-02-create-tfx-pipeline.ipynb) -
This notebook demonstrates how to create and test the TFX pipeline and how to submit pipeline runs to AI Platform (Unfied) Pipelines.

Before experimenting with the notebooks make sure that you have prepared the BigQuery environment and trained and extracted item embeddings using the procedures described in the ScaNN library based solution.



## License

Copyright 2020 Google LLC

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License. You may obtain a copy of the License at: http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. 

See the License for the specific language governing permissions and limitations under the License.

**This is not an official Google product but sample code provided for an educational purpose**



In both variants, [BigQuery ML Matrix Factorization](https://cloud.google.com/bigquery-ml/docs/reference/standard-sql/bigqueryml-syntax-create-matrix-factorization)
model is used to train item embeddings, which are then used to create and deploy a scalable and high performance approximate nearest neighbors search index and deploy it as an online service. In the first variant, the index is built using the ScaNN library and deployed using AI Platform Prediction. In the second variant, the managed ANN service is used to both create and deploy the index.

The prescriptive guidance for implementing the systems has been structured as series of tasks. The first three tasks that describe the process of creating item embeddings are the same for both variants. 

1. Compute pointwise mutual information (PMI) between items based on their co-occurrences.
2. Train item embeddings using BigQuery ML Matrix Factorization, with item PMI as implicit feedback.
3. Post-process and export the embeddings from BigQuery ML model to a BigQuery table.

After the item embeddings have been created and exported to the BigQuery table, you can follow the below tasks to implement the first variant of the system: 

4. Implement an embedding lookup model using Keras and deploy it to AI Platform Prediction.
5. Create the approximate nearest neighbors index using the ScaNN library
6. Deploy the index using AI Platform Prediction.

Or the following tasks for the second variant:

7. Create an ANN index
8. Deploy the index to the ANN endpoint

The following diagram summarizes the workflow to implement the first variant: