Codebase Structure
==================

The codebase is organized in a modular, datatype / feature centric way so that adding a feature for a new datatype is pretty straightforward and requires isolated code changes. All the datatype specific logic lives in the corresponding feature module all of which are under `ludwig/features/`.

Feature classes contain raw data preprocessing logic specific to each data type. All input (output) features implement `build_input` (`build_output`) method which is used to build encodings (decode outputs). Output features also contain datatype-specific logic to compute output measures such as loss, accuracy, etc.

Encoders and decoders are modularized as well (they are under `ludwig/models/modules`) so that they can be used by multiple features. For example sequence encoders are shared among text, sequence, and timeseries features.

Various model architecture components which can be reused are also split into dedicated modules, for example convolutional modules, fully connected modules, etc.

Bulk of the training logic resides in `ludwig/models/model.py` which initializes a tensorflow session, feeds the data, and executes training.

Adding an Encoder
=================

 1. Add a new encoder class
---------------------------

Source code for encoders lives under `ludwig/models/modules`.
New encoder objects should be defined in the corresponding files, for example all new sequence encoders should be added to `ludwig/models/modules/sequence_encoders.py`.

All the encoder parameters should be provided as arguments in the constructor with their default values set. For example `RNN` encoder takes the following list of arguments in its constructor:

```python
def __init__(
    self,
    should_embed=True,
    vocab=None,
    representation='dense',
    embedding_size=256,
    embeddings_trainable=True,
    pretrained_embeddings=None,
    embeddings_on_cpu=False,
    num_layers=1,
    state_size=256,
    cell_type='rnn',
    bidirectional=False,
    dropout=False,
    initializer=None,
    regularize=True,
    reduce_output='last',
    **kwargs
):
```

Typically all the dependencies are initialized in the encoder's constructor (in the case of the RNN encoder these are EmbedSequence and RecurrentStack modules) so that at the end of the constructor call all the layers are fully described.

Actual creation of tensorflow variables takes place inside the `__call__` method of the encoder. All encoders should have the following signature:

```python
__call__(
    self,
    input_placeholder,
    regularizer,
    dropout,
    is_training
)
```

__Inputs__

- __input_placeholder__ (tf.Tensor): input tensor.
- __regularizer__ (A (Tensor -> Tensor or None) function): regularizer function passed to `tf.get_variable` method.
- __dropout__ (tf.Tensor(dtype: tf.float32)): dropout rate.
- __is_training__ (tf.Tensor(dtype: tf.bool), default: `True`): boolean indicating whether this is a training dataset.


__Return__

- __hidden__ (tf.Tensor(dtype: tf.float32)): feature encodings.
- __hidden_size__ (int): feature encodings size.

Encoders are initialized as class member variables in input feature object constructors and called inside `build_input` methods.


 2. Add the new encoder class to the corresponding encoder registry
-------------------------------------------------------------------

Mapping between encoder keywords in the model definition and encoder classes is done by encoder registries: for example sequence encoder registry is defined in `ludwig/features/sequence_feature.py`

```
sequence_encoder_registry = {
    'stacked_cnn': StackedCNN,
    'parallel_cnn': ParallelCNN,
    'stacked_parallel_cnn': StackedParallelCNN,
    'rnn': RNN,
    'cnnrnn': CNNRNN,
    'embed': EmbedEncoder
}
```

Adding a Decoder
================

 1. Add a new decoder class
---------------------------

Souce code for decoders lives under `ludwig/models/modules`.
New decoder objects should be defined in the corresponding files, for example all new sequence decoders should be added to `ludwig/models/modules/sequence_decoders.py`.

All the decoder parameters should be provided as arguments in the constructor with their default values set. For example `Generator` decoder takes the following list of arguments in its constructor:

```python
__init__(
    self,
    cell_type='rnn',
    state_size=256,
    embedding_size=64,
    beam_width=1,
    num_layers=1,
    attention_mechanism=None,
    tied_embeddings=None,
    initializer=None,
    regularize=True,
    **kwargs
)
```

Decoders are initialized as class member variables in output feature object constructors and called inside `build_output` methods.

 2. Add the new decoder class to the corresponding decoder registry
