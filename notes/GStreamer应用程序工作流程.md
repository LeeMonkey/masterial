# GStreamer  

[toc]

***

## 1 入门例程

### 1.1 代码展示

```cpp
#include <gst/gst.h>

int main(int argc, char *argv[])
{
    GstElement *pipeline;
    GstBus *bus;
    GstMessage *msg;

    // 1.初始化GStreamer
     gst_init(&argc, &argv);

    // 2.创建Pipeline
    pipeline = gst_parse_launch("playbin uri=https://www.freedesktop.org/software/gstreamer-sdk/data/media/sintel_trailer-480p.webm", NULL);

    // 3.开始播放
    gst_element_set_state(pipeline, GST_STATE_PLAYING);

    // 4.等待直到error或者eos
    bus = gst_element_gt_bus(pipeline);
    msg = gst_bus_timed_pop_filtered(bus, GST_CLOCK_TIME_NONE,
        GST_MESSAGE_ERROR | GST_MESSAGE_EOS);

    // 5.释放资源
    if(msg != NULL)
        gst_message_unref(msg);
    gst_object_unref(bus);
    gst_element_set_state(pipeline, GST_STATE_NULL);
    gst_object_unref(pipeline);
    return 0;
}
```

### 1.2 源码分析

#### 1) GStreamer初始化

```cpp
gst_init(&argc, &argv);
```

首先我们调用了gstreamer的初始化函数，该初始化函数必须在其他gstreamer接口之前被调用，gst_init会负责以下资源的初始化：

- 初始化GStreamer库
- 注册内部element
- 加载插件列表，扫描列表中及相应路径下的插件
- 解析并执行命令行参数  

在不需要gst_init处理命令行参数时，我们可以讲NULL作为其参数，例如：gst_init(NULL, NULL);

#### 2) 创建Pipeline

```cpp
pipeline = gst_parse_launch ("playbin uri=https://www.freedesktop.org/software/gstreamer-sdk/data/media/sintel_trailer-480p.webm", NULL);
```

##### gst_parse_launch

在基本介绍中我们了解了Pipeline的概念，在pipeline中，首先通过"source" element获取媒体数据，然后通过一个或多个element对编码数据进行解码，最后通过"sink" element输出声音和画面。通常在创建较复杂的pipeline时，我们需要通过gst_element_factory_make来创建element，然后将其加入到GStreamer Bin中，并连接起来。当pipeline比较简单并且我们不需要对pipeline中的element进行过多的控制时，我们可以采用gst_parse_launch 来简化pipeline的创建。  
这个函数能够巧妙的将pipeline的文本描述转化为pipeline对象，我们也经常需要通过文本方式构建pipeline来查看GStreamer是否支持相应的功能，因此GStreamer提供了gst-launch-1.0命令行工具，极大的方便了pipeline的测试。

##### playbin

我们知道pipeline中需要添加特定的element以实现相应的功能，在本例中，我们通过gst_parse_launch创建了只包含一个element的Pipeline。  
我们刚提到pipeline需要有"source"、"sink" element，为什么这里只需要一个playbin就够了呢？是因为**playbin element内部会根据文件的类型自动去查找所需要的"source"，"decoder"，"sink"并将它们连接起来，同时提供了部分接口用于控制pipeline中相应的element。**  
在playbin后，我们跟了一个uri参数，指定了我们想要播放的媒体文件地址，**playbin会根据uri所使用的协议（"https://"，"ftp://"，"file://"等）自动选择合适的source element**此例中通过https方式）获取数据。  

#### 3) 设置播放状态

```cpp
// Start playing
gst_element_set_state (pipeline, GST_STATE_PLAYING);
```

这一行代码引入了一个新的概念“状态”（state）。每个GStreamer element都有相应都状态，我们目前可以简单的把状态与播放器的播放/暂停按钮联系起来，只有当状态处于PLAYING时，pipeline才会播放/处理数据。
**这里gst_element_set_state通过pipeline，将playbin的状态设置为PLAYING**，使playbin开始播放视频文件。

#### 4) 等待播放结束

```cpp
// Wait until error or EOS
bus = gst_element_get_bus (pipeline);
msg = gst_bus_timed_pop_filtered (bus, GST_CLOCK_TIME_NONE, GST_MESSAGE_ERROR | GST_MESSAGE_EOS);
```

这几行会等待pipeline播放结束或者播放出错。我们知道GStreamer框架会通过bus，将所发生的事件通知到应用程序，因此，这里首先取得pipeline的bus对象，通过gst_bus_timed_pop_filtered 以同步的方式等待bus上的ERROR或EOS（End of Stream）消息，该函数收到消息后才会返回。
我们会在下一篇文章中继续介绍消息相关的内容。
到目前为止，GStreamer会处理视频播放的所有工作（数据获取，解码，音视频同步，输出）。当到达文件末端（EOS）或出错（直接关闭播放窗口，断开网络）时，播放会自动停止。我们也可以在终端通过ctrl+c中断程序的执行。

#### 5) 释放资源

```cpp
// Free resources
if (msg != NULL)
  gst_message_unref (msg);

gst_object_unref (bus);
gst_element_set_state (pipeline, GST_STATE_NULL);
gst_object_unref (pipeline);
```

这里我们将不再使用的msg和bus对象进行销毁，并将pipeline状态设置为NULL（在NULL状态时GStreamer会释放为pipeline分配的所有资源），最后销毁pipeline对象。由于GStreamer是继承自GObject，所以需要通过gst_object_unref 来减少引用计数，当对象的引用计数为0时，函数内部会自动释放为其分配的内存。
不同接口会对返回的对象进行不同的处理，我们需要详细的阅读API文档，来决定我们是否需要对返回的对象进行释放。

***

## 2 基本概念

### 2.1 Element

根据功能将Element分为三类：

#### 1) source element

只能生成数据不能用于接受数据的element，例如用于文件读取的filesrc等。
<center><img src="https://img2018.cnblogs.com/blog/1647252/201906/1647252-20190617144041322-1759670774.png" /></center>

#### 2) sink element

只能接收数据，不能产生数据的element，例如用于播放声音的alsasink等。
<center><img src="https://img2018.cnblogs.com/blog/1647252/201906/1647252-20190617144107949-2019852860.png" /></center>

#### 3) filter-like element

既能接收数据，又能生成数据的element称为filter-like element，例如分离器，解码器，音量控制器。
<center><img src="https://img2018.cnblogs.com/blog/1647252/201906/1647252-20190617144135498-747765959.png" /></center>

这些element可能包含多个src pad，也可能包含多个sink pad，例如mp4的demuxer(qtdemux)会将mp4文件中的音频和视频分离到audio src pad和video src pad。而mp4的muxer(mp4mux)则相反，会将audio sink pad和video sink pad的数据合并到一个src pad，再经其他element将数据写入文件或发送到网络。

#### 4) 连接element

当我们有很多element时，我们需要将他们按照数据的串数路径串联起来，src pad只能连接到sink pad，这样才能够实现相应的功能。
<center><img src="https://img2018.cnblogs.com/blog/1647252/201906/1647252-20190617144244135-1609912147.png" /></center>

### 2.2 Bin和Pipeline

Element，Bin和Pipeline之间的继承关系：

```cpp
GObject
    ╰──GInitiallyUnowned
        ╰──GstObject
            ╰──GstElement
                ╰──GstBin
                    ╰──GstPipeline
```

创建多个element后，需要对element进行状态/资源管理，如果每次状态改变时，都需要依次去操作每个element，所以需要用**bin作为一个容器**，将多个element添加到bin，当操作bin时，bin会将相应的操作转发到内部所有的element中，我们可以将bin认为是一个新的逻辑element，由bin来管理其内部element的状态及资源，同时转发其产生的消息。
<center><img src="https://img2018.cnblogs.com/blog/1647252/201906/1647252-20190617144354342-1162294711.png" /></center>

Bin实现了容器的功能，那pipeline又有什么功能呢？  
在多媒体应用中，音视频同步是一个基本的功能，需要支持这样的功能，所有的element必须要有一个相同的时钟，这样才能保证各个音频和视频在同一时间输出。**pipeline就会为其内部所有的element选择一个相同的时钟**，同时还为应用提供了bus系统，用于消息的接收。

### 2.3 Bus (类似于队列)

pipeline会提供一个Bus，这个pipeline上所有的element都可以用这个bus向应用程序发送消息。Bus主要是为了解决多线程之间消息处理的问题。由于GStreamer内部可能会创建多个线程，如果没有Bus，应用程序可能同时收到从多个线程的消息，如果应用程序再发送线程中通过回调去处理消息，应用程序可能阻塞播放线程，造成播放卡顿，死锁等其他问题。为了解决这类问题，GStreamer通常是将多个线程的消息发送到Bus系统，由应用程序从bus中取出消息，然后进行处理。**Bus在这里扮演了消息队列的角色**，通过bus解耦了GStreamer框架和应用程序对消息的处理，降低了应用程序的复杂度。

### 2.4 源码分析

element创建和使用示例：

```cpp
#include <gst/gst.h>

int main (int argc, char *argv[])
{
  GstElement *pipeline, *source, *filter, *sink;
  GstBus *bus;
  GstMessage *msg;
  GstStateChangeReturn ret;

  // 1. Initialize GStreamer
  gst_init (&argc, &argv);

  // 2. Create the elements
  source = gst_element_factory_make ("videotestsrc", "source");
  filter = gst_element_factory_make ("timeoverlay", "filter");
  sink = gst_element_factory_make ("autovideosink", "sink");

  // 3. Create the empty pipeline
  pipeline = gst_pipeline_new ("test-pipeline");

  if (!pipeline || !source || !filter || !sink) {
    g_printerr ("Not all elements could be created.\n");
    return -1;
  }

  // 4. Build the pipeline
  gst_bin_add_many (GST_BIN (pipeline), source, filter, sink, NULL);
  if (gst_element_link_many (source, filter, sink, NULL) != TRUE) {
    g_printerr ("Elements could not be linked.\n");
    gst_object_unref (pipeline);
    return -1;
  }

  // 5. Modify the source's properties
  g_object_set (source, "pattern", 0, NULL);

  // 6. Start playing
  ret = gst_element_set_state (pipeline, GST_STATE_PLAYING);
  if (ret == GST_STATE_CHANGE_FAILURE) {
    g_printerr ("Unable to set the pipeline to the playing state.\n");
    gst_object_unref (pipeline);
    return -1;
  }

  // 7. Wait until error or EOS
  bus = gst_element_get_bus (pipeline);
  msg =
      gst_bus_timed_pop_filtered (bus, GST_CLOCK_TIME_NONE,
      GST_MESSAGE_ERROR | GST_MESSAGE_EOS);

  // 8. Parse message
  if (msg != NULL) {
    GError *err;
    gchar *debug_info;

    switch (GST_MESSAGE_TYPE (msg)) {
      case GST_MESSAGE_ERROR:
        gst_message_parse_error (msg, &err, &debug_info);
        g_printerr ("Error received from element %s: %s\n",
            GST_OBJECT_NAME (msg->src), err->message);
        g_printerr ("Debugging information: %s\n",
            debug_info ? debug_info : "none");
        g_clear_error (&err);
        g_free (debug_info);
        break;
      case GST_MESSAGE_EOS:
        g_print ("End-Of-Stream reached.\n");
        break;
      default:
        // We should not reach here because we only asked for ERRORs and EOS
        g_printerr ("Unexpected message received.\n");
        break;
    }
    gst_message_unref (msg);
  }

  // 9. Free resources
  gst_object_unref (bus);
  gst_element_set_state (pipeline, GST_STATE_NULL);
  gst_object_unref (pipeline);
  return 0;
}
```

#### 1) 创建Element

```cpp
// Create the elements
source = gst_element_factory_make ("videotestsrc", "source");
filter = gst_element_factory_make ("timeoverlay", "filter");
sink = gst_element_factory_make ("autovideosink", "sink");
```

在对GStreamer进行初始化后，**我们可以通过gst_element_factory_make创建element**。**第一个参数是element的类型**，可以通过这个字符串，找到对应的类型，从而创建element对象。**第二个参数指定了创建element的名字**，当我们没有保存创建element的对象指针时，我们可以通过gst_bin_get_by_name从pipeline中取得该element的对象指针。如果第二个参数为NULL，则GStreamer内部会为该element自动生成一个唯一的名字。
我们在当前示例中创建了3个element：videotestsrc，timeoverlay，autovideosink，其作用分别为：

- videotestsrc是一个source element，用于产生视频数据，通常用于调试。

- timeoverlay是一个filter-like element，可以在视频数据中叠加一个时间字符串。

- autovideosink上一个sink element，用于自动选择视频输出设备，创建视频显示窗口，并显示其收到的数据。

#### 2) 创建Pipeline

```cpp
pipeline = gst_pipeline_new("test-pipeline");
```

Pipeline通过gst_pipeline_new创建，参数为pipeline的名字。

pipeline提供播放所必须的时钟以及消息处理，所以应把我们创建的element添加到pipeline中。

```cpp
// Build the pipeline
gst_bin_add_many (GST_BIN (pipeline), source, filter, sink, NULL);
if (gst_element_link_many (source, filter, sink, NULL) != TRUE) {
  g_printerr ("Elements could not be linked.\n");
  gst_object_unref (pipeline);
  return -1;
}
```

pipeline继承自bin，所有bin的方法适用于pipeline，通过宏GST_BIN将子类转换成父类，宏内部会对其类型进行检查。使用gst_bin_add_many将多个element添加到pipeline中，这个函数接受任意多个参数，最后用NULL表示参数列表结束，如果一次加入一个可以用gst_bin_add函数。
多个element通过gst_element_link_many连接，gst_element_link_many会根据参数顺序一次将element连接起来。
**注意**：只用同一个bin/pipeline才能够连接在一起，在连接前需要将所需的element加入到bin/pipeline中。

test-pipeline可以表示成：

<center><img src="https://img2018.cnblogs.com/blog/1647252/201906/1647252-20190617144639387-1437925669.png"/></center>

#### 3) 设置element属性

```cpp
// Modify the source's properties
g_object_set(source, "pattern", 0, NULL);
```

大部分的element都有自己的属性。**只读属性常用于查询element的状态。可修改属性常用于控制element的行为**。

由于GstElement继承于GObject，同时GObject对象系统提供了 g_object_get()用于读取属性，g_object_set()用于修改属性，g_object_set()支持以NULL结束的属性-值的键值对，所以可以一次修改element的多个属性。

