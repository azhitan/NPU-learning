# 总览

- `UCP`：统一计算平台，定义了一套统一的异构编程接口，供应用程序接口（Application Programming Interface，以下简称API）实现对计算平台上所有资源的调�[...]

  https://doc.oe.horizon.auto/static/image/diagram_arch.c7232816f1.png<img width="2021" height="1209" alt="image" src="https://github.com/user-attachments/assets/a7b1f3a3-6a6b-4615-9191-0b1086f3c8dd" [...]


- UCP 优势：
  - 高度抽象，对于单算子的功能，不会收到硬件带来的干扰，可以通过制定backend选择需要执行的硬件，降低硬件部署难度。
  - 集成度高，一套接口完成所有需求开发。
  
# 工作模式

- 两种工作模式
  - 直连模式
  - 中继模式，中继模式下，支持多进程任务的统一调度，使用中继模式前，首先启动 `ucp_service`，service文件位于 `deps_aarch64/ucp/bin/service/` 路径下， ��[...]

# 接口使用流程

## 任务执行三种方式

### 同步执行

创建任务时，将taskHandle对应的参数传入nullptr，可立即同步执行

```
#include "hobot/hb_ucp.h"
#include "hobot/vp/hb_vp_rotate.h"

int main() {
    // step1: execute a rotate task
    hbVPImage src_img;  //do something to init src image
    hbVPImage dst_img;  //do something to init dst image
    hbVPRotate(nullptr, &dst_img, &src_img, HB_VP_ROTATE_90_CLOCKWISE);
    return 0;
}
```

### 异步执行

创建任务时，需要提前将`taskHandle`参数初始化为`nullptr`； 提交UCP任务（hbUCPSubmitTask）后，在线程指定位置执行接口（hbUCPWaitTaskDone），等待任务完成�[...]

```
#include "hobot/hb_ucp.h"
#include "hobot/vp/hb_vp_rotate.h"

int main() {
    // step1: create a rotate task
    hbUCPTaskHandle_t task_handle{nullptr};
    hbVPImage src_img;  //do something to init src image
    hbVPImage dst_img;  //do something to init dst image
    hbVPRotate(&task_handle, &dst_img, &src_img, HB_VP_ROTATE_90_CLOCKWISE);

    // step2：complete the commit and execution of the task
    hbUCPSchedParam sched_param;
    sched_param.backend = HB_UCP_DSP_CORE_0;
    sched_param.priority = 0;
    hbUCPSubmitTask(task_handle, &sched_param);
    hbUCPWaitTaskDone(task_handle, 100);
    hbUCPReleaseTask(task_handle);
    return 0;
}
```

### 注册回调函数执行

创建任务时，需要提前将taskHandle参数初始化为nullptr； 回调函数需要在任务提交（hbUCPSubmitTask）前注册，否则注册接口会报错。

```
#include "hobot/hb_ucp.h"
#include "hobot/vp/hb_vp_rotate.h"

typedef struct {
  std::mutex mutex;
  std::condition_variable cv;
  int32_t status;
} UserData;

void CallBack(hbUCPTaskHandle_t handle, int32_t status, void *userdata) {
  auto data = static_cast<UserData *>(userdata);
  data->status = status;
  data->cv.notify_all();
}

void ProcessThread(void *userdata) {
  auto data = static_cast<UserData *>(userdata);
  {
    std::unique_lock<std::mutex> lk{data->mutex};
    data->cv.wait(lk);
  }
  // do something here
}

int main() {
    // step1: create a rotate task
    hbUCPTaskHandle_t task_handle{nullptr};
    hbVPImage src_img;  //do something to init src image
    hbVPImage dst_img;  //do something to init dst image
    hbVPRotate(&task_handle, &dst_img, &src_img, HB_VP_ROTATE_90_CLOCKWISE);

    // step2: create userdata and custom process thread
    UserData userdata;
    userdata.status = 0;
    std::thread worker(ProcessThread, &userdata);

    // step3：set callback function before submit task
    hbUCPSetTaskDoneCb(task_handle, CallBack, &userdata);

    // step4: submit task
    hbUCPSchedParam sched_param;
    HB_UCP_INITIALIZE_SCHED_PARAM(&sched_param);
    sched_param.backend = HB_UCP_DSP_CORE_0;

    hbUCPSubmitTask(task_handle, &sched_param);

    worker.join();
    hbUCPReleaseTask(task_handle);
    return 0;
}
```
- UCP任务调度统一按照优先级调度策略，提交任务时可指定任务执行的优先级。
- 同步执行不支持配置任务的控制参数和后端选择，UCP会根据当前任务的可执行backend，以及硬件负载信息，选择合适的硬件执行。
- 异步执行支持配置任务的控制参数和后端选择，不指定则基于各后端负载均衡选择。
  

## 概念与模型部署

| 概念 | 模型部署 |
| --- | --- |
| 上板模型 | hbm |
| 对照基准 | quantized_model.bc（含 quantized_removed_model.bc） |
| 运行时框架 | UCP |
| 推理接口 | hbDNNInferV2 / hbUCPSubmitTask / hbUCPWaitTaskDone |
| 内存申请 | hbUCPMalloc / hbUCPMallocCached |
| 张量属性 | hbDNNTensorProperties（validShape / stride / alignedByteSize） |
| 性能评测 | hrt_model_exec perf |
| 精度评测 | hrt_model_exec infer / hbm_infer |
| 资源监控 | hrt_ucp_monitor |
| 调度追踪 | UCP Trace |
