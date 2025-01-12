# AWS Last Mile Route Sequence Optimization

This repository demonstrates our solution to the [Amazon Last Mile Routing Research Challenge](https://routingchallenge.mit.edu/), which aims to integrate real-life experience of Amazon drivers into the solution of optimal route planning. The solution is based on our paper **[Learning from Drivers to Tackle the Amazon Last Mile Routing Research Challenge](https://arxiv.org/abs/2205.04001)**, and can be deployed as an Amazon SageMaker Processing Job. The diagram below shows two example sequences generated from the same Amazon route with 100 packages. The sequence on the left (*driver friendly*) is calculated by our solution and the one on the right (*cost optimal*) is generated by a conventional [Traveling Salesperson Problem](https://en.wikipedia.org/wiki/Travelling_salesman_problem) (TSP) solver. While the cost optimal sequence on the right requires less travel time in theory, it presents several narrow and sharp `V` turns and leaps. This may appear unappealing to drivers who are not ready to reverse course. The sequence on the left, in comparison, appears to form a convex hull that avoids those `V` turns altogether. For example, the first stop that enters the delivery cluster from the South East side is far away from the last stop that leaves the cluster in the South West. This sequence can thus be selected to improve driving experience.
| Driver friendly sequence             |  Time optimal sequence |
:-------------------------:|:-------------------------:
![](route_driver.png)  |  ![](route_cost.png)
travel time: 2.01 hours | travel time: 1.80 hours

: *Image caption - Blue solid circles are locations of Amazon delivery packages. Red arrows show the direction for each transit. The **actual depo location** is far down below (out of sight), and is shown as red solid circles for the purpose of visualization. Travel time reported here does not include service time spent at each delivery stop. Road networks are overlaid as gray lines and curves. Note that roadnetwork-mapped driving routes that result from these two sequences are not shown here.*

# Method Overview
Our solution hierarchically integrates [Markov model](https://en.wikipedia.org/wiki/Markov_model) training, online policy search (i.e. [Rollout](https://www.amazon.com/dp/1886529078/)), and off-the-shelf TSP solvers to produce driver friendly routes for last mile planning. The choice of the underlying TSP solver is flexible. For example, [our paper]((https://arxiv.org/)) reported the evaluation score of 0.0374 using [LKH](http://akira.ruc.dk/~keld/research/LKH/). This repository uses [OR-tools](https://github.com/google/or-tools) for simpler integration, and obtains a nearly identical score of 0.0372. These results are comparable to what the **top three** teams have achieved on the public [leaderboard](https://routingchallenge.mit.edu/last-mile-routing-challenge-team-performance-and-leaderboard/). More importantly, our method is completely learning-based and data-driven. The training process is lightweight and fast - it takes 4 ~ 5 seconds to finish training the PPM model using 6,000 route sequences on a single CPU instance. Since it does not require any hand-crafted heuristics, our method has the potential to solve large-scale route sequence planning problem in a continuously changing operating environment.

<img src="method.png" alt="An overview of our method" width="800"/>

Our method involves training and inference. The training phase produces a [Prediction by Partial Matching](https://en.wikipedia.org/wiki/Prediction_by_partial_matching) (PPM for short) model to extract sequential patterns from Travelling *Zones*. These patterns encode driver preferences and behaviour, and will be used to produce driver friendly routes. Training is generally fast. For example, it takes less than 5 seconds on the AWS `ml.m5.4xlarge` instance. During the inference phase, **Step 1** - the trained PPM model auto-regressively generates new zone sequences guided by the [Rollout method](https://web.mit.edu/dimitrib/www/rollout.pdf). **Step 2** - for each zone in the generated zone sequence, we use [OR-tools](https://github.com/google/or-tools) to solve a small TSP instance to produce *Stop* (namely, package delivery) sequence. **Step 3** - we join stop sequences from all zones as per the zone sequence order to form the *global* stop sequence as the final solution. The following pseudocode captures the main idea of our method.
```python
# Training
ppm_model = train_ppm_model(ground_truth)
# Inference
for route in all_routes:
    all_stops = []
    zone_set = get_zone_set(route)       # Step 0
    sorted_zones = ppm_rollout_sort_zone(zone_set, ppm_model) # Step 1
    for zone in sorted_zones:
        sorted_stops = TSP_solve(zone)   # Step 2
        all_stops += sorted_stops        # Step 3
```

# Folder structure
Python source files and shell scripts are structured as follows: 
``` bash
${repo_root_directory}         # e.g. amazon-sagemaker-amazon-routing-challenge-sol
└── aro/                       # Name of the Python module of our solution
    └── model/                 # model files
        ├── __init__.npy       # empty module file 
        ├── ortools_helper.py  # interface to invoke the OR-tools TSP solver
        ├── ppm.py             # implementation of the PPM model
        └── zone_utils.py      # Implementation of Step 1 and Step 2
├── example_inference_job.sh   # submit a SageMaker processing job
├── example_inference.sh       # test route sequence generation locally
├── inference_job.py           # Wrap inference.py as a SageMaker job
├── inference.py               # Glue code that combines step 0 to step 4
├── preprocessing.py           # Generate input files for training and inference
├── requirements_dev.txt       # External libraries used locally
├── requirements.txt           # External libraries rquired to run SageMaker jobs
├── setup.py                   # Installation file
└── train.py                   # Training script to train the PPM model
```

# Quick Start
The Quick Start instructions below are based on `macOS` and `Linux` OS with [AWS CLI installed](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).

## Step 1. Install
```bash
# Get the source code
git clone https://github.com/aws-samples/amazon-sagemaker-amazon-routing-challenge-sol.git
cd amazon-sagemaker-amazon-routing-challenge-sol

# Setup Python Environment
conda create --name aro python=3.8
conda activate aro # `aro` is the name of the python virtual environment, feel free to change it
# if on a SageMaker notebook instance, uncomment and execute the following command instead
# source activate aro

# Install the current version of the package
pip install -r requirements_dev.txt
pip install -e .
```

## Step 2. Obtain data
More specific instructions will be provided here once the dataset associated with the [Amazon Last Mile Routing Research Challenge](https://routingchallenge.mit.edu/) is made publicly avaialble at [Open Data on AWS](https://registry.opendata.aws/). 
The following code snippets assume you have downloaded the `train` and `evaluation` datasets to your local machine at `/tmp/Final_March_15_Data` and `/tmp/Final_June_18_Data` respectively.

## Step 3. Preprocess data
The datasets contain package (aka *stop* ) information, such as destination locations, parcel specifications, customer preferred time windows, expected service times, and zone identifiers. The preprocesing step converts this information from JSON to [Parquet](https://parquet.apache.org/) and [Numpy array format](https://numpy.org/doc/stable/reference/generated/numpy.lib.format.html) for easy access.
```bash
train_data_dir=Final_March_15_Data
eval_data_dir=Final_June_18_Data
mkdir data
# please replace the example source directory `/tmp` with actual path for downloaded datasets
mv /tmp/${train_data_dir} data/
mv /tmp/${eval_data_dir} data/

# generate package information in Parquet (training)
python preprocessing.py --act gen_route --data_dir  data/${train_data_dir}
# generate travel time matrix for all stops in a route (training)
python preprocessing.py --act gen_dist_mat --data_dir  data/${train_data_dir}
# generate zone information for each stop (training)
python preprocessing.py --act gen_zone_list --data_dir  data/${train_data_dir}
# generate ground-truth zone sequence for each route
python preprocessing.py --act gen_actual_zone --data_dir  data/${train_data_dir}
# generate package information in Parquet (evaluation)
python preprocessing.py --act gen_route --data_dir  data/${eval_data_dir}
# generate travel time matrix for all stops in a route (evaluation)
python preprocessing.py --act gen_dist_mat --data_dir  data/${eval_data_dir}
# generate zone information for each stop (evaluation)
python preprocessing.py --act gen_zone_list --data_dir  data/${eval_data_dir}
```

## Step 4. Upload pre-processed data to S3
We upload pre-processed data to Amazon S3 in order to run SageMaker processing jobs during the inference phase.
```bash
export bucket_name=my-route-bucket # set `${bucket_name}` to your own S3 bucket name
export s3_data_prefix=lmc # set `${s3_data_prefix}` to your own S3 data prefix
aws s3 sync data/${train_data_dir}/ s3://${bucket_name}/data/${s3_data_prefix}/${train_data_dir}/
aws s3 sync data/${eval_data_dir}/ s3://${bucket_name}/data/${s3_data_prefix}/${eval_data_dir}/
```
After completing this step, the prefix structure in your S3 bucket appears as follows:
``` bash
${train_or_eval_data_dir}                # e.g. `Final_March_15_Data` OR `Final_June_18_Data`
├── distance_matrix/                     # a directory with all distance matrix files
│   ├── ${route_id_0}_raw_w_st.npy       # distance matrix file produced by preprocessing.py 
│   ├── ...                              # more distance matrix files
│   └── ${route_id_N}_raw_w_st.npy       # distance matrix file
├── model_apply_inputs/                  # challenge input files
├── model_apply_outputs/                 # output json results here
├── model_build_inputs/                  # challenge input files
├── model_score_inputs/                  # challenge input files
├── model_score_outputs/                 # output json score here
├── processed/                           # output processed parquet file here
└── zone_list                            # A directory with all zone files
    ├── ${route_id_0}_zone_w_st.joblib   # zone file produced by preprocessing.py            
    ├── ...                              # more zone files
    ├── ${route_id_N}_zone_w_st.joblib   # the last zone file
    └── actual_zone-{mode}.csv           # ground-truth zone sequence file produced 
                                         # by preprocessing.py
```

## Step 5. Train the PPM model locally
We train the [Prediction by Partial Matching (PPM)](https://en.wikipedia.org/wiki/Prediction_by_partial_matching) model. In our work, the PPM model is used as a sequential probability model for generating synthetic zone sequences.
```bash
# train the PPM model
python train.py --train_zdf_fn data/${train_data_dir}/zone_list/actual_zone-train.csv
```

## Step 6. Upload the trained model to S3
We upload the PPM model to S3 so that the subsequent SageMake processing job can access the model to generate zone sequences.
```bash
# optional - set `${s3_model_prefix}` with your own S3 model prefix
export s3_model_prefix=almrc 
aws s3 cp aro_ppm_train_model.joblib s3://${bucket_name}/models/${s3_model_prefix}/
```

## Step 7. Generate sequence locally as a test
We run the inferernce for route generation locally (e.g. on your laptop or desktop) for the purpose of debugging or understanding the inner workings of our approach.
```bash
# test inference locally
./example_inference.sh
```
Please change your script as per any debugging information revealed in the output or error statements.

## Step 8. Run sequence generation as a SageMaker processing job
Now we are ready to generate routes by submititng an Amazon SageMaker processing job running on a `ml.m5.4xlarge` instance.
```bash
# please set environment variables (e.g. ${bucket_name}) 
# in `example_inference_job.sh` before running it
./example_inference_job.sh
```
Once submission is successful, we can navigate the browser to the Amazon SageMaker Processing jobs console to check if a job named `ppm-rollout-2022-xxx` is indeed running.

## Step 9. Check submission file
It generally takes less than an hour to complete generating 3,052 route sequences on an `ml.m5.4xlarge` instance. Once the job status becomes `completed`, we can check the generated sequences for all routes by running the following command.
```bash
aws s3 ls \
 s3://${bucket_name}/data/${s3_data_prefix}/${eval_data_dir}/model_apply_outputs/eval-ppm-rollout
```

## Step 10. Obtain evaluation scores
Once the submission file is downloaded, follow the evaluation instructions at https://github.com/MIT-CAVE/rc-cli
to calculate the evaluation score, which should be around `0.0372` ~ `0.0376`

## Step 11. Integrate this example into your last mile planning
If you are interested in potential applications of this example for your last mile planning, please feel free to contact the authors.
* [Chen Wu](https://github.com/chenwuperth)
* [Yin Song](https://github.com/yinsong1986)
* [Verdi March](https://github.com/verdimrc)
* [Eden Duthie](https://github.com/edenduthie)

# Security
See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

# License
This code is licensed under the Apache-2.0 License. See the LICENSE file.

# Acknowledgments
This code uses [OR-tools](https://github.com/google/or-tools), which is distributed under the Apache-2.0 License.