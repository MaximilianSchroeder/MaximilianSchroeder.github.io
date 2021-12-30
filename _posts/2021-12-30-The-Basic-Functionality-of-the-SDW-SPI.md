---
title: "The Basic Functionality of the SDW-API"
date: 2021-12-30
tags:
excerpt:
categories: [Tutorials]
---

# Introduction
While I was working on a project, I required euro area (EA) data. One obvious go to address for such tasks is the ECB's Statistical Data Warehouse (SDW). Unfortunately, there was - to the best of my knowledge - no efficient and easy-to-use way to import data from SDW directly into `python`. Downloading individual excel spreadsheets and merging data manually, however, is not only time consuming, but also error prone. For this reason, I created a basic SDW-API that automates such tasks. The SDW-API can be found either on [pypi](https://pypi.org/project/sdw-api/0.1.2/) or in my corresponding [GitHub repository](https://github.com/MaximilianSchroeder/SDW_API). Installation using `pip install` is described below. This "tutorial" essentially replicates the ReadMe that is provided with the package and explains its basic functionality.

## 1.0 Basic Functionality

The package provides a basic API for the ECB's Statistical Data Warehouse (SDW).
In its current version, a few features are still missing. Nonetheless, the package already allows for downloading data seamlessly. An option allows saving the downloaded data directly into a `.xlsx` spreadsheet.

### 1.1 Installation
The package is available via pip. To install the SDW_API simply run `pip install sdw-api` in your command prompt. Please note that in its current version `python` >=3.9 is required. If you do not want to upgrade your python, you can of course launch a virtual environment instead and install the package there.

### 1.2 Downloading Data
The package consists of one main class called `SDW_API`, which handles the data download and basic data treatment automatically. Once the package is downloaded, it can be imported using the following statement:

```python
from sdw_api import SDW_API
```

The SDW_API class takes the following input arguments:

```python
SDW_API(ticker_list, start=None, end=None, outpath=None, filename=None, target_freq=None,method=None)
```
They can be separated into two groups:

* Required/Positional arguments:
  * `ticker_list`: A python list containing the data series tickers, series keys, or labels. These are equivalent to the ones used on the SDW website.
* Keyword Arguments:
  * `start`: This argument can be used, if a start date is to be set. The start date has to be in `YYYY-MM-DD` format. If this argument is specified, only data with a time stamp that is more recent than the start date is retrieved. If the argument is `None`, the entire available history will be downloaded.
  * `end`: This argument can be used, if an end date is to be set. The end date has to be in `YYYY-MM-DD` format. If this argument is specified, only data with a time stamp that is older than the end date is retrieved.
  * `outpath`: If the resulting date is to be saved as `.xlsx` an output path can be specified. This argument sets the directory where the data is to be saved.
  * `filename`: This argument allows to specify a unique filename for the output file. If neither `outpath` nor `filename` are set, the output file is not saved.
  * `target_freq`: This setting allows for defining a desired output frequency of the final DataFrame or spreadsheet. The class automatically detects the data frequency of the individual data series in `ticker_list`. If `ticker_list` contains time series at monthly as well as quarterly frequency, the highest frequency is assumed as a default. In this example, the output DataFrame will thus be at monthly frequency. In this case, setting  `target_freq` to "Q" overwrites the default. The final DataFrame is then at quarterly frequency.
  * `method`: If  `target_freq` is set to "Q", but the `ticker_list` also contains time series at monthly frequency, this option allows for setting an aggregation method for the time series at higher frequency. At the moment, only the average is implemented, which is also the default option.

## 2.0 Example

Let's assume we want to download Euro area (EA) HICP excluding food and energy ('ICP.M.U2.Y.XEF000.3.INX'), EA GDP ('MNA.Q.Y.I8.W2.S1.S1.B.B1GQ._Z._Z._Z.EUR.LR.N'), the historical close of the EONIA at monthly frequency ('FM.M.U2.EUR.4F.MM.EONIA.HSTA'), and EA total employment in hours worked ('ENA.Q.Y.I8.W2.S1.S1._Z.EMP._Z._T._Z.HW._Z.N') from January 2000 (i.e. '2000-01-01') until now. The string given in parentheses is the series' `Series Key`. It is the unique address that SDW assigns each series and can be found on SDW in the series' `Series Level Information`.

