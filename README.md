# Handwritten Korean Character Recognition with TensorFlow and Android

Hangul, the Korean alphabet, has 19 consonant and 21 vowel letters.
Combinations of these letters give a total of 11,172 possible Hangul
syllables/characters. However, only a small subset of these are typically used.

This journey will cover the creation process of an Android application that
will utilize a TensorFlow model trained to recognize Korean syllables.
In this application, users will be able to draw a Korean syllable on their
phone, and the application will attempt to infer what the character is by using
the trained model. Furthermore, users will be able to form words or sentences in
the application which they can then translate using the
[Watson Language Translator](https://www.ibm.com/watson/services/language-translator/)
service.

![Demo App](doc/source/images/hangul_tensordroid_demo1.gif "Android application")

The following steps will be covered:
1. Generating image data using free Hangul-supported fonts found online and
   elastic distortion.
2. Converting images to TFRecords format to be used for input and training of
   the model.
3. Training and saving the model.
4. Using the saved model in a simple Android application.
5. Connecting the Watson Language Translator service to translate the characters.


## Prerequisites

Make sure you have the python requirements for this journey installed on you
system. From the root of the repository, run:

```
pip install -r requirements.txt
```


## Generating Image Data

In order to train a decent model, having copious amounts of data is necessary.
However, getting a large enough dataset of actual handwritten Korean characters
is challenging to find and cumbersome to create.

One way to deal with this data issue is to programmatically generate the data
yourself, taking advantage of the abundance of Korean font files found online.
So, that is exactly what we will be doing.

Provided in the tools directory of this repo is
[hangul-image-generator.py](./tools/hangul-image-generator.py).
This script will use fonts found in the fonts directory to create several images
for each character provided
in the given labels file. The default labels file is
[2350-common-hangul.txt](./labels/2350-common-hangul.txt)
which contains 2350 frequent characters derived from the
[KS X 1001 encoding](https://en.wikipedia.org/wiki/KS_X_1001). Other label files
are [256-common-hangul.txt](./labels/256-common-hangul.txt) and
[512-common-hangul.txt](./labels/512-common-hangul.txt). These were adapted from
the top 6000 Korean words compiled by the National Institute of Korean Language
listed [here](https://www.topikguide.com/download/6000_korean_words.htm).
If you don't have a powerful machine to train on, using a smaller label set can
help reduce the amount of model training time later on.

The [fonts](./fonts) folder is currently empty, so before you can generate the
Hangul dataset, you must first download
several font files as described in the fonts directory [README](./fonts/README.md).
For my dataset, I used around 22 different font files, but more can always be
used to improve your dataset. Once your fonts directory is populated,
then you can proceed with the actual image generation:

```
python ./tools/hangul-image-generator.py
```

Optional flags for this are:

* `--label-file` for specifying a different label file (perhaps with less characters).
  Default is _./labels/2350-common-hangul.txt_.
* `--font-dir` for specifying a different fonts directory. Default is _./fonts_.
* `--output-dir` for specifying the output directory to store generated images.
  Default is _./image-data_.

Depending on how many labels and fonts there are, this script may take a while
to complete. In order to bolster the dataset, random elastic distortions are also
performed on each generated character image. An example is shown below, with the
original character displayed first, followed by three elastic distortions.

![Normal Image](doc/source/images/hangul_normal.jpeg "Normal font character image")
![Distorted Image 1](doc/source/images/hangul_distorted1.jpeg "Distorted font character image")
![Distorted Image 2](doc/source/images/hangul_distorted2.jpeg "Distorted font character image")
![Distorted Image 3](doc/source/images/hangul_distorted3.jpeg "Distorted font character image")

Once the script is done, the output directory will contain a _hangul-images_ folder
which will hold all the 64x64 JPEG images. The output directory will also contain a
_labels-map.csv_ file which will map all the image paths to their corresponding
labels.


## Converting Images to TFRecords

The TensorFlow standard input format is TFRecords, so in order to better feed in
data to a TensorFlow model, let's first create several TFRecords files from our
images. A [script](./tools/convert-to-tfrecords.py) is provided that will do this
for us.

This script will first partition the data so that we have a training set and
also a testing set (15% testing, 85% training). Then it will read in
all the image and label data based on the _labels-map.csv_ file that was
generated above.

To run the script, you can simply do:

```
python ./tools/convert_to_tfrecords.py
```

Optional flags for this are:

* `--image-label-csv` for specifying the CSV file that maps image paths to labels.
  Default is _./image-data/labels-map.csv_
* `--label-file` for specifying the labels that correspond to your training set.
  This is used by the script to determine the number of classes.
  Default is _./labels/2350-common-hangul.txt_.
* `--output-dir` for specifying the output directory to store TFRecords files.
  Default is _./tfrecords-output_.
* `--num-shards-train` for specifying the number of shards to divide training set
  TFRecords into. Default is _3_.
* `--num-shards-test` for specifying the number of shards to divide testing set
  TFRecords into. Default is _1_.

Once this script has completed, you should have sharded TFRecords files in the
output directory _./tfrecords-output_.

```
$ ls ./tfrecords-output
test1.tfrecords    train1.tfrecords    train2.tfrecords    train3.tfrecords
```

## Training the Model

Now that we have a lot of data, it is time to actually use it. In the root of
the project is [hangul-model.py](./hangul-model.py). This script will handle
creating an input pipeline for reading in TFRecords files and producing random
batches of images and labels. Next, a convolutional neural network (CNN) is
defined, and training is performed. After training, the model is exported
so that it can be used in our Android application.

The model here is similar to the MNIST model described on the TensorFlow
[website](https://www.tensorflow.org/get_started/mnist/pros). A third
convolutional layer is added to extract more features to help classify for the
much greater number of classes.

To run the script, simply do the following from the root of the project:

```
python ./hangul_model.py
```

Optional flags for this script are:

* `--label-file` for specifying the labels that correspond to your training set.
  This is used by the script to determine the number of classes to classify for.
  Default is _./labels/2350-common-hangul.txt_.
* `--tfrecords-dir` for specifying the directory containing the TFRecords shards.
  Default is _./tfrecords-output_.
* `--output-dir` for specifying the output directory to store model checkpoints,
   graphs, and Protocol Buffer files. Default is _./saved-model_.

Note: In this script there is a NUM_TRAIN_STEPS variable defined at the top.
This should be increased with more data (or vice versa). The number of steps
should cover several iterations over all of the training data (epochs).
For example, if I had 200,000 images in my training set, one epoch would be
_200000/100 = 2000_ steps where _100_ is the batch size. So, if I wanted to
train for 30 epochs, I would simply do _2000*30 = 60000_ training steps.

Depending on how many images you have, this will likely take a long time to
train (several hours to maybe even a day), especially if only training on a laptop.
If you have access to GPUs, these will definitely help speed things up, and you
should certainly install the TensorFlow version with GPU support (supported on
[Ubuntu](https://www.tensorflow.org/install/install_linux) and
[Windows](https://www.tensorflow.org/install/install_windows) only).

On my Windows desktop computer with an Nvidia GTX 1080 graphics card, training
about 200,000 images with the script defaults took about an hour and a half.

Another alternative is to use a reduced label set (i.e. 256 vs 2350 Hangul
characters) which can reduce the computational complexity quite a bit.

As the script runs, you should hopefully see the printed training accuracies
grow towards 1.0, and you should also see a respectable testing accuracy after
the training. When the script completes, the exported model we should use will
be saved, by default, as `./saved-model/optimized_hangul_tensorflow.pb`. This is
a [Protocol Buffer](https://en.wikipedia.org/wiki/Protocol_Buffers) file
which represents a serialized version of our model with all the learned weights
and biases. This specific one is optimized for inference-only usage.

## Trying Out the Model

Before we jump into making an Android application with our newly saved model,
let's first try it out. Provided is a script that will load your model and use it
for inference on a given image. Try it out on images of your own, or download some
of the sample images below. Just make sure each image is 64x64 pixels with a
black background and white character color.

```
python ./tools/classify-hangul.py <Image Path>
```

Optional flags for this are:

* `--label-file` for specifying a different label file. This is used to map indices
  in the one-hot label representations to actual characters.
  Default is _./labels/2350-common-hangul.txt_.
* `--graph-file` for specifying your saved model file.
  Default is _./saved-model/optimized_hangul_tensorflow.pb_.

***Sample Images:***

![Sample Image 1](doc/source/images/hangul_sample1.jpeg "Sample image")
![Sample Image 2](doc/source/images/hangul_sample2.jpeg "Sample image")
![Sample Image 3](doc/source/images/hangul_sample3.jpeg "Sample image")
![Sample Image 4](doc/source/images/hangul_sample4.jpeg "Sample image")
![Sample Image 5](doc/source/images/hangul_sample5.jpeg "Sample image")

After running the script, you should see the top five predictions and their
corresponding scores. Hopefully the top prediction matches what your character
actually is.

***Note***: If running this script on Windows, in order for the Korean characters
to be displayed on the console, you must first change the active code page to
support UTF-8. Just run:

```
chcp 65001
```

Then you must change the console font to be one that supports Korean text
(like Batang, Dotum, or Gulim).


## Creating the Android Application

With the saved model, a simple Android application can be created that will be
able to classify handwritten Hangul that a user has drawn.