我们这里通过g_object_set()来修改videotestsrc的pattern属性。pattern属性可以控制测试图像的类型，可以尝试将0修改为1，查看输出结果有何不同。

我们可以通过gst-inspect-1.0 videotestsrc命令来查看pattern所支持的所有值。

#### 4) 设置播放装填

```cpp
// Start playing
  ret = gst_element_set_state (pipeline, GST_STATE_PLAYING);
  if (ret == GST_STATE_CHANGE_FAILURE) {
    g_printerr ("Unable to set the pipeline to the playing state.\n");
    gst_object_unref (pipeline);
    return -1;
  }
```

在完成pipeline的创建以及属性的修改后，我们将pipeline的状态设置为PLAYING，这里与上一文章中的示例相同，只增加来错误处理，其他的返回值处理将在后续章节讲述。

***

## 3 媒体类型与Pad

### 3.1 Pad

我们知道，pad是element之间的数据的接口，一个src pad只能与一个sink pad相连。每个element可以通过pad过滤数据，接收自己支持的数据类型。Pad通过Pad Capabilities（简称为Pad Caps）来描述支持的数据类型。例如：

- 表示分辨率为300x200，帧率为30fps的RGB视频的Caps：  
  "video/x-raw,format=RGB,width=300,height=200,framerate=30/1"

- 表示采样位宽为16位，采样率44.1kHz，双通道PCM音频的Caps：
  "audio/x-raw,format=S16LE,rate=44100,channels=2"

- 或者直接描述编码数据格式Voribis，VP8："audio/x-vorbis" "video/x-vp8"

　　一个Pad可以支持多种类型的Caps（比如一个video sink可以同时支持RGB或YUV格式的数据），同时可以指定Caps支持的数据范围（比如一个audio sink可以支持1~48k的采样率）。但是，在一个Pipeline中，Pad之间所传输的数据类型必须是唯一的。GStreamer在进行element连接时，会通过协商（negotiation）的方式选择一个双方都支持的类型。

　　因此，为了能使两个Element能够正确的连接，双方的Pad Caps之间必须有交集，从而在协商阶段选择相同的数据类型，这就是Pad Caps的主要作用。在实际使用中，我们可以通过gst-inspect工具查看Element所支持的Pad Caps，从而才能知道在连接出错时如何处理。

#### 1) Pad Templates(模板)

我们曾使用gst_element_factory_make()接口创建Element，这个接口内部也会先创建一个Element 工厂，再通过工厂方法创建一个Element。由于大部分Element都需要创建类似的Pad，于是GStreame定义了Pad Template，Pad Template被包含中Element工厂中，在创建Element时，用于快速创建Pad。  

　　Pad Template包含了一个Pad所能支持的所有Caps。通过Pad Template，我们可以快速的判断两个pad是否能够连接（比如两个elements都只提供了sink template，这样的element之间是无法连接的，这样就没必要进一步判断Pad Caps）。

　　由于Pad Template属于Element工厂，所以我们可以直接使用gst-inspect查看其属性，但Element实际的Pad会根据Element所处的不同状态来进行实例化，具体的Pad Caps会在协商后才会被确定。

#### 2) Pad Availability(有效性)

上面的例子中显示的Pad Template都是一直存在的（Availability: Always），创建的Pad也是一直有效的。但有些Element会根据输入数据以及后续的Element动态增加或删除Pad，因此GStreamer提供了3种Pad有效性的状态：Always，Sometimes，On request。

##### Always Pad

在element被初始化后就存在的Pad，被称之为always pad或static pad。

##### Sometimes Pad

根据输入数据的不同而产生的pad，被称为sometimes pad，常见于各种文件格式解析器。例如用于解析mp4文件的qtdemux："gst-inspect-1.0 qtdemux"

```shell
Pad Templates:
  SINK template: 'sink'
    Availability: Always
    Capabilities:
      video/quicktime
      video/mj2
      audio/x-m4a
      application/x-3gp

  SRC template: 'video_%u'
    Availability: Sometimes
    Capabilities:
      ANY

  SRC template: 'audio_%u'
    Availability: Sometimes
    Capabilities:
      ANY

  SRC template: 'subtitle_%u'
    Availability: Sometimes
    Capabilities:
      ANY
```

##### Request Pad

按需创建的pad被称为request pad，常见于合并或生成多路数据。例如,用于1到N转换的tee："gst-inspect-1.0 tee"

```shell
Pad Templates:
  ...
  SRC template: 'src_%u'
    Availability: On request
      Has request_new_pad() function: gst_tee_request_new_pad
    Capabilities:
      ANY
```

当我们需要将同一路视频流同时进行显示和存储，这时候我们就需要用到tee，在创建tee element的时候，我们不知道pipeline需要多少个src pad，需要后续element来请求一个src pad。

### 3.2 示例代码

GStreamer提供了gst-inspect工具来查看element所提供的Pad Templates，但无法查看element在不同状态时其Pad所支持的数据类型，通过下面的代码，我们可以看到Pad Caps在不同状态下的变化。

```cpp
#include <gst/gst.h>

// Functions below print the Capabilities in a human-friendly format 
static gboolean print_field (GQuark field, const GValue * value, gpointer pfx) {
  gchar *str = gst_value_serialize (value);

  g_print ("%s  %15s: %s\n", (gchar *) pfx, g_quark_to_string (field), str);
  g_free (str);
  return TRUE;
}

static void print_caps (const GstCaps * caps, const gchar * pfx) {
  guint i;

  g_return_if_fail (caps != NULL);

  if (gst_caps_is_any (caps)) {
    g_print ("%sANY\n", pfx);
    return;
  }
  if (gst_caps_is_empty (caps)) {
    g_print ("%sEMPTY\n", pfx);
    return;
  }

  for (i = 0; i < gst_caps_get_size (caps); i++) {
    GstStructure *structure = gst_caps_get_structure (caps, i);

    g_print ("%s%s\n", pfx, gst_structure_get_name (structure));
    gst_structure_foreach (structure, print_field, (gpointer) pfx);
  }
}

// Prints information about a Pad Template, including its Capabilities 
static void print_pad_templates_information (GstElementFactory * factory) {
  const GList *pads;
  GstStaticPadTemplate *padtemplate;

  g_print ("Pad Templates for %s:\n", gst_element_factory_get_longname (factory));
  if (!gst_element_factory_get_num_pad_templates (factory)) {
    g_print ("  none\n");
    return;
  }

  pads = gst_element_factory_get_static_pad_templates (factory);
  while (pads) {
    padtemplate = pads->data;
    pads = g_list_next (pads);

    if (padtemplate->direction == GST_PAD_SRC)
      g_print ("  SRC template: '%s'\n", padtemplate->name_template);
    else if (padtemplate->direction == GST_PAD_SINK)
      g_print ("  SINK template: '%s'\n", padtemplate->name_template);
    else
      g_print ("  UNKNOWN!!! template: '%s'\n", padtemplate->name_template);

    if (padtemplate->presence == GST_PAD_ALWAYS)
      g_print ("    Availability: Always\n");
    else if (padtemplate->presence == GST_PAD_SOMETIMES)
      g_print ("    Availability: Sometimes\n");
    else if (padtemplate->presence == GST_PAD_REQUEST)
      g_print ("    Availability: On request\n");
    else
      g_print ("    Availability: UNKNOWN!!!\n");

    if (padtemplate->static_caps.string) {
      GstCaps *caps;
      g_print ("    Capabilities:\n");
      caps = gst_static_caps_get (&padtemplate->static_caps);
      print_caps (caps, "      ");
      gst_caps_unref (caps);

    }

    g_print ("\n");
  }
}

// Shows the CURRENT capabilities of the requested pad in the given element 
static void print_pad_capabilities (GstElement *element, gchar *pad_name) {
  GstPad *pad = NULL;
  GstCaps *caps = NULL;

  // Retrieve pad 
  pad = gst_element_get_static_pad (element, pad_name);
  if (!pad) {
    g_printerr ("Could not retrieve pad '%s'\n", pad_name);
    return;
  }

  // Retrieve negotiated caps (or acceptable caps if negotiation is not finished yet) 
  caps = gst_pad_get_current_caps (pad);
  if (!caps)
    caps = gst_pad_query_caps (pad, NULL);

  // Print and free 
  g_print ("Caps for the %s pad:\n", pad_name);
  print_caps (caps, "      ");
  gst_caps_unref (caps);
  gst_object_unref (pad);
}

int main(int argc, char *argv[]) {
  GstElement *pipeline, *source, *sink;
  GstElementFactory *source_factory, *sink_factory;
  GstBus *bus;
  GstMessage *msg;
  GstStateChangeReturn ret;
  gboolean terminate = FALSE;

  // Initialize GStreamer 
  gst_init (&argc, &argv);

  // Create the element factories 
  source_factory = gst_element_factory_find ("audiotestsrc");
  sink_factory = gst_element_factory_find ("autoaudiosink");
  if (!source_factory || !sink_factory) {
    g_printerr ("Not all element factories could be created.\n");
    return -1;
  }

  // Print information about the pad templates of these factories 
  print_pad_templates_information (source_factory);
  print_pad_templates_information (sink_factory);

  // Ask the factories to instantiate actual elements 
  source = gst_element_factory_create (source_factory, "source");
  sink = gst_element_factory_create (sink_factory, "sink");

  // Create the empty pipeline 
  pipeline = gst_pipeline_new ("test-pipeline");

  if (!pipeline || !source || !sink) {
    g_printerr ("Not all elements could be created.\n");
    return -1;
  }

  // Build the pipeline 
  gst_bin_add_many (GST_BIN (pipeline), source, sink, NULL);
  if (gst_element_link (source, sink) != TRUE) {
    g_printerr ("Elements could not be linked.\n");
    gst_object_unref (pipeline);
    return -1;
  }

  // Print initial negotiated caps (in NULL state) 
  g_print ("In NULL state:\n");
  print_pad_capabilities (sink, "sink");

  // Start playing 
  ret = gst_element_set_state (pipeline, GST_STATE_PLAYING);
  if (ret == GST_STATE_CHANGE_FAILURE) {
    g_printerr ("Unable to set the pipeline to the playing state (check the bus for error messages).\n");
  }

  // Wait until error, EOS or State Change 
  bus = gst_element_get_bus (pipeline);
  do {
    msg = gst_bus_timed_pop_filtered (bus, GST_CLOCK_TIME_NONE, GST_MESSAGE_ERROR | GST_MESSAGE_EOS |
        GST_MESSAGE_STATE_CHANGED);

    // Parse message 
    if (msg != NULL) {
      GError *err;
      gchar *debug_info;

      switch (GST_MESSAGE_TYPE (msg)) {
        case GST_MESSAGE_ERROR:
          gst_message_parse_error (msg, &err, &debug_info);
          g_printerr ("Error received from element %s: %s\n", GST_OBJECT_NAME (msg->src), err->message);
          g_printerr ("Debugging information: %s\n", debug_info ? debug_info : "none");
          g_clear_error (&err);
          g_free (debug_info);
          terminate = TRUE;
          break;
        case GST_MESSAGE_EOS:
          g_print ("End-Of-Stream reached.\n");
          terminate = TRUE;
          break;
        case GST_MESSAGE_STATE_CHANGED:
          // We are only interested in state-changed messages from the pipeline 
          if (GST_MESSAGE_SRC (msg) == GST_OBJECT (pipeline)) {
            GstState old_state, new_state, pending_state;
            gst_message_parse_state_changed (msg, &old_state, &new_state, &pending_state);
            g_print ("\nPipeline state changed from %s to %s:\n",
                gst_element_state_get_name (old_state), gst_element_state_get_name (new_state));
            // Print the current capabilities of the sink element 
            print_pad_capabilities (sink, "sink");
          }
          break;
        default:
          // We should not reach here because we only asked for ERRORs, EOS and STATE_CHANGED 
          g_printerr ("Unexpected message received.\n");
          break;
      }
      gst_message_unref (msg);
    }
  } while (!terminate);

  // Free resources 
  gst_object_unref (bus);
  gst_element_set_state (pipeline, GST_STATE_NULL);
  gst_object_unref (pipeline);
  gst_object_unref (source_factory);
  gst_object_unref (sink_factory);
  return 0;
}
```

#### 1) 源码分析

##### 输出可读信息

print_field, print_caps和print_pad_templates_information实现类似功能，打印GStreamer的数据结构，可以查看相应GStreamer GstCaps 接口了解更多信息。

```cpp
// Shows the CURRENT capabilities of the requested pad in the given element
static void print_pad_capabilities(GstElement *element, gchar *pad_name) {
  GstPad *pad = NULL;
  GstCaps *caps = NULL;

  // Retrieve pad
  pad = gst_element_get_static_pad(element, pad_name);
  if (!pad){
    g_printerr("Could not retrieve pad '%s'\n", pad_name);
    return;
  }

  // Retrieve negotiated caps (or acceptable caps if negotiation is not finished yet)
  caps = gst_pad_get_current_caps(pad);
  if (!caps)
    caps = gst_pad_query_caps (pad, NULL);

  // Print and free
  g_print ("Caps for the %s pad:\n", pad_name);
  print_caps (caps, "      ");
  gst_caps_unref (caps);
  gst_object_unref (pad);
}
```

因为我们使用的source和sink都具有static（always）pad，所以这里使用gst_element_get_static_pad()获取Pad， 其他情况可以使用gst_element_foreach_pad()或gst_element_iterate_pads()获取动态创建的Pad。
接着使用gst_pad_get_current_caps()获取pad当前的caps，根据不同的element状态会有不同的结果，甚至可能不存在caps。如果没有，我们通过gst_pad_query_caps()获取当前可以支持的caps，当element处于NULL状态时，这个caps为Pad Template所支持的caps，其值可随状态变化而变化。

##### 获取Element工厂

```cpp
// Create the element factories 
source_factory = gst_element_factory_find ("audiotestsrc");
sink_factory = gst_element_factory_find ("autoaudiosink");
if (!source_factory || !sink_factory) {
  g_printerr ("Not all element factories could be created.\n");
  return -1;
}

// Print information about the pad templates of these factories 
print_pad_templates_information (source_factory);
print_pad_templates_information (sink_factory);

// Ask the factories to instantiate actual elements 
source = gst_element_factory_create (source_factory, "source");
sink = gst_element_factory_create (sink_factory, "sink");
```

##### 处理State-Change消息

Pipeline的创建过程与其他示例相同，此例新增了状态变化的处理。