Assuming the package is imported, let's first download the data at monthly frequency:

```python
# set the tickers to be downloaded
ticker_list = ['ICP.M.U2.Y.XEF000.3.INX',
               'MNA.Q.Y.I8.W2.S1.S1.B.B1GQ._Z._Z._Z.EUR.LR.N',
               'FM.M.U2.EUR.4F.MM.EONIA.HSTA',
               'ENA.Q.Y.I8.W2.S1.S1._Z.EMP._Z._T._Z.HW._Z.N']

# set a start date
start = '2000-01-01'            

# initialize the API
example = SDW_API(ticker_list, start=start)

# download the data and compose DataFrame
example()

# access the output data
example.data
```

The head of the resulting DataFrame looks something like this:

|                     |   ICP.M.U2.Y.XEF000.3.INX |   MNA.Q.Y.I8.W2.S1.S1.B.B1GQ._Z._Z._Z.EUR.LR.N |   FM.M.U2.EUR.4F.MM.EONIA.HSTA |   ENA.Q.Y.I8.W2.S1.S1._Z.EMP._Z._T._Z.HW._Z.N |
|:--------------------|--------------------------:|-----------------------------------------------:|-------------------------------:|----------------------------------------------:|
| 2000-01-31 00:00:00 |                   79.8723 |                                  nan           |                        3.04286 |                                 nan           |
| 2000-02-29 00:00:00 |                   79.8981 |                                  nan           |                        3.27571 |                                 nan           |
| 2000-03-31 00:00:00 |                   79.9401 |                                    2.22607e+06 |                        3.51043 |                                   5.83027e+07 |
| 2000-04-30 00:00:00 |                   79.9912 |                                  nan           |                        3.685   |                                 nan           |
| 2000-05-31 00:00:00 |                   80.0094 |                                  nan           |                        3.92    |                                 nan           |

To generate output data at quarterly frequency instead, the following commands can be used:

```python
# set the tickers to be downloaded
ticker_list = ['ICP.M.U2.Y.XEF000.3.INX',
               'MNA.Q.Y.I8.W2.S1.S1.B.B1GQ._Z._Z._Z.EUR.LR.N',
               'FM.M.U2.EUR.4F.MM.EONIA.HSTA',
               'ENA.Q.Y.I8.W2.S1.S1._Z.EMP._Z._T._Z.HW._Z.N']

# set a start date
start = '2000-01-01'            
target_freq = 'Q'

# initialize the API
example = SDW_API(ticker_list, start=start, target_freq=target_freq)

# download the data and compose DataFrame
example()

# access the output data
example.data
```

The monthly series have now been aggregated to quarterly frequency automatically:

|                     |   ICP.M.U2.Y.XEF000.3.INX |   MNA.Q.Y.I8.W2.S1.S1.B.B1GQ._Z._Z._Z.EUR.LR.N |   FM.M.U2.EUR.4F.MM.EONIA.HSTA |   ENA.Q.Y.I8.W2.S1.S1._Z.EMP._Z._T._Z.HW._Z.N |
|:--------------------|--------------------------:|-----------------------------------------------:|-------------------------------:|----------------------------------------------:|
| 2000-03-31 00:00:00 |                   79.9035 |                                    2.22607e+06 |                        3.27634 |                                   5.83027e+07 |
| 2000-06-30 00:00:00 |                   80.0513 |                                    2.24645e+06 |                        3.96652 |                                   5.85199e+07 |
| 2000-09-30 00:00:00 |                   80.2987 |                                    2.2588e+06  |                        4.43939 |                                   5.87186e+07 |
| 2000-12-31 00:00:00 |                   80.5968 |                                    2.27377e+06 |                        4.80622 |                                   5.89292e+07 |
| 2001-03-31 00:00:00 |                   80.7685 |                                    2.29697e+06 |                        4.84326 |                                   5.90698e+07 |
