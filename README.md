# MLLess

MLLess is a prototype system of a FaaS-based ML training on IBM Cloud Functions. MLLess is built on top
of [Lithops](https://github.com/lithops-cloud/lithops), a serverless computing framework. This repository contains
MLLess' source as well as examples and a configuration guide.

## Getting started

Before installing and configuring MLLess, make sure that Python 3.8.x and Docker are installed and configured on the
client machine. MLLess runs on Python 3.8.x, and a Docker container is required to build MLLess' Cython modules.

MLLess also requires a Redis and a RabbitMQ instances to communicate the serverless functions and share intermediate
state. These can be serverless services or dedicated servers, depending on the cloud provider. 
In case you are manually provisioning the servers, follow the configuration guide at [Setting up RabbitMQ and Redis](#setting-up-rabbitmq-and-redis). Ensure that both services are up and running before configuring MLLess.

### Client installation

1. Clone MLLess' repository on your local machine:

```bash
git clone https://github.com/pablogs98/MLLess
```

2. From the repository's root directory, install MLLess' Python dependencies:

```bash
pip3 install -r requirements.txt
```

3. Run MLLess' setup script to compile all Cython modules. This step **requires Docker to be installed**.

```bash
python3 setup.py build_ext --inplace
```

### Configuration

1. Before proceeding,
   follow [Lithops' configuration guide](https://github.com/lithops-cloud/lithops/blob/master/config/README.md) and
   configure the IBM Cloud Functions serverless compute backend and the IBM Cloud Object Storage backend.
2. Add the dataset bucket name's and Redis and RabbitMQ's configuration keys to Lithops' configuration file. For MLLess
   to work, the Redis instance should not be running in _protected mode_, hence it does not require authentication.

```yaml
rabbit_mq:
  rabbitmq_host: xxx.xxx.xxx.xxx
  rabbitmq_port: xxxxx
  rabbitmq_username: xxxxxxxx
  rabbitmq_password: xxxxxxxx

redis_hosts:
  - xxx.xxx.xxx.xxx

buckets:
  datasets: xxxxxxxx
```

### Datasets

The datasets used for experimentation are publicly available and can be downloaded in the following locations:

| Model      | Dataset |
| ----------- | ----------- |
| Logistic regression | [Criteo display advertising challenge dataset](http://labs.criteo.com/2014/02/kaggle-display-advertising-challenge-dataset/)       |
| Probabilistic matrix factorisation   | [Movielens 10M dataset](https://grouplens.org/datasets/movielens/10m/) and [Movielens 20M dataset](https://grouplens.org/datasets/movielens/20m/)|

The datasets must be partitioned in batches in a particular format and uploaded to IBM Cloud Object Storage 
(see [Partitioning a dataset](#partitioning-a-dataset)).

### Setting up RabbitMQ and Redis
This section explains how to manually set up RabbitMQ and Redis servers. This process is not necessary if you are using serverless RabbitMQ and Redis instances. 

The following guide has been tested on Ubuntu Server 20.04.

#### RabbitMQ installation and configuration

In the RabbitMQ server:

1. Install RabbitMQ Server:
```bash
apt install rabbitmq-server 
```

2. Create a RabbitMQ user:
```bash
rabbitmqctl add_user username password
```

3. Set the user's role and permissions:
```bash
rabbitmqctl set_user_tags username administrator
rabbitmqctl set_permissions -p / username ".*" ".*" ".*"
```

RabbitMQ's default port is 5672 and will bind to all interfaces unless it is manually specified in RabbitMQ's configuration file.

#### Redis installation and configuration

In the Redis server:

1. Install Redis Server:
```bash
apt install redis
```

2. Modify Redis' configuration file ```/etc/redis/redis.conf```. Change the following lines:
```text
bind 127.0.0.1 ::1   # Comment this line
...
protected-mode yes   # Change to protected-mode no
```

3. Restart Redis' daemon:
```bash
systemctl restart redis-server
```

## Examples

From a shell, make ```PYTHONPATH``` point to MLLess' root directory before running the examples.

### Running a ML training job

Once MLLess is configured, running a ML training job only requires the user to import the desired ML model and train it
as follows:

```python
from MLLess.algorithms import ProbabilisticMatrixFactorisation

# Dataset specific parameters
dataset = 'movielens/movielens-20m-12000'
n_minibatches = 1666
n_users = 138493
n_items = 27278

# Create and run PMF job
model = ProbabilisticMatrixFactorisation(n_workers=24,
                                         n_users=n_users, n_items=n_items, n_factors=20,
                                         end_threshold=0.68, significance_threshold=0.7,
                                         learning_rate=8.0, max_epochs=20)
model.fit(dataset, n_minibatches)
model.generate_stats('movielens-20m_results')
```

This example creates and executes a Probabilistic Matrix Factorisation job with MLLess' significance filter
enabled ```significance_threshold=0.7```, which stops once the convergence threshold of 0.68 MSE is achieved. It also
generates a set of files which contain execution statistics such as the total execution time, training cost or loss and
time per step.

Before starting the training, the desired dataset must be stored in IBM Cloud Object Storage.

For other examples, check out the [examples](examples) folder.

### Partitioning a dataset

MLLess requires datasets to be partitioned in batches in a certain format and stored in IBM Cloud Object Storage. We
provide a utility code which partitions the datasets used during our experimentation. The batch size, the separator and
the partitioned dataset's filenames must be passed as parameters. The following example shows how to partition the
Movielens-20M dataset.

```python
from MLLess.utils.preprocessing import Preprocessing
from MLLess.utils.preprocessing import DatasetTypes

dataset_type = DatasetTypes.MOVIELENS
dataset_path = 'ml-20m/ratings.csv'
minibatch_size = 4000
dataset_name = f'movielens/movielens-20m-{minibatch_size}'
separator = ','

p = Preprocessing(dataset_path, dataset_type, separator)
n_minibatches = p.partition(minibatch_size, dataset_name)
print(n_minibatches)
```

## References

Marc Sànchez-Artigas, Pablo Gimeno-Sarroca. MLLess: Towards Enhancing Cost Efficiency in Serverless Machine Learning
Training. **ACM Middleware 2021**.