-------------------------------------------------------------------

Mapping between decoder keywords in the model definition and decoder classes is done by decoder registries: for example sequence decoder registry is defined in `ludwig/features/sequence_feature.py`

```python
sequence_decoder_registry = {
    'generator': Generator,
    'tagger': Tagger
}
```

Adding a new Feature Type
=========================

 1. Add a new feature class
---------------------------

Souce code for feature classes lives under `ludwig/features`.
Input and output feature classes are defined in the same file, for example `CategoryInputFeature` and `CategoryOutputFeature` are defined in `ludwig/features/category_feature.py`.

An input features inherit from the `InputFeature` and corresponding base feature classes, for example `CategoryInputFeature` inherits from `CategoryBaseFeature` and `InputFeature`.

Similarly, output features inherit from the `OutputFeature` and corresponding base feature classes, for example `CategoryOutputFeature` inherits from `CategoryBaseFeature` and `OutputFeature`.

Feature parameters are provided in a dictionary of key-value pairs as an argument to the input or output feature constructor which contains default parameter values as well.

All input and output features should implement `build_input` and `build_output` methods correspondingly with the following signatures:


### build_input

```python
build_input(
    self,
    regularizer,
    dropout_rate,
    is_training=False,
    **kwargs
)
```

__Inputs__


- __regularizer__ (A (Tensor -> Tensor or None) function): regularizer function passed to `tf.get_variable` method.
- __dropout_rate__ (tf.Tensor(dtype: tf.float32)): dropout rate.
- __is_training__ (tf.Tensor(dtype: tf.bool), default: `True`): boolean indicating whether this is a training dataset.


__Return__

- __feature_representation__ (dict): the following dictionary

```python
{
    'type': self.type, # str
    'representation': feature_representation, # tf.Tensor(dtype: tf.float32)
    'size': feature_representation_size, # int
    'placeholder': placeholder # tf.Tensor(dtype: tf.float32)
}
```

### build_output

```python
build_output(
    self,
    hidden,
    hidden_size,
    regularizer=None,
    **kwargs
)
```

__Inputs__

- __hidden__ (tf.Tensor(dtype: tf.float32)): output feature representation.
- __hidden_size__ (int): output feature representation size.
- __regularizer__ (A (Tensor -> Tensor or None) function): regularizer function passed to `tf.get_variable` method.

__Return__
- __train_mean_loss__ (tf.Tensor(dtype: tf.float32)): mean loss for train dataset.
- __eval_loss__ (tf.Tensor(dtype: tf.float32)): mean loss for evaluation dataset.
- __output_tensors__ (dict): dictionary containing feature specific output tensors (predictions, probabilities, losses, etc).

 2. Add the new feature class to the corresponding feature registry
-------------------------------------------------------------------

Input and output feature registries are defined in `ludwig/features/feature_registries.py`.


Hyper-parameter optimization
============================

The hyper-parameter optimization design in Ludwig is based on two abstract interfaces: `HyperoptStrategy` and `HyperoptExecutor`. 

`HyperoptStrategy` represents the strategy adopted for sampling hyper-parameters values.
 Which strategy to use is defined in the `strategy` section of the model definition.