```cpp
case GST_MESSAGE_STATE_CHANGED:
  // We are only interested in state-changed messages from the pipeline 
  if (GST_MESSAGE_SRC(msg) == GST_OBJECT(pipeline)) {
    GstState old_state, new_state, pending_state;
    gst_message_parse_state_changed(msg, &old_state, &new_state, &pending_state);
    g_print("\nPipeline state changed from %s to %s:\n",
        gst_element_state_get_name (old_state), gst_element_state_get_name (new_state));
    // Print the current capabilities of the sink element 
    print_pad_capabilities(sink, "sink");
  }
  break;
```

因为我们在gst_bus_timed_pop_filtered()中加入了GST_MESSAGE_STATE_CHANGED，所以我们会收到状态变化的消息。在状态变化时，输出sink element的pad caps中当前状态的信息。

接着使用gst_pad_get_current_caps()获取pad当前的caps，根据不同的element状态会有不同的结果，甚至可能不存在caps。如果没有，我们通过gst_pad_query_caps()获取当前可以支持的caps，当element处于NULL状态时，这个caps为Pad Template所支持的caps，其值可随状态变化而变化。

#### 2) 输出分析

```shell
Pad Templates for Audio test source:
  SRC template: 'src'
    Availability: Always
    Capabilities:
      audio/x-raw
                 format: { S16LE, S32LE, F32LE, F64LE }
                 layout: interleaved
                   rate: [ 1, 2147483647 ]
               channels: [ 1, 2 ]

Pad Templates for Auto audio sink:
  SINK template: 'sink'
    Availability: Always
    Capabilities:
      ANY
```

首先是"audiotestsrc"和"autoaudiosink"的pad templates信息，这个与gst-inspect的输出相同。

```shell
In NULL state:
Caps for the sink pad:
    ANY
```

NULL状态为Element的初始化状态，此时，"autoaudiosink"的sink pad caps与Pad Template相同，支持所有的格式。

```shell
Pipeline state changed from NULL to READY:
Caps for the sink pad:
      audio/x-raw
                 format: { S16LE, S16BE, F32LE, F32BE, S32LE, S32BE, S24LE, S24BE, S24_32LE, S24_32BE, U8 }
                 layout: interleaved
                   rate: [ 1, 2147483647 ]
               channels: [ 1, 32 ]
      audio/x-alaw
                   rate: [ 1, 2147483647 ]
               channels: [ 1, 32 ]
      audio/x-mulaw
                   rate: [ 1, 2147483647 ]
               channels: [ 1, 32 ]
```

状态从NULL转到READY时，GStreamer会获取音频输出设备所支持的所有类型，这里可以看到sink pad caps列出了输出设备所能支持的类型。

```shell
Pipeline state changed from READY to PAUSED:
Caps for the sink pad:
      audio/x-raw
                 format: S16LE
                 layout: interleaved
                   rate: 44100
               channels: 1

Pipeline state changed from PAUSED to PLAYING:
Caps for the sink pad:
      audio/x-raw
                 format: S16LE
                 layout: interleaved
                   rate: 44100
               channels: 1
```

状态从READY转到PAUSED时，GStreamer会协商一个所有element都支持的类型。当进入PLAYING状态时，sink会采用协商后的类型进行数据传输。

***

## 4 媒体类型与Pad

在以前的文章中，我们了解到了2种播放文件的方式：一种是在知道了文件的类型及编码方式后，手动创建所需Element并构造Pipeline；另一种是直接使用playbin，由playbin内部动态创建所需Element并连接Pipeline。很明显使用playbin的方式更加灵活，我们不需要在一开始就创建各种Pipeline，只需由playbin内部根据文件类型，自动构造Pipeline。 在了解了Pad的作用后，本文通过一个例子来了解如何通过Pad事件动态的连接Pipeline，为了解playbin内部是如何动态创建Pipeline打下基础。

### 4.1 动态连接Pipeline

在本章的例子中，我们在将Pipeline设置为PLAYING状态之前，不会将所有的Element都连接起来，这种处理方式是可以的，但需要额外的处理。如果在设置PLAYING状态后不做任何操作，数据无法到达Sink，Pipeline会直接抛出一个错误并退出。如果在收到相应事件后，对其进行处理，并将Pipeline连接起来，Pipeline就可以正常工作。

我们常见的媒体，音频和视频都是通过某一种容器格式被包含在同一个文件中。播放时，我们需要将音视频数据分离出来，通常将具备这种功能的模块称为分离器（demuxer）。

GStreamer针对常见的容器提供了相应的demuxer，如果一个容器文件中包含多种媒体数据（例如：一路视频，两路音频），这种情况下，demuxer会为些数据分别创建不同的Source Pad，每一个Source Pad可以被认为一个处理分支，可以创建多个分支分别处理相应的数据。

gst-launch-1.0 filesrc location=sintel_trailer-480p.ogv ! oggdemux name=demux ! queue ! vorbisdec ! autoaudiosink demux. ! queue ! theoradec ! videoconvert ! autovideosink

通过上面的命令播放文件时，会创建具有2个分支的Pipeline：

<center><img src="https://img2018.cnblogs.com/blog/1647252/201905/1647252-20190530112147332-1498662949.png"/></center>

使用demuxer需要注意的一点是：demuxer只有在收到足够的数据时才能确定容器中包含哪些媒体信息，因此demuxer开始没有Source Pad，所以其他的Element无法在Pipeline创建时就连接到demuxer。
**解决这种问题的办法是**：在创建Pipeline时，我们只将Source Element到demuxer之间的Elements连接好，然后设置Pipeline状态为PLAYING，当demuxer收到足够的数据可以确定文件总包含哪些媒体流时，demuxer会创建相应的Source Pad，并通过事件告诉应用程序。**我们可以通过监听demuxer的事件**，在新的Source Pad被创建时，我们根据数据类型，创建相应的Element，再将其连接到Source Pad，形成完整的Pipeline。

### 4.2 示例代码

为了简化逻辑，我们在本示例中会忽略视频的Source Pad，仅连接音频的Source Pad。

```cpp
#include <gst/gst.h>

// Structure to contain all our information, so we can pass it to callbacks 
typedef struct _CustomData {
  GstElement *pipeline;
  GstElement *source;
  GstElement *convert;
  GstElement *sink;
} CustomData;

// Handler for the pad-added signal 
static void pad_added_handler (GstElement *src, GstPad *pad, CustomData *data);

int main(int argc, char *argv[]) {
  CustomData data;
  GstBus *bus;
  GstMessage *msg;
  GstStateChangeReturn ret;
  gboolean terminate = FALSE;

  // Initialize GStreamer 
  gst_init (&argc, &argv);

  // Create the elements 
  data.source = gst_element_factory_make ("uridecodebin", "source");
  data.convert = gst_element_factory_make ("audioconvert", "convert");
  data.sink = gst_element_factory_make ("autoaudiosink", "sink");

  // Create the empty pipeline 
  data.pipeline = gst_pipeline_new ("test-pipeline");

  if (!data.pipeline || !data.source || !data.convert || !data.sink) {
    g_printerr ("Not all elements could be created.\n");
    return -1;
  }

  // Build the pipeline. Note that we are NOT linking the source at this
   * point. We will do it later. 
  gst_bin_add_many (GST_BIN (data.pipeline), data.source, data.convert , data.sink, NULL);
  if (!gst_element_link (data.convert, data.sink)) {
    g_printerr ("Elements could not be linked.\n");
    gst_object_unref (data.pipeline);
    return -1;
  }

  // Set the URI to play 
  g_object_set (data.source, "uri", "https://www.freedesktop.org/software/gstreamer-sdk/data/media/sintel_trailer-480p.webm", NULL);

  // Connect to the pad-added signal 
  g_signal_connect (data.source, "pad-added", G_CALLBACK (pad_added_handler), &data);

  // Start playing 
  ret = gst_element_set_state (data.pipeline, GST_STATE_PLAYING);
  if (ret == GST_STATE_CHANGE_FAILURE) {
    g_printerr ("Unable to set the pipeline to the playing state.\n");
    gst_object_unref (data.pipeline);
    return -1;
  }

  // Listen to the bus 
  bus = gst_element_get_bus (data.pipeline);
  do {
    msg = gst_bus_timed_pop_filtered (bus, GST_CLOCK_TIME_NONE,
        GST_MESSAGE_STATE_CHANGED | GST_MESSAGE_ERROR | GST_MESSAGE_EOS);

    // Parse message 
    if (msg != NULL) {
      GError *err;
      gchar *debug_info;

      switch (GST_MESSAGE_TYPE (msg)) {
        case GST_MESSAGE_ERROR:
          gst_message_parse_error (msg, &err, &debug_info);
          g_printerr ("Error received from element %s: %s\n", GST_OBJECT_NAME (msg->src), err->message);
          g_printerr ("Debugging information: %s\n", debug_info ? debug_info : "none");
          g_clear_error (&err);
          g_free (debug_info);
          terminate = TRUE;
          break;
        case GST_MESSAGE_EOS:
          g_print ("End-Of-Stream reached.\n");
          terminate = TRUE;
          break;
        case GST_MESSAGE_STATE_CHANGED:
          // We are only interested in state-changed messages from the pipeline 
          if (GST_MESSAGE_SRC (msg) == GST_OBJECT (data.pipeline)) {
            GstState old_state, new_state, pending_state;
            gst_message_parse_state_changed (msg, &old_state, &new_state, &pending_state);
            g_print ("Pipeline state changed from %s to %s:\n",
                gst_element_state_get_name (old_state), gst_element_state_get_name (new_state));
          }
          break;
        default:
          // We should not reach here 
          g_printerr ("Unexpected message received.\n");
          break;
      }
      gst_message_unref (msg);
    }
  } while (!terminate);

  // Free resources 
  gst_object_unref (bus);
  gst_element_set_state (data.pipeline, GST_STATE_NULL);
  gst_object_unref (data.pipeline);
  return 0;
}

// This function will be called by the pad-added signal 
static void pad_added_handler (GstElement *src, GstPad *new_pad, CustomData *data) {
  GstPad *sink_pad = gst_element_get_static_pad (data->convert, "sink");
  GstPadLinkReturn ret;
  GstCaps *new_pad_caps = NULL;
  GstStructure *new_pad_struct = NULL;
  const gchar *new_pad_type = NULL;

  g_print ("Received new pad '%s' from '%s':\n", GST_PAD_NAME (new_pad), GST_ELEMENT_NAME (src));

  // If our converter is already linked, we have nothing to do here 
  if (gst_pad_is_linked (sink_pad)) {
    g_print ("We are already linked. Ignoring.\n");
    goto exit;
  }

  // Check the new pad's type 
  new_pad_caps = gst_pad_get_current_caps (new_pad);
  new_pad_struct = gst_caps_get_structure (new_pad_caps, 0);
  new_pad_type = gst_structure_get_name (new_pad_struct);
  if (!g_str_has_prefix (new_pad_type, "audio/x-raw")) {
    g_print ("It has type '%s' which is not raw audio. Ignoring.\n", new_pad_type);
    goto exit;
  }

  // Attempt the link 
  ret = gst_pad_link (new_pad, sink_pad);
  if (GST_PAD_LINK_FAILED (ret)) {
    g_print ("Type is '%s' but link failed.\n", new_pad_type);
  } else {
    g_print ("Link succeeded (type '%s').\n", new_pad_type);
  }

 exit:
  // Unreference the new pad's caps, if we got them 
  if (new_pad_caps != NULL)
    gst_caps_unref (new_pad_caps);

  // Unreference the sink pad 
  gst_object_unref (sink_pad);
}
```

### 4.3 源码分析

```cpp
// Create the elements
data.source = gst_element_factory_make("uridecodebin", "source")
data.convert = gst_element_factory_make("audioconvert", "convert")
data.sink = gst_element_factory_make("autoaudiosink", "sink")
```

首先创建了所需的Element：

- uridecodebin中会内部实例化所需的Elements（source，demuxer，decoder）将URI所指向的媒体文件中的各种媒体数据分别提取出来。因为其包含了demuxer，所以Source Pad在初始化阶段无法访问，只有在收到相应事件后去动态连接Pad。
- audioconvert用于在不同的音频数据格式之间进行转换。由于不同的声卡支持的数据类型不尽相同，所以在某些平台需要对音频数据类型进行转换。
- autoaudiosink会自动查找声卡设备，并将音频数据传输到声卡上进行输出。

```cpp
if(!gst_element_link(data.convert, data.sink)) {
  g_printerr("Elements could not be linked.\n");
  gst_object_unref(data.pipeline);
  return -1;
}
```

接着将converter和sink连接起来，注意，这里我们没有连接source与convert，是因为uridecodebin在pipeline初始阶段还没有Source Pad。

```cpp
g_object_set(data.source, "uri", "https://www.freedesktop.org/software/gstreamer-sdk/data/media/sintel_trailer-480p.webm", NULL);
```

这里设置了播放文件的uri，uridecodebin会自动解析该地址，并读取媒体数据。

#### 1) 监听事件

```cpp
// Connect to the pad-added signal
g_signal_connect (data.source, "pad-added", G_CALLBACK (pad_added_handler), &data);
```

GSignals在GStreamer中扮演这至关重要的角色，信号使你能在你所关心到事件发生后得到通知。在GLib中的信号通过信号名来进行识别，每个GObject对象都有其自己的信号。
在上面这行代码中，我们通过g_signal_connect将pad_added_handler回调连接到uridecodebin的"pad-added"信号上，同时附带回调函数的私有参数。
GStreamer不会处理我们传入到data指针，只会将其作为参数传递给回调函数，这是传递私有数据给回调函数的常用方式。  
在我们连接了"pad-added"的信号后，我们就可以将Pipeline的状态设置为PLAYING并按原有方式处理自己所关心到消息。

#### 2) 回调处理

当Source Element收集到足够的信息，能产生数据时，它会创建Source Pad并且触发"pad-added"信号，这时，我们的回调函数就会被调用。

```cpp
static void pad_added_handler (GstElement *src, GstPad *new_pad, CustomData *data)
```

这里是我们实现的回调函数，回调函数为什么要定义成这种格式？
因为我们的回调函数是为了处理信号所携带的信号，所以必须用符合信号的数据类型，否则不能正确处理相应数据，可通过gst-inspect-1.0查看uridecodebin可以看到信号所需要的回电函数的格式：

```shell
$ gst-inspect-1.0 uridecodebin
...
Element Signals:
  "pad-added" :  void user_function (GstElement* object,
                                     GstPad* arg0,
                                     gpointer user_data);
...
```

