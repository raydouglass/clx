# <div align="left"><img src="https://rapids.ai/assets/images/rapids_logo.png" width="90px"/>&nbsp;&nbsp;Cyber Log Accelerators (CLX)</div>

**NOTE:** For the latest stable [README.md](https://github.com/rapidsai/clx/blob/master/README.md) ensure you are on the `master` branch.

CLX ("clicks") provides a collection of [RAPIDS](https://rapids.ai/) examples for security analysts, data scientists, and engineers to quickly get started applying RAPIDS and GPU acceleration to real-world cybersecurity use cases.

The goal of CLX is to:

1. Allow cyber data scientists and SecOps teams to generate workflows, using cyber-specific GPU-accelerated primitives and methods, that let them interact with code using security language,
1. Make available pre-built use cases that demonstrate CLX and RAPIDS functionality that are ready to use in a Security Operations Center (SOC),
1. Accelerate log parsing in a flexible, non-regex method. and
1. Provide SIEM integration with GPU compute environments via RAPIDS and effectively extend the SIEM environment.

## Getting Started with Python and Notebooks
CLX is targeted towards cybersecurity data scientists, senior security analysts, threat hunters, and forensic investigators. Data scientists can use CLX in traditional Python files and Jupyter notebooks. The notebooks folder contains example use cases and workflow instantiations. It's also easy to get started using CLX with RAPIDS with Python. The code below reads cyber alerts, aggregates them by day, and calculates the rolling z-score value across multiple days to look for outliers in volumes of alerts. Expanded code is available in the alert analysis notebook.

```python
import cudf
import s3fs
from os import path

# download data
if not path.exists("./splunk_faker_raw4"):
    fs = s3fs.S3FileSystem(anon=True)
    fs.get("rapidsai-data/cyber/clx/splunk_faker_raw4", "./splunk_faker_raw4")

# read in alert data
gdf = cudf.read_csv('./splunk_faker_raw4')
gdf.columns = ['raw']

# parse the alert data using CLX built-in parsers
from clx.parsers.splunk_notable_parser import SplunkNotableParser

snp = SplunkNotableParser()
parsed_gdf = cudf.DataFrame()
parsed_gdf = snp.parse(gdf, 'raw')

# define function to round time to the day
def round2day(epoch_time):
    return int(epoch_time/86400)*86400

# aggregate alerts by day
parsed_gdf['time'] = parsed_gdf['time'].astype(int)
parsed_gdf['day'] = parsed_gdf.time.applymap(round2day)
day_rule_gdf= parsed_gdf[['search_name','day','time']].groupby(['search_name', 'day']).count().reset_index()
day_rule_gdf.columns = ['rule', 'day', 'count']

# import the rolling z-score function from CLX statistics
from clx.analytics.stats import rzscore

# pivot the alert data so each rule is a column
def pivot_table(gdf, index_col, piv_col, v_col):
    index_list = gdf[index_col].unique()
    piv_gdf = cudf.DataFrame()
    piv_gdf[index_col] = index_list
    for group in gdf[piv_col].unique():
        temp_df = gdf[gdf[piv_col] == group]
        temp_df = temp_df[[index_col, v_col]]
        temp_df.columns = [index_col, group]
        piv_gdf = piv_gdf.merge(temp_df, on=[index_col], how='left')
    piv_gdf = piv_gdf.set_index(index_col)
    return piv_gdf.sort_index()

alerts_per_day_piv = pivot_table(day_rule_gdf, 'day', 'rule', 'count').fillna(0)

# create a new cuDF with the rolling z-score values calculated
r_zscores = cudf.DataFrame()
for rule in alerts_per_day_piv.columns:
    x = alerts_per_day_piv[rule]
    r_zscores[rule] = rzscore(x, 7) #7 day window

```

## Getting Started With Workflows

In addition to traditional Python files and Jupyter notebooks, CLX also includes structure in the form of a workflow. A workflow is a series of data transformations performed on a [GPU dataframe](https://github.com/rapidsai/cudf) that contains raw cyber data, with the goal of surfacing meaningful cyber analytical output. Multiple I/O methods are available, including Kafka and on-disk file stores.

Example flow workflow reading and writing to file:

```python
from clx.workflow import netflow_workflow

source = {
   "type": "fs",
   "input_format": "csv",
   "input_path": "/path/to/input",
   "schema": ["firstname","lastname","gender"],
   "delimiter": ",",
   "required_cols": ["firstname","lastname","gender"],
   "dtype": ["str","str","str"],
   "header": "0"
}
dest = {
   "type": "fs",
   "output_format": "csv",
   "output_path": "/path/to/output"
}
wf = netflow_workflow.NetflowWorkflow(source=source, destination=dest, name="my-netflow-workflow")
wf.run_workflow()
```

For additional examples, browse our complete [API documentation](https://rapidsai.github.io/clx/), or check out our more detailed [notebooks](https://github.com/rapidsai/clx/tree/master/notebooks).



## Getting CLX
### Intro
There are 3 ways to get CLX :
1. [Quick Start with CLX Docker Container](#quick)
1. [Conda Installation](#conda)
1. [Build from Source](#source)

<a name="quick"></a>

## Quick Start Docker Container

Prerequisites

* NVIDIA Pascal™ GPU architecture or better
* CUDA 10.0+ compatible NVIDIA driver
* Ubuntu 16.04/18.04 or CentOS 7
* Docker CE v18+
* nvidia-docker v2+

Pull the RAPIDS image suitable to your environment and build CLX image.

```aidl
docker pull rapidsai/rapidsai-dev-nightly:0.14-cuda10.1-devel-ubuntu18.04-py3.7
docker build --build-arg image=rapidsai/rapidsai-dev-nightly:0.14-cuda10.1-devel-ubuntu18.04-py3.7 -t clx:latest .
```

### Docker Container without SIEM Integration

Start the container and the notebook server. There are multiple ways to do this, depending on what version of Docker you have.

#### Preferred - Docker CE v19+ and nvidia-container-toolkit
```aidl
docker run  --gpus '"device=0"' \
  --rm -d \
  -p 8888:8888 \
  -p 8787:8787 \
  -p 8686:8686 \
  clx:latest
```

#### Legacy - Docker CE v18 and nvidia-docker2
```aidl
docker run --runtime=nvidia \
  --rm -d \
  -p 8888:8888 \
  -p 8787:8787 \
  -p 8686:8686 \
  clx:latest
```

### Docker Container with SIEM Integration

If you want a CLX container with SIEM integration (including data ingest), follow the steps above to build the CLX image. Then use `docker-compose` to start multiple containers running CLX, Kafka, and Zookeeper. 

```aidl
docker-compose up
```



<a name="conda"></a>

## Conda Install 
It is easy to install CLX using conda. You can get a minimal conda installation with Miniconda or get the full installation with Anaconda.

Install and update CLX using the conda command:

```
conda install -c rapidsai-nightly -c nvidia -c pytorch -c conda-forge -c defaults clx
```



<a name="source"></a>

## Building from Source and Contributing

For contributing guildelines please reference our [guide for contributing](https://github.com/rapidsai/clx/blob/master/CONTRIBUTING.md).
