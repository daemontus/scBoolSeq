
# scBoolSeq

scRNA-Seq data binarisation and synthetic generation from Boolean dynamics.

## Installation

### Conda

```
conda install -c conda-forge -c colomoto scboolseq
```

### Pip

You need `R` installed, see the specification of the R dependencies below.

```
pip install scboolseq
```

### R

Having R installed, copying and executing the following snippet will install all of `scBoolSeq`'s dependencies:

```R
r_dependencies <- c("mclust", "diptest", "moments", "magrittr", "tidyr", "dplyr", 
    "tibble", "bigmemory", "doSNOW", "foreach", "glue")
# try to load all the dependencies :
.deps_loaded <- sapply(r_dependencies, require, character.only=T)
# retrieve those which are not installed
.not_installed <- Filter(function(x) !x, .deps_loaded)
.to_install <- names(.not_installed)
install.packages(.to_install)
```


## Usage

### Command line

scBoolSeq provides a rich CLI allowing programmatic access to its main functionalities, namely the `binarization` of RNA-Seq data and the 
generation of synthetic RNA-Seq data `synthesis` reflecting activation states from Boolean Network simulations. Once correctly instaled, 
the tool's and subcommand's help explain all the possible parameters. Some minimal examples are here presented.

#### Main CLI

```bash
$ scBoolSeq -h
usage: scBoolSeq <command> [<args>]

Available commands:
	* binarize	 Binarize a RNA-Seq dataset.
	* synthesize	 Simulate a RNA-Seq experiment from Boolean dynamics.
	* from_file	 Repeat a binarization or synthethic generation experiment, based on a config file.

NOTE on TSV/CSV file specs:
* If '.csv', the file is assumed to use the standard separator for columns ','.
* The index (gene or sample identifiers) is assumed to be the first column.
* The scBoolSeq is designed with consistency in mind. 
  The `output` (binarized or synthetic expression frame) will have the same disposition 
  (genes x observations | observations x genes) as the `input`. 
  If a `reference` is specified, its disposition must match the `input`'s.

scBoolSeq: bulk and single-cell RNA-Seq data binarization and synthetic generation from Boolean dynamics.

positional arguments:
  command     Subcommand to run

optional arguments:
  -h, --help  show this help message and exit
```

#### Binarization

Minimal example of binarization, specifying some optional parameters.

```bash
curl -fOL https://github.com/pinellolab/STREAM/raw/master/stream/tests/datasets/Nestorowa_2016/data_Nestorowa.tsv.gz

ls
# data_Nestorowa.tsv.gz
time scBoolSeq binarize data_Nestorowa.tsv.gz --genes-are-rows\
--output Nestorowa_binarized.csv --n-threads 10 --dump-config --dump-criteria
# ________________________________________________________
# Executed in   34.49 secs   fish           external 
#   usr time   30.04 secs  1211.00 micros   30.04 secs 
#   sys time    3.90 secs  171.00 micros    3.89 secs 

ls
# data_Nestorowa.tsv.gz    scBoolSeq_criteria_data_Nestorowa_2022-04-27_15h14m27.tsv
# Nestorowa_binarized.csv  scBoolSeq_experiment_config_2022-04-27_15h14m27.toml

# Visualize the binarized expression frame. 
# Note that some entries are undefined (NaN)
# These might be discarded genes for which no binarization or synthesis can occur,
# or observations which did not pass the thresholds to be set to 0 or 1.
python -c 'import pandas as pd; pd.read_csv("Nestorowa_binarized.csv", index_col=0).iloc[0:7, 0:7]'
#             Clec1b  Kdm3a  Coro2b  8430408G22Rik  Clec9a  Phf6  Usp14
# HSPC_025       NaN    1.0     NaN            NaN     NaN   0.0    0.0
# HSPC_031       NaN    1.0     NaN            NaN     NaN   0.0    0.0
# HSPC_037       NaN    0.0     1.0            NaN     NaN   0.0    1.0
# LT-HSC_001     NaN    0.0     1.0            NaN     NaN   1.0    0.0
# HSPC_001       NaN    0.0     1.0            NaN     NaN   1.0    0.0
# HSPC_008       1.0    1.0     NaN            NaN     NaN   1.0    0.0
# HSPC_014       NaN    0.0     NaN            NaN     NaN   0.0    1.0
```

#### Synthetic generation from Boolean states

