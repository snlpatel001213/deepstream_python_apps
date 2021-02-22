# RUNTIME SOURCE ADDITION DELETION REFERENCE APP USING DEEP-STREAMSDK 5.0 IN PYTHON

This app demonstrate how to add or remove streams at runtime in Python.

## Prerequisites:

- DeepStreamSDK 5.0
- Python 3.6
- Gst-python

## To run:
​     `python3 deepstream_runtime_add_remove.py [uri1] [uri2] ... [uriN]`
​    **e.g.**
​     `python3 deepstream_runtime_add_remove.py rtsp://127.0.0.1/video1 rtsp://127.0.0.1/video2 rtsp://127.0.0.1/video3`

This document describes the sample deepstream-runtime-add-remove application.

This sample builds on top of the deepstream-test3 sample to demonstrate how to:

* At runtime after a timeout a source will be added periodically. All the components are reconfigured during addition/deletion

* After reaching of max provided streams each source is deleted periodically.

## Stream Addition Logic

This sample accepts one or more rtsp video streams as input. For the first stream creates a source bin for each input and connects the  bins to an instance of the "nvstreammux" element, which forms the batch of frames. The batch of frames is fed to "nvinfer" for batched inference. The batched buffer is composited into a 2D tile array using "nvmultistreamtiler." 

After addition of the first stream, a function [GObject.timeout_add_seconds](https://developer.gnome.org/glib/stable/glib-The-Main-Event-Loop.html#g-timeout-add-seconds)  can be used to check the availability of new streams.  [GObject.timeout_add_seconds](https://developer.gnome.org/glib/stable/glib-The-Main-Event-Loop.html#g-timeout-add-seconds)  allow a function to be called at regular intervals with the default priority, [G_PRIORITY_DEFAULT](https://developer.gnome.org/glib/stable/glib-The-Main-Event-Loop.html#G-PRIORITY-DEFAULT:CAPS). The function is called repeatedly until it returns [`FALSE`](https://developer.gnome.org/glib/stable/glib-Standard-Macros.html#FALSE:CAPS), at which point the timeout is automatically destroyed and the function will not be called again. New stream can be added by querying a SQL database or by reading Json file when this function is triggered. In our case for simplicity we have taken all the stream as command line argument to the app. The first stream will play when app starts and remaining streams will be added periodically.

When a new stream is added source bin is created and connected with the sink pad of streammux in runtime. This is how new stream get added. A snippet of the sample code is as given below.

```python
source_bin=create_source_bin(number_sources, uri_name) # function to create new source bin
source_to_bin_map[number_sources] = source_bin # keeping a map of stream, this will help us in removal phase
pipeline.add(source_bin)
padname="sink_%u" %number_sources
streammux.set_property('batch-size', number_sources) # increasing streammux batch size
sinkpad= streammux.get_request_pad(padname) 
srcpad=source_bin.get_static_pad("src")
srcpad.link(sinkpad) # linking source and sync
source_bin.set_state(Gst.State.PLAYING) # set the bin state to play
```

## Stream Removal Logic

The stream removal process has following events : 

1. Get the sink pad for the stream id.
2. A flush stop event is sent to the sink so that it stop accepting data.
3. A release pad request makes the element free of the given pad and finally removed.
4. number of active sources are reduced and so streammux batch size is also reduced.

```python
padname="sink_%u" %i 
sinkpad= streammux.get_request_pad(padname)
Gst.Pad.send_event(sinkpad, Gst.Event.new_flush_stop (False))
Gst.Element.release_request_pad(streammux, sinkpad)
Gst.Bin.remove (pipeline, source_to_bin_map[i])
number_sources =  number_sources - 1
streammux.set_property('batch-size', max(1,number_sources))
```

