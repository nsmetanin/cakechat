# CakeChat: Emotional Generative Dialog System

CakeChat is a dialog system that is able to express emotions in a text conversation. [Try it online!](https://cakechat.replika.ai/)

![Demo](https://user-images.githubusercontent.com/764902/34832660-92570bfe-f6fe-11e7-9802-db2f8730a997.png)

It is written in [Theano](http://deeplearning.net/software/theano/) and [Lasagne](https://github.com/Lasagne/Lasagne). It uses end-to-end trained embeddings of 5 different emotions to generate responses conditioned by a given emotion.

The code is flexible and allows to condition a response by an arbitrary categorical variable defined for some samples in the training data. With CakeChat you can, for example, train your own persona-based neural conversational model[<sup>\[5\]</sup>](#f5) or create an emotional chatting machine without external memory[<sup>\[4\]</sup>](#f4).


## Table of contents

  * [Network architecture and features](#network-architecture-and-features)
  * [Quick start](#quick-start)
  * [Setup](#setup)
    * [Docker](#docker)
      * [CPU-only setup](#cpu-only-setup)
      * [GPU-enabled setup](#gpu-enabled-setup)
    * [Manual setup](#manual-setup)
  * [Getting the model](#getting-the-model)
    * [Using a pre-trained model](#using-a-pre-trained-model)
    * [Training your own model](#training-your-own-model)
    * [Existing training datasets](#existing-training-datasets)
  * [Running the system](#running-the-system)
    * [Local HTTP\-server](#local-http-server)
      * [HTTP\-server API description](#http-server-api-description)
    * [Gunicorn HTTP\-server](#gunicorn-http-server)
    * [Telegram bot](#telegram-bot)
  * [Repository overview](#repository-overview)
    * [Important tools](#important-tools)
    * [Important configuration settings](#important-configuration-settings)
  * [Example use cases](#example-use-cases)
  * [References](#references)
  * [Credits & Support](#credits--support)
  * [License](#license)


## Network architecture and features

![Network architecture](https://user-images.githubusercontent.com/4047271/34774960-71c7bfa0-f622-11e7-812b-cdb84472577a.png)


* Model:
    * Hierarchical Recurrent Encoder-Decoder (HRED) architecture for handling deep dialog context[<sup>\[7\]</sup>](#f7)
    * Multilayer RNN with GRU cells. First layer of the utterance-level encoder is always bidirectional.
    * Thought vector is fed into decoder on each decoding step.
    * Decoder can be conditioned on any string label. For example: emotion label or id of a person talking.
* Word embedding layer:
    * May be initialized using w2v model trained on your own corpus.
    * Embedding layer may either stay fixed of be fine-tuned along with all other weights of the network.
* Decoding
    * 4 different response generation algorithms: "sampling", "beamsearch", "sampling-reranking" and "beamsearch-reranking".
    Reranking of the generated candidates is performed according to the log-likelihood or MMI-criteria[<sup>\[3\]</sup>](#f3).
    See [configuration settings description](#important-configuration-settings) for details.
* Metrics:
    * Perplexity
    * n-gram distinct metrics adjusted to the samples size[<sup>\[3\]</sup>](#f3).
    * Lexical similarity between samples of the model and some fixed dataset.
    Lexical similarity is a cosine distance between TF-IDF vector of responses generated by the model and tokens
    in the dataset.
    * Ranking metrics: mean average precision and mean recall@k.[<sup>\[8\]</sup>](#f8)


## Quick start

Run a CPU-only pre-built docker image & start the CakeChat serving the model on 8080 port:

```(bash)
docker run --name cakechat-dev -p 127.0.0.1:8080:8080 -it lukalabs/cakechat:latest \
    bash -c "python bin/cakechat_server.py"
```

(Or) with the GPU-enabled image:

```(bash)
nvidia-docker run --name cakechat-gpu-dev -p 127.0.0.1:8080:8080 -it lukalabs/cakechat-gpu:latest \
    bash -c "USE_GPU=0 python bin/cakechat_server.py"
```

That's it! Now you can try it by running `python tools/test_api.py -f localhost -p 8080 -c "Hi! How are you?"` from the host command line.

## Setup

### Docker

This is the easiest way to set up the environment and install all the dependencies.

#### CPU-only setup

1. Install [Docker](https://docs.docker.com/engine/installation/)

2. Build a docker image

Build a CPU-only image:

```(bash)
docker build -t cakechat:latest -f dockerfiles/Dockerfile.cpu dockerfiles/
```

3. Start the container

Run a docker container in the CPU-only environment
```(bash)
docker run --name <CONTAINER_NAME> -it cakechat:latest
```

#### GPU-enabled setup

1. Install [nvidia-docker](https://github.com/NVIDIA/nvidia-docker) for the GPU support.

2. Build a GPU-enabled docker image:

```(bash)
nvidia-docker build -t cakechat-gpu:latest -f dockerfiles/Dockerfile.gpu dockerfiles/
```

3. Start the container

Run a docker container in the GPU-enabled environment:

```(bash)
nvidia-docker run --name <CONTAINER_NAME> -it cakechat-gpu:latest
```

That's it! Now you can train your model and chat with it.


### Manual setup

If you don't want to deal with docker images and containers, you can always simply run (with `sudo`, `--user` or inside your **[virtualenv](https://virtualenv.pypa.io/en/stable/)**):

```(bash)
pip install -r requirements.txt
```

Most likely this will do the job.
**NB:** This method only provides a CPU-only environment. To get a GPU support, you'll need to build and install **[libgpuarray](http://deeplearning.net/software/libgpuarray/installation.html)** by yourself (see [Dockerfile.gpu](dockerfiles/Dockerfile.gpu) for example).


## Getting the model

### Using a pre-trained model

Run `python tools/download_model.py` to download our pre-trained model.

The model is trained with **context size 3** where
the encoded sequence contains **30 tokens or less** and
the decoded sequence contains **32 tokens or less**.
Both encoder and decoder contain **2 GRU layers** with **512 hidden units** each.

The model was trained on a Twitter preprocessed conversational data.
To clean up the data, we removed URLs, retweets and citations.
Also we removed mentions and hashtags that are not preceded by normal words or punctuation marks
and filtered out all messages that contains more than 30 tokens.  
Then we marked out each utterance with our emotions classifier that predicts one of the
5 emotions: "neutral", "joy", "anger", "sadness" and "fear".
To mark-up your own corpus with emotions you can use, for example, [DeepMoji tool](https://github.com/bfelbo/DeepMoji)
or any other emotions classifier that you have.

### Training your own model

1. Put your training text corpus to [`data/corpora_processed/`](data/corpora_processed/).
Each line of the corpus file should be a JSON object containing a list of dialog messages sorted in chronological order.
Code is fully language-agnostic — you can use any unicode texts in datasets.
Refer to our dummy corpus to see the input format [`data/corpora_processed/train_processed_dialogs.txt`](data/corpora_processed/train_processed_dialogs.txt).

2. The following datasets are used for validation and early stopping:

* [`data/corpora_processed/val_processed_dialogs.txt`](data/corpora_processed/val_processed_dialogs.txt)(dummy example) - for the context sensitive dataset
* [`data/quality/context_free_validation_set.txt`](data/quality/context_free_validation_set.txt) - for the context-free validation dataset
* [`data/quality/context_free_questions.txt`](data/quality/context_free_questions.txt) - is used for generating responses for logging and computing distinct-metrics
* [`data/quality/context_free_test_set.txt`](data/quality/context_free_test_set.txt) - is used for computing metrics of the trained model, e.g. ranking metrics

3. Set up training parameters in [`cakechat/config.py`](cakechat/config.py).
See [configuration settings description](#important-configuration-settings) for more details.
4. Run `python tools/prepare_index_files.py` to build the index files with tokens and conditions from the training corpus.
5. Run `python tools/train.py`. Don't forget to set `USE_GPU=<GPU_ID>` environment variable (with GPU_ID as from **nvidia-smi**) if you want to use GPU.
Use `SLICE_TRAINSET=N` to train the model on a subset of the first N samples of your training data to speed up preprocessing for debugging.
6. You can also set `IS_DEV=1` to enable the "development mode". It uses a reduced number of model parameters (decreased hidden layer dimensions, input and output sizes of token sequences, etc.), performs verbose logging and disables Theano graph optimizations. Use this mode for debugging.
7. Weights of your model will be saved in `data/nn_models/`.

### Existing training datasets
You can train a dialog model on any text conversational dataset available to you. A great overview of existing conversational datasets can be found here: https://breakend.github.io/DialogDatasets/


## Running the system

### Local HTTP-server

Run a server that processes HTTP-requests with given input messages (contexts) and returns response messages of the model:

```(bash)
python bin/cakechat_server.py
```

Specify `USE_GPU=<GPU_ID>` environment variable if you want to use a certain GPU.

Wait until the model is compiled.
**Don't forget to run [`tools/download_model.py`](tools/download_model.py) prior to running [`bin/cakechat_server.py`](bin/cakechat_server.py) if you want to start an API with our pre-trained model.**

To make sure everything works fine, test the model on the following conversation:

> – Hi, Eddie, what's up?  
> – Not much, what about you?  
> – Fine, thanks. Are you going to the movies tomorrow?


```(bash)
python tools/test_api.py -f 127.0.0.1 -p 8080 \
    -c "Hi, Eddie, what's up?" \
    -c "Not much, what about you?" \
    -c "Fine, thanks. Are you going to the movies tomorrow?"
```

#### HTTP-server API description

##### /cakechat_api/v1/actions/get_response
JSON parameters are:

|Parameter|Type|Description|
|---|---|---|
|context|list of strings|List of previous messages from the dialogue history (max. 3 is used)|
|emotion|string, one of enum|One of {'neutral', 'anger', 'joy', 'fear', 'sadness'}. An emotion to condition the response on. Optional param, if not specified, 'neutral' is used|

##### Request
```
POST /cakechat_api/v1/actions/get_response
data: {
 'context': ['Hello', 'Hi!', 'How are you?'],
 'emotion': 'joy'
}
```

##### Response OK
```
200 OK
{
 'response': 'I\'m fine!'
}
```

### Gunicorn HTTP-server

We recommend to use [Gunicorn](http://gunicorn.org/) for serving the API of your model at a production scale.

Run a server that processes HTTP-queries with input messages and returns response messages of the model:

```(bash)
cd bin && gunicorn cakechat_server:app -w 1 -b 127.0.0.1:8080 --timeout 2000
```

You may need to install gunicorn from pip: `pip install gunicorn`.


### Telegram bot

You can also test your model in a Telegram bot:
[create a telegram bot](https://core.telegram.org/bots#3-how-do-i-create-a-bot) and run

`python tools/telegram_bot.py --token <YOUR_BOT_TOKEN>`


## Repository overview

* `cakechat/dialog_model/` - contains computational graph, training procedure and other model utilities
* `cakechat/dialog_model/inference/` - algorithms for response generation
* `cakechat/dialog_model/quality/` - code for metrics calculation and logging
* `cakechat/utils/` - utilities for text processing, w2v training, etc.
* `cakechat/api/` - functions to run http server: API configuration, error handling
* `tools/` - scripts for training, testing and evaluating your model


### Important tools

* [`bin/cakechat_server.py`](bin/cakechat_server.py) - 
Runs an HTTP-server that returns response messages of the model given dialog contexts and an emotion. See [run section](#gunicorn-http-server) for details.
* [`tools/train.py`](tools/train.py) - 
Trains the model on your data. You can use the `--reverse` option to train the model used in "\*-reranking" response generation algorithms for more accurate predictions.
* [`tools/prepare_index_files.py`](tools/prepare_index_files.py) - 
Prepares index for the most commonly used tokens and conditions. Use this script before training the model.
* [`tools/quality/ranking_quality.py`](tools/quality/ranking_quality.py) - 
Computes ranking metrics of a dialog model.
* [`tools/quality/prediction_distinctness.py`](tools/quality/prediction_distinctness.py) - 
Computes distinct-metrics of a dialog model.
See the [features section](#network=architecture=and-features) for details about the metrics.
* [`tools/quality/condition_quality.py`](tools/quality/condition_quality.py) - 
Computes metrics on different subsets of a data according to the condition value.
* [`tools/generate_predictions.py`](tools/generate_predictions.py) - 
Evaluates the model. Generates predictions of a dialog model on the set of given dialog contexts and then computes metrics.
Note that you should have a reverse-model in the `data/nn_models` directory, if you want to use "\*-reranking" prediction modes.
* [`tools/generate_predictions_for_condition.py`](tools/generate_predictions_for_condition.py) - 
Generates predictions for a given condition value.
* [`tools/test_api.py`](tools/test_api.py) - 
Example code to send requests to a running HTTP-server.
* [`tools/download_model.py`](tools/download_model.py) - 
Downloads the pre-trained model and index files associated with it. Also compiles the whole model once to create Theano cache.
* [`tools/telegram_bot.py`](tools/telegram_bot.py) - 
Runs a Telegram bot that uses a trained model.


### Important configuration settings

All the configuration parameters for the network architecture, training, predicting and logging steps are defined in [`cakechat/config.py`](cakechat/config.py).
Some inference parameters used in an HTTP-server are defined in [`cakechat/api/config.py`](cakechat/api/config.py).

* Network architecture and size
    * `HIDDEN_LAYER_DIMENSION` is the main parameter that defines the number of hidden units in recurrent layers.
    * `WORD_EMBEDDING_DIMENSION` and `CONDITION_EMBEDDING_DIMENSION` define the number of hidden units that each
    token/condition are mapped into.
    Together they sum up to the dimension of input vector passed to the encoder RNN.
    * Number of units of the output layer of the decoder is defined by the number of tokens in the dictionary in the
    tokens_index directory.
* Decoding algorithm:
    * `PREDICTION_MODE_FOR_TESTS` defines how the responses of the model are generated. The options are the following:
        -  **sampling** – response is sampled from output distribution token-by-token. 
        For every token the temperature transform is performed prior to sampling. 
        You can control the temperature value by tuning `DEFAULT_TEMPERATURE` parameter.
        - **sampling-reranking** – multiple candidate-responses are generated using sampling procedure described above.
        After that the candidates are ranked according to their MMI-score[<sup>\[3\]</sup>](#f3)
        You can tune this mode by picking `SAMPLES_NUM_FOR_RERANKING` and `MMI_REVERSE_MODEL_SCORE_WEIGHT` parameters.
        - **beamsearch** – candidates are sampled using [beam search algorithm](https://en.wikipedia.org/wiki/Beam_search).
        The candidates are ordered according to their log-likelihood score computed by the beam search procedure.
        - **beamsearch-reranking** – same as above, but the candidates are re-ordered after the generation in the same way as
        in sampling-reranking mode.
        
    Note that there are other parameters that affect the response generation process.
    See `REPETITION_PENALIZE_COEFFICIENT`, `NON_PENALIZABLE_TOKENS`, `MAX_PREDICTIONS_LENGTH`.

## Example use cases

By providing additional condition labels within a dataset entries, you can build the following models:
* [A Persona-Based Neural Conversation Model][5] — a model that allows to condition responses on a persona ID to make them lexically similar to the given persona's linguistic style.
* [Emotional Chatting Machine][4]-like model — a model that allows to condition responses on an emotion to provide emotional styles (anger, sadness, joy, etc).
* [Topic Aware Neural Response Generation][6]-like model — a model that allows to condition responses on a certain topic to keep the topic-aware conversation.

To make use of these extra conditions, please refer to the section [Training your own model](#training-your-own-model). Just set the "condition" field in the [training set](data/corpora_processed/train_processed_dialogs.txt) to one of the following: **persona ID**, **emotion** or **topic** label, update the index files and start the training.

## References

* <a name="f1"/><sup>\[1\]</sup> [A Neural Conversational Model][1]
* <a name="f2"/><sup>\[2\]</sup> [How NOT To Evaluate Your Dialogue System][2]
* <a name="f3"/><sup>\[3\]</sup> [A Diversity-Promoting Objective Function for Neural Conversation Models][3]
* <a name="f4"/><sup>\[4\]</sup> [Emotional Chatting Machine: Emotional Conversation Generation with Internal and External Memory][4]
* <a name="f5"/><sup>\[5\]</sup> [A Persona-Based Neural Conversation Model][5]
* <a name="f6"/><sup>\[6\]</sup> [Topic Aware Neural Response Generation][6]
* <a name="f7"/><sup>\[7\]</sup> [A Hierarchical Recurrent Encoder-Decoder For Generative Context-Aware Query Suggestion][7]
* <a name="f8"/><sup>\[8\]</sup> [Quantitative Evaluation of User Simulation Techniques for Spoken Dialogue Systems][8]

[1]: https://arxiv.org/pdf/1506.05869.pdf
[2]: https://arxiv.org/pdf/1603.08023.pdf
[3]: https://arxiv.org/pdf/1510.03055.pdf
[4]: https://arxiv.org/pdf/1704.01074.pdf
[5]: https://arxiv.org/pdf/1603.06155.pdf
[6]: https://arxiv.org/pdf/1606.08340v2.pdf
[7]: https://arxiv.org/pdf/1507.02221.pdf
[8]: http://mi.eng.cam.ac.uk/~sjy/papers/scgy05.pdf

## Credits & Support
**CakeChat** is developed and maintained by the [Replika team](https://replika.ai): [Michael Khalman](https://github.com/mihaha), [Nikita Smetanin](https://github.com/nsmetanin), [Artem Sobolev](https://github.com/artsobolev), [Nicolas Ivanov](https://github.com/nicolas-ivanov), [Artem Rodichev](https://github.com/rodart) and [Denis Fedorenko](https://github.com/sadreamer). Demo by [Oleg Akbarov](https://github.com/olegakbarov), [Alexander Kuznetsov](https://github.com/alexkuz) and [Vladimir Chernosvitov](http://chernosvitov.com/).

All issues and feature requests can be tracked here - [GitHub Issues](https://github.com/lukalabs/cakechat/issues).

## License
© 2018 Luka, Inc. Licensed under the Apache License, Version 2.0. See LICENSE file for more details.
