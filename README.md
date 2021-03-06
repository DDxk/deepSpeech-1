# TensorFlow implementation of deepSpeech
End-to-end speech recognition using distributed TensorFlow

This repository contains TensorFlow code for an end-to-end speech recognition engine using Deep Neural Networks inspired by Baidu's DeepSpeech model, that can train on multiple GPUs. This work was developed at Ford Research and Innovation Center Palo Alto.

This software is released under a BSD license. The license to this software does not apply to TensorFlow, which is available under the Apache 2.0 license, or the third party pre-requisites listed below, which are available under their own respective licenses.

Pre-requisites
-------------
* Python 3.5
* TensorFlow - version: 1.2+
* python-levenshtein - to compute Character-Error-Rate
* python_speech_features - to generate mfcc features
* PySoundFile - to read FLAC files
* scipy - helper functions for windowing
* tqdm - for displaying a progress bar
* kenlm - for language modeling [optional]

Getting started
------------------
*Step 1: Create a virtualenv and install all dependencies.*

With anaconda, you can use:
```
$ conda create -n 'SpeechRecog' python=3.5.0
$ source activate SpeechRecog
(SpeechRecog)$ pip install python-Levenshtein
(SpeechRecog)$ pip install python_speech_features
(SpeechRecog)$ pip install PySoundFile
(SpeechRecog)$ pip install scipy
(SpeechRecog)$ pip install tqdm
(SpeechRecog)$ pip install kenlm
(SpeechRecog)$ pip install asr.align

# Install TensorFlow by following instructions here:
https://www.tensorflow.org/install/
For GPU support, make sure you have installed CUDA and cuDNN using the instructions in the above link.

# Update ~/.bashrc to reflect path for CUDA.
Add these lines to the ~/.bashrc:
export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:/usr/local/cuda/lib64:/usr/local/cuda/extras/CUPTI/lib64"
export CUDA_HOME=/usr/local/cuda

# On a Linux machine with GPU support, use:
(SpeechRecog)$ export TF_BINARY_URL=https://storage.googleapis.com/tensorflow/linux/gpu/tensorflow_gpu-1.5.0-cp34-cp34m-linux_x86_64.whl
(SpeechRecog)$ pip install --upgrade $TF_BINARY_URL

# If you see a sndfile Error:
sudo apt-get install libsndfile1

```
*Step 2: Clone this git repo.*
```
(SpeechRecog)$ git clone https://github.com/FordSpeech/deepSpeech.git
(SpeechRecog)$ cd deepSpeech
```

Preprocessing the data
----------------------
*Step 1: Download and unpack the LibriSpeech data*
```
Inside the github repo that you have cloned run:
$ mkdir -p data/librispeech
$ cd data/librispeech
$ wget http://www.openslr.org/resources/12/train-clean-100.tar.gz
$ wget http://www.openslr.org/resources/12/dev-clean.tar.gz
$ wget http://www.openslr.org/resources/12/test-clean.tar.gz
$ mkdir audio
$ cd audio
$ tar xvzf ../train-clean-100.tar.gz LibriSpeech/train-clean-100 --strip-components=1
$ tar xvzf ../dev-clean.tar.gz LibriSpeech/dev-clean  --strip-components=1
$ tar xvzf ../test-clean.tar.gz LibriSpeech/test-clean  --strip-components=1
```
*Step 2: Run this command to preprocess the audio and generate TFRecord files.*

The computed mfcc features will be stored within TFRecords files inside data/librispeech/processed/
```
(SpeechRecog)$ cd ../../../code/
(SpeechRecog)$ python preprocess_LibriSpeech.py
```

Training a model
----------------
```
(SpeechRecog)$python deepSpeech_train.py --num_rnn_layers 3 --rnn_type 'bi-dir' --initial_lr 3e-4
--max_steps 30000 --train_dir PATH_TO_SAVE_CHECKPOINT_FILE

# To continue training from a saved checkpoint file
(SpeechRecog)$python deepSpeech_train.py --checkpoint_dir PATH_TO_SAVED_CHECKPOINT_FILE --max_steps 40000
```
The script train.sh contains commands to train on utterances in sorted order for the first epoch and then to resume training on shuffled utterances.
Note that during the first epoch, the cost will increase and it will take longer to train on later steps because the utterances are presented in sorted order to the network.

Monitoring training
--------------------
Since the training data is fed through a shuffled queue, to check validation loss a separate graph needs to be set up in a different session and potentially on an additional GPU. This graph is fed with the valildation data to compute predictions. The deepSpeech_test.py script initializes the graph from a previously saved checkpoint file and computes the CER on the eval_data every 5 minutes by default. It saves the computed CER values in the models/librispeech/eval folder. By calling tensorboard with logdir set to models/librispeech, it is possible to monitor validation CER and training loss during training.
```
(SpeechRecog)$python deepSpeech_test.py --eval_data 'val' --checkpoint_dir PATH_TO_SAVED_CHECKPOINT_FILE
(SpeechRecog)$tensorboard --logdir PATH_TO_SUMMARY
```
Testing a model
----------------
```
(SpeechRecog)$python deepSpeech_test.py --eval_data 'test' --checkpoint_dir PATH_TO_SAVED_CHECKPOINT_FILE
```

Using a Language Model:
-------------------------
Language Models can help improve the CER of models by acting as spell checker or even providing the most probable word for a sentence. We show one possible implementation of using a language model during inference as a spell checker after decoding.

Download the kenlm source code and create a language model (.klm file) based on the the source corpus which you are transcribing for. (In this case the Librispeech transcritions)
http://victor.chahuneau.fr/notes/2012/07/03/kenlm.html has an excellent tutorial for creating your first language model.

Use the --lm flag to point to the location of your language model. This will enable deepSpeech_test.py to create a LM() class from lm.py and use the language model for simple spell checking.
```
(SpeechRecog)$python deepSpeech_test.py --eval_data 'test' --checkpoint_dir PATH_TO_SAVED_CHECKPOINT_FILE --lm PATH_TO_LANGUAGE_MODEL
```

Using a Custom Decoder:
---------------------------

In lm_decoder.py we have created two classes, GreedyDecoder()and PrefixDecoder().
The GreedyDecoder() implements basic greedy decoding of the output probabilities from deepSpeech.

```
(SpeechRecog)$python deepSpeech_test.py --eval_data 'test' --checkpoint_dir PATH_TO_SAVED_CHECKPOINT_FILE --decoder greedy
```

The PrefixDecoder() implements the ctc prefix beam search decoding algorithm mentioned in https://arxiv.org/abs/1408.2873. This implementation requires a language model.

```
(SpeechRecog)$python deepSpeech_test.py --eval_data 'test' --checkpoint_dir PATH_TO_SAVED_CHECKPOINT_FILE --lm PATH_TO_LANGUAGE_MODEL --decoder prefix
```

Note: Both implementations are simply examples and do not yield optimal results. As both of these decoders are written in python they are slower than a custom tensorflow op implementation.
These decoders are great starting points for creating your own custom decoder, implementing other decoder algorithms, or understanding how a language model can be incorporated with a decoder.

Thanks to Mihail Eric for help with the algorithm and discussions in creating the prefix decoder.
