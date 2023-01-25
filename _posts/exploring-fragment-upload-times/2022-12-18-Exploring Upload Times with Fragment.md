---
title: Exploring Upload Times in Fragment
published: true
---

Last updated: Jan 25, 2023

---

## [](#header-2)Background

For the final project in this course, I wanted to incorporate a machine learning model into one of my own personal projects. Over the summer, I built an application that allows users to upload large files and store them on GitHub in a way that mimics Google Drive. This would essentially give them a large capactiy of file storage in the cloud at no cost. In order to upload large files without going over the maximum repository size on GitHub, the application splits up your file, distributes it across multiple repositories, and stitches it back up when you want to download it.

However, uploading files using Git instead of a regular HTTPS bulk upload resulted in upload times that were extremely large.

<figure>
<img src="https://firebasestorage.googleapis.com/v0/b/portfolio-7ec02.appspot.com/o/blog%2Ffragment-upload-times%2F1.png?alt=media&token=fd5484b8-6afe-41c2-bdf0-a4d862ce7020" alt="kernel central of operating system">
</figure>

## [](#header-2)Goals

The goal for this project was to build a machine learning model that could predict upload times based on certain split sizes for files. Once deployed on a server, the application could make calls to the server to see just how to split the file up. The hope was that the configuration that the machine learning model comes up with reduces the overall upload time of the file. These were some of the main goals I had:

- Build a model that can predict a file upload
- Export the model for use elsewhere
- Integrate it with the Fragment client app

## [](#header-2)Dataset

The dataset I used came from my own MongoDB database that stores logs of user-uploaded files. This data was collected from September 2022 to the beginning of December 2022 and contains around 5,000 instances. When a user uploads their file, it gets split at a random file size before being uploaded using Git. Once the upload is complete, a small packet of data gets logged to the database:

- `file_size_bytes`: The size of the file in bytes
- `upload_time`: The total time the upload takes
- `upload_speed_bytes`: The upload speed (not the internet speed necessarily)
- `file_type`: The type of file (scrambled for user safety)
- `split_size`: The size of the pieces of the files
- `timestamp`: Time of upload

