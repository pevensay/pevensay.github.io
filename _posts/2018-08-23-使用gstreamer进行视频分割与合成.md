---
layout:     post
title:      使用gstreamer进行视频分割与合成
subtitle:    ""
date:       2018-08-23
author:     ech069
header-img: img/15.jpg
catalog: true
tags:
  - linux
---

## 将视频分割为图片
### 将文件类型视频切割为图片
1. 命令行形式
```
gst-launch-1.0 filesrc location="output.mp4" ! decodebin ! queue ! autovideoconvert ! pngenc ! multifilesink location="pig_pictures/pig_pictures1/frame%d.png"
```
2. 源文件形式
```
#include <gst/gst.h>
#include <glib.h>
#include <stdlib.h>
//#include <Python.h>
#include <stdio.h>


static gboolean 
bus_call (GstBus     *bus,
          GstMessage *msg,
          gpointer    data)
{
  GMainLoop *loop = (GMainLoop *) data;

  switch (GST_MESSAGE_TYPE (msg)) {

    case GST_MESSAGE_EOS:
      g_print ("End of stream\n");
      g_main_loop_quit (loop);
      break;

    case GST_MESSAGE_ERROR: {
      gchar  *debug;
      GError *error;

      gst_message_parse_error (msg, &error, &debug);
      g_free (debug);

      g_printerr ("Error: %s\n", error->message);
      g_error_free (error);

      g_main_loop_quit (loop);
      break;
    }
    default:
      break;
  }

  return TRUE;
}

static void
on_pad_added (GstElement *element,
              GstPad     *pad,
              gpointer    data)
{
  GstPad *sinkpad;
  GstElement *decoder = (GstElement *) data;

  /* We can now link this pad with the vorbis-decoder sink pad */
  g_print ("Dynamic pad created, linking demuxer/decoder\n");

  sinkpad = gst_element_get_static_pad (decoder, "sink");

  gst_pad_link (pad, sinkpad);

  gst_object_unref (sinkpad);
}



int
main (int   argc,
      char *argv[])
{
  GMainLoop *loop;

  GstElement *pipeline, *source, *demuxer, *decoder, *conv, *jpg, *sink;
  GstBus *bus;
  guint bus_watch_id;

  /* Initialisation */
  gst_init (&argc, &argv);

  loop = g_main_loop_new (NULL, FALSE);


  /* Check input arguments */
  // if (argc != 2) {
  //   g_printerr ("Usage: %s <Ogg/Vorbis filename>\n", argv[0]);
  //   return -1;
  // }


  /* Create gstreamer elements */
  pipeline = gst_pipeline_new ("audio-player");
  // decodebin ! queue ! autovideoconvert ! pngenc ! multifilesink 
  source   = gst_element_factory_make ("filesrc",       "file-source");
  demuxer  = gst_element_factory_make ("decodebin",      "ogg-demuxer");
  decoder  = gst_element_factory_make ("queue",     "vorbis-decoder");
  conv     = gst_element_factory_make ("autovideoconvert",  "converter");
  jpg      = gst_element_factory_make ("jpegenc","jpg");
  sink     = gst_element_factory_make ("multifilesink", "audio-output");

  if (!pipeline || !source || !demuxer || !decoder || !conv || !jpg || !sink) {
    g_printerr ("One element could not be created. Exiting.\n");
    return -1;
  }

  /* Set up the pipeline */

  /* we set the input filename to the source element */
  g_object_set (G_OBJECT (source), "location", "/home/echo/TEST/test/output.mp4", argv[1], NULL);
  g_object_set (G_OBJECT (sink), "location", "/home/echo/TEST/test/pig_pictures/pig_pictures3/frame%d.jpg", argv[1], NULL);

  /* we add a message handler */
  bus = gst_pipeline_get_bus (GST_PIPELINE (pipeline));
  bus_watch_id = gst_bus_add_watch (bus, bus_call, loop);
  gst_object_unref (bus);

  /* we add all elements into the pipeline */
  /* file-source | ogg-demuxer | vorbis-decoder | converter | alsa-output */
  gst_bin_add_many (GST_BIN (pipeline),
                    source, demuxer, decoder, conv, jpg, sink, NULL);

  /* we link the elements together */
  /* file-source -> ogg-demuxer ~> vorbis-decoder -> converter -> alsa-output */
  gst_element_link (source, demuxer);
  gst_element_link_many (decoder, conv, jpg, sink, NULL);
  g_signal_connect (demuxer, "pad-added", G_CALLBACK (on_pad_added), decoder);

  /* note that the demuxer will be linked to the decoder dynamically.
     The reason is that Ogg may contain various streams (for example
     audio and video). The source pad(s) will be created at run time,
     by the demuxer when it detects the amount and nature of streams.
     Therefore we connect a callback function which will be executed
     when the "pad-added" is emitted.*/


  /* Set the pipeline to "playing" state*/
  g_print ("Now playing: %s\n", argv[1]);
  gst_element_set_state (pipeline, GST_STATE_PLAYING);


  /* Iterate */
  g_print ("Running...\n");
  g_main_loop_run (loop);


  /* Out of the main loop, clean up nicely */
  g_print ("Returned, stopping playback\n");
  gst_element_set_state (pipeline, GST_STATE_NULL);

  g_print ("Deleting pipeline\n");
  gst_object_unref (GST_OBJECT (pipeline));
  g_source_remove (bus_watch_id);
  g_main_loop_unref (loop);
  return 0;
}
```
其中
g_object_set (G_OBJECT (source), “location”, “视频文件存放地址”, argv[1], NULL);

g_object_set (G_OBJECT (sink), “location”, “图片存放地址”, argv[1], NULL);

然后输入命令：
```
gcc split.c -o a.out `pkg-config --cflags --libs gstreamer-1.0` 
./a.out
```
### 将摄像头内容切割为jpg图片
1. 命令行形式
```
gst-launch-1.0 v4l2src device=/dev/video0 ! queue ! videorate ! video/x-raw,framerate=25/1 ! jpegenc ! multifilesink location="pictures/frame%06d.jpg"
```
2. 源文件形式
不会（sad。。。。。。

## 将图片合成视频
### 将jpg图片合成为ogg视频
1. 命令行形式
```
gst-launch-1.0 multifilesrc location="pig_pictures/pig_pictures3/frame%d.jpg" caps="image/jpeg,framerate=25/1" ! jpegdec ! theoraenc ! oggmux ! filesink location="pig_pictures/out.ogg"
```
2. 源文件形式
不会（5555555

### 编译.c文件和python文件的命令
```
gcc -I/usr/include/python2.7/ basic-tutorial-1.c -o basic-tutorial-1 `pkg-config --cflags --libs gstreamer-1.0` -L/usr/lib/ -lpython2.7
```
