# Malet: a tool for machine learning experiment

🔨 **Malet** (**Ma**chine **L**earning **E**xperiment **T**ool) is a tool for efficient machine learning experiment execution, logging, analysis, and plot making.

The following features are provided:

- Simple YAML-based hyperparameter configuration w/ grid search syntax
- Experiment logging and resuming system
- User-friendly command-line tool for flexible graphing and easy data extraction from experiment logs
- Efficient parallelization by splitting a sequence of experiments over GPU jobs

## Installation

You can install Malet using pip,

```bash
pip install malet
```

or from this repository.

```bash
pip install git+https://github.com/edong6768/Malet.git
```

## Dependencies

- absl-py 1.0.0
- numpy 1.22.0
- pandas 2.0.3
- matplotlib 3.7.0
- ml-collections 0.1.0
- seaborn 0.11.2

## Documentation

### Contents

**[Quick start](#quick-start)**

1. [Prerequisite](#1-prerequisite)
2. [Running experiments](#2-running-experiments)
3. [Plot making](#3-plot-making)

**[Advanced topics](#advanced-topics)**

1. [Advanced gridding in yaml](#advanced-gridding-in-yaml)
2. [Advanced plot making](#advanced-plot-making)
3. [Parallel friendly grid splitting](#parallel-friendly-grid-splitting)
4. [Saving logs in intermediate epochs](#saving-logs-in-intermediate-epochs)
5. [Merging multiple log files](#merging-multiple-log-files)

## Quick start

### 1. Prerequisite

#### Experiment Folder

Using Malet starts with making a folder with a single yaml config file.
Various files resulting from some experiment is saved in this single folder.
We advise to create a folder for each experiment under ```experiments``` folder.

```yaml
experiments/
└── {experiment folder}/
    ├── exp_config.yaml : experiment config yaml file            (User created)
    ├── log.tsv         : log file for saving experiment results (generated by malet.experiment)
    ├── (log_splits)    : folder for splitted logs               (generated by malet.experiment)
    └── figure          : folder for figures                     (generated by malet.plot)
```

#### Pre-existing training pipeline

Say you have some training pipeline that takes in a configuration (any object w/ dictionary-like interface).
We require you to return the result of the training so it gets logged.

```python
def train(config, ...):
    ...
    # training happens here
    ...
    metric_dict = {
        'train_accuracies': train_accuracies,
        'val_accuracies': val_accuracies,
        'train_losses': train_losses,
        'val_losses': val_losses,
    }
    return metric_dict
```

### 2. Running experiments

#### Experiment config yaml

You can configure as you would do in the yaml file.
But we provide useful special keyword `grid`, used as follows:

```yaml
# static configs
model: LeNet5
dataset: mnist

num_epochs: 100
batch_size: 128
optimizer: adam

# grided fields
grid:
    seed: [1, 2, 3]
    lr: [0.0001, 0.001, 0.01, 0.1]
    weight_decay: [0.0, 0.00005, 0.0001]
```

Specifying list of config values under `grid` lets you run all possible combination (*i.e.* grid) of your configurations, with field least frequently changing in the order of declaration in `grid`.

#### Running experiments

The following will run the `train_fn` on grid of configs based on `{exp_folder_path}` and `train_fn`.

```python
from functools import partial
from malet.experiment import Experiment

train_fn = partial(train, ...{other arguments besides config}..)
metric_fields =  ['train_accuracies', 'val_accuracies', 'train_losses', 'val_losses']
experiment = Experiment({exp_folder_path}, train_fn, metric_fields)
experiment.run()
```

Note that you need to partially apply your original function so that you pass in a function with only `config` as its argument.

#### Experiment logs

The experiment log will be automatically saved in the `{exp_folder_path}` as `log.tsv`, where the static configs and the experiment log are eached saved in yaml and tsv like structure respectively.
You can retrieve these data in python using `ExperimentLog` in `malet.experiment` as follows:

```python
from malet.experiment import ExperimentLog

log = ExperimentLog.from_tsv({tsv_file})

static_configs = log.static_configs
df = log.df
```

Experiment logs also enables resuming to the most recently run config when a job is suddenly killed.
Note that this only enable you to resume from the begining of the training.
For resuming from intermediate log checkpoints, check out [Saving logs in intermediate epochs](#saving-logs-in-intermediate-epochs).

### 3. Plot making

Running `malet.plot` lets you make plots based on `log.tsv` in the experiment folder.

```bash
malet-plot \
-exp_folder ../experiments/{exp_folder} \
-mode curve-epoch-train_accuracy
```

The key intuition for using this is to *leave only two field in the dataframe for the x-axis and the y-axis* by

1. **specifying a specific value** (*e.g.* model, dataset, optimizer, etc.),
2. **averaging over** (seed),
3. or **choose value with best metric** (other hyperparameters),

which will leave only one value for each field.
This can be done using the following arguments.

#### Data related arguments

1. **`-mode`**: Mode consists of mode of the plot (currently only has 'curve' and 'bar'), the field for x-axis, and the metric to use for y-axis.

    ```bash
    -mode {plot_mode}-{x_field}-{metric}
    ```

    Any other field except `x_field` and `seed` (always averaged over) is automatically chosen value with best metric.
    To specify a value of a field, you can use the following `-filter` argument.
2. **`-filter`**: Use to explicitly choose only certain subset values of some field.

    ```bash
    -filter '{field1} {v1} {v2} / {field2} {v3} {v4} ...'
    ```

    Here,

    - `epoch` - from `explode`ing list-type metric
    - `metric` - from `melt`ing different metrics column name into a new column

    are automatically generated.

3. **`-multi_line_field`**: Specify the fields to plot multiple lines over.

    ```bash
    -multi_line_field '{field1} {field2} ...'
    ```

4. **`-best_at_max`** (Default: False): Specify whether chosen metric is best when largest (e.g. accuracy).

    ```bash
    -best_at_max
    -nobest_at_max
    ```

#### Styling arguments

1. **`-colors`**: There are two color mode `''`, `'cont'`, where one uses rainbow like colors and the other uses more continuous colors.

    ```bash
    -colors 'cont'
    ```

2. **`-annotate`**: Option to add annotation based on field specified in `annotate_fields`.

    ```bash
    -annotate
    ```

3. **`-annotate_fields`**: Field to annotate.

    ```bash
    -annotate_fields '{field1} {field2} ...'
    ```

4. **`-plot_config`**: The path for a yaml file to configure all aspects the plot.

    ```bash
    -plot_config {plot_config_path}
    ```

    In this yaml, you can specify the `line_style` and `ax_style` under each mode as follows:

    ```yaml
    'curve-epoch-train_accuracy':
      annotate: false
      std_plot: fill
      line_style: 
        linewidth: 4
        marker: 'D'
        markersize: 10

      ax_style:
        frame_width: 2.5
        fig_size: 7
        legend: [{'fontsize': 20}]
        grid: [true, {'linestyle': '--'}]
        tick_params:
          - axis: both
            which: major
            labelsize: 25
            direction: in
            length: 5
    ```

    - `line_style`: Style of the plotted line (`linewidth`, `marker`, `markersize`, `markevery`)
    - `ax_style`: Style of the figure. [Most attribute of `matplotlib.axes.Axes` object](https://matplotlib.org/stable/api/_as_gen/matplotlib.axes.Axes.html#matplotlib.axes.Axes) can be set as follows:

      ```yaml
      yscale: [parg1, parg2, {'kwarg1': v1, 'kwarg2': v2}]
      ```

      is equivalent to running

      ```python
      ax.set_yscale(parg1, parg2, kwarg1=v1, kwarg2=v2)
      ```

For more details, go to [Advanced plot making](#advanced-plot-making) section.

## Advanced topics

### Advanced gridding in yaml

#### 1. List comprehension

This serves similar functionality as list comprehensions in python, and used as follows:

```yaml
lr: [10**{-i};1:1:5]
```

**Syntax:**

```yaml
[{expression};{start}:{step}:{end}]
```

where expression should be any python-interpretable using symbols `i, +, -, *, /, [], ()` and numbers.
This is equivalent to python expression

```python
[{expression} for i in range({start}, {end}, {step})]
```

#### 2. Grid sequences

We can execute sequence of grids by passing in a list of dictionary instead of a dictionary under `grid` keyword as follows

```yaml
grid:
    - optimizer: sgd
      lr: [0.001, 0.01]
      seed: [1, 2, 3]

    - optimizer: adam
      lr: [0.005]
      seed: [1, 2, 3]
```

#### 3. Grouping

Grouping lets you group two different fields so it gets treated as a single field in the grid.

```yaml
grid:
    group:
        optimizer: [[sgd], [adam]]
        lr: [[0.001, 0.01], [0.005]]
    seed: [1, 2, 3]
```

**Syntax:**

```yaml
grid:
    group:
        cfg1: [A1, B1]
        cfg2: [A2, B2]
    cfg3: [1, 2, 3]
```

is syntactically equivalent to

```yaml
grid:
    - cfg1: A1
      cfg2: A2
      cfg3: [1, 2, 3]

    - cfg1: B1
      cfg2: B2
      cfg3: [1, 2, 3]
```

Here the two config fields `cfg1` and `cfg2` has grouped values `(A1, A2)` and `(B1, B2)` that acts like a single config field and arn't gridded seperately (`A1-2, B1-2` are lists of values.)

You can also create several groupings with list of dictionary under `group` keyword as follows.

```yaml
grid:
    group:
        - cfg1: [A1, B1]
          cfg2: [A2, B2]
        - cfg3: [C1, D1]
          cfg4: [C2, D2]
    cfg5: [1, 2, 3]
```

### Advanced Plot making

#### 1. Other arguments for `malet.plot`

- `-best_ref_x_field`: On defualt, each point in `x_field` get its own optimal hyperparameter set, which is sometimes undesirable.
This argument lets you specify on which value of `x_field` to choose the best hyperparamter.

    ```bash
    -best_ref_x_field {x_field_value}
    ```

- `-best_ref_ml_field`: Likewise, we might want to use the same hyperparameter for all lines in `multi_line_field` with best hyperparameter chosen from a single value in `multi_line_field`.

    ```bash
    -best_ref_ml_field {ml_field_value}
    ```

- `-best_ref_metric_field`: To plot one metric with the hyperparameter set chosen based on another, pass the name of the metric of reference in `metric_field_value`.

    ```bash
    -best_ref_metric_field {metric_field_value}
    ```

#### 2. Advanced yaml plot config

#### More details on ax_style keyword

Unlike other fields, `frame_width, fig_size, tick_params, legend, grid` are not attributes of `Axes` but are enabled for convinience.
From these, `frame_width` and `fig_size` should be set as a number, while others can be similarly used like the rest of the attributes in `Axes`.

#### Default style

You can change the default plot style by adding the `default_style` keyword in the yaml file.

```yaml
'default_style':
  annotate: false
  std_plot: fill
  line_style: 
    linewidth: 4
    marker: 'D'
    markersize: 10

  ax_style:
    frame_width: 2.5
    fig_size: 7
    legend: [{'fontsize': 20}]
    grid: [true, {'linestyle': '--'}]
    tick_params:
      - axis: both
        which: major
        labelsize: 25
        direction: in
        length: 5
```

#### Mode aliases

You can specify a set of arguments for `malet.plot` in the yaml file and give it an alias you can pass in to `mode` argument.

```yaml
'sam_rho':
  mode: curve-rho-val-accuracy
  multi_line_field: optimizer
  filter: 'optimizer sgd sam'
  annotate: True
  colors: ''
  
  std_plot: bar

  ax_style:
    title: ['SGD vs SAM', {'size': 27}]
    xlabel: ['$\rho$', {'size': 30}]
    ylabel: ['Val Accuracy (%)', {'size': 30}]
```

```bash
malet-plot \
-exp_folder ../experiments/{exp_folder} \
-plot_config {plot_config_path} \
-mode sam_rho
```

When using mode aliases, the conflicting argument passed within the shell will be ignored.

#### Style hierarchy

If conflicting style is passed in, we use the specifications given in the highest priority, given as the following:

```python
default_style < {custom style} < {mode alias}
```

#### 3. Custom dataframe processing

The `legend` and the `tick` are automatically determined based on the processed dataframe within `draw_metric` function.
You can pass in a function to the `preprcs_df` keyword argument in `draw_metric` with the following arguments and return values:

```python
def preprcs_df(df, legend):
    ...
    # Process df and legend
    ...
    return processed_df, processed_legend
```

We advise to assign a new mode for each `preprcs_df`.

#### 4. Custom plotting using `avgbest_df` and `ax_draw` in `plot_utils.metric_drawer`

Much of what `malet.plot` does comes from `avgbest_df` and `ax_draw`.

#### avgbest_df(df, metric_field, avg_over=None,  best_over=tuple(),  best_of=dict(), best_at_max=True)

- Paramters:
  - **df** (`pandas.DataFrame`) : Base dataframe to operate over. All hyperparameters should be set as `MultiIndex`.
  - **metric_field** (`str`) : Column name of the metric. Used to evaluate best hyperparameter.
  - **avg_over** (`str`) : `MultiIndex` level name to average over.
  - **best_over** (`List[str]`) : List of `MultiIndex` level names to find value yielding best values of `metric_field`.
  - **best_of** (`Dict[str, Any]`) : Dictionary of pair `{MultiIndex name}: {value in MultiIndex}` to find best hyperparameter of. The other values in `{MultiIndex name}` will follow the best hyperparamter found for these values.
  - **best_at_max** (`bool`) : `True` when larger metric is better, and `False` otherwise.
- Returns: Processed DataFrame (`pandas.DataFrame`)

#### ax_draw(ax, df, label, annotate=True, std_plot='fill', unif_xticks=False, plot_config = {'linewidth':4, 'color':'orange', 'marker':'D', 'markersize':10, 'markevery':1})

- Paramters:
  - **ax** (`matplotlib.axes.Axes`) : Axes to plot in.
  - **df** (`pandas.DataFrame`) : Dataframe used for the plot.
    This dataframe should have one named index for the x-axis and one column for the y-axis.
  - **label** (`str`) : label for drawn line to be used in the legend.
  - **std_plot** (`Literal['none','fill','bar']`) : Style of standard error drawn in to the plot.
  - **unif_xticks** (`bool`) : When `True`, the xticks will be uniformly distanced regardless of its value.
  - **plot_config** (`Dict[str, Any]`) : Dictionary of configs to use when plotting the line (e.g. linewidth, color, marker, markersize, markevery).
- Returns: Axes (`matplotlib.axes.Axes`) with single line added based on `df`.

### Parallel friendly grid splitting

When using GPU resource allocation programs such as Slurm, you might want to split multiple hyperparameter configurations over different GPU jobs in parallel.
We provide two methods of spliting the grid as arguments of `Experiment`.
We advise to use flags to pass these as argument of your `train.py` file.

```python
from absl import app, flags
from malet.experiment import Experiment

...

FLAGS = flags.FLAGS
def main(argv):
  ...
  experiment = Experiment({exp_folder_path}, train_fn, metric_fields,
                          total_splits=FLAGS.total_splits,
                          curr_splits=FLAGS.curr_splits,
                          auto_update_tsv=FLAGS.auto_update_tsv,
                          configs_save=FLAGS.configs_save)
  experiment.run()

if __name__=='__main__':
  flags.DEFINE_string('total_splits', '1')
  flags.DEFINE_string('curr_splits', '0')
  flags.DEFINE_bool('auto_update_tsv', False)
  flags.DEFINE_bool('configs_save', False)
  app.run(main)
```

#### 1. Partitioning

1. Uniform Partitioning (Pass in number)

    This method of splits the experiments uniformally given the following arguments

    1. number of total partition (`total_splits`),
    2. batch index to allocate to this script (`curr_splits`).

    Each sbatch script needs to be using different `curr_splits` numbers (=0~total-1).

    ```bash
    splits = 4
    echo "run sbatch slurm_train $n"
    for ((i=0;i<n;i++))
    do
      python train.py ./experiments/{exp_folder} \
        --workdir=./logdir \
        --total_splits=splits \
        --curr_splits=$i
    done
    ```

2. Field Partitioning (Pass in field name)

    This method of splits the experiments based on some field given in the following arguments

    1. name of the field to split over (`total_splits`),
    2. string of field values seperated by ' ' to allocate to this current split script (`curr_splits`).

    Each sbatch script needs different field values (whitespace seperated strings for multiple values) in `curr_splits`.

    ```bash
    python experiment_util.py ./experiments/{exp_folder} \
        --total_splits 'optimizer' \
        --curr_splits= 'sgd'

    python experiment_util.py ./experiments/{exp_folder} \
        --total_splits 'optimizer' \
        --curr_splits= 'rmsprop adam'
    ```

    Both of these split methods result in multiple `.tsv` files, which is saved in `{exp_folder}/log_splits/split_{i}.tsv` folder.

**Comments on `auto_update_tsv` argument.**

`auto_update_tsv` is used for 'Current run checking' stated in the next section, but using it in 'Partitioning' doesn't cause problems.
However we advise to not use it by adding since additional read/writing adds unnessacery computation time, especially as the `log.tsv` file grows larger.

#### 2. Queueing

With this method, each jobs, once finished running its config, runs the next config in the queue of the unrun configs.
More precisly, it skips any configs that finished running or are currently running.
The key to doing this is `configs_save=True`, which saves the configs to the `{exp_folder}/log.tsv` file before a config is run, enabling other jobs to know what config is currently running and skip it.

```bash
python experiment_util.py ./experiments/{exp_folder} --workdir=./logdir \
--auto_update_tsv \
--configs_save
```

This method requires the keyword `auto_update_tsv=True` in `Experiment` to automatically read/write tsv files after a job starts/finishes running a config.

One adventage of 'Queueing' over 'Partitioning' is that you can freely allocate/deallocate new GPUs while running an experiment.

#### 3. Use Both (Partitioning + Queueing)

However as `log.tsv` grows larger, read/write time gets larger which cause various conflicts across different GPU jobs. One workaround is to use 'Partitioning' to split experiments to be saved in seperate `log_splits/split_{i}.tsv` to keep the `.tsv` files small, while using 'Queueing' in each splits to freely allocate GPU jobs to leverage the advantages of both methods.

```bash
splits = 4
echo "run sbatch slurm_train $n"
for ((i=0;i<n;i++))
do
  python experiment_util.py ./experiments/{exp_folder} \
    --workdir=./logdir \
    --total_splits=splits \
    --curr_splits=$i \
    --auto_update_tsv \
    --configs_save
done
```

### Saving logs in intermediate epochs

We checkpoint training state so that we can resume training in the event of an unexpected termination.
We can also checkpoint the experiment log so that we don't have to retrain a certain config to re-evaluate the metrics.

#### Training pipeline

For this, we need to add `exp_log` argument in `train` function for checkpointing the experiment log, where you can use it to add the following code for retrieveing/saving intermediate metric dictionary from/to the `tsv` file.

```python
import os

def get_ckpt_dir(config):
    ...
    return ckpt_dir

def get_ckpt(ckpt_dir):
    ...
    return ckpt

def train(config, experiment, ...):

    ... # set up
    
    # retrieve model/trainstate checkpoint if there exists
    # these are just placeholders for the logic
    ckpt_epoch = 0
    ckpt_dir = get_ckpt_dir(config)
    if os.path.exists(ckpt_dir)
      ckpt = get_ckpt(ckpt_dir)
      ckpt_epoch = ckpt.epoch
    
    ############# retrieve log checkpoint if there exists #############
    metric_dict = {
        'train_accuracies': [],
        'val_accuracies': [],
        'train_losses': [],
        'val_losses': [],
    }
    if config in experiment.log:
      metric_dict = experiment.get_log_checkpoint(config, metric_dict)
    ###################################################################
    ...
    # training happens here
    for epoch in range(config.ckpt_epoch, config.epochs):
      
      ... # train
      
      ... # update metric_dict

      if not epoch % config.ckpt_every:

        ... # train state, model checkpoint

        ####################### checkpoint log #######################
        experiment.update_log(metric_dict, configs=config) 
        ##############################################################
    ...

    return metric_dict
```

The `ExperimentLog.get_log_checkpoint` method retrieves the `metric_dict` based on the `status` field in the dataframe.
|status|Description|Behavior when resumed|
|:----:|-----------|--------|
| `R`  | Currently running | Get skipped |
| `C`  | Completed | Get skipped |
| `F`  | Failed while running | Rerun and `metric_dict` is retrieved |

Note that with some external halt (e.g. computer shut down, slurm job cancellation), malet won't be able to log the status as `F` (failed). 
In these cases, you need to **manually find the row in the `log.tsv` file corresponding to the halted job and change the `status` from `R` (running) to `F` (falied)**.

#### Running experiment

```python
from functools import partial
from malet.experiment import Experiment

train_fn = partial(train, ...{other arguments besides config & exp_log}..)
metric_fields =  ['train_accuracies', 'val_accuracies', 'train_losses', 'val_losses']
experiment = Experiment({exp_folder_path}, train_fn, metric_fields, 
                        checkpoint=True, auto_update_tsv=True) 
experiment.run()
```

You should add `checkpoint=True, auto_update_tsv=True` when instanciating `Experiment`.

### Merging multiple log files

There are two methods for merging multiple log files.

#### 1. Merge all logs in a folder

```python
from malet.experiment import ExperimentLog

ExperimentLog.merge_folder({log_folder_path})
```

#### 2. Merge a portion of the logs in a folder

```python
from malet.experiment import ExperimentLog

names = ["log1", "log2", ..., "logn"]
ExperimentLog.merge_tsv(names, {log_folder_path})
```

Both methods automatically merges and saves as `log_merged.tsv` in the folder.
These methods are helpful after running splitted experiments, where merging is required for using plot tools.