![](https://firebasestorage.googleapis.com/v0/b/portfolio-7ec02.appspot.com/o/blog%2Ffragment-upload-times%2F2.png?alt=media&token=c77b45f8-b865-4f3d-8671-22f6bdf8744d)

## [](#header-2)Model

While my initial tests and prototypes were done using a regression model from `scikit-learn` I ultimately chose to use `TensorFlow` to build my model. One of the basic models that TensorFlow has is `keras`, which allows the user to add hidden layers in between the input and output. The specific keras model I used is called a `Sequential` model. They are a type of model that is composed of a linear stack of layers. The layers are connected sequentially, meaning that the output of one layer is the input of the next layer. This type of model is usually used for a variety of tasks, such as image classification, natural language processing, and time series forecasting. However, I used the model to predict a numerical output.

Although I initially tried to use a linear regression model, I ultimately settled on the keras deep neural network. Keras Sequential models are more powerful than linear regression models as they can model complex non-linear relationships between variables. Furthermore, Keras Sequential models are more accurate than linear regression models as they can capture non-linear relationships between variables.

## [](#header-2)Building the Model

In order to ensure that I had enough data for training and for testing, I split the data with 80% used for training and 20% used for testing, similar to what we had been doing in the homework assignments. I also chose to only use the quantitative variables which included `file_size_bytes`, `upload_speed_bytes`, and `split_size` as predictors for `upload_time`. Many of these values were on different scales and had values ranging from the low millions of bytes to billions. As a result, I had to add a normalization layer to the model to ensure that accuracy would be preserved. Once this was done, the model could be built as follows:

```python
model = keras.Sequential([layers.Dense(64, activation=tf.nn.relu,
                         input_shape=[len(train_dataset.keys())]),
                         layers.Dense(64, activation=tf.nn.relu),
                         layers.Dense(1)])
```

This model has 3 different layers: the first and second each have 64 nodes while the third layer is the output layer with 1 node. The model also requires an activation function, and I chose to use Tensorflow's ReLU. ReLU is a type of non-linear activation function that is used to introduce non-linearity into the model.

The model also allows you to specify an optimizer. I chose to use the `Adam` optimizaer which is well-suited for problems with noisy or sparse gradients, as it can automatically adjust its learning rate for each parameter:

```python
optimizer = tf.keras.optimizers.Adam(0.001)
model.compile(loss='mse', optimizer=optimizer, metrics=['mae', 'mse'])
```

## [](#header-2)Training and Fitting the Model

I fit the model using 500 epochs to begin with. However, TensorFlow allows you to specify an early stop in training if it detect the model might be overfitting or if the mean error is not dropping anymore:

```python
early_stop = keras.callbacks.EarlyStopping(monitor='val_loss', patience=10)
```

After 500 epochs, the mean error output can be seen below. The output suggests that 500 epochs is overfitting and that the model is sufficient with under 100:

| Mean Sq Error                                                                                                                                                                     | Mean Abs Error                                                                                                                                                                  |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ![](https://firebasestorage.googleapis.com/v0/b/portfolio-7ec02.appspot.com/o/blog%2Ffragment-upload-times%2Fdownload-1.png?alt=media&token=d0bd323c-a52a-4288-ac88-1d0912d66f55) | ![](https://firebasestorage.googleapis.com/v0/b/portfolio-7ec02.appspot.com/o/blog%2Ffragment-upload-times%2Fdownload.png?alt=media&token=11e72a27-67a6-470d-b212-40c94420cadd) |

## [](#header-2)Model Testing

In order to test the model, we can feed the validation data through it. When looking at the predicted upload speed, we're expecting a linear pattern that is similar to the normalized data we fed in. Additionally, to know that the model is sufficient, we expect the Prediction Upload Error to be roughly unimodal and normal. All of these conditions are met:

| File Size vs. Predicted Upload                                                                                                                                                    | Prediction Upload Error Distribution                                                                                                                                              |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ![](https://firebasestorage.googleapis.com/v0/b/portfolio-7ec02.appspot.com/o/blog%2Ffragment-upload-times%2Fdownload-2.png?alt=media&token=b9c04407-4fc5-4a1f-8aef-3bc9ed884feb) | ![](https://firebasestorage.googleapis.com/v0/b/portfolio-7ec02.appspot.com/o/blog%2Ffragment-upload-times%2Fdownload-3.png?alt=media&token=2e99eb3f-0a77-41e4-9c84-c853ae0d7793) |

## [](#header-2)Exporting the Model

Because the server is written in TypeScript, we need to use TensorflowJS to export the model in a format that is able to be used by the server. The following code exports the model and produces two files: `model.json` and a shard binary file which together contains data about the model.

```python
tfjs.converters.save_keras_model(model, '.')
```

## [](#header-2)Using the Model

On the server side, we can load in the model by importing the JSON file we exported from the Jupyter notebook. After loading the model, predictions can also be made using any input data we specify. However, the input values must also be normalized, which the model takes into account when exported.

```typescript
model = await tensorflow.loadLayersModel(
  "file://src/mlmodel/static/model.json"
);
const data = [[file_size_bytes, upload_speed_bytes, MB(generated_split_size)]];

const dataframe = make_dataframe(data, LABELS);
let normedData = normalize(dataframe);
let prediction = model.predict(normedData.tensor) as tensorflow.Tensor;
```

Now that the model has been loaded onto the server, we can make a prediction by making a request to a URL endpoint. This example can be seen in the file `demo.ipynb` which is attached as part of this project.

## [](#header-2)Conclusions

After integrating the model in the server and using it as part of the Fragment application, the main conclusion we can make is that the model does improve upload time over the previous method of generating a configuration to split the file. In the examples below, we can compare the durations of uploading a 75MB file using the old method vs. using the model.

After a demo run with the old model, Fragment took around `94 seconds` to upload the file:

<figure>
<img src="https://firebasestorage.googleapis.com/v0/b/portfolio-7ec02.appspot.com/o/blog%2Ffragment-upload-times%2Fa1.png?alt=media&token=24a1e54e-ba4a-465c-be70-6012f759b28a" alt="kernel central of operating system">
</figure>

However, after using the machine learning model from this project, the model recommended to split the file into 21MB sizes pieces. This time, Fragment uploaded the file in 48 seconds, which was significantly faster. This was also better than the time of 68 seconds that the model predicted.

<figure>
<img src="https://firebasestorage.googleapis.com/v0/b/portfolio-7ec02.appspot.com/o/blog%2Ffragment-upload-times%2Fa2.png?alt=media&token=8fa48e3a-f714-44f1-b4a0-358b59a8d1ff" alt="kernel central of operating system">
</figure>

Overall, we can conclude that the model improves upload times when integrated in the Fragment client app. Additionally, this improvement becomes more drastic the larger the file is: the bigger the file, the more time you save by using the model over the traditional upload process.

## [](#header-2)Implications

This project was extremely insightful for me as it allowed me to connect what I learned in class during the semester with my personal projects outside of class. Additionally, I am thrilled that the model actually improved the performance of my application because I was skeptical about the potential outcomes before I started the project.

This project has implications that reach much further than just my grade. When the new build is pushed, the model will hopefully improve the experience of the many people across the world who currently upload and store files using Fragment. After collecting data from other users who have used the model, I hope to feed this data back into the model in order to improve the predictions. This hopefully means that that the product experience improves over time.