```bash
cat minimal_boolean_example.csv 
# the output is not commented out so that it can be copied
# and perhaps be read with `x = pandas.read_clipboard(sep=',', index_col=0)`
,HSPC_025,HSPC_031,HSPC_037,LT-HSC_001,HSPC_001,HSPC_008,HSPC_014,HSPC_020,HSPC_026,HSPC_038,LT-HSC_002,HSPC_002,HSPC_009,HSPC_015,HSPC_021
Kdm3a,1.0,1.0,0.0,0.0,0.0,1.0,0.0,0.0,0.0,0.0,0.0,0.0,1.0,0.0,1.0
Coro2b,1.0,1.0,1.0,1.0,1.0,0.0,1.0,0.0,1.0,0.0,0.0,0.0,0.0,1.0,1.0
8430408G22Rik,1.0,0.0,0.0,1.0,0.0,1.0,1.0,1.0,1.0,0.0,0.0,0.0,1.0,0.0,1.0
Clec9a,1.0,0.0,0.0,1.0,1.0,0.0,1.0,0.0,1.0,1.0,0.0,0.0,0.0,0.0,0.0
Phf6,0.0,0.0,0.0,1.0,1.0,1.0,0.0,1.0,1.0,1.0,0.0,1.0,0.0,1.0,0.0


# Generate 20 samples per boolean state, using 12 threads
# setting the random number generator's seed ensures reproductiblility.
time scBoolSeq synthesize --genes-are-rows minimal_boolean_example_T.csv --reference data_Nestorowa.tsv.gz\
--n-samples 20 --output new_synthetic.tsv --n-threads 12 --rng-seed 1234
# ________________________________________________________
# Executed in   43.85 secs   fish           external 
#    usr time   22.08 secs    0.00 millis   22.08 secs 
#    sys time    3.65 secs    3.31 millis    3.65 secs 

# visualize the newly generated synthetic scRNA-Seq experiment
python -c 'import pandas as pd; pd.read_csv("new_synthetic.tsv", index_col=0, sep="\t").iloc[0:3, 0:7]'
#                HSPC_025  HSPC_031  HSPC_037  LT-HSC_001  HSPC_001  HSPC_008  HSPC_014
# Kdm3a          7.328819  8.536391  0.000000    0.000000  0.821561  7.030519  1.891949
# Coro2b         0.000000  0.000000  6.457878    5.479887  0.000000  0.000000  5.503554
# 8430408G22Rik  0.000000  0.005110  0.000000    0.000000  0.000000  6.428994  0.000000
```

### Python API

Here a minimal example is presented, using the same dataset as the CLI usage guide.
For further information, please check the documentation.

```python
import pandas as pd
from scboolseq import scBoolSeq

# read in the normalized expression data
nestorowa = pd.read_csv("data_Nestorowa.tsv.gz", index_col=0, sep="\t")
nestorowa.iloc[1:5, 1:5] 
#                HSPC_031  HSPC_037  LT-HSC_001  HSPC_001
# Kdm3a          6.877725  0.000000    0.000000  0.000000
# Coro2b         0.000000  6.913384    8.178374  9.475577
# 8430408G22Rik  0.000000  0.000000    0.000000  0.000000
# Clec9a         0.000000  0.000000    0.000000  0.000000
#
# NOTE : here, genes are rows and observations are columns

# scBoolSeq expects genes to be columns, thus we transpose the DataFrame.
scbool_nest = scBoolSeq(data=nestorowa.T, r_seed=1234)

##
## Binarization
##

scbool_nest.fit() # compute binarization criteria

binarized = scbool_nestorowa.binarize(nestorowa.T)
binarized.iloc[1:5, 1:5] 
#             Kdm3a  Coro2b  8430408G22Rik  Phf6
# HSPC_031      1.0     NaN            NaN   0.0
# HSPC_037      0.0     1.0            NaN   0.0
# LT-HSC_001    0.0     1.0            NaN   1.0
# HSPC_001      0.0     1.0            NaN   1.0


##
## Synthetic RNA-Seq generation from Boolean states
##

scbool_nestorowa.simulation_fit() # compute simulation criteria

# we generate Boolean states by randomly (equiprobably) binarize undetermined
# values from the previous binarization.
from scboolseq.simulation import random_nan_binariser
fully_bin = binarized.iloc[1:5, 1:5].pipe(random_nan_binariser) 
fully_bin 
#             Kdm3a  Coro2b  8430408G22Rik  Phf6
# HSPC_031      1.0     0.0            1.0   0.0
# HSPC_037      0.0     1.0            1.0   0.0
# LT-HSC_001    0.0     1.0            0.0   1.0
# HSPC_001      0.0     1.0            1.0   1.0

# create a synthetic frame, with two samples per boolean state,
# fixing the rng's seed for reproducibility
scbool_nestorowa.simulate(fully_bin, n_threads=4, seed=1234, n_samples=2) 
#               Kdm3a    Coro2b  8430408G22Rik      Phf6
# HSPC_031    7.328819  0.000000       8.087928  0.923352
# HSPC_037    1.003712  6.843611       7.003577  0.000000
# LT-HSC_001  0.000000  0.000000       0.000000  5.174053
# HSPC_001    1.672793  0.000000       0.000000  4.481709
# HSPC_031    8.536391  1.060373       0.000000  3.267464
# HSPC_037    1.055816  5.479887       0.000000  3.836276
# LT-HSC_001  0.000000  0.000000       0.000000  8.131221
# HSPC_001    2.451340  0.000000       0.000000  9.969012

```

## Contributors

* [Gustavo Magaña López](https://github.com/gmagannaDevelop)
