# People Counter apps

-This app is optimized to work with minimum latency to work on edge. 
-The model is optimized with Intel [Open Vino toolkit](https://software.intel.com/content/www/us/en/develop/tools/openvino-toolkit.html). 

## Explaining Custom Layers

-First the model optimizer searches the list of known layers for each layer in the input model. 
-The inference engine loads the layers from the input model IR files into the device plugin, which searches a list of known layer implementations for the device. 

-If our model architecure contains layers that are not in the list of known layers, the Inference Engine then considers the 
layer to be unsupported and reports the specified error. 
-To see the layers that are supported by each device plugin for the Inference Engine, refer to 
the [Supported Devices documentation]
(https://docs.openvinotoolkit.org/2019_R1.1/_docs_IE_DG_supported_plugins_Supported_Devices.html).


## Models Tested

Tested the app with each of the following three models:

- Model 1: SSD Mobilenet
  - [Model Source](https://github.com/tensorflow/models/blob/master/research/object_detection/g3doc/detection_model_zoo.md)
  - Converted the model to an IR with the following arguments
  
```
python mo_tf.py --input_model frozen_inference_graph.pb --tensorflow_object_detection_api_pipeline_config pipeline.config --reverse_input_channels --tensorflow_use_custom_operations_config extensions/front/tf/ssd_v2_support.json
```
-Verdict:
  - This model was not suitable for this app because the output was not accurate as expected. 
  
- Model 2: SSD Inception V2
  - [Model Source](http://download.tensorflow.org/models/object_detection/ssd_inception_v2_coco_2018_01_28.tar.gz)
  - Converted the model to an IR with the following arguments
  
```
python mo_tf.py --input_model frozen_inference_graph.pb --tensorflow_object_detection_api_pipeline_config pipeline.config --reverse_input_channels --tensorflow_use_custom_operations_config extensions/front/tf/ssd_v2_support.json
```
-Verdict:
  - Again this model was not appropriate for the app because it has high latency while making predictions.
  
- Model 3: SSD Coco MobileNet V1
  - [Model Source](http://download.tensorflow.org/models/object_detection/ssd_inception_v2_coco_2018_01_28.tar.gz)
  - Converted the model to an IR with the following arguments

```
python mo_tf.py --input_model frozen_inference_graph.pb --tensorflow_object_detection_api_pipeline_config pipeline.config --reverse_input_channels --tensorflow_use_custom_operations_config extensions/front/tf/ssd_v2_support.json
```
-Verdict:
  - Once again the model was not suitable for this app because of low inference accuracy. 

## The Models

Due to the above mentioned issues I finally chosen the models from the OpenVino Model zoo and found these 2 models suitable for our app.
 
- [person-detection-retail-0002](https://docs.openvinotoolkit.org/latest/person-detection-retail-0002.html)
- [person-detection-retail-0013](https://docs.openvinotoolkit.org/latest/_models_intel_person_detection_retail_0013_description_person_detection_retail_0013.html)

Also, found that [person-detection-retail-0013](https://docs.openvinotoolkit.org/latest/_models_intel_person_detection_retail_0013_description_person_detection_retail_0013.html) has higher accuracy so prefered this one.

### How to Download the Model

Download all the pre-requisite libraries and source the openvino installation using the following commands:

```sh
pip install requests pyyaml -t /usr/local/lib/python3.5/dist-packages && clear && 
source /opt/intel/openvino/bin/setupvars.sh -pyver 3.5
```

Then, Navigate to the directory containing the Model Downloader:

```sh
cd /opt/intel/openvino/deployment_tools/open_model_zoo/tools/downloader
```

You will notice a `downloader.py` file, and can use the `-h` argument with it to see available arguments, `--name` for model name, and `--precisions`, used when only certain precisions are desired, are the important arguments. 

Use the following command to download the model

```sh
sudo ./downloader.py --name person-detection-retail-0013 --precisions FP16 -o /home/workspace
```

## Performing Inference as shown

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

If succesfully executed, then you will received a message-

```sh
Mosca Server Started.
```

Open another terminal

These commands will compile the UI 

```sh
cd webservice/ui
npm install
```

After installation, run:

```sh
npm run dev
```

If succesfully executed, you will received a message-

```sh
webpack: Compiled successfully
```

Open another terminal and run:

This will set up the `ffmpeg` 

```sh
sudo ffserver -f ./ffmpeg/server.conf
```

Now, execute the following command in another terminal

The command says, the testing video provided in the `resources/` folder and run it on the port `3004`

```sh
python main.py -i resources/Pedestrian_Detect_2_1_1.mp4 -m person-detection-retail-0013/FP32/person-detection-retail-0013.xml -l /opt/intel/openvino/deployment_tools/inference_engine/lib/intel64/libcpu_extension_sse4.so -d CPU -pt 0.6 | ffmpeg -v warning -f rawvideo -pixel_format bgr24 -video_size 768x432 -framerate 24 -i - http://0.0.0.0:3004/fac.ffm
```
