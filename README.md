# PGTNet: A Process Graph Transformer Network for Remaining Time Prediction of Business Process Instances
This is the supplementary githob repository of the paper: "PGTNet: A Process Graph Transformer Network for Remaining Time Prediction of Business Process Instances".

Our approach consists of a data transformation from an event log to a graph dataset, and training a neural network based on the [GraphGPS: General Powerful Scalable Graph Transformers](https://github.com/rampasek/GraphGPS) recipe. 

**<a name="part1">1. Set up a Python environement to work with GPS Graph Transformers:</a>**

Based on the [instructions](https://github.com/rampasek/GraphGPS#python-environment-setup-with-conda), you need to set up a Python environement with Conda:
```
conda create -n graphgps python=3.10
conda activate graphgps

conda install pytorch=1.13 torchvision torchaudio pytorch-cuda=11.7 -c pytorch -c nvidia
conda install pyg=2.2 -c pyg -c conda-forge
pip install pyg-lib -f https://data.pyg.org/whl/torch-1.13.0+cu117.html

pip install pytorch-lightning yacs torchmetrics
pip install performer-pytorch
pip install tensorboardX
pip install ogb
pip install wandb

pip install pm4py
#pip install PyYAML #Requirement already satisfied
#pip install numpy  #Requirement already satisfied
#pip install scikit-learn #Requirement already satisfied

conda clean --all
```
Note that we included pip install command required for working with event log data (i.e., [pm4py](https://pm4py.fit.fraunhofer.de/) library). For other libraries, namely PyYAML numpy scikit-learn requirements should be already satisfied. Therefore, we do not need to install them separately (commented in the above piece of commands). 

**<a name="part2">2. Clone repositories and download event logs:</a>**

Once you setup a conda environement, clone the [GPS Graph Transformer repository](https://github.com/rampasek/GraphGPS) (It is called the  **GPS repository** in the remaining of this README file) as well as the current repository (i.e., the **PGTNet repository**). The **PGTNet repository** should be placed in one folder with the directory that is used as the root directory of **GPS repository**. The structure should be like this:
```
GraphGPS
│
├── configs
├── graphgps
├── PGTNet
├── run
├── tests
├── unittests
├── main.py
├──
├──
└── README.md
```
Now, we are ready to download all event logs that are used in our experiments. Note that, downloading event logs and converting them to graph datasets are not mandatory steps for training PGTNet because we already uploaded the resultant graph dataset [here](https://github.com/keyvan-amiri/PGTNet/tree/main/conversion/transformation). In case you want to start with training PGTNet, you can skip this step as well as the next step, and refer to [training](https://github.com/keyvan-amiri/PGTNet#part4) step. However, we have provided our source code for the sake of transparency. This source code also facilitates the use of PGTNet for other predictive process monitoring tasks (e.g., next activity prediction, next timestamp prediction, suffix prediction, outcome prediction), and indeed for a broader range of event logs. Our source code for conversion can also be adjusted to accomodate different graph representation of event prefixes.

To download all event logs run the following command:
```
python data-acquisition.py
```
All datasets are publicly available at [the 4TU Research Data repository](https://data.4tu.nl/categories/13500?categories=13503). The **data-acquisition.py** script download all event logs, and convert them into .xes format. It also generates additional event logs (BPIC12C, BPIC12W, BPIC12CW, BPIC12A, BPIC12O) from BPIC12 event log. Links that are used for downloading event logs are saved in **4TU-links.yaml** file. You can easily add other links to include more event logs into your experiments, or adjust the links if in future they would be relocated. Our implementation currently supports event logs in xes xes.gz and csv formats. In case your dataset is in csv format you might need to explicitly define labels that are used for mandatory event attributes, namely activity identifier, timestamp identifier, and case identifier (similar to what is done for Helpdesk event log in data-acquisition.py). 

**<a name="part3">3. Converting an event log into a graph dataset:</a>**

This section of the implementation focuses on the conversion of an event log into a graph dataset. We already uploaded the resultant graph dataset [here](https://github.com/keyvan-amiri/PGTNet/tree/main/conversion/transformation). Therefore, this step can be skipped if you are not intreseted in conducting more experiments with feature engineering. In this case, generated graph dataset are automatically downloaded and will be used to train and evaluaate PGTNet for remaining time prediction. In order to convert an event log into its corresponding graph dataset, you need to run the same python script with specific arguments:
```
python GTconvertor.py conversion_configs envpermit.yaml --overwrite true
```
The first argument (i.e., conversion_configs) is a name of directory in which all required configuration files are located. The second argument (i.e., envpermit.yaml) is the name of configuration file that is used for conversion. We will discuss this argument with more details in the followings. The last argument called overwrite is a Boolean variable which provides some flexibility. If it is set to false, and you have already converted the event log into its corresponding graph dataset the script simply skip repeating the task. 

**Conversion Configuration Files:**

Each conversion configuration file defines global variables specific to the dataset. These variables are used for converting the event log into its corresponding graph dataset and include:
1. `raw_dataset`: name of the raw dataset (i.e., event log).
2. `event_attributes`, `event_num_att`, `case_attributes`, `case_num_att`: Categorical and numerical attribute names at both the event-level and case-level. The implementation provides the opportunity to experiment with different combinations for these variables. Therefore, it is easy to conduct ablation studies or investigate contribution of different attributes to the accuracy of predictions. 
3. `train_val_test_ratio`: Training, validation, and test data ratio. By default, we use a 0.64-0.16-0.20 data split ratio. This means that we sort all traces based on the timestamps of their first events, and then use the first 64% for training set, the next 16% for validation set and the last 20% for test set. This is equivalent to holdout data split. Later, we will discuss how we can use cross-fold validation data split using training configuration files.
4. A boolean attribute called `target_normalization`. When `target_normalization` is set to True (the default value), the target attribute is normalized based on the duration of the longest case, ensuring values fall within the range of zero to one. This normalization proved to be helpful because the target attribuite often has a highly skewed distribution.

**The output for conversion step:** The resultant graph dataset will be saved in a seperate folder within **datasets** in the root directory for **GPS repository**. We will discuss the structure of the resultant graph dataset later. Note that, Running the `GTconvertor.py` script produces several additional output files, including:
1. Encoders: One-hot encoders for both case-level and event-level attributes, implemented using scikit-learn.
2. Activity Classes Dictionary: A dictionary that defines activity classes.
3. Filtered Cases: A list of case IDs for cases that do not have at least three events.
4. Histogram: A PNG file that visualizes the distribution of target attribute values.
All additional ouputs are saved in a separate folder called **transformation** in the root directory of **PGTNet repository**. 

**Note:** We provide additional text files describing general statistics for different graph datasets. See: [General statistics for graph datasets](https://github.com/keyvan-amiri/PGTNet/tree/main/graph_dataset_statistics).

**Dataset Structure:** Each graph dataset which represent set of event prefixes (obtained from the event log) is a [PyG data object](https://pytorch-geometric.readthedocs.io/en/latest/modules/data.html). In this graph dataset each attributed directed graph corresponds to an event prefix: an unfinished business process instance.

For each graph dataset, three separate files are generated for the training, validation, and test sets. These files are formatted as graph dataset objects compatible with PyTorch Geometric library. Loading a graph dataset to train a GPS graph transformer is done using one single zip file including all these three parts. While our evaluation relies on cross-validation data split, we initially create separate graph dataset files for direct use in the holdout approach. Modifying data split approach can be easily done by using a variable called `split_mode` in the relevant training configuration file. 


**<a name="part4">4. Training a PGTNet for remaining time prediction:</a>**

To train and evaluate PGTNet, we employ the implementation of [GraphGPS: General Powerful Scalable Graph Transformers](https://github.com/rampasek/GraphGPS). However, in order to use it for remaining time prediction of business process instances, you need to adjust some part of the original implementation. This can be achieved by running the following command:
```
python file_transfer.py
```
This script copies 5 important python scripts including are required adjustments:

a. In `main.py` , the most important change is that a train mode called 'event-inference' is added to customize the inference step.

b. In `master_loader.py`, the most important change is that several new dataset classes are added to handle graph representation of event logs.

c. The python script `GTeventlogHandler.py` includes multiple `InMemoryDataset` Pytorch Geometric classes. We created one seperate class for each event log.

d. The python scripts `linear_edge_encoder.py` and `two_layer_linear_edge_encoder.py` are specifically designed for edge embedding in the remaining cycle time prediction problem.

The `file_transfer.py` script also copy all required configuration files for training and evaluation of PGTNet to the relevant folder in **GPS repository**.

Once abovementioned adjustments are done, training PGTNet is straightforward. Training is done using the relevant .yml configuration file which specifies all hyperparameters and training parameters. All configuration files required to train PGTNet based on the event logs used in our experiments are collected [here](https://github.com/keyvan-amiri/PGTNet/tree/main/training_configs). For training PGTNet, you need to run the following:
```
python main.py --cfg configs/GPS/bpic2015m1-GPS+LapPE+RWSE-ckptbest.yaml run_multiple_splits [0,1,2,3,4] seed 42
```
As we mentioned in our paper, to evaluate robustness of our approach we trained and evaluated PGTNet using three different random seeds. These random seeds are 42,56,89. Each time you want to train PGTNet for specific event log and specific seed number, you should adjust the configuration file name, and the seed number in this command.

**<a name="part5">5. Inference with PGTNet:</a>**

The inference (i.e., get prediction of PGTNet for all examples in the test set) can be achieved similar to the training step. To do so, run commands like: 
```
python main.py --cfg configs/GPS/bpic2015m1-GPS+LapPE+RWSE-ckptbest-eventinference.yaml run_multiple_splits [0,1,2,3,4] seed 42
```
All configuration files required to inference with a PGTNet based on the event logs used in our experiments are collected [here](https://github.com/keyvan-amiri/PGTNet/tree/main/evaluation_configs).
