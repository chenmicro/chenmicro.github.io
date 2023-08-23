---
title:  "基于深度学习的简单验证码识别"
categories:
  - code
tags:
  - deep learning
toc: true
---

The recognition of captcha is a popular visual problem (at least in the early 2010s) that can be solved by deep learning, which can obviously improve the personal experience of fighting for various kinds of limited resources online in the student life. Thus, the technique can bring astonishing advantages in the zero-sum activities.
In order to counter back automated technology in real human verification, the form of captchas has been iterated several times. For example, in the early captchas, the characters were located at fixed positions in images. People can recognize the captchas simply by splitting the characters. However, the updated captchas have randomly positioned characters in the images, and thus cannot be segmented manually. Fortunately, new technologies were proposed to solve the problems.
This post aims to introduce basic components and structures of neural networks by explaining effective models for some well-known datasets.

## Single digit recognition by CNN

[MNIST](https://www.kaggle.com/hojjatk/mnist-dataset) is a basic dataset in the vision category, which only contains handwritten digits.
Its simplicity makes it an appropriate problem for beginners to learn CNN.
Among the notebooks in Kaggle, a neural network containing two convolutional layers can achieve 99% accuracy on the dataset.

The structure of a typical network is shown as follows:

| Layer num | Operation          | Output             | Trainable Params   |
| --------- | ------------------ | ------------------ | ------------------ |
| 0         |                    | (None, 28, 28, 1)  |                    |
| 1         | Conv (5×5×30)      | (None, 24, 24, 30) | (5×5×1+1)×30=780   |
| 2         | MaxPool  (2×2)     | (None, 12, 12, 30) | 0                  |
| 3         | Conv (3×3×15)      | (None, 10, 10, 15) | (3×3×30+1)×15=4065 |
| 4         | MaxPool  (2×2)     | (None, 5, 5, 15)   | 0                  |
| 5         | Dropout (p=0.2)    | (None, 5, 5, 15)   | 0                  |
| 6         | Flatten            | (None, 375)        | 0                  |
| 7         | Dense (128,relu)   | (None, 128)        | (375+1)×128=48128  |
| 8         | Dense (50,relu)    | (None, 50)         | (128+1)×50=6450    |
| 9         | Dense (10,softmax) | (None, 10)         | (50+1)×10=510      |

The are two kinds of params in each trainable layer: **weight** and **bias**.

By sharing weights during computation, each channel in a **convolution** layer can extract the same type of features at all locations of the input signal.
Meanwhile, the independence between number of params and the shape of the input can avoid an explosion in the number of params in a convolution layer. 
For example, the layer 3 contains only 4065 trainable params, while layer 7 contains 48k params since it is related to the shape of inputs in the dense layer. 

Besides, according to the theory of neural network, a single **Conv** layer or a single **Dense** layer cannot achieve non-linear classification. Therefore, this model uses 2 **Conv** layers and 3 **Dense** layers.

Finally, a neural network also contains non-trainable layers, including **MaxPool** and **Dropout**. These layers can effectively improve the training speed and stability.

Actually, the **Normalization** layers, especially the **BatchNorm** layer, are another kind of useful layers in the CNN[^Ioffe2015]. By adding an extra step to enforce normalization of the output data, the training of the model can be effectively accelerated. However, it should be noted that **BatchNorm** only normalizes each channel *C* and does not distinguish between *W* and *H* dimensions.

## Multi char recognition by CTC

In this scenario, when testing captchas, the server can tell us whether the code is right, but it won't tell us the exact location of each character.

CTC[^Graves2006](Connectionist Temporal Classification) is a kind of loss function. It is a suitable choice when the alignment between inputs and labels is unknown. The CTC maximizes the probability by computing the possibilities of all paths between input sequences and output labels, which can be solved efficiently by dynamic programming[^Hannun2017].

A common network based on CTC loss function contains two parts: 
(1) Convolution layers to down sample the images and collect vision features, 
and (2) Recurrent layers to extract sequence features.

Similar to the convolution layer, the trainable params in the recurrent layer can also be shared (along sequence *L*), including weight and bias for input *x* and hidden state *h*. Detailed information for several common recurrent units (RNN, LSTM, and GRU) can be found in pytorch [documentation](https://pytorch.org/docs/1.13/nn.html#recurrent-layers).

The [CAPTCHA Images](https://www.kaggle.com/datasets/fournierp/captcha-version-2-images) dataset is a suitable choice to do experiments, and an example model can be found in [keras](https://keras.io/examples/vision/captcha_ocr).

## Reference

[^Ioffe2015]: Ioffe, Sergey, and Christian Szegedy. "Batch normalization: Accelerating deep network training by reducing internal covariate shift." International conference on machine learning. pmlr, 2015.

[^Graves2006]: Graves, Alex, et al. "Connectionist temporal classification: labelling unsegmented sequence data with recurrent neural networks." Proceedings of the 23rd international conference on Machine learning. 2006.

[^Hannun2017]: Hannun, Awni. "Sequence modeling with ctc." Distill 2.11 (2017): e8.