- src指针，指向触发这个事件的GstElement对象实例，这里是uridecodebin。GStreamer中的信号处理函数的第一个参数均为触发事件的对象指针。
- new_pad指针，指向src中被创建的GstPad对象实例，这通常是我们需要连接的Pad。
- data指针，指向我们在连接信号时所传的CustomData对象。

```cpp
GstPad* sink_pad = gst_element_get_static_pad(data->convert, "sink");
```

首先从CustomData中取得convert指针，并通过gst_element_get_static_pad()获取其Sink Pad。我们需要将这个Sink Pad连接到uridecodebin创建的new_pad中。

```cpp
// If our converter is already linked, we have nothing to do here
if (gst_pad_is_linked (sink_pad)) {
  g_print ("We are already linked. Ignoring.\n");
  goto exit;
}
```

由于uridecodebin可能会创建多个Pad，在每次有Pad被创建时，回调函数都会被调用。上面这段代码就是为了避免重复连接Pad。

```cpp
// Check the new pad's type
new_pad_caps = gst_pad_get_current_caps (new_pad, NULL);
new_pad_struct = gst_caps_get_structure (new_pad_caps, 0);
new_pad_type = gst_structure_get_name (new_pad_struct);
if (!g_str_has_prefix (new_pad_type, "audio/x-raw")) {
  g_print ("It has type '%s' which is not raw audio. Ignoring.\n", new_pad_type);
  goto exit;
}
```

由于我们在当前示例中只处理audio相关的数据（我们开始之创建autoaudiosink），所以这里对Pad所产生的数据类型进行了过滤，对于非音频Pad（视频及字幕）直接忽略。
gst_pad_get_current_caps()可以获取当前Pad的能力（这里是new_pad输出数据的能力），所有能力被存储在GstCaps结构体中。Pad所支持的所有Caps可以通过gst_pad_query_caps()得到，由于一个Pad可能包含多个Caps，因此GstCaps可以包含一个或多个GstStructure，每个都代表所支持的不同数据的能力。通过gst_pad_get_current_caps()获取到的当前Caps只会包含一个GstStructure用于表示唯一的数据类型，如果无法获取到当前所使用到Caps，该函数会直接返回NULL。
由于我们一致在本例中new_pad只包含一个音频Cap，所以我们直接通过gst_caps_get_structure()来取得第一个GstStructure。接着再通过gst_structure_get_name()获取该Cap支持的数据类型，如果不是音频（audio/x-raw），直接忽略

```cpp
// Attempt the link
ret = gst_pad_link (new_pad, sink_pad);
if (GST_PAD_LINK_FAILED (ret)) {
  g_print ("Type is '%s' but link failed.\n", new_pad_type);
} else {
  g_print ("Link succeeded (type '%s').\n", new_pad_type);
}
```

对于音频的Source Pad，我们使用gst_pad_link()将其与Sink Pad进行连接，使用方式与gst_element_link()相同，指定Source和Sink Pad，其所属的Element必须位于同一个Bin或Pipeline。
到目前为止，我们完成了Pipeline的建立，数据会继续再后续的Element中进行音频的播放，直到产生ERROR或EOS。

#### 3) GStreamer的状态

我们已经知道Pipeline在我们将状态设置为PLAYING之前是不会进入播放状态，实际上PLAYING状态只是GStreamer状态中一个，GStreamer总共包含4个状态：

1. NULL：NULL状态时所有Element被创建后的初始状态。
2. READY：READY状态表明GStreamer已经完成所需资源的检查，可以进入PAUSED状态。
3. PAUSED：Element处于暂停状态，表明其可以开始接受数据。Sink Element在接收了一个buffer后就会进入等待状态。
4. PLAYING：Element处于播放状态，时钟处于运行中，数据被依次处理。

GStreamer的状态必须按照上面的顺序进行切换，例如：不能直接从NULL切换到PLAYING状态，NULL必须依次切换到READY，PAUSED后才能切换到PLAYING状态，**当我们直接设置Pipeline的状态为PLAYING时，GStreamer内部会依次为我们切换到PLAYING状态**。

```cpp
case GST_MESSAGE_STATE_CHANGED:
  // We are only interested in state-changed messages from the pipeline 
  if (GST_MESSAGE_SRC (msg) == GST_OBJECT (data.pipeline)) {
    GstState old_state, new_state, pending_state;
    gst_message_parse_state_changed (msg, &old_state, &new_state, &pending_state);
    g_print ("Pipeline state changed from %s to %s:\n",
        gst_element_state_get_name (old_state), gst_element_state_get_name (new_state));
  }
  break;
```

***

## 5 播放时间控制

### 5.1 GStreamer查询机制

GStreamer提供了GstQuery的查询机制，用于查询Element或Pad的相应信息。例如：查询当前的播放速率，产生的延迟，是否支持跳转等。可查看GstQuery文档了解所支持的类型。

要查询所需的信息，首先需要构造一个查询的类型，然后使用Element或Pad的查询接口获取数据，最终再解析相应结果。 下面的例子介绍了如何使用GstQuery查询Pipeline的总时间：

```cpp
GstQuery *query = gst_query_new_duration (GST_FORMAT_TIME);
   gboolean res = gst_element_query (pipeline, query);
   if (res) {
     gint64 duration;
     gst_query_parse_duration (query, NULL, &duration);
     g_print ("duration = %"GST_TIME_FORMAT, GST_TIME_ARGS (duration));
   } else {
     g_print ("duration query failed...");
   }
   gst_query_unref (query);
```

### 5.2 示例代码

在本示例中，我们通过查询Pipeline是否支持跳转（seeking），如果支持跳转（有些媒体不支持跳转，例如实时视频），我们会在播放10秒后跳转到其他位置。
在以前的示例中，我们在Pipeline开始执行后，只等待ERROR和EOS消息，然后退出。本例中，我们会在消息等在中设置等待超时时间，超时后，我们会去查询当前播放的时间，用于显示，这与播放器的进度条类似。

```cpp
#include <gst/gst.h>

// Structure to contain all our information, so we can pass it around 
typedef struct _CustomData {
  GstElement *playbin;  // Our one and only element 
  gboolean playing;      // Are we in the PLAYING state? 
  gboolean terminate;    // Should we terminate execution? 
  gboolean seek_enabled; // Is seeking enabled for this media? 
  gboolean seek_done;    // Have we performed the seek already? 
  gint64 duration;       // How long does this media last, in nanoseconds 
} CustomData;

// Forward definition of the message processing function 
static void handle_message (CustomData *data, GstMessage *msg);

int main(int argc, char *argv[]) {
  CustomData data;
  GstBus *bus;
  GstMessage *msg;
  GstStateChangeReturn ret;

  data.playing = FALSE;
  data.terminate = FALSE;
  data.seek_enabled = FALSE;
  data.seek_done = FALSE;
  data.duration = GST_CLOCK_TIME_NONE;

  // Initialize GStreamer 
  gst_init (&argc, &argv);

  // Create the elements 
  data.playbin = gst_element_factory_make ("playbin", "playbin");

  if (!data.playbin) {
    g_printerr ("Not all elements could be created.\n");
    return -1;
  }

  // Set the URI to play 
  g_object_set (data.playbin, "uri", "https://www.freedesktop.org/software/gstreamer-sdk/data/media/sintel_trailer-480p.webm", NULL);

  // Start playing 
  ret = gst_element_set_state (data.playbin, GST_STATE_PLAYING);
  if (ret == GST_STATE_CHANGE_FAILURE) {
    g_printerr ("Unable to set the pipeline to the playing state.\n");
    gst_object_unref (data.playbin);
    return -1;
  }

  // Listen to the bus 
  bus = gst_element_get_bus (data.playbin);
  do {
    msg = gst_bus_timed_pop_filtered (bus, 100 * GST_MSECOND,
        GST_MESSAGE_STATE_CHANGED | GST_MESSAGE_ERROR | GST_MESSAGE_EOS | GST_MESSAGE_DURATION_CHANGED);

    // Parse message 
    if (msg != NULL) {
      handle_message (&data, msg);
    } else {
      // We got no message, this means the timeout expired 
      if (data.playing) {
        gint64 current = -1;

        // Query the current position of the stream 
        if (!gst_element_query_position (data.playbin, GST_FORMAT_TIME, &current)) {
          g_printerr ("Could not query current position.\n");
        }

        // If we didn't know it yet, query the stream duration 
        if (!GST_CLOCK_TIME_IS_VALID (data.duration)) {
          if (!gst_element_query_duration (data.playbin, GST_FORMAT_TIME, &data.duration)) {
            g_printerr ("Could not query current duration.\n");
          }
        }

        // Print current position and total duration 
        g_print ("Position %" GST_TIME_FORMAT " / %" GST_TIME_FORMAT "\r",
            GST_TIME_ARGS (current), GST_TIME_ARGS (data.duration));

        // If seeking is enabled, we have not done it yet, and the time is right, seek 
        if (data.seek_enabled && !data.seek_done && current > 10 * GST_SECOND) {
          g_print ("\nReached 10s, performing seek...\n");
          gst_element_seek_simple (data.playbin, GST_FORMAT_TIME,
              GST_SEEK_FLAG_FLUSH | GST_SEEK_FLAG_KEY_UNIT, 30 * GST_SECOND);
          data.seek_done = TRUE;
        }
      }
    }
  } while (!data.terminate);

  // Free resources 
  gst_object_unref (bus);
  gst_element_set_state (data.playbin, GST_STATE_NULL);
  gst_object_unref (data.playbin);
  return 0;
}

static void handle_message (CustomData *data, GstMessage *msg) {
  GError *err;
  gchar *debug_info;

  switch (GST_MESSAGE_TYPE (msg)) {
    case GST_MESSAGE_ERROR:
      gst_message_parse_error (msg, &err, &debug_info);
      g_printerr ("Error received from element %s: %s\n", GST_OBJECT_NAME (msg->src), err->message);
      g_printerr ("Debugging information: %s\n", debug_info ? debug_info : "none");
      g_clear_error (&err);
      g_free (debug_info);
      data->terminate = TRUE;
      break;
    case GST_MESSAGE_EOS:
      g_print ("End-Of-Stream reached.\n");
      data->terminate = TRUE;
      break;
    case GST_MESSAGE_DURATION_CHANGED:
      // The duration has changed, mark the current one as invalid 
      data->duration = GST_CLOCK_TIME_NONE;
      break;
    case GST_MESSAGE_STATE_CHANGED: {
      GstState old_state, new_state, pending_state;
      gst_message_parse_state_changed (msg, &old_state, &new_state, &pending_state);
      if (GST_MESSAGE_SRC (msg) == GST_OBJECT (data->playbin)) {
        g_print ("Pipeline state changed from %s to %s:\n",
            gst_element_state_get_name (old_state), gst_element_state_get_name (new_state));

        // Remember whether we are in the PLAYING state or not 
        data->playing = (new_state == GST_STATE_PLAYING);

        if (data->playing) {
          // We just moved to PLAYING. Check if seeking is possible 
          GstQuery *query;
          gint64 start, end;
          query = gst_query_new_seeking (GST_FORMAT_TIME);
          if (gst_element_query (data->playbin, query)) {
            gst_query_parse_seeking (query, NULL, &data->seek_enabled, &start, &end);
            if (data->seek_enabled) {
              g_print ("Seeking is ENABLED from %" GST_TIME_FORMAT " to %" GST_TIME_FORMAT "\n",
                  GST_TIME_ARGS (start), GST_TIME_ARGS (end));
            } else {
              g_print ("Seeking is DISABLED for this stream.\n");
            }
          }
          else {
            g_printerr ("Seeking query failed.");
          }
          gst_query_unref (query);
        }
      }
    } break;
    default:
      // We should not reach here 
      g_printerr ("Unexpected message received.\n");
      break;
  }
  gst_message_unref (msg);
}
```

### 5.3 源码分析

示例前部分内容与其他示例类似，构造Pipeline并使其进入PLAYING状态。之后开始监听Bus上的消息。

```cpp
msg = gst_bus_timed_pop_filtered (bus, 100 * GST_MSECOND,
    GST_MESSAGE_STATE_CHANGED | GST_MESSAGE_ERROR | GST_MESSAGE_EOS | GST_MESSAGE_DURATION_CHANGED);
```

与以前示例相似，在gst_bus_timed_pop_filterd()中加入了超时时间(100ms),这使得此函数如果在100毫秒内没有收到任何消息就会返回超时(msg == NULL)，我们会在超时中去更新当前时间，如果返回相应消息（msg != NULL），我们在handle_message中处理相应消息。

GStreamer内部有统一的时间类型（GstClockTime），时间计算方式为：GstClockTime = 数值 x 时间单位。GStreamer提供了3种时间单位（宏定义）：GST_SECOND（秒），GST_MSECOND（毫秒），GST_NSECOND（纳秒）。例如：

- 10秒： 10 * GST_SECOND
- 100毫秒：100 * GST_MSECOND
- 100纳秒：100 * GST_NSECOND

#### 1) 刷新播放时间

```cpp
// We got no message, this means the timeout expired 
if (data.playing) {
// Query the current position of the stream 
if (!gst_element_query_position (data.pipeline, GST_FORMAT_TIME, &current)) {
  g_printerr ("Could not query current position.\n");
}
// If we didn't know it yet, query the stream duration 
if (!GST_CLOCK_TIME_IS_VALID (data.duration)) {
  if (!gst_element_query_duration (data.pipeline, GST_FORMAT_TIME, &data.duration)) {
     g_printerr ("Could not query current duration.\n");
  }
}
```

我们首先判断Pipeline的状态，仅在PLAYING状态时才更新当前时间，在非PLAYING状态时查询可能失败。这部分逻辑每秒大概会执行10次，频率足够用于界面的刷新。这里我们只将查询到的时间输出到终端。
GstElement封装了相应的接口分别用于查询当前时间（gst_element_query_position）和总时间（gst_element_query_duration）。

```cpp
// Print current position and total duration
g_print ("Position %" GST_TIME_FORMAT " / %" GST_TIME_FORMAT "\r",
    GST_TIME_ARGS (current), GST_TIME_ARGS (data.duration));
```

这里使用GST_TIME_FORMAT 和GST_TIME_ARGS 帮助我们方便地将GstClockTime的值转换为："时：分：秒"格式的字符串输出。

