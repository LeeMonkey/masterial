# ONNX转TensorRT  

## 将ONNX模型加载到TensorRT网络中  

onnx模型可以通过ONNX parser加载到TensorRT网络。这个parser利用两个参数进行初始化，分别为将要写入模型的TensorRT网络和logger对象

```c++
    auto parser = nvonnxparser::createParse(*network, sample::gLogger.getTRTLogger);
```

然后，将ONNX模型和日志级别一起传递到parser中

```c++
    if (!parser->parseFromFile(model_file, static_cast<int>(sample::gLogger.getReportableSeverity())))
    {
        string msg("failed to parse onnx file");
        sample::gLogger->log(nvinfer1::ILogger::Severity::kERROR, msg.c_str());
        exit(EXIT_FAILURE);
    }
```

可通过下面方法查看网络其他信息

```c++
    parser->reportParsingInfo();
```

通过解析网络构建TensorRT网络后，TensorRT engine就可以被创建用于运行推理

## 创建engine

创建engine之前，先生成一个builder，初始化builder需要传入一个logger对象，该logger是为TensorRT创建的用来输出日志信息：`IBuilder* builder = createInferBuilder(sample::gLogger);`

接着，从构建完成的网络中创建engine：`nvinfer1::ICudaEngine* engine = builder->buildCudaEngine(*network);`

创建engine后，验证engine运行结果是否正确。