A `HyperoptStrategy` uses the `parameters` defined in the `hyperopt` section of the YAML model definition and a `goal` , either to minimize or maximize.
Each sub-class of `HyperoptStrategy` that implements its abstract methods samples parameters according to their definition and type differently (see [User Guide](user_guide.md#hyper-parameter-optimization) for details), like using a random search (implemented in `RandomStrategy`), or a grid serach (implemented in `GridStrategy`, or bayesian optimization or evolutionary techniques.
 
`HyperoptExecutor` represents the method used to execute the hyper-parameter optimization, independently of how the values for the hyperparameters are sampled.
Available implementations are a serial executor that executes the training with the different sampled hyper-parameters values one at a time (implemented in `SerialExecutor`), a parallel executor that runs the training using sampled hyper-parameters values in parallel on the same machine (implemented in the `ParallelExecutor`), and a [Fiber](https://uber.github.io/fiber/)-based executor that enables to run the training using sampled hyper-parameters values in parallel on multiple machines within a cluster. 
A `HyperoptExecutor` uses a `HyperoptStrategy` to sample hyper-parameters values, usually initializes an execution context, like a multithread pool fo instance, and executes the hyper-parameter optimization according to the strategy.
First, a new batch of parameters values is sampled from the `HyperoptStrategy`.
Then, sampled parameters values are merged with the basic model definition parameters specified, with the sampled parameters values overriding the ones in the basic model definition they refer to.
Training is executed using the merged model definition and training and validation losses and metrics are collected.
A `(sampled_parameters, statistics)` pair is provided to the `HyperoptStrategy.update` function and the loop is repeated until all the samples are sampled.
At the end, `HyperoptExecutor.execute` returns a list of dictionaries that include a parameter sample, its metric score, and its training and test statistics.
The returned list is printed and saved to disk, so that it can also be used as input to [hyper-parameter optimization visualizations](user_guide.md#hyper-parameter-optimization-visualization).


Adding a HyperoptStrategy
-------------------------

### 1. Add a new strategy class

The source code for the base `HyperoptStrategy` class is in the `ludwig/utils/hyperopt_utils.py` module.
Classes extending the base class should be defined in the same module.

#### `__init__`
```python
def __init__(self, goal: str, parameters: Dict[str, Any]):
```

The parameters of the base `HyperoptStrategy` class constructor are:
- `goal` which indicates if to minimize or maximize a metric or a loss of any of the output features on any of the splits which is defined in the `hyperopt` section
- `parameters` which contains all hyper-parameters to optimize with their types and ranges / values.

Example:
```python
goal = "minimize"
parameters = {
    "training.learning_rate": {
        "type": "float",
        "low": 0.001,
        "high": 0.1,
        "steps": 4,
        "scale": "linear"
    },
    "combiner.num_fc_layers": {
        "type": "int",
        "low": 2,
        "high": 6,
        "steps": 3
    }
}

strategy = GridStrategy(goal, parameters)
```

#### `sample`
```python
def sample(self) -> Dict[str, Any]:
```

`sample` is a method that yields a new sample according to the strategy.
It returns a set of parameters names and their values.
If `finished()` returns `True`, calling `sample` would return a `IndexError`.

Example returned value:
```
{'training.learning_rate': 0.005, 'combiner.num_fc_layers': 2, 'utterance.cell_type': 'gru'}
```

#### `sample_batch`
```python
def sample_batch(self, batch_size: int = 1) -> List[Dict[str, Any]]:
```

`sample_batch` method returns a list of sampled parameters of length equal to or less than `batch_size`.
If `finished()` returns `True`, calling `sample_batch` would return a `IndexError`. 

Example returned value:
```
[{'training.learning_rate': 0.005, 'combiner.num_fc_layers': 2, 'utterance.cell_type': 'gru'}, {'training.learning_rate': 0.015, 'combiner.num_fc_layers': 3, 'utterance.cell_type': 'lstm'}]
```

#### `update`
```python
def update(
    self,
    sampled_parameters: Dict[str, Any],
    metric_score: float
):
```

`update` updates the strategy with the results of previous computation.
- `sampled_parameters` is a dictionary of sampled parameters.
- `metric_score` is the value of the optimization metric obtained for the specified sample.

It is not needed for stateless strategies like grid and random, but is needed for stateful strategies like bayesian and evolutionary ones.

Example:
```python
sampled_parameters = {
    'training.learning_rate': 0.005,
    'combiner.num_fc_layers': 2, 
    'utterance.cell_type': 'gru'
} 
metric_score = 2.53463

strategy.update(sampled_parameters, statistics)
```

#### `update_batch`
```python
def update_batch(
    self,
    parameters_metric_tuples: Iterable[Tuple[Dict[str, Any], float]]
):
```

`update_batch` updates the strategy with the results of previous computation in batch.
- `parameters_metric_tuples` a list of pairs of sampled parameters and their respective metric value.

It is not needed for stateless strategies like grid and random, but is needed for stateful strategies like bayesian and evolutionary ones.

Example:
```python
sampled_parameters = [
    {
        'training.learning_rate': 0.005,
        'combiner.num_fc_layers': 2, 
        'utterance.cell_type': 'gru'
    },
    {
        'training.learning_rate': 0.015,
        'combiner.num_fc_layers': 5, 
        'utterance.cell_type': 'lstm'
    }
]
metric_scores = [2.53463, 1.63869]

strategy.update_batch(zip(sampled_parameters, metric_scores))
```

#### `finished`
```python
def finished(self) -> bool:
```

The `finished` method return `True` when all samples have been sampled, return `False` otherwise.


### 2. Add the new strategy class to the corresponding strategy registry

The `strategy_registry` contains a mapping between `strategy` names in the `hyperopt` section of model definition and `HyperoptStartegy` sub-classes.
To make a new strategy available, add it to the registry:
```
strategy_registry = {
    "random": RandomStrategy,
    "grid": GridStrategy,
    "new_strategy_name": NewStrategyClass
}
```


Adding a HyperoptExecutor
-------------------------

### 1. Add a new executor class

The source code for the base `HyperoptExecutor` class is in the `ludwig/utils/hyperopt_utils.py` module.
Classes extending the base class should be defined in the module.

#### `__init__`
```python
def __init__(
    self,
    hyperopt_strategy: HyperoptStrategy,
    output_feature: str,
    metric: str,
    split: str
)
```

The parameters of the base `HyperoptExecutor` class constructor are
- `hyperopt_strategy` is a `HyperoptStrategy` object that will be used to sample hyper-parameters values
- `output_feature` is a `str` containing the name of the output feature that we want to optimize the metric or loss of. Available values are `combined` (default) or the name of any output feature provided in the model definition. `combined` is a special output feature that allows to optimize for the aggregated loss and metrics of all output features.
- `metric` is the metric that we want to optimize for. The default one is `loss`, but depending on the tye of the feature defined in `output_feature`, different metrics and losses are available. Check the metrics section of the specific output feature type to figure out what metrics are available to use.
- `split` is the split of data that we want to compute our metric on. By default it is the `validation` split, but you have the flexibility to specify also `train` or `test` splits.

Example:
```python
goal = "minimize"
parameters = {
            "training.learning_rate": {
                "type": "float",
                "low": 0.001,
                "high": 0.1,
                "steps": 4,
                "scale": "linear"
            },
            "combiner.num_fc_layers": {
                "type": "int",
                "low": 2,
                "high": 6,
                "steps": 3
            }
        }
output_feature = "combined"
metric = "loss"
split = "validation"

grid_strategy = GridStrategy(goal, parameters)

executor = SerialExecutor(grid_strategy, output_feature, metric, split)
```

#### `execute`
```python
def execute(
    self,
    model_definition,
    data_df=None,
    data_train_df=None,
    data_validation_df=None,
    data_test_df=None,
    data_csv=None,
    data_train_csv=None,
    data_validation_csv=None,
    data_test_csv=None,
    data_hdf5=None,
    data_train_hdf5=None,
    data_validation_hdf5=None,
    data_test_hdf5=None,
    train_set_metadata_json=None,
    experiment_name="hyperopt",
    model_name="run",
    model_load_path=None,
    model_resume_path=None,
    skip_save_training_description=False,
    skip_save_training_statistics=False,
    skip_save_model=False,
    skip_save_progress=False,
    skip_save_log=False,
    skip_save_processed_input=False,
    skip_save_unprocessed_output=False,
    skip_save_test_predictions=False,
    skip_save_test_statistics=False,
    output_directory="results",
    gpus=None,
    gpu_fraction=1.0,
    use_horovod=False,
    random_seed=default_random_seed,
    debug=False,
    **kwargs
):
```

The `execute` method executes the hyper-parameter optimization.
It can leverage the `train_and_eval_on_split` function to obtain training and eval statistics and the `self.get_metric_score` function to extract the metric score from the eval results according to `self.output_feature`, `self.metric` and `self.split`.


### 2. Add the new executor class to the corresponding executor registry

The `executor_registry` contains a mapping between `executor` names in the `hyperopt` section of model definition and `HyperoptExecutor` sub-classes.
To make a new executor available, add it to the registry:
```
executor_registry = {
    "serial": SerialExecutor,
    "parallel": ParallelExecutor,
    "fiber": FiberExecutor,
    "new_executor_name": NewExecutorClass
}
```


Adding a new Integration
========================

Ludwig provides an open-ended method of third-party system
integration. This makes it easy to integrate other systems or services
with Ludwig without having users do anything other than adding a flag
to the command line interface.

To contribute an integration, follow these steps:

1. Create a Python file in `ludwig/contribs/` with an obvious name. In this example, it is called `mycontrib.py`.
2. Inside that file, create a class with the following structure, renaming `MyContribution` to a name that is associated with the third-party system:

```python
class MyContribution():
    @staticmethod
    def import_call(argv, *args, **kwargs):
	# This is called when your flag is used before any other
	# imports.

    def experiment(self, *args, **kwargs):
	# See: ludwig/experiment.py and ludwig/cli.py

    def experiment_save(self, *args, **kwargs):
	# See: ludwig/experiment.py

    def train_init(self, experiment_directory, experiment_name, model_name,
                   resume, output_directory):
	# See: ludwig/train.py

    def train(self, *args, **kwargs):
	# See: ludwig/train.py and ludwig/cli.py

    def train_model(self, *args, **kwargs):
	# See: ludwig/train.py

    def train_save(self, *args, **kwargs):
	# See: ludwig/train.py

    def train_epoch_end(self, progress_tracker):
	# See: ludwig/models/model.py

    def predict(self, *args, **kwargs):
	# See: ludwig/predict.py and ludwig/cli.py

    def predict_end(self, test_stats):
        # See: ludwig/predict.py

    def test(self, *args, **kwargs):
	# See: ludwig/test.py and ludwig/cli.py

    def visualize(self, *args, **kwargs):
	# See: ludwig/visualize.py and ludwig/cli.py

    def visualize_figure(self, fig):
	# See ludwig/utils/visualization_utils.py

    def serve(self, *args, **kwargs):
	# See ludwig/utils/serve.py and ludwig/cli.py

    def collect_weights(self, *args, **kwargs):
        # See ludwig/collect.py and ludwig/cli.py

    def collect_activations(self, *args, **kwargs):
        # See ludwig/collect.py and ludwig/cli.py
```

If your integration does not handle a particular action, you can simply remove the method, or do nothing (e.g., `pass`).

If you would like to add additional actions not already handled by the
above, add them to the appropriate calling location, add the
associated method to your class, and add them to this
documentation. See existing calls as a pattern to follow.

3. In the file `ludwig/contribs/__init__.py` add an import in this pattern, using your names:

```python
from .mycontrib import MyContribution
```

4. In the file `ludwig/contribs/__init__.py` in the `contrib_registry["classes"]` dictionary, add a key/value pair where the key is your flag, and the value is your class name, like:

```python
contrib_registry = {
    ...
    "classes": {
        ...,
        "myflag": MyContribution,
    }
}
```

5. Submit your contribution as a pull request to the Ludwig github repository.

Style Guidelines
================
We expect contributions to mimic existing patterns in the codebase and demonstrate good practices: the code should be concise, readable, PEP8-compliant, and conforming to 80 character line length limit.

Tests
=====

We are using ```pytest``` to run tests. 
Current test coverage is limited to several integration tests which ensure end-to-end functionality but we are planning to expand it.

Checklist
---------

Before running tests, make sure 
1. Your environment is properly setup.
2. You have write access on the machine. Some of the tests require saving data to disk.

Running tests
-------------

To run all tests, just run
```python -m pytest``` from the ludwig root directory.
Note that you don't need to have ludwig module installed and in this case
code change will take effect immediately.


To run a single test, run
``` 
python -m pytest path_to_filename -k "test_method_name"
```

Example
-------

```
python -m pytest tests/integration_tests/test_experiment.py -k "test_visual_question_answering"
```
