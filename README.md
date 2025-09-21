# End-to-End MLOps for Classification: Bank Marketing Case Study

## Introduction: Classification Problems and Applications

Classification is one of the most common problems in machine learning, where the goal is to assign an observation to one of several predefined categories. Typical applications include:

- **Anomaly Detection**: A specialized form of classification used to detect unusual patterns or rare events, such as fraudulent transactions, equipment failures, or abnormal customer behavior.  
- **Customer Retention**: Predicting whether a customer will churn or remain active.  
- **Credit Risk Assessment**: Classifying loan applicants as low-risk or high-risk.  
- **Healthcare**: Detecting diseases from patient records or medical images.  
- **Marketing**: Identifying customers most likely to respond positively to a campaign.  

By framing real-world challenges as classification problems, organizations can leverage data-driven insights to optimize decision-making, reduce risk, and increase operational efficiency.  

This project demonstrates how classification can be applied to a real-world banking dataset to predict customer responses to marketing campaigns. Beyond model training, the focus is on building a robust **MLOps pipeline** that ensures scalability, reliability, and continuous improvement.

--- 

- [Problem description](#Problem-description) 
- [Experiment tracking and model registry](#Experiment-tracking-and-model-registry)
- [Workflow orchestration](#Workflow-orchestration)
- [Model deployment](#Model-deployment)
- [Model monitoring](#Model-monitoring)
- [Cloud](#Cloud)
- [Reproducibility](#Reproducibility)

### Problem description

In 2012 a Portuguese banking institution collected data for several direct marketing campaigns it conducted in order to analyze it and to build machine learning models that can increase the efficiency of future marketing campaigns.

A marketing campaign is a concentrated effort by the bank in which it contacts its customers by phone and asks them to subscribe to a term deposit. Term deposits, aka certificate depoists, are deposits by customers that are made for a specific period of time and tradionally return more interest than savings accounts. They provide a guarantee for the banks that the money will remain available for a known period of time, which helps them better manage their available capitol.

In this project, I'll be using the this dataset to create an end-to-end MLOps dashboard to train, deploy, and monitor an ML model that predicts whether a customer subscribes to a term deposit. The data set can be downloaded [here](https://archive.ics.uci.edu/dataset/222/bank+marketing).

The original dataset has 16 features, and one target variable: 

| Variable Name | Role     | Type        | Description | Missing Values |
|---------------|----------|-------------|-------------|----------------|
| age           | feature  | integer     | age         | no             |
| job           | feature  | categorical | occupation  | no             |
| marital       | feature  | categorical | marital status | no             |
| education     | feature  | categorical | education level            | no             |
| default       | feature  | binary      | has credit in default? | no |
| balance       | feature  | integer     | average yearly balance | no |
| housing       | feature  | binary      | has housing loan? | no |
| loan          | feature  | binary      | has personal loan? | no |
| contact       | feature  | categorical | contact communication type ('cellular','telephone') | yes |
| day_of_week   | feature  | date        | last contact day of the week | no |
| month         | feature  | date        | last contact month of year ('jan', 'feb', 'mar', ...) | no |
| duration      | feature  | integer     | last contact duration, in seconds | no |
| campaign      | feature  | integer     | number of contacts performed during this campaign and for this client (includes last contact) | no |
| pdays         | feature  | integer     | number of days that passed by after the client was last contacted from a previous campaign (-1 means client was not previously contacted) | yes |
| previous      | feature  | integer     | number of contacts performed before this campaign and for this client | no |
| poutcome      | feature  | categorical | outcome of the previous marketing campaign ('failure','nonexistent','success') | yes |
| y             | target   | binary      | has the client subscribed a term deposit? | |:

I'm discarding the following 5 features: marital, education, default, loan, duration. For more details on how the features relate to the target, check out the EDA notebook [here](./EDA.ipynb).


Tech stack:  

1. MLflow for experiment tracking and resgistration  
2. Mage for orchestration  
3. Evidently + Grafana for monitoring  
4. Docker + Flask API + Streamlit for deployment  
5. Docker-compose for running multi-container applications  
6. Postgres database to store tracking, orchestration, and monitoring data

In the worfklow, I train multiple binary classifiers, deploy the best performing one, and monitor its performance. 

Below are the details.

### Experiment tracking and model registry

MLflow is used to: 

1. Track the model's name, hyperparameter valuess, accuracy score, model object  
2. Add the trained models to the MLflow model registry  
3. Transition the top 3 models to the staging phase  
4. Transition the top 1 model to production and save it as an artifact*.  

❕ PS: I'm also saving the preprocessing code as a binary file with pickle, but I'm not using MLflow for this because I only need to save it once. 

### Workflow orchestration

In Mage, the following pipelines are created:

**Prepare**  

- Ingests the data
- Splits the data into training and validation sets
- Prepares the data by selecting the relevant columns, type casting,and imputing missing values
- Transforms it using a dict vectorizor

**Train**  
- Initiates parallel training cycles for different models (RandomForrest, XGBClassifier, LogisticRegression, SVC, MLPClassifier)
- Logs the relevant details using MLflow
- Registers the trained models in MLflow's model registry

**Deploy**  
- Transitions the top 3 models to the staging phase
- Transitions the top 1 model to the production phase
- Creates a Flask API python script 
- Creates a Dockerfile with the python script and gunicorn that can be deployed on the cloud
- Runs the the Flask API
- Runs a streamlit interface with the API that can be used to test the model

**Test**

- Loads the test data using the same ingest-prepare-build blocks from the Prepare pipeline 
- Transforms the loaded data
- Loads the production model 
- Makes predictions and prints+saves the accuracy metric

**Monitoring**

- Creates table `grafana.evidently_metrics` 
- Loads the reference data (the validation data set) and the new test data
- Creates a report that shows prediction drift, data drift, and missing values

**Retrain**

- Triggers a retraining cycle if alerted

### Model deployment

The "Deploy" pipeline creates 2 payloads:  

- [predict.py](./mlops/payloads/predict.py) script: The production model is wrapped in a Flask API 
- [predict.dockerfile](./mlops/payloads/predict.dockerfile): The predict script containerized in a docker container

To build the docker container run this command

```bash
docker build -t predict:v01 -f ./mlops/payloads/predict.dockerfile .
```

### Model monitoring

- The Grafana dashboard monitors the following metrics:
    - Model performance: Bar chart showing train and validation accuracy (source: MLflow)
    - Share of Missing Values: Trend showing share of missing values per month (source: Evidently)
    - Number of Drifted Columns: Bar chart showing the number of drifted columsn per month, and a dotted-line shows the currently accepted threshold (source: Evidently) 
    - Prediction Drift: Bar Chart showing the prediction drift value per month (source: Evidently)
    - Pipeline Runs: Table showing information on pipeline runs (source: Mage)
    - Alerts: List showing the defined alerts and their status (source: Grafana)
- Currently, prediction drift is the only defined alert. The alert fires if a prediction drift higher than 0.1 is detected in any of the months. 
    - When the alert is fired it sends a webhook notification to Mage on port 6790
    - In the Retrain pipeline a listener listens to port 6790
    - If the listener receives a POST request it creates an alert log file *and triggers a train pipeline to retrain the models*


### Cloud

⚠️ Currently, the project is setup to be deployed locally (...)

### Reproducibility

#### Prerequisite

To install the prerequisites you can follow the instructions [here](./docs/setup.md) 

#### Deployment

Next, clone this rep and move into the project directory

```bash
git clone https://github.com/el-grudge/bank-marketing.git && cd bank-marketing
```

Once inside, run this command to start the services (Postgres, adminer, Grafana, Mage, MLflow):

```bash
docker-compose up
```

❕To capture back the prompt use the `--d` option when running the above command.

To validate that the services are up and running running this command

```bash
docker ps
```

You should see an output like this

![dockerps](./images/docker-status.png)

Now, go to [Mage](http://localhost:6789) and navigate to the pipeline page where you will see the 6 deployed pipelines

![pipelines](./images/pipelines.png)

Run the pipelines in the following order:

1. [Prepare](http://localhost:6789/pipelines/prepare/triggers): Will ingest and prepare the data  
2. [Train](http://localhost:6789/pipelines/train/triggers): Will train multiple classifiers, track the training data, the models, their hyperparameters, and metrics, and add them to the model registry  
3. [Deploy](http://localhost:6789/pipelines/deploy/triggers): Will transition the top 3 models to the staging stage, then move the top model to the production phase. Will also create the Flask API and inference endpoint using the production model, and the docker container that hosts the Flask API  
4. [Test](http://localhost:6789/pipelines/test/triggers): Will load and prepare the test data, load the production model, and use it for inference on the test data  
5. [Monitor](http://localhost:6789/pipelines/monitor/triggers): Will compare the validation data (reference) with the test data (current) and look for prediction drift, column drift, and number of missing values. Will create a table and store this information in it  
6. [Retrain](http://localhost:6789/pipelines/retrain/triggers): Will trigger a new training run  

To view the logged models and the model registry go to [MLflow](http://localhost:5000).

Tracking
![mlflow_tracking](./images/mlflow_tracking.png)

Model registry
![mlflow_registry](./images/mlflow_registry.png)

To view the monitoring dashboard go to [Grafana](http://localhost:3000).

![grafana](./images/grafana.png)

❗Grafana's initial credentials are admin/admin.

To interact with the model through streamlit dashboard go to [Streamlit app](http://localhost:8501)

![streamlit](./images/streamlit.png)
