# Download and Prepare Dataset

## Download

We use the **Taobao** and **Tmall** datasets. 

You can download the Taobao dataset from this [link](https://tianchi.aliyun.com/dataset/649). 
Please download ``UserBehavior.csv.zip`` (906M) from this website, unzip it, then you will get ``UserBehavior.csv`` (3.5G). 
Put it in this path: ``data/taobao/UserBehavior.csv``.

You can download the Tmall dataset from this [link](https://tianchi.aliyun.com/dataset/42). 
Please download ``data_format1 .zip`` (361M). 
Note that there exists a space in the file name, and it may cause some trouble. 
So it's highly recommended to first rename it as ``data_format1.zip``. 
Unzip it, then you will get four files. Among them, we only need ``user_log_format1.csv`` (1.8G).
Put it in this path: ``data/tmall/user_log_format1.csv``.

## Preprocess
To our knowledge, some versions of numpy will raise error like *The requested array has an inhomogeneous shape after 1 dimensions.* 
We believe we have fixed this issue. In case you still meet this error, you can simply add ``dtype=object`` when reading or saving a npy file to solve it.

Taobao:

```
cd preprocess
python taobao_v4.py
```

You will get ``train_{0001~0048}_{data_num}.npy``, ``val.npy`` and ``test.npy`` in the data/taobao folder.

The training dataset is very large, so we divide and store it in many files, and load them sequentially in the training process. This can avoid "out of memory" error.

Tmall:

```
python tmall_sort_log.py
python tmall_v4.py
```

The data in ``user_log_format1.csv`` has not been sorted by time. So we need to sort them first. Other operations are quite similar to Taobao dataset.



## Data Process Details

The following information may be helpful if you want to follow our code and do some further analysis.

### data format

After pre-processing, each sample will contain the following feature:

- uid: the ID of the user (not used in our code).
- target_iid: the item ID of the candidate.
- target_cid: the category ID of the candidate.
- label: whether the user will click the candidate.
- hist_iid: a list of item IDs that the user clicked in the history.
- hist_cid: a list of category IDs that the user clicked in the history.

### information about hist_iid and hist_cid

- They are sorted by time. hist_iid[-1] indicates the latest item that the user clicked; while hist_iid[0] refers to an item that is clicked a long time ago.
- Clip: the maximum length is 200. If a user history is too long, only recent 200 behaviors are reserved. 
- Pad：Pad with 0 if user history is shorter than 200.

### train/val/test split

- We filter out the users with history length less than 210.
- For a user with 300 behaviors ([1~300], heiger index means more recent), the train/val/test dataset will contain:
  - train: for j in range(19):  
    - i = 298 - 5 * j
    - given: [i-200 ~ i-1]
    - to predict: [i]
  - val:
    - given: [99 ~ 298]
    - to predict: [299]
  - test:
    - given: [100 ~ 299]
    - to predict: [300]

As we pointed out in our paper, in recommendation system, there are too many items and relatively too little training data, 
making the models suffer from "under-fitting". So we sample more than one piece of data from one user to expand the training dataset.
However, this would break the "independent identical distribution" principle, reducing the quality of data.

Thus, we finally make a trade-off and process the data like that, mainly for the two reason: 
for each user, there are 20 pieces of data visible in the training process; there is no padding in the test dataset. 
Experiments show that under this setting, all models are likely to perform better.

It's not easy to balance data quality and data quantity. If you would like to follow our research,
you can use our setting, or define your own processing method.

### Candidate Sampling

- The negative candidate is randomly sampled. 
- All item that **has not been clicked in the history** may be chosen.
- The probability is proportional to the frequency that it occurs in the whole dataset.
- There are two times negative candidates than positive ones.
