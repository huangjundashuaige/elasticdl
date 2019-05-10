# ElasticDL Client: Submit EDL job to mac kubernetes docker-for-desktop from laptop.

## Check Environment

make sure the kubernetes docker-for-desktop (not minikube) already installed on your laptop.

## Download ElasticDL Source Code
```bash
git clone https://github.com/wangkuiyi/elasticdl.git
cd elasticdl
```

## Build ElasticDL Dev Docker Image
```bash
docker build -t elasticdl:dev -f dockerfile/elasticdl.dev .
```
The k8s example use `elasticdl:dev` Docker image as the base master/worker image.


## Write a Keras Model

save the following file as `edl_k8s_examples/mnist_model.py`, or just use the one in the directory.

```python
import tensorflow as tf
import numpy as np


class MnistModel(tf.keras.Model):
    def __init__(self, channel_last=True):
        super(MnistModel, self).__init__(name='mnist_model')
        if channel_last:
            self._reshape = tf.keras.layers.Reshape((28, 28, 1))
        else:
            self._reshape = tf.keras.layers.Reshape((1, 28, 28))
        self._conv1 = tf.keras.layers.Conv2D(
            32, kernel_size=(3, 3), activation='relu')
        self._conv2 = tf.keras.layers.Conv2D(
            64, kernel_size=(3, 3), activation='relu')
        self._batch_norm = tf.keras.layers.BatchNormalization()
        self._maxpooling = tf.keras.layers.MaxPooling2D(
            pool_size=(2, 2))
        self._dropout = tf.keras.layers.Dropout(0.25)
        self._flatten = tf.keras.layers.Flatten()
        self._dense = tf.keras.layers.Dense(10)

    def call(self, inputs, training=False):
        x = self._reshape(inputs)
        x = self._conv1(x)
        x = self._conv2(x)
        x = self._batch_norm(x, training=training)
        x = self._maxpooling(x)
        if training:
            x = self._dropout(x, training=training)
        x = self._flatten(x)
        x = self._dense(x)
        return x

    @staticmethod
    def input_shapes():
        return (1, 28, 28)

    @staticmethod
    def input_names():
        return ['image']
        
    @staticmethod
    def loss(output, labels):
        return tf.reduce_mean(
            tf.nn.sparse_softmax_cross_entropy_with_logits(
                logits=output, labels=labels['label']))

    @staticmethod
    def optimizer(lr=0.1):
        return tf.train.GradientDescentOptimizer(lr)

    @staticmethod
    def input_fn(records):
        image_list = []
        label_list = []
        # deserialize
        for r in records:
            parsed = np.frombuffer(r, dtype="uint8")
            label = parsed[-1]
            image = np.resize(parsed[:-1], new_shape=(28, 28))
            image = image.astype(np.float32)
            image /= 255
            label = label.astype(np.int32)
            image_list.append(image)
            label_list.append(label)

        # batching
        batch_size = len(image_list)
        images = np.concatenate(image_list, axis=0)
        images = np.reshape(images, (batch_size, 28, 28))
        labels = np.array(label_list)
        return {'image': images, 'label': labels}
```

## Submit EDL job


```bash
python elasticdl/client/client.py \
    --model_file=edl_k8s_examples/mnist_model.py \
    --model_class=MnistModel \
    --train_data_dir=/data/mnist/train \
    --num_epoch=1 \
    --minibatch_size=10 \
    --record_per_task=100 \
    --num_worker=1 \
    --grads_to_wait=2
```

## Check the pod status

```bash
kubectl get pods
kubectl logs ${pod_name}
```