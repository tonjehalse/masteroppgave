# Masteroppgave

Used for the master's thesis TDT4900, spring 2027, by Tonje Halse. Much is the same as in this repo: https://github.com/eivindmidtbo/masteroppgave, and my repo is essentially a fork. That repo is again a fork of this repo: https://github.com/bjorafla/master.

In our thesis paper, the bucketing strategy referred to as *strict* is called *original* in the codebase.





# Setup

Tested locally on Windows 11 with Python 3.14. The original code was run on NTNU's IDUN cluster (Linux, Python 3.13).

### 1. Install a C compiler

`traj-dist` and `cityhash` are compiled during installation, so a C compiler is required.

### 2. Create a virtual environment and install dependencies

From the repository root:

```powershell
py -3.14 -m venv .venv
.\.venv\Scripts\Activate.ps1 
python -m pip install --upgrade pip setuptools wheel
pip install -r requirements.txt
```

### 3. Verify the installation

Run from the repository root:

```powershell
python -c "from computation import similarity; print('repo imports OK')"
```

### Datasets

The raw datasets are not included in the repository. Place them at:

- Porto ([ECML/PKDD 2015 taxi trajectories, Kaggle](https://www.kaggle.com/c/pkdd-15-predict-taxi-service-trajectory-i)): `dataset/porto/porto_raw.csv`
- Rome ([CRAWDAD roma/taxi](https://ieee-dataport.org/open-access/crawdad-romataxi)): `dataset/rome/rome_raw.txt`

# Troubleshooting and issues

- Fixed traj-dist package issues by following: https://github.com/bguillouet/traj-dist/issues/28
  - Forced integer division in frechet.pyx as shown in: https://stackoverflow.com/questions/64932145/cython-compile-error-cannot-assign-type-double-to-int-using-mingw64-in-win


### Folder structure

| **Folder Name**    | **Description**                             |
| ------------------ | ------------------------------------------- |
| `computation`      | Contains all computational processes.       |
| `dataset`          | Includes all datasets and generated hashes. |
| `results_hashed`   | Results of computations on hashed data.     |
| `results_analysis`   | Analysis **notebooks**.     |
| `results_true`     | Results of computations on true data.       |
| `schemes`          | Contains LSH schemes.                       |
| `traj-dist-master` | External trajectory similarity library.     |
| `utils`            | Helper functions.                           |
| `visualization`    | Includes visualizations.                    |

### Often used files

#### Analysis

| **File Name**                                                                                                                                         | **Type** | **Description**                                             |
| ----------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | ----------------------------------------------------------- |
| [`bucketing_compute_disk_similarity_values_hybrid.ipynb`](computation/similarity_values/hashed/bucketing_compute_disk_similarity_values_hybrid.ipynb) | Analysis | Notebook for analysis on the hybrid version for disk scheme |
| [`bucketing_compute_disk_similarity_values.ipynb`](computation/similarity_values/hashed/bucketing_compute_disk_similarity_values.ipynb)               | Analysis | Notebook for analysis on the disk scheme                    |
| [`bucketing_compute_grid_similarity_values_hybrid.ipynb`](computation/similarity_values/hashed/bucketing_compute_grid_similarity_values_hybrid.ipynb) | Analysis | Notebook for analysis on the hybrid version for grid scheme |
| [`bucketing_compute_grid_similarity_values.ipynb`](computation/similarity_values/hashed/bucketing_compute_grid_similarity_values.ipynb)               | Analysis | Notebook for analysis on the grid scheme                    |
| [`bucketing_correlation_disk.ipynb`](result_analysis/correlation/bucketing_correlation_disk.ipynb)                                                    | Analysis/Correlation | Notebook for correlation analysis on the disk scheme        |
| [`bucketing_correlation_grid.ipynb`](result_analysis/correlation/bucketing_correlation_grid.ipynb)                                                    | Analysis/Correlation | Notebook for correlation analysis on the grid scheme        |

#### Runtime

| **File Name**                                                                                                    | **Type**       | **Description**                                                                |
| ---------------------------------------------------------------------------------------------------------------- | -------------- | ------------------------------------------------------------------------------ |
| [`compute_hashed_bucketing_runtimes_hybrid.ipynb`](computation/runtimes/hashed/compute_hashed_bucketing_runtimes_hybrid.ipynb) | RUNTIME        | Notebook for measuring runtime for hybrid version of different schemes with bucketing            |
| [`compute_hashed_bucketing_runtimes.ipynb`](computation/runtimes/hashed/compute_hashed_bucketing_runtimes.ipynb) | RUNTIME        | Notebook for measuring runtime for different schemes with bucketing            |
| [`compute_hashed_runtimes.ipynb`](computation/runtimes/hashed/compute_hashed_runtimes.ipynb)                     | RUNTIME        | Notebook for measuring runtime for different schemes                           |
| [`compute_true_runtimes.ipynb`](computation/runtimes/true/compute_true_runtimes.ipynb)                           | RUNTIME        | Notebook for measuring runtime for similarity computation on TRUE trajectories |
| [`measure_similarities.py`](utils/helpers/measure_similarities.py)                                               | RUNTIME/HELPER | Sheet containing functions for measuring runtime                               |

### Helper Files

| **File Name**                                                | **Type** | **Description**                                 |
| ------------------------------------------------------------ | -------- | ----------------------------------------------- |
| [`lsh_bucketing.py`](schemes/lsh_bucketing.py)               | Helper   | Helper functions for bucketing logic            |
| [`bucket_evaluation.py`](utils/helpers/bucket_evaluation.py) | Helper   | Helper functions for bucketing analysis         |
| [`similarity.py`](computation/similarity.py)                 | Helper   | Sheet containing wrapper functions for analysis |
| [`disk_correlation_bucketing.py`](result_analysis/disk_correlation_bucketing.py)| Helper   | Sheet containing correlation functions for disk scheme |
| [`grid_correlation_bucketing.py`](result_analysis/grid_correlation_bucketing.py)| Helper   | Sheet containing correlation functions for grid scheme |