```cpp
// If seeking is enabled, we have not done it yet, and the time is right, seek
if (data.seek_enabled && !data.seek_done && current > 10 * GST_SECOND) {
  g_print ("\nReached 10s, performing seek...\n");
  gst_element_seek_simple (data.pipeline, GST_FORMAT_TIME,
      GST_SEEK_FLAG_FLUSH | GST_SEEK_FLAG_KEY_UNIT, 30 * GST_SECOND);
  data.seek_done = TRUE;
}
```

我们同时在超时处理中判断是否需要进行seek操作（在播放到10s时，自动跳转到30s），这里我们直接在Pipeline对象上使用gst_element_seek_simple()来执行跳转操作。
gst_element_seek_simple所需要的参数为：

element ： 需要执行seek操作的Element，这里是Pipeline。
format：执行seek的类型，这里使用GST_FORMAT_TIME表示我们基于时间的方式进行跳转。其他支持的类型可以查看 GstFormat。
seek_flags ：通过标识指定seek的行为。 常用的标识如下，其他支持的flag详见GstSeekFlags。

- GST_SEEK_FLAG_FLUSH：在执行seek前，清除Pipeline中所有buffer中缓存的数据。这可能导致Pipeline在填充的新数据被显示之前出现短暂的等待，但能提高应用更快的响应速度。如果不指定这个标志，Pipeline中的所有缓存数据会依次输出，然后才会播放跳转的位置，会导致一定的延迟。
- GST_SEEK_FLAG_KEY_UNIT：对于大多数的视频，如果跳转的位置不是关键帧，需要依次解码该帧所依赖的帧（I帧及P帧）后，才能解码此非关键帧。使用这个标识后，seek会自动从最近的I帧开始播放。这个标识降低了seek的精度，提高了seek的效率。
- GST_SEEK_FLAG_ACCURATE：一些媒体文件没有提供足够的索引信息，在这种文件中执行seek操作会非常耗时，针对这类文件，GStreamer通过内部计算得到需要跳转的位置，大部分的计算结果都是正确的。如果seek的位置不能达到所需精度时，可以增加此标识。但需要注意的是，使用此标识可能会导致seek耗费更多时间来寻找精确的位置。

seek_pos ：需要跳转的位置，前面指定了seek的类型为时间，所以这里是30秒。

#### 2) 消息处理

我们在handle_message接口中处理所有Pipeline上的消息，ERROR和EOS与之前示例处理方式相同，此例中新增了以下内容：

```cpp
case GST_MESSAGE_DURATION_CHANGED:
  // The duration has changed, mark the current one as invalid
  data->duration = GST_CLOCK_TIME_NONE;
  break;
```

在文件的总时间发生变化时，我们会收到此消息，这里简单的将总长度标记为非法值，在下次更新时间进行查询。

```cpp
case GST_MESSAGE_STATE_CHANGED: {
  GstState old_state, new_state, pending_state;
  gst_message_parse_state_changed (msg, &old_state, &new_state, &pending_state);
  if (GST_MESSAGE_SRC (msg) == GST_OBJECT (data->pipeline)) {
    g_print ("Pipeline state changed from %s to %s:\n",
        gst_element_state_get_name (old_state), gst_element_state_get_name (new_state));

    // Remember whether we are in the PLAYING state or not
    data->playing = (new_state == GST_STATE_PLAYING);
```

跳转和时间查询操作仅在PAUSED和PLAYING状态时才能得到正确的结果，因为所有的Element只能在这2个状态才能接收处理seek和query的指令。这里会保存播放的状态便于后续使用，并且在进入PLAYING状态时查询当前所播放的文件/流是否支持跳转操作：

```cpp
if (data->playing) {
  // We just moved to PLAYING. Check if seeking is possible 
  GstQuery *query;
  gint64 start, end;
  query = gst_query_new_seeking (GST_FORMAT_TIME);
  if (gst_element_query (data->pipeline, query)) {
    gst_query_parse_seeking (query, NULL, &data->seek_enabled, &start, &end);
    if (data->seek_enabled) {
      g_print ("Seeking is ENABLED from %" GST_TIME_FORMAT " to %" GST_TIME_FORMAT "\n",
          GST_TIME_ARGS (start), GST_TIME_ARGS (end));
    } else {
      g_print ("Seeking is DISABLED for this stream.\n");
    }
  }
  else {
    g_printerr ("Seeking query failed.");
  }
  gst_query_unref (query);
}
```

这里的查询步骤与文章开始介绍的方式相同：

- 首先，通过gst_query_new_seeking()钩爪一个跳转的查询对象，使用GST_FORMAT_TIME作为参数，表明我们需要知道当前的文件是否支持通过时间进行跳转。我们同样可以使用GST_FORMAT_BYTES作为参数，用于查询是否可以根据文件的偏移量来进行跳转，但这种使用方式不常见。
- 接着，将查询对象传入gst_element_query()查询，并得到结果。
- 最后，通过gst_query_parse_seeking()解析是否支持跳转及所支持的范围。

**一定要记住在使用完后释放查询对象。**

***

## 6 获取媒体信息

### 6.1 GStreamer元数据

GStreamer将元数据分为两类：

- 流信息(Stream-info)：用于描述流的属性。例如：编码类型，分辨率，采样率等。
Stream-info可以通过Pipeline中所有的GstCap获取，使用方式在媒体类型与Pad中有描述，本文将不在复述。

- 流标签(Stream-tag)：用于描述非技术行的信息。例如：作者，标题，专辑等。
Stream-tag可以通过GstBus，监听GST_MESSAGE_TAG消息，从消息中提取相应信息。
需要注意的是，Gstreamer可能出发多次GST_MESSAGE_TAG消息，应用程序可以通过gst_tag_list_merge()合并多个标签，再在适当的时间显示，当切换媒体文件时，需要清空缓存。
使用此函数时，需要采用GST_TAG_MERGE_PREPEND，这样后续更新的元数据会有更高的优先级。

### 6.2 示例代码

```cpp
#include <gst/gst.h>

static void print_one_tag (const GstTagList * list, const gchar * tag, gpointer user_data)
{
  int i, num;

  num = gst_tag_list_get_tag_size (list, tag);
  for (i = 0; i < num; ++i) {
    const GValue *val;

    // Note: when looking for specific tags, use the gst_tag_list_get_xyz() API,
     * we only use the GValue approach here because it is more generic 
    val = gst_tag_list_get_value_index (list, tag, i);
    if (G_VALUE_HOLDS_STRING (val)) {
      g_print ("\t%20s : %s\n", tag, g_value_get_string (val));
    } else if (G_VALUE_HOLDS_UINT (val)) {
      g_print ("\t%20s : %u\n", tag, g_value_get_uint (val));
    } else if (G_VALUE_HOLDS_DOUBLE (val)) {
      g_print ("\t%20s : %g\n", tag, g_value_get_double (val));
    } else if (G_VALUE_HOLDS_BOOLEAN (val)) {
      g_print ("\t%20s : %s\n", tag,
          (g_value_get_boolean (val)) ? "true" : "false");
    } else if (GST_VALUE_HOLDS_BUFFER (val)) {
      GstBuffer *buf = gst_value_get_buffer (val);
      guint buffer_size = gst_buffer_get_size (buf);

      g_print ("\t%20s : buffer of size %u\n", tag, buffer_size);
    } else if (GST_VALUE_HOLDS_DATE_TIME (val)) {
      GstDateTime *dt = g_value_get_boxed (val);
      gchar *dt_str = gst_date_time_to_iso8601_string (dt);

      g_print ("\t%20s : %s\n", tag, dt_str);
      g_free (dt_str);
    } else {
      g_print ("\t%20s : tag of type '%s'\n", tag, G_VALUE_TYPE_NAME (val));
    }
  }
}

static void on_new_pad (GstElement * dec, GstPad * pad, GstElement * fakesink)
{
  GstPad *sinkpad;

  sinkpad = gst_element_get_static_pad (fakesink, "sink");
  if (!gst_pad_is_linked (sinkpad)) {
    if (gst_pad_link (pad, sinkpad) != GST_PAD_LINK_OK)
      g_error ("Failed to link pads!");
  }
  gst_object_unref (sinkpad);
}

int main (int argc, char ** argv)
{
  GstElement *pipe, *dec, *sink;
  GstMessage *msg;
  gchar *uri;

  gst_init (&argc, &argv);

  if (argc < 2)
    g_error ("Usage: %s FILE or URI", argv[0]);

  if (gst_uri_is_valid (argv[1])) {
    uri = g_strdup (argv[1]);
  } else {
    uri = gst_filename_to_uri (argv[1], NULL);
  }

  pipe = gst_pipeline_new ("pipeline");

  dec = gst_element_factory_make ("uridecodebin", NULL);
  g_object_set (dec, "uri", uri, NULL);
  gst_bin_add (GST_BIN (pipe), dec);

  sink = gst_element_factory_make ("fakesink", NULL);
  gst_bin_add (GST_BIN (pipe), sink);

  g_signal_connect (dec, "pad-added", G_CALLBACK (on_new_pad), sink);

  gst_element_set_state (pipe, GST_STATE_PAUSED);

  while (TRUE) {
    GstTagList *tags = NULL;

    msg = gst_bus_timed_pop_filtered (GST_ELEMENT_BUS (pipe),
        GST_CLOCK_TIME_NONE,
        GST_MESSAGE_ASYNC_DONE | GST_MESSAGE_TAG | GST_MESSAGE_ERROR);

    if (GST_MESSAGE_TYPE (msg) != GST_MESSAGE_TAG) // error or async_done 
      break;

    gst_message_parse_tag (msg, &tags);

    g_print ("Got tags from element %s:\n", GST_OBJECT_NAME (msg->src));
    gst_tag_list_foreach (tags, print_one_tag, NULL);
    g_print ("\n");
    gst_tag_list_unref (tags);

    gst_message_unref (msg);
  }

  if (GST_MESSAGE_TYPE (msg) == GST_MESSAGE_ERROR) {
    GError *err = NULL;

    gst_message_parse_error (msg, &err, NULL);
    g_printerr ("Got error: %s\n", err->message);
    g_error_free (err);
  }

  gst_message_unref (msg);
  gst_element_set_state (pipe, GST_STATE_NULL);
  gst_object_unref (pipe);
  g_free (uri);
  return 0;
}
```

**示例输出**：

```shell
$ ./basic-tutorial-6 sintel_trailer-480p.ogv
Got tags from element fakesink0:
                       title : Sintel Trailer
                      artist : Durian Open Movie Team
                   copyright : (c) copyright Blender Foundation | durian.blender.org
                     license : Creative Commons Attribution 3.0 license
            application-name : ffmpeg2theora-0.24
                     encoder : Xiph.Org libtheora 1.1 20090822 (Thusnelda)
                 video-codec : Theora
             encoder-version : 3

Got tags from element fakesink0:
            container-format : Ogg
```

#### 源码分析

本例中使用uridecodebin解析媒体文件，Pipeline的构造与其他示例相同，下面介绍Tag相关的处理逻辑。

```cpp
static void
print_one_tag (const GstTagList * list, const gchar * tag, gpointer user_data)
{
  int i, num;

  num = gst_tag_list_get_tag_size (list, tag);
  for (i = 0; i < num; ++i) {
    const GValue *val;

    // Note: when looking for specific tags, use the gst_tag_list_get_xyz() API,
     * we only use the GValue approach here because it is more generic 
    val = gst_tag_list_get_value_index (list, tag, i);
    if (G_VALUE_HOLDS_STRING (val)) {
      g_print ("\t%20s : %s\n", tag, g_value_get_string (val));
    } 
...
}
```

此函数用于输出一个标签的值。GStreamer会将多个标签都放在同一个GstTagList中。每一个标签可以包含多个值，所以首先通过gst_tag_list_get_tag_size ()接口及标签名（tag）获取其值的数量，然后再获取相应的值。
本例使用GValue来进行通用的处理，所以需要先判断数据的类型，再通过GValue接口获取。实际处理标签时，可以根据规范（例如ID3Tag）得到标签值的类型，直接通过GstTagList接口获取，例如：当标签名为title时，我们可以直接使用gst_tag_list_get_string()取得title的字符串，不需要再通过GValue转换，详细使用方式可参考GstTagList文档。

```cpp
static void
on_new_pad (GstElement * dec, GstPad * pad, GstElement * fakesink)
{
  GstPad *sinkpad;

  sinkpad = gst_element_get_static_pad (fakesink, "sink");
  if (!gst_pad_is_linked (sinkpad)) {
    if (gst_pad_link (pad, sinkpad) != GST_PAD_LINK_OK)
      g_error ("Failed to link pads!");
  }
  gst_object_unref (sinkpad);
}
...
  sink = gst_element_factory_make ("fakesink", NULL);
  gst_bin_add (GST_BIN (pipe), sink);
  g_signal_connect (dec, "pad-added", G_CALLBACK (on_new_pad), sink);
```

由于我们只需要提取相应的媒体信息，不需要关心具体的数据，所以这里使用fakesink，fakesink会直接丢弃掉所有收到的数据。同时在此处监听了"pad-added"的信号，用于动态连接Pipeline，这种处理方式已在动态连接Pipeline中进行了详细的介绍。

```cpp
while (TRUE) {
    GstTagList *tags = NULL;

    msg = gst_bus_timed_pop_filtered (GST_ELEMENT_BUS (pipe),
        GST_CLOCK_TIME_NONE,
        GST_MESSAGE_ASYNC_DONE | GST_MESSAGE_TAG | GST_MESSAGE_ERROR);

    if (GST_MESSAGE_TYPE (msg) != GST_MESSAGE_TAG) // error or async_done 
      break;

    gst_message_parse_tag (msg, &tags);

    g_print ("Got tags from element %s:\n", GST_OBJECT_NAME (msg->src));
    gst_tag_list_foreach (tags, print_one_tag, NULL);
    g_print ("\n");
    gst_tag_list_unref (tags);

    gst_message_unref (msg);
  }
```

与其他示例相同，这里也采用gst_bus_timed_pop_filtered()获取Bus上的GST_MESSAGE_TAG，再通过gst_message_parse_tag()从消息中将标签拷贝到GstTagList中，再通过gst_tag_list_foreach ()依次输出所有的标签，随后释放GstTagList。
**需要注意的是，如果GstTagList中不包含任何标签信息，gst_tag_list_foreach ()中的回调函数不会被调用**。
从上面的介绍可以发现，Stream-tag主要是通过监听GST_MESSAGE_TAG后，根据相应接口提取元数据。在使用的过程中需要注意数据的释放。

### 6.3 GstDiscoverer

