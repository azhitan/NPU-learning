# 总览

- UCP：统一计算平台，定义了一套统一的异构编程接口，供应用程序接口（Application Programming Interface，以下简称API）实现对计算平台上所有资源的调用。 UCP将SOC上的功能硬件抽象出来并进行封装，对外提供基于功能的API，用于创建相应的UCP任务（如VP算子任务），并支持设置硬件Backend提交至UCP调度器， UCP可基于硬件资源，完成SOC上任务的统一调度。 具体提供了以下几个功能： 视觉处理(Vision Process)、 神经网络模型推理（Neural Network）、 高性能计算库（High Performance Library）、 自定义算子插件开发。

- UCP 优势：
  - 高度抽象，对于单算子的功能，不会收到硬件带来的干扰，可以通过制定backend选择需要执行的硬件，降低硬件部署难度。
  - 集成度高，一套接口完成所有需求开发。
  
