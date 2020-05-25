# People Counter application

This app is optimized to work with little latency to work
on edge. The model is optimized with Intel [Open Vino toolkit](https://software.intel.com/content/www/us/en/develop/tools/openvino-toolkit.html). This repo also demonstrates how inference could take 
place with a video.

## Explaining Custom Layers

-First the model optimizer searches the list of known layers for 
each layer in the input model. 
-The inference engine thenn loads the  
layers from the input model IR files into the specified device plugin, 
which will search a list of known layer implementations for the device. 

-If your model architecure contains layer or layers that are not in the 
list of known layers for the device, the Inference Engine considers the 
layer to be unsupported and reports an error. To see the layers that 
are supported by each device plugin for the Inference Engine, refer to 
the [Supported Devices documentation](https://docs.openvinotoolkit.org/2019_R1.1/_docs_IE_DG_supported_plugins_Supported_Devices.html).


## Model Tested

Tested with each of the following three models:

- Model 1: SSD Mobilenet
  - [Model Source](https://github.com/tensorflow/models/blob/master/research/object_detection/g3doc/detection_model_zoo.md)
  - I converted the model to an Intermediate Representation with the following arguments
  
```
python mo_tf.py --input_model frozen_inference_graph.pb --tensorflow_object_detection_api_pipeline_config pipeline.config --reverse_input_channels --tensorflow_use_custom_operations_config extensions/front/tf/ssd_v2_support.json
```

  - The model was insufficient for the app because it wasn't pretty accurate while doing inference. 

  - I tried to improve the model for the app by using some transfer learning techniques, I tried to retrain few of the model layers with some additional data but that did not work too well for this use case.
  
- Model 2: SSD Inception V2]
  - [Model Source](http://download.tensorflow.org/models/object_detection/ssd_inception_v2_coco_2018_01_28.tar.gz)
  - I converted the model to an Intermediate Representation with the following arguments
  
```
python mo_tf.py --input_model frozen_inference_graph.pb --tensorflow_object_detection_api_pipeline_config pipeline.config --reverse_input_channels --tensorflow_use_custom_operations_config extensions/front/tf/ssd_v2_support.json
```
  
  - The model was insufficient for the app because it had pretty high latency in making predictions ~155 ms whereas the model I now use just takes ~40 ms. It made accurate predictions but due to a very huge tradeoff in inference time, the model could not be used.
  - I tried to improve the model for the app by reducing the precision of weights, however this had a very huge impact on the accuracy.

- Model 3: SSD Coco MobileNet V1
  - [Model Source](http://download.tensorflow.org/models/object_detection/ssd_inception_v2_coco_2018_01_28.tar.gz)
  - I converted the model to an Intermediate Representation with the following arguments

```
python mo_tf.py --input_model frozen_inference_graph.pb --tensorflow_object_detection_api_pipeline_config pipeline.config --reverse_input_channels --tensorflow_use_custom_operations_config extensions/front/tf/ssd_v2_support.json
```

  - The model was insufficient for the app because it had a very low inference accuracy. I particularly observed a trend that it was unable to identify people with their back facing towards the camera making this model unusable.

## The Model

As having explained above the issues I faced with some other models so I ended up using models from the OpenVino Model zoo, I particularly found two models which seemed best for the job
 
- [person-detection-retail-0002](https://docs.openvinotoolkit.org/latest/person-detection-retail-0002.html)
- [person-detection-retail-0013](https://docs.openvinotoolkit.org/latest/_models_intel_person_detection_retail_0013_description_person_detection_retail_0013.html)

These models are in fact based on the MobileNet model, the MobileNet model performed well for me considering latency and size apart of few inference errors. These models have fixed that error.

I found that [person-detection-retail-0013](https://docs.openvinotoolkit.org/latest/_models_intel_person_detection_retail_0013_description_person_detection_retail_0013.html) had a higher overall accuracy so used it.

### Downloading model

Download all the pre-requisite libraries and source the openvino installation using the following commands:

```sh
pip install requests pyyaml -t /usr/local/lib/python3.5/dist-packages && clear && 
source /opt/intel/openvino/bin/setupvars.sh -pyver 3.5
```

Navigate to the directory containing the Model Downloader:

```sh
cd /opt/intel/openvino/deployment_tools/open_model_zoo/tools/downloader
```

Within there, you'll notice a `downloader.py` file, and can use the `-h` argument with it to see available arguments, `--name` for model name, and `--precisions`, used when only certain precisions are desired, are the important arguments. Use the following command to download the model

```sh
sudo ./downloader.py --name person-detection-retail-0013 --precisions FP16 -o /home/workspace
```

## Performing Inference

Open a new terminal

Execute the following commands:

```sh
  cd webservice/server
  npm install
```
After installation, run:

```sh
  cd node-server
  node ./server.js
```

If succesful you should receive a message-

```sh
Mosca Server Started.
```

Open another terminal

These commands will compile the UI for you

```sh
cd webservice/ui
npm install
```

After installation, run:

```sh
npm run dev
```

If succesful you should receive a message-

```sh
webpack: Compiled successfully
```

Open another terminal and run:

This will set up the `ffmpeg` for you

```sh
sudo ffserver -f ./ffmpeg/server.conf
```

Finally execute the following command in another terminal

This peice of code specifies the testing video provided in `resources/` folder and run it on port `3004`

```sh
python main.py -i resources/Pedestrian_Detect_2_1_1.mp4 -m person-detection-retail-0013/FP32/person-detection-retail-0013.xml -l /opt/intel/openvino/deployment_tools/inference_engine/lib/intel64/libcpu_extension_sse4.so -d CPU -pt 0.6 | ffmpeg -v warning -f rawvideo -pixel_format bgr24 -video_size 768x432 -framerate 24 -i - http://0.0.0.0:3004/fac.ffm
```