获取媒体信息是一个常用的功能，因此GStreamer通过GstDiscoverer提供了一组实用接口。使用时无需关心内部Pipeline的创建，只需通过gst_discoverer_new()创建实例，使用gst_discoverer_discover_uri()指定URI，监听相应信号后，即可在回调函数中得到相应的元数据，使用时需要额外连接libgstpbutils-1.0库。GStreamer同时基于GstDiscoverer提供了gst-discoverer-1.0工具，使用方式如下：

```shell
$ gst-discoverer-1.0 sintel_trailer-480p.mp4
Analyzing file:///home/xleng/video/sintel_trailer-480p.mp4
Done discovering file:///home/xleng/video/sintel_trailer-480p.mp4

Topology:
  container: Quicktime
    audio: MPEG-4 AAC
    video: H.264 (High Profile)

Properties:
  Duration: 0:00:52.209000000
  Seekable: yes
  Live: no
  Tags:
      audio codec: MPEG-4 AAC audio
      maximum bitrate: 128000
      datetime: 1970-01-01T00:00:00Z
      title: Sintel Trailer
      artist: Durian Open Movie Team
      copyright: (c) copyright Blender Foundation | durian.blender.org
      description: Trailer for the Sintel open movie project
      encoder: Lavf52.62.0
      container format: ISO MP4/M4A
      video codec: H.264 / AVC
      bitrate: 535929
```

***

## 7 播放速率控制

### 7.1 GStreamer Seek与Step事件

