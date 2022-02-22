# Datasets

> In addition to Uber movement speed data, we include two time series datasets in this project. You can use both two datasets by following the Python codes below.

- [Energy consumption dataset](https://doi.org/10.6084/m9.figshare.c.5245457)

```python
import numpy as np
np.random.seed(1000)

dense_mat = np.load('energy_10w.npz')['arr_0']
dim = dense_mat.shape

missing_rate = 0.5

### Random missing (RM)
sparse_mat = dense_mat * np.round(np.random.rand(dim[0], dim[1]) + 0.5 - missing_rate)
```

- [NYC taxi dataset](https://transdim.github.io/dataset/NYC-taxi/)

```python
import numpy as np
np.random.seed(1000)

dense_mat = np.sum(np.load('yellow_taxi_tensor.npz')['arr_0'], axis = 1)
dim = dense_mat.shape

missing_rate = 0.5

### Random missing (RM)
sparse_mat = dense_mat * np.round(np.random.rand(dim[0], dim[1]) + 0.5 - missing_rate)
```