# tracebase
Insufficient sampling and imputation treatment to traffic network states from mobile sensors

<br>

## Open Source

- Uber Movement: [Uber movement speed data of New York City](https://movement.uber.com/explore/new_york/speeds)
- Todd W. Schneider (2015). [Analyzing 1.1 billion NYC taxi and Uber trips, with a vengeance](https://toddwschneider.com/posts/analyzing-1-1-billion-nyc-taxi-and-uber-trips-with-a-vengeance/). [[GitHub](https://github.com/toddwschneider)]
- [OSMnx: Python for street networks. Retrieve, model, analyze, and visualize street networks and other spatial data from OpenStreetMap.](https://github.com/gboeing/osmnx)
- Geoff Boeing (2016). [OSMnx: Python for Street Networks](https://geoffboeing.com/2016/11/osmnx-python-street-networks/). Blog post.

<br>

## Data Processing

### Download Movement Speed Data

1. Open the download page of [Uber movement project](https://movement.uber.com/). Take an example like New York, please try [https://movement.uber.com/explore/new_york/speeds](https://movement.uber.com/explore/new_york/speeds).
2. Set the product as `speeds` and one specific time period.
3. Download data and save it on your computer.

### Extract Roads/Streets

Please download `movement-speeds-hourly-new-york-2019-1.csv` (movement speed data file of New York City in January 2019).

```python
import pandas as pd
import numpy as np

data = pd.read_csv('movement-speeds-hourly-new-york-2019-1.csv')
road = data.drop_duplicates(['osm_way_id', 'osm_start_node_id', 'osm_end_node_id'])
road = road.drop(['year', 'month', 'day', 'hour', 'utc_timestamp', 'segment_id', 'start_junction_id', 
                  'end_junction_id', 'speed_mph_mean', 'speed_mph_stddev'], axis = 1)
road.to_csv('road.csv')
```

In New York City, there are 98,210 roads/streets.

### Construct Speed Matrix

This process is time-consuming.

```python
import numpy as np
import pandas as pd

month = 1
data = pd.read_csv('movement-speeds-hourly-new-york-2019-{}.csv'.format(month))
road = pd.read_csv('road.csv')
tensor = np.zeros((road.shape[0], max(data.day.values), 24))
k = 0
for i in range(road.shape[0]):
    temp = data[(data['osm_way_id'] == road.osm_way_id.iloc[i]) 
                & (data['osm_start_node_id'] == road.osm_start_node_id.iloc[i]) 
                & (data['osm_end_node_id'] == road.osm_end_node_id.iloc[i])]
    for j in range(temp.shape[0]):
        tensor[k, temp.day.iloc[j] - 1, temp.hour.iloc[j]] = temp.speed_mph_mean.iloc[j]
    k += 1
    if (k % 1000) == 0:
        print(k)
mat = tensor.reshape([road.shape[0], max(data.day.values) * 24])
np.save('hourly_speed_mat_2019_{}.npy'.format(month), mat)

del data, tensor
```

The matrix's row corresponds to one specific road/street, while the column corresponds to one specific hour.

<br>

## Data Analysis

### Analyze missing rates

```python
## Build a speed matrix for the whole year of 2019 in NYC
mat = np.load('hourly_speed_mat_2019_1.npy')
for month in range(2, 13):
    mat = np.append(mat, np.load('hourly_speed_mat_2019_{}.npy'.format(month)), axis = 1)

## Calculate missing rates
print('The missing ratte of speed matrix is:')
print(len(np.where(mat == 0)[0]) / (mat.shape[0] * mat.shape[1]))

N, T = mat.shape
sample_rate = np.zeros(T)
for t in range(T):
    pos = np.where(mat[:, t] == 0)
    sample_rate[t] = len(pos[0]) / N
sample_rate = sample_rate[: 52 * 7 * 24].reshape([52, 24 * 7])
whole_rate = np.mean(sample_rate, axis = 0)
```

### Draw Missing Rates

```python
rate = len(np.where(mat == 0)[0]) / (mat.shape[0] * mat.shape[1])
print(rate)
```

```python
import matplotlib.pyplot as plt

plt.rcParams['font.size'] = 12
fig = plt.figure(figsize = (8, 2))
ax = fig.add_subplot(1, 1, 1)
plt.plot(whole_rate, color = 'red', linewidth = 1.8)
upper = whole_rate + np.std(sample_rate, axis = 0)
lower = whole_rate - np.std(sample_rate, axis = 0)
x_bound = np.append(np.append(np.append(np.array([0, 0]), np.arange(0, 7 * 24)), 
                              np.array([7 * 24 - 1, 7 * 24 - 1])), np.arange(7 * 24 - 1, -1, -1))
y_bound = np.append(np.append(np.append(np.array([upper[0], lower[0]]), lower), 
                              np.array([lower[-1], upper[-1]])), np.flip(upper))
plt.fill(x_bound, y_bound, color = 'red', alpha = 0.2)
plt.axhline(y = rate, color = 'gray', alpha = 0.5, linestyle='dashed')
plt.xticks(np.arange(0, 24 * 7 + 1, 1 * 24))
plt.xlabel('Time (hour)')
plt.ylabel('Missing rate')
plt.grid(axis = 'both', linestyle='dashed', linewidth = 0.1, color = 'gray')
ax.tick_params(direction = "in")
ax.set_xlim([-1, 7 * 24])
# ax.set_ylim([0.6, 1])
plt.show()
# fig.savefig("Seattle_missing_rate_stat.pdf", bbox_inches = "tight")
```

### Analyze Mean and Standard Deviation in Data

```python
import numpy as np
import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt

data = pd.read_csv('movement-speeds-hourly-seattle-2019-1.csv'.format(month))

fig = plt.figure(figsize = (8, 3))
plt.rcParams['font.size'] = 12

ax = fig.add_subplot(1, 2, 1)
sns.distplot(data.speed_mph_mean.values, kde_kws = dict(linewidth = 2.5), color = '#069AF3')
ax.tick_params(direction = "in")
ax.set_xlim([0, 80])
plt.xlabel('Speed (mph)')
plt.ylabel('Probability')

ax = fig.add_subplot(1, 2, 2)
sns.distplot(data.speed_mph_stddev.values, kde_kws = dict(linewidth = 2.5), color = '#069AF3')
ax.tick_params(direction = "in")
ax.set_xlim([0, 41])
plt.xlabel('Speed (mph)')
plt.ylabel('Probability')
fig.tight_layout()

plt.show()
# fig.savefig("Seattle_Jan_mean_std_speed_dist.pdf", bbox_inches = "tight")
```

<br>

## References

- A. M. Avila, I. Mezić (2020). [Data-driven analysis and forecasting of highway traffic dynamics](https://www.nature.com/articles/s41467-020-15582-5). Nature Communications, 11: 2090.
- Reese Pathak, Rajat Sen, Nikhil Rao, N. Benjamin Erichson, Michael I. Jordan, Inderjit S. Dhillon (2021). [Cluster-and-Conquer: A Framework for Time-Series Forecasting](https://arxiv.org/pdf/2110.14011.pdf). arXiv:2110.14011.
- Oliver Borchers (2019). [A Hitchhiker’s Guide to Mixture Density Networks](https://towardsdatascience.com/a-hitchhikers-guide-to-mixture-density-networks-76b435826cca). Blog post.
- Hao Zhang, Clarence W. Rowley, Eric A. Deem, Louis N. Cattafesta (2019). [Online Dynamic Mode Decomposition for Time-Varying Systems](http://cwrowley.princeton.edu/papers/Zhang-2019a.pdf). SIAM J. Applied Dynamical Systems, 18(3):  1586–1609.
- Christopher M. Bishop (1994). [Mixture Density Networks](https://publications.aston.ac.uk/id/eprint/373/1/NCRG_94_004.pdf).
- Wen-Ting Wang, Hsin-Cheng Huang (2017). [Regularized Principal Component Analysis for Spatial Data](https://doi.org/10.1080/10618600.2016.1157483). Journal of Computational and Graphical Statistics, 26(1): 14-25.
- Huiling Qin, Xianyuan Zhan, Yuanxun Li, Xiaodu Yang, Yu Zheng (2021).[Network-Wide Traffic States Imputation Using Self-interested Coalitional Learning](http://urban-computing.com/pdf/Network-Wide%20Traffic%20States%20Imputation%20Using%20Self-interested%20Coalitional%20Learning.pdf). KDD 2021.