快进(Fast-Forward)，快退(Fast-Rewind)和慢放(Slow-Motion)都是通过修改播放的速率来达到相应的目的。在GStreamer中，将1倍速作为正常的播放速率，将大于1倍速的2倍，4倍，8倍等倍速称为快进，慢放则是播放速率的绝对值小于1倍速，当播放速率小于0时，则进行倒放。
在GStreamer中，我们通过seek与step事件来控制Element的播放速率及区域。Step事件允许跳过指定的区域并设置后续的播放速率（此速率必须大于0）。Seek事件允许跳转到播放文件中的的任何位置，并且播放速率可以大于0或小于0。
在播放时间控制中，我们使用gst_element_seek_simple 来快速的跳转到指定的位置，此函数是对seek事件的封装。实际使用时，我们首先需要构造一个seek event，设置seek的绝对起始位置和停止位置，停止位置可以设置为0，这样会执行seek的播放速率直到结束。同时可以支持按buffer的方式进行seek，以及设置不同的标志指定seek的行为。
Step事件相较于Seek事件需要更少的参数，更易用于修改播放速率，但是不够灵活。Step事件只会作用于最终的sink，Seek事件则可以作用于Pipeline中所有的Element。Step操作的效率高于Seek。
在GStreamer中，单帧播放(Frame Stepping）与快进相同，也是通过事件实现。单帧播放通常在暂停的状态下，构造并发送step event每次播放一帧。
需要注意的是，seek event需要直接作用于sink element（eg：audio sink或video sink），如果直接将seek event作用于Pipeline，Pipeline会自动将事件转发给所有的sink，如果有多个sink，就会造成多次seek。通常是先获取Pipeline中的video-sink或audio-sink，然后发送seek event到指定的sink，完成seek的操作。 Seek时间的构造及发送示例如下：

```cpp
GstEvent *event;
   gboolean result;
   ...
   // construct a seek event to play the media from second 2 to 5, flush
   // the pipeline to decrease latency.
   event = gst_event_new_seek (1.0,
      GST_FORMAT_TIME,
      GST_SEEK_FLAG_FLUSH,
      GST_SEEK_TYPE_SET, 2 * GST_SECOND,
      GST_SEEK_TYPE_SET, 5 * GST_SECOND);
   ...
   result = gst_element_send_event (video_sink, event);
   if (!result)
     g_warning ("seek failed");
   ...
```

### 7.2 示例代码

下面通过一个完整的示例，来查看GStreamer是如何通过seek和step达到相应的播放速度。

```cpp
#include <string.h>
#include <stdio.h>
#include <gst/gst.h>

typedef struct _CustomData
{
  GstElement *pipeline;
  GstElement *video_sink;
  GMainLoop *loop;

  gboolean playing;             // Playing or Paused 
  gdouble rate;                 // Current playback rate (can be negative) 
} CustomData;

// Send seek event to change rate 
static void send_seek_event (CustomData * data)
{
  gint64 position;
  GstEvent *seek_event;

  // Obtain the current position, needed for the seek event 
  if (!gst_element_query_position (data->pipeline, GST_FORMAT_TIME, &position)) {
    g_printerr ("Unable to retrieve current position.\n");
    return;
  }

  // Create the seek event 
  if (data->rate > 0) {
    seek_event =
        gst_event_new_seek(data->rate, GST_FORMAT_TIME,
        GST_SEEK_FLAG_FLUSH | GST_SEEK_FLAG_ACCURATE, GST_SEEK_TYPE_SET,
        position, GST_SEEK_TYPE_END, 0);
  } else {
    seek_event =
        gst_event_new_seek(data->rate, GST_FORMAT_TIME,
        GST_SEEK_FLAG_FLUSH | GST_SEEK_FLAG_ACCURATE, GST_SEEK_TYPE_SET, 0,
        GST_SEEK_TYPE_SET, position);
  }

  if (data->video_sink == NULL) {
    // If we have not done so, obtain the sink through which we will send the seek events 
    g_object_get (data->pipeline, "video-sink", &data->video_sink, NULL);
  }

  // Send the event 
  gst_element_send_event (data->video_sink, seek_event);

  g_print ("Current rate: %g\n", data->rate);
}

// Process keyboard input 
static gboolean handle_keyboard (GIOChannel * source, GIOCondition cond, CustomData * data)
{
  gchar *str = NULL;

  if (g_io_channel_read_line (source, &str, NULL, NULL,
          NULL) != G_IO_STATUS_NORMAL) {
    return TRUE;
  }

  switch (g_ascii_tolower (str[0])) {
    case 'p':
      data->playing = !data->playing;
      gst_element_set_state (data->pipeline,
          data->playing ? GST_STATE_PLAYING : GST_STATE_PAUSED);
      g_print ("Setting state to %s\n", data->playing ? "PLAYING" : "PAUSE");
      break;
    case 's':
      if (g_ascii_isupper (str[0])) {
        data->rate *= 2.0;
      } else {
        data->rate /= 2.0;
      }
      send_seek_event (data);
      break;
    case 'd':
      data->rate *= -1.0;
      send_seek_event (data);
      break;
    case 'n':
      if (data->video_sink == NULL) {
        // If we have not done so, obtain the sink through which we will send the step events 
        g_object_get (data->pipeline, "video-sink", &data->video_sink, NULL);
      }

      gst_element_send_event (data->video_sink,
          gst_event_new_step (GST_FORMAT_BUFFERS, 1, ABS (data->rate), TRUE,
              FALSE));
      g_print ("Stepping one frame\n");
      break;
    case 'q':
      g_main_loop_quit (data->loop);
      break;
    default:
      break;
  }

  g_free (str);

  return TRUE;
}

int main (int argc, char *argv[])
{
  CustomData data;
  GstStateChangeReturn ret;
  GIOChannel *io_stdin;

  // Initialize GStreamer 
  gst_init (&argc, &argv);

  // Initialize our data structure 
  memset (&data, 0, sizeof (data));

  // Print usage map 
  g_print ("USAGE: Choose one of the following options, then press enter:\n"
      " 'P' to toggle between PAUSE and PLAY\n"
      " 'S' to increase playback speed, 's' to decrease playback speed\n"
      " 'D' to toggle playback direction\n"
      " 'N' to move to next frame (in the current direction, better in PAUSE)\n"
      " 'Q' to quit\n");

  // Build the pipeline 
  data.pipeline =
      gst_parse_launch
      ("playbin uri=https://www.freedesktop.org/software/gstreamer-sdk/data/media/sintel_trailer-480p.webm",
      NULL);

  // Add a keyboard watch so we get notified of keystrokes 
#ifdef G_OS_WIN32
  io_stdin = g_io_channel_win32_new_fd (fileno (stdin));
#else
  io_stdin = g_io_channel_unix_new (fileno (stdin));
#endif
  g_io_add_watch (io_stdin, G_IO_IN, (GIOFunc) handle_keyboard, &data);

  // Start playing 
  ret = gst_element_set_state (data.pipeline, GST_STATE_PLAYING);
  if (ret == GST_STATE_CHANGE_FAILURE) {
    g_printerr ("Unable to set the pipeline to the playing state.\n");
    gst_object_unref (data.pipeline);
    return -1;
  }
  data.playing = TRUE;
  data.rate = 1.0;

  // Create a GLib Main Loop and set it to run 
  data.loop = g_main_loop_new (NULL, FALSE);
  g_main_loop_run (data.loop);

  // Free resources 
  g_main_loop_unref (data.loop);
  g_io_channel_unref (io_stdin);
  gst_element_set_state (data.pipeline, GST_STATE_NULL);
  if (data.video_sink != NULL)
    gst_object_unref (data.video_sink);
  gst_object_unref (data.pipeline);
  return 0;
}
```

**源码分析**
本例中，Pipeline的创建与其他示例相同，通过playbin播放文件，采用GLib的I/O接口来处理键盘输入。

```cpp
// Process keyboard input
static gboolean handle_keyboard (GIOChannel *source, GIOCondition cond, CustomData *data) {
  gchar *str = NULL;

  if (g_io_channel_read_line (source, &str, NULL, NULL, NULL) != G_IO_STATUS_NORMAL) {
    return TRUE;
  }

  switch (g_ascii_tolower (str[0])) {
  case 'p':
    data->playing = !data->playing;
    gst_element_set_state (data->pipeline, data->playing ? GST_STATE_PLAYING : GST_STATE_PAUSED);
    g_print ("Setting state to %s\n", data->playing ? "PLAYING" : "PAUSE");
    break;
```

在终端输入P时，使用gst_element_set_state()设置播放状态。

```cpp
case 's':
  if (g_ascii_isupper (str[0])) {
    data->rate *= 2.0;
  } else {
    data->rate /= 2.0;
  }
  send_seek_event (data);
  break;
case 'd':
  data->rate *= -1.0;
  send_seek_event (data);
  break;
```

通过S和s增加和降低播放速度，d用于改变播放方向（倒放），这里在修改rate后，调用send_seek_event实现真正的处理。

```cpp
// Send seek event to change rate
static void send_seek_event (CustomData *data) {
  gint64 position;
  GstEvent *seek_event;

  // Obtain the current position, needed for the seek event 
  if (!gst_element_query_position (data->pipeline, GST_FORMAT_TIME, &position)) {
    g_printerr ("Unable to retrieve current position.\n");
    return;
  }
```

这个函数会构造一个SeekEvent发送到Pipeline以调节速率。因为Seek Event会跳转到指定的位置，但我们在此示例中只想改变速率，不跳转到其他位置，所以首先通过gst_element_query_position ()获取当前的播放位置。

```cpp
// Create the seek event 
if (data->rate > 0) {
  seek_event = gst_event_new_seek (data->rate, GST_FORMAT_TIME, GST_SEEK_FLAG_FLUSH | GST_SEEK_FLAG_ACCURATE,
      GST_SEEK_TYPE_SET, position, GST_SEEK_TYPE_END, 0);
} else {
  seek_event = gst_event_new_seek (data->rate, GST_FORMAT_TIME, GST_SEEK_FLAG_FLUSH | GST_SEEK_FLAG_ACCURATE,
      GST_SEEK_TYPE_SET, 0, GST_SEEK_TYPE_SET, position);
}
```

通过gst_event_new_seek()创建SeekEvent，设置新的rate，flag，起始位置，结束位置。需要注意的是，起始位置需要小于结束位置。

```cpp
if (data->video_sink == NULL) {
  // If we have not done so, obtain the sink through which we will send the seek events
  g_object_get (data->pipeline, "video-sink", &data->video_sink, NULL);
}
// Send the event
gst_element_send_event (data->video_sink, seek_event);
```

正如上文提到的，为了避免Pipeline执行多次的seek，我们在此处获取video-sink，并向其发送SeekEvent。我们直到执行Seek时才获取video-sink是因为实际的sink有可能会根据不同的媒体类型，在PLAYING状态时才创建。
以上部分内容就是速率的修改，关于单帧播放的情况，实现方式更加简单：

```cpp
case 'n':
  if(data->video_sink == NULL) {
    // If we have not done so, obtain the sink through which we will send the step events
    g_object_get(data->pipeline, "video-sink", &data->video_sink, NULL);
  }

  gst_element_send_event(data->video_sink, 
    gst_event_new_step(GST_FORMAT_BUFFERS, 1, ABS(data->rate), TRUE, FALSE));
  g_print("Stepping one frame\n");
  break;
```

我们通过gst_event_new_step()创建了StepEvent，并指定只跳转一帧，且不改变当前速率。单帧播放通常都是先暂停，然后再进行单帧播放。
以上就是通过GStreamer实现播放速率的控制，实际中，有些Element对倒放支持不是特别好，不能达到理想效果。

***

## 8 多线程

### 8.1 GStreamer多线程

GStreamer框架是一个支持多线程的框架，线程会根据Pipeline的需要自动创建和销毁，例如，将媒体流与应用线程解耦，应用线程不会被GStreamer的处理阻塞。而且，GStreamer的插件还可以创建自己所需的线程用于媒体的处理，例如：在一个4核的CPU上，视频解码插件可以创建4个线程来最大化利用CPU资源。
此外，在创建Pipeline时，我们还可以指定某个Pipeline的分支在不同的线程中执行（例如，使audio、video同时在不同的线程中进行解码）。这是通过queue Element来实现的，queue的sink pad仅仅将数据放入队列，另外一个线程从队列中取出数据，并传递到下一个Element。queue通常也被用于作为数据缓冲，缓冲区大小可以通过queue的属性进行配置。

<center><img src="https://img2018.cnblogs.com/blog/1647252/201909/1647252-20190929160937045-425468738.jpg" /></center>

在上面的示例Pipeline中，source是audiotestsrc，会产生一个相应的audio信号，然后使用tee Element将数据分为两路，一路被用于播放，通过声卡输出，另一路被用于转换为视频波形，用于输出到屏幕。
示例图中的红色阴影部分表示位于同一个线程中，queue会创建单独的线程，所以上面的Pipeline使用了3个线程完成相应的功能。拥有多个sink的Pipeline通常需要多个线程，因为在多个sync间进行同步的时候，sink会阻塞当前所在线程直到所等待的事件发生。

### 8.2 示例代码

示例代码将创建上图所示的Pipeline：

```cpp
#include <gst/gst.h>

int main(int argc, char *argv[]) {
  GstElement *pipeline, *audio_source, *tee, *audio_queue, *audio_convert, *audio_resample, *audio_sink;
  GstElement *video_queue, *visual, *video_convert, *video_sink;
  GstBus *bus;
  GstMessage *msg;
  GstPad *tee_audio_pad, *tee_video_pad;
  GstPad *queue_audio_pad, *queue_video_pad;

  // Initialize GStreamer 
  gst_init (&argc, &argv);

  // Create the elements 
  audio_source = gst_element_factory_make ("audiotestsrc", "audio_source");
  tee = gst_element_factory_make ("tee", "tee");
  audio_queue = gst_element_factory_make ("queue", "audio_queue");
  audio_convert = gst_element_factory_make ("audioconvert", "audio_convert");
  audio_resample = gst_element_factory_make ("audioresample", "audio_resample");
  audio_sink = gst_element_factory_make ("autoaudiosink", "audio_sink");
  video_queue = gst_element_factory_make ("queue", "video_queue");
  visual = gst_element_factory_make ("wavescope", "visual");
  video_convert = gst_element_factory_make ("videoconvert", "csp");
  video_sink = gst_element_factory_make ("autovideosink", "video_sink");

  // Create the empty pipeline 
  pipeline = gst_pipeline_new ("test-pipeline");

  if (!pipeline || !audio_source || !tee || !audio_queue || !audio_convert || !audio_resample || !audio_sink ||
      !video_queue || !visual || !video_convert || !video_sink) {
    g_printerr ("Not all elements could be created.\n");
    return -1;
  }

  // Configure elements 
  g_object_set (audio_source, "freq", 215.0f, NULL);
  g_object_set (visual, "shader", 0, "style", 1, NULL);

  // Link all elements that can be automatically linked because they have "Always" pads 
  gst_bin_add_many (GST_BIN (pipeline), audio_source, tee, audio_queue, audio_convert, audio_resample, audio_sink,
      video_queue, visual, video_convert, video_sink, NULL);
  if (gst_element_link_many (audio_source, tee, NULL) != TRUE ||
      gst_element_link_many (audio_queue, audio_convert, audio_resample, audio_sink, NULL) != TRUE ||
      gst_element_link_many (video_queue, visual, video_convert, video_sink, NULL) != TRUE) {
    g_printerr ("Elements could not be linked.\n");
    gst_object_unref (pipeline);
    return -1;
  }

  // Manually link the Tee, which has "Request" pads 
  tee_audio_pad = gst_element_get_request_pad (tee, "src_%u");
  g_print ("Obtained request pad %s for audio branch.\n", gst_pad_get_name (tee_audio_pad));
  queue_audio_pad = gst_element_get_static_pad (audio_queue, "sink");
  tee_video_pad = gst_element_get_request_pad (tee, "src_%u");
  g_print ("Obtained request pad %s for video branch.\n", gst_pad_get_name (tee_video_pad));
  queue_video_pad = gst_element_get_static_pad (video_queue, "sink");
  if (gst_pad_link (tee_audio_pad, queue_audio_pad) != GST_PAD_LINK_OK ||
      gst_pad_link (tee_video_pad, queue_video_pad) != GST_PAD_LINK_OK) {
    g_printerr ("Tee could not be linked.\n");
    gst_object_unref (pipeline);
    return -1;
  }
  gst_object_unref (queue_audio_pad);
  gst_object_unref (queue_video_pad);

  // Start playing the pipeline 
  gst_element_set_state (pipeline, GST_STATE_PLAYING);

  // Wait until error or EOS 
  bus = gst_element_get_bus (pipeline);
  msg = gst_bus_timed_pop_filtered (bus, GST_CLOCK_TIME_NONE, GST_MESSAGE_ERROR | GST_MESSAGE_EOS);

  // Release the request pads from the Tee, and unref them 
  gst_element_release_request_pad (tee, tee_audio_pad);
  gst_element_release_request_pad (tee, tee_video_pad);
  gst_object_unref (tee_audio_pad);
  gst_object_unref (tee_video_pad);

  // Free resources 
  if (msg != NULL)
    gst_message_unref (msg);
  gst_object_unref (bus);
  gst_element_set_state (pipeline, GST_STATE_NULL);

  gst_object_unref (pipeline);
  return 0;
}
```

#### 源码分析

```cpp
// Create the elements
audio_source = gst_element_factory_make("audiotestsrc", "audio_source");
tee = gst_element_factory_make("tee", "tee");
audio_queue = gst_element_factory_make("queue", "audio_queue");
audio_convert = gst_element_factory_make("audioconvert", "audio_convert");
audio_resample = gst_element_factory_make("audioresample", "audio_resample");
audio_sink = gst_element_factory_make("autoaudiosink", "audio_sink");
video_queue = gst_element_factory_make("queue", "video_queue");
visual = gst_element_factory_make("wavescope", "visual");
video_convert = gst_element_factory_make("videoconvert", "video_convert");
video_sink = gst_element_factory_make("autovideosink", "video_sink");
```

首先创建所需的Element：audiotestsrc会产生测试的音频波形数据，wavescope会将输入的音频数据转换为波形图像。audioconvert，audioresample，videoconvert保证了Pipeline中各个Element之间的数据可以相互兼容，使得Pipeline能够被正确的link起来，如果不需要对数据进行转换，这些Element会直接将数据发送到下一个Element，这种情况下的性能影响可以忽略不计。

```cpp
// Configure elements
g_object_set (audio_source, "freq", 215.0f, NULL);
g_object_set (visual, "shader", 0, "style", 1, NULL);
```

这些修改相应Element的参数，使得输出结果更直观。"freq"会设置audiotestsrc输出波形的频率为215Hz，设置"shader"和"style"使得波形更加连续。其他的参数可以通过gst-inspect查看。

```cpp
// Link all elements that can be automatically linked because they have "Always" pads
gst_bin_add_many (GST_BIN (pipeline), 
  audio_source, 
  tee, 
  audio_queue, 
  audio_convert, 
  audio_sink,
  video_queue, 
  visual, 
  video_convert, 
  video_sink, 
  NULL);
if (gst_element_link_many (audio_source, tee, NULL) != TRUE ||
    gst_element_link_many (audio_queue, audio_convert, audio_sink, NULL) != TRUE ||
    gst_element_link_many (video_queue, visual, video_convert, video_sink, NULL) != TRUE) {
  g_printerr ("Elements could not be linked.\n");
  gst_object_unref (pipeline);
  return -1;
}
```

这里我们使用gst_element_link_many将多个Element连接起来，需要注意的是，这里我们只连接了拥有Always Pad的Element。虽然gst_element_link_many() 能够在内部处理Request Pad的情况，但我们仍然需要单独释放Request Pad，如果直接使用此函数连接所有的Element，这样容易忘记释放Request Pad。因此我们使用下面的代码单独处理Request Pad。

```cpp
// Manually link the Tee, which has "Request" pads
tee_audio_pad = gst_element_get_request_pad (tee, "src_%u");
g_print ("Obtained request pad %s for audio branch.\n", gst_pad_get_name (tee_audio_pad));
queue_audio_pad = gst_element_get_static_pad (audio_queue, "sink");
tee_video_pad = gst_element_get_request_pad (tee, "src_%u");
g_print ("Obtained request pad %s for video branch.\n", gst_pad_get_name (tee_video_pad));
queue_video_pad = gst_element_get_static_pad (video_queue, "sink");
if (gst_pad_link (tee_audio_pad, queue_audio_pad) != GST_PAD_LINK_OK ||
    gst_pad_link (tee_video_pad, queue_video_pad) != GST_PAD_LINK_OK) {
  g_printerr ("Tee could not be linked.\n");
  gst_object_unref (pipeline);
  return -1;
}
gst_object_unref (queue_audio_pad);
gst_object_unref (queue_video_pad);
```

为了能够连接到Request Pad，我们需要主动的向Element取得相应的Pad。由于一个Element可以提供不同的Request Pad，所以我们需要指定所需的"Pad Template"，Element提供的Pad Template可以通过gst-inspect查看。从下面的结果可以发现，tee提供了2种类型的模板，"sink"和"src_%u"。

```cpp
$ gst-inspect-1.0  tee
...
Pad Templates:
  SRC template: 'src_%u'
    Availability: On request
      Has request_new_pad() function: gst_tee_request_new_pad
    Capabilities:
      ANY

  SINK template: 'sink'
    Availability: Always
    Capabilities:
      ANY
...
```

由于我们这里需要的是2个Source Pad，所以我们通过gst_element_get_request_pad (tee, "src_%u")获取两个Request Pad分别用于audio和video。queue的Sink Pad是Alwasy Pad，所以我们直接使用gst_element_get_static_pad 获取其Sink Pad。最后再通过gst_pad_link()将其连接起来，在gst_element_link()和gst_element_link_many()内部也是使用此函数连接两个Element的Pad。

***需要注意的是，我们通过Element获取到的Pad的引用计数会自动增加，因此我们需要调用gst_object_unref()释放相关的引用，对于Request Pad，我们需要在Pipeline执行完成后进行释放。***

```cpp
// Release the request pads from the Tee, and unref them 
gst_element_release_request_pad (tee, tee_audio_pad);
gst_element_release_request_pad (tee, tee_video_pad);
gst_object_unref (tee_audio_pad);
gst_object_unref (tee_video_pad);
```

除了播放完成后正常的资源释放外，我们还要对Request进行释放，需要首先调用gst_element_release_request_pad()，最后再释放相应的对象。

***

## 9 Appsrc及Appsink

GStreamer提供了多种方法使得应用程序与GStreamer Pipeline之间可以进行数据交互，我们这里介绍的是最简单的一种方式：appsrc与appsink。

- appsrc：

用于将应用程序的数据发送到Pipeline中。应用程序负责数据的生成，并将其作为GstBuffer传输到Pipeline中。
appsrc有2种模式，**拉模式和推模式**。在拉模式下，appsrc会在需要数据时，通过指定接口从应用程序中获取相应数据。在推模式下，则需要由应用程序主动将数据推送到Pipeline中，应用程序可以指定在Pipeline的数据队列满时是否阻塞相应调用，或通过监听enough-data和need-data信号来控制数据的发送。

- appsink：
  
用于从Pipeline中提取数据，并发送到应用程序中。
appsrc和appsink需要通过特殊的API才能与Pipeline进行数据交互，相应的接口可以查看官方文档，在编译的时候还需要连接gstreamer-app库。

### 9.1 GstBuffer

在GStreamer Pipeline中的plugin间传输的数据块被成为buffer，在GStreamer内部对应于GstBuffer。Buffer由Source Pad产生，并由Sink Pad消耗。一个Buffer只表示一块数据，不同的buffer可能包含不同大小，不同时间长度的数据。同时，某些Element中可能对Buffer进行拆分或合并，所以GstBuffer中可能包含不止一个内存数据，实际的内存数据在GStreamer系统中通过GstMemory对象进行描述，因此，GstBuffer可以包含多个GstMemory对象。
每个GstBuffer都有相应的时间戳以及时间长度，用于描述这个buffer的解码时间以及显示时间。

### 9.2 示例代码

本例在GStreamer基础教程08 - 多线程示例上进行扩展，首先使用appsrc替代audiotestsrc用于产生audio数据，另外增加一个新的分支，将tee产生的数据发送到应用程序，由应用程序决定如何处理收到的数据。Pipeline的示意图如下：

<center><img src="https://img2018.cnblogs.com/blog/1647252/201909/1647252-20190930103238628-1994366509.jpg"/></center>

```cpp
#include <gst/gst.h>
#include <gst/audio/audio.h>
#include <string.h>

#define CHUNK_SIZE 1024   // Amount of bytes we are sending in each buffer 
#define SAMPLE_RATE 44100 // Samples per second we are sending 

// Structure to contain all our information, so we can pass it to callbacks 
typedef struct _CustomData {
  GstElement *pipeline, *app_source, *tee, *audio_queue, *audio_convert1, *audio_resample, *audio_sink;
  GstElement *video_queue, *audio_convert2, *visual, *video_convert, *video_sink;
  GstElement *app_queue, *app_sink;

  guint64 num_samples;   // Number of samples generated so far (for timestamp generation) 
  gfloat a, b, c, d;     // For waveform generation 

  guint sourceid;        // To control the GSource 

  GMainLoop *main_loop;  // GLib's Main Loop 
} CustomData;

// This method is called by the idle GSource in the mainloop, to feed CHUNK_SIZE bytes into appsrc.
 * The idle handler is added to the mainloop when appsrc requests us to start sending data (need-data signal)
 * and is removed when appsrc has enough data (enough-data signal).
 
static gboolean push_data (CustomData *data) {
  GstBuffer *buffer;
  GstFlowReturn ret;
  int i;
  GstMapInfo map;
  gint16 *raw;
  gint num_samples = CHUNK_SIZE / 2; // Because each sample is 16 bits 
  gfloat freq;

  // Create a new empty buffer 
  buffer = gst_buffer_new_and_alloc (CHUNK_SIZE);

  // Set its timestamp and duration 
  GST_BUFFER_TIMESTAMP (buffer) = gst_util_uint64_scale (data->num_samples, GST_SECOND, SAMPLE_RATE);
  GST_BUFFER_DURATION (buffer) = gst_util_uint64_scale (num_samples, GST_SECOND, SAMPLE_RATE);

  // Generate some psychodelic waveforms 
  gst_buffer_map (buffer, &map, GST_MAP_WRITE);
  raw = (gint16 *)map.data;
  data->c += data->d;
  data->d -= data->c / 1000;
  freq = 1100 + 1000 * data->d;
  for (i = 0; i < num_samples; i++) {
    data->a += data->b;
    data->b -= data->a / freq;
    raw[i] = (gint16)(500 * data->a);
  }
  gst_buffer_unmap (buffer, &map);
  data->num_samples += num_samples;

  // Push the buffer into the appsrc 
  g_signal_emit_by_name (data->app_source, "push-buffer", buffer, &ret);

  // Free the buffer now that we are done with it 
  gst_buffer_unref (buffer);

  if (ret != GST_FLOW_OK) {
    // We got some error, stop sending data 
    return FALSE;
  }

  return TRUE;
}

// This signal callback triggers when appsrc needs data. Here, we add an idle handler
 * to the mainloop to start pushing data into the appsrc 
static void start_feed (GstElement *source, guint size, CustomData *data) {
  if (data->sourceid == 0) {
    g_print ("Start feeding\n");
    data->sourceid = g_idle_add ((GSourceFunc) push_data, data);
  }
}

// This callback triggers when appsrc has enough data and we can stop sending.
 * We remove the idle handler from the mainloop 
static void stop_feed (GstElement *source, CustomData *data) {
  if (data->sourceid != 0) {
    g_print ("Stop feeding\n");
    g_source_remove (data->sourceid);
    data->sourceid = 0;
  }
}

// The appsink has received a buffer 
static GstFlowReturn new_sample (GstElement *sink, CustomData *data) {
  GstSample *sample;

  // Retrieve the buffer 
  g_signal_emit_by_name (sink, "pull-sample", &sample);
  if (sample) {
    // The only thing we do in this example is print a * to indicate a received buffer 
    g_print ("*");
    gst_sample_unref (sample);
    return GST_FLOW_OK;
  }

  return GST_FLOW_ERROR;
}

// This function is called when an error message is posted on the bus 
static void error_cb (GstBus *bus, GstMessage *msg, CustomData *data) {
  GError *err;
  gchar *debug_info;

  // Print error details on the screen 
  gst_message_parse_error (msg, &err, &debug_info);
  g_printerr ("Error received from element %s: %s\n", GST_OBJECT_NAME (msg->src), err->message);
  g_printerr ("Debugging information: %s\n", debug_info ? debug_info : "none");
  g_clear_error (&err);
  g_free (debug_info);

  g_main_loop_quit (data->main_loop);
}

int main(int argc, char *argv[]) {
  CustomData data;
  GstPad *tee_audio_pad, *tee_video_pad, *tee_app_pad;
  GstPad *queue_audio_pad, *queue_video_pad, *queue_app_pad;
  GstAudioInfo info;
  GstCaps *audio_caps;
  GstBus *bus;

  // Initialize cumstom data structure 
  memset (&data, 0, sizeof (data));
  data.b = 1; // For waveform generation 
  data.d = 1;

  // Initialize GStreamer 
  gst_init (&argc, &argv);

  // Create the elements 
  data.app_source = gst_element_factory_make ("appsrc", "audio_source");
  data.tee = gst_element_factory_make ("tee", "tee");
  data.audio_queue = gst_element_factory_make ("queue", "audio_queue");
  data.audio_convert1 = gst_element_factory_make ("audioconvert", "audio_convert1");
  data.audio_resample = gst_element_factory_make ("audioresample", "audio_resample");
  data.audio_sink = gst_element_factory_make ("autoaudiosink", "audio_sink");
  data.video_queue = gst_element_factory_make ("queue", "video_queue");
  data.audio_convert2 = gst_element_factory_make ("audioconvert", "audio_convert2");
  data.visual = gst_element_factory_make ("wavescope", "visual");
  data.video_convert = gst_element_factory_make ("videoconvert", "video_convert");
  data.video_sink = gst_element_factory_make ("autovideosink", "video_sink");
  data.app_queue = gst_element_factory_make ("queue", "app_queue");
  data.app_sink = gst_element_factory_make ("appsink", "app_sink");

  // Create the empty pipeline 
  data.pipeline = gst_pipeline_new ("test-pipeline");

  if (!data.pipeline || !data.app_source || !data.tee || !data.audio_queue || !data.audio_convert1 ||
      !data.audio_resample || !data.audio_sink || !data.video_queue || !data.audio_convert2 || !data.visual ||
      !data.video_convert || !data.video_sink || !data.app_queue || !data.app_sink) {
    g_printerr ("Not all elements could be created.\n");
    return -1;
  }

  // Configure wavescope 
  g_object_set (data.visual, "shader", 0, "style", 0, NULL);

  // Configure appsrc 
  gst_audio_info_set_format (&info, GST_AUDIO_FORMAT_S16, SAMPLE_RATE, 1, NULL);
  audio_caps = gst_audio_info_to_caps (&info);
  g_object_set (data.app_source, "caps", audio_caps, "format", GST_FORMAT_TIME, NULL);
  g_signal_connect (data.app_source, "need-data", G_CALLBACK (start_feed), &data);
  g_signal_connect (data.app_source, "enough-data", G_CALLBACK (stop_feed), &data);

  // Configure appsink 
  g_object_set (data.app_sink, "emit-signals", TRUE, "caps", audio_caps, NULL);
  g_signal_connect (data.app_sink, "new-sample", G_CALLBACK (new_sample), &data);
  gst_caps_unref (audio_caps);

  // Link all elements that can be automatically linked because they have "Always" pads 
  gst_bin_add_many (GST_BIN (data.pipeline), data.app_source, data.tee, data.audio_queue, data.audio_convert1, data.audio_resample,
      data.audio_sink, data.video_queue, data.audio_convert2, data.visual, data.video_convert, data.video_sink, data.app_queue,
      data.app_sink, NULL);
  if (gst_element_link_many (data.app_source, data.tee, NULL) != TRUE ||
      gst_element_link_many (data.audio_queue, data.audio_convert1, data.audio_resample, data.audio_sink, NULL) != TRUE ||
      gst_element_link_many (data.video_queue, data.audio_convert2, data.visual, data.video_convert, data.video_sink, NULL) != TRUE ||
      gst_element_link_many (data.app_queue, data.app_sink, NULL) != TRUE) {
    g_printerr ("Elements could not be linked.\n");
    gst_object_unref (data.pipeline);
    return -1;
  }

  // Manually link the Tee, which has "Request" pads 
  tee_audio_pad = gst_element_get_request_pad (data.tee, "src_%u");
  g_print ("Obtained request pad %s for audio branch.\n", gst_pad_get_name (tee_audio_pad));
  queue_audio_pad = gst_element_get_static_pad (data.audio_queue, "sink");
  tee_video_pad = gst_element_get_request_pad (data.tee, "src_%u");
  g_print ("Obtained request pad %s for video branch.\n", gst_pad_get_name (tee_video_pad));
  queue_video_pad = gst_element_get_static_pad (data.video_queue, "sink");
  tee_app_pad = gst_element_get_request_pad (data.tee, "src_%u");
  g_print ("Obtained request pad %s for app branch.\n", gst_pad_get_name (tee_app_pad));
  queue_app_pad = gst_element_get_static_pad (data.app_queue, "sink");
  if (gst_pad_link (tee_audio_pad, queue_audio_pad) != GST_PAD_LINK_OK ||
      gst_pad_link (tee_video_pad, queue_video_pad) != GST_PAD_LINK_OK ||
      gst_pad_link (tee_app_pad, queue_app_pad) != GST_PAD_LINK_OK) {
    g_printerr ("Tee could not be linked\n");
    gst_object_unref (data.pipeline);
    return -1;
  }
  gst_object_unref (queue_audio_pad);
  gst_object_unref (queue_video_pad);
  gst_object_unref (queue_app_pad);

  // Instruct the bus to emit signals for each received message, and connect to the interesting signals 
  bus = gst_element_get_bus (data.pipeline);
  gst_bus_add_signal_watch (bus);
  g_signal_connect (G_OBJECT (bus), "message::error", (GCallback)error_cb, &data);
  gst_object_unref (bus);

  // Start playing the pipeline 
  gst_element_set_state (data.pipeline, GST_STATE_PLAYING);

  // Create a GLib Main Loop and set it to run 
  data.main_loop = g_main_loop_new (NULL, FALSE);
  g_main_loop_run (data.main_loop);

  // Release the request pads from the Tee, and unref them 
  gst_element_release_request_pad (data.tee, tee_audio_pad);
  gst_element_release_request_pad (data.tee, tee_video_pad);
  gst_element_release_request_pad (data.tee, tee_app_pad);
  gst_object_unref (tee_audio_pad);
  gst_object_unref (tee_video_pad);
  gst_object_unref (tee_app_pad);

  // Free resources 
  gst_element_set_state (data.pipeline, GST_STATE_NULL);
  gst_object_unref (data.pipeline);
  return 0;
}
```

**源码分析**
与上一示例相同，首先对所需Element进行实例化，同时将Element的Always Pad连接起来，并与tee的Request Pad相连。此外我们还对appsrc及appsink进行了相应的配置：

```cpp
// Configure appsrc
gst_audio_info_set_format (&info, GST_AUDIO_FORMAT_S16, SAMPLE_RATE, 1, NULL);
audio_caps = gst_audio_info_to_caps (&info);
g_object_set (data.app_source, "caps", audio_caps, NULL);
g_signal_connect (data.app_source, "need-data", G_CALLBACK (start_feed), &data);
g_signal_connect (data.app_source, "enough-data", G_CALLBACK (stop_feed), &data);
```

首先需要对appsrc的caps进行设定，指定我们会产生何种类型的数据，这样GStreamer会在连接阶段检查后续的Element是否支持此数据类型。这里的 caps必须为GstCaps对象，我们可以通过gst_caps_from_string()或gst_audio_info_to_caps()得到相应的实例。
我们同时监听了"need-data"与"enough-data"事件，这2个事件由appsrc在需要数据和缓冲区满时触发，使用这2个事件可以方便的控制何时产生数据与停止数据。

```cpp
// Configure appsink
g_object_set (data.app_sink, "emit-signals", TRUE, "caps", audio_caps, NULL);
g_signal_connect (data.app_sink, "new-sample", G_CALLBACK (new_sample), &data);
gst_caps_unref (audio_caps);
```

对于appsink，我们监听"new-sample"事件，用于appsink在收到数据时的处理。同时我们需要显示的使能"new-sample"事件。因为这个事件默认是处于关闭状态。
Pipeline的播放，停止及消息处理与其他示例相同，不再复述。我们接下来将查看我们监听事件的回调函数。

```cpp
// This signal callback triggers when appsrc needs data. Here, 
// we add an idle handler to the mainloop to start pushing data into the appsrc 
static void start_feed (GstElement *source, guint size, CustomData *data) {
  if (data->sourceid == 0) {
    g_print("Start feeding\n");
    data->sourceid = g_idle_add((GSourceFunc) push_data, data);
  }
}
```

appsrc会在其内部的数据队列即将缺乏数据时调用此回调函数，这里我们通过注册一个GLib的idle函数来向appsrc填充数据，GLib的主循环在"idle"状态时会循环调用push_data，用于向appsrc填充数据，这只是一种向appsrc填充数据的方式，我们可以在任意线程中向appsrc填充数据。
我们保存了g_idle_add()的返回值，以便后续用于停止数据写入。

```cpp
// This callback triggers when appsrc has enough data and we can stop sending.
// We remove the idle handler from the mainloop
static void stop_feed (GstElement *source, CustomData *data) {
  if (data->sourceid != 0) {
    g_print ("Stop feeding\n");
    g_source_remove (data->sourceid);
    data->sourceid = 0;
  }
}
```

stop_feed函数会在appsrc内部数据队列满时被调用。这里我们仅仅通过g_source_remove() 将先前注册的idle处理函数从GLib的主循环中移除（idle处理函数是被实现为一个GSource）。

```cpp
// This method is called by the idle GSource in the mainloop, to feed CHUNK_SIZE bytes into appsrc.
// The ide handler is added to the mainloop when appsrc requests us to start sending data (need-data signal)
// and is removed when appsrc has enough data (enough-data signal).
static gboolean push_data (CustomData *data) {
  GstBuffer *buffer;
  GstFlowReturn ret;
  int i;
  gint16 *raw;
  gint num_samples = CHUNK_SIZE / 2; // Because each sample is 16 bits
  gfloat freq;

  // Create a new empty buffer
  buffer = gst_buffer_new_and_alloc (CHUNK_SIZE);

  // Set its timestamp and duration
  GST_BUFFER_TIMESTAMP (buffer) = gst_util_uint64_scale (data->num_samples, GST_SECOND, SAMPLE_RATE);
  GST_BUFFER_DURATION (buffer) = gst_util_uint64_scale (num_samples, GST_SECOND, SAMPLE_RATE);

  // Generate some psychodelic waveforms
  raw = (gint16 *)GST_BUFFER_DATA (buffer);
```

此函数会将真实的数据填充到appsrc的队列中，首先通过gst_buffer_new_and_alloc()分配一个GstBuffer对象，然后通过产生的采样数量计算这块buffer所对应的时间戳及事件长度。
gst_util_uint64_scale(val, num, denom)函数用于计算val * num / denom，此函数内部会对数据范围进行检测，避免溢出的问题。
GstBuffer的数据指针可以通过GST_BUFFER_DATA宏获取，在写数据时需要避免超出内存分配大小。本文将跳过audio波形生成的函数，其内容不是本文介绍的重点。

```cpp
// Push the buffer into the appsrc
g_signal_emit_by_name(data->app_source, "push-buffer", buffer, &ret);

// Free the buffer now that we are done with it
gst_buffer_unref(buffer);
```

在我们准备好数据后，我们这里通过"push-buffer"事件通知appsrc数据就绪，并释放我们申请的buffer。 另外一种方式为通过调用gst_app_src_push_buffer()向appsrc填充数据，这种方式就需要在编译时链接gstreamer-app-1.0库，同时gst_app_src_push_buffer()会接管GstBuffer的所有权，调用者无需释放buffer。在所有数据都发送完成后，我们可以调用gst_app_src_end_of_stream()向Pipeline写入EOS事件。

```cpp
// The appsink has received a buffer
static GstFlowReturn new_sample (GstElement *sink, CustomData *data) {
  GstSample *sample;
  // Retrieve the buffer
  g_signal_emit_by_name (sink, "pull-sample", &sample);
  if (sample) {
    // The only thing we do in this example is print a to indicate a received buffer
    g_print ("*");
    gst_sample_unref (sample);
    return GST_FLOW_OK;
  }
  return GST_FLOW_ERROR;
}
```

当appsink得到数据时会调用new_sample函数，我们使用"pull-sample"信号提取sample，这里仅输出一个"*"表明此函数被调用。除此以外，我们同样可以使用gst_app_sink_pull_sample()获取Sample。得到GstSample之后，我们可以通过gst_sample_get_buffer()得到Sample中所包含的GstBuffer，在使用GST_BUFFER_DATA，GST_BUFFER_SIZE等接口访问其中的数据。使用完成后，得到GstSample同样需要通过gst_sample_unref()进行释放。
需要注意的是，在某些Pipeline里得到的GstBuffer可能会和source中填充的GstBuffer有所差异，因为Pipeline中的Element可能对Buffer进行各种处理。(此例中不存在这种情况，因为在appsrc与appsink之间只存在一个tee)
