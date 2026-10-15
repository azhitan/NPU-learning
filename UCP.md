# 总览

- `UCP`：统一计算平台，定义了一套统一的异构编程接口，供应用程序接口（Application Programming Interface，以下简称API）实现对计算平台上所有资源的调用。 UCP将SOC上的功能硬件抽象出来并进行封装，对外提供基于功能的API，用于创建相应的UCP任务（如VP算子任务），并支持设置硬件Backend提交至UCP调度器， UCP可基于硬件资源，完成SOC上任务的统一调度。 具体提供了以下几个功能： 视觉处理(Vision Process)、 神经网络模型推理（Neural Network）、 高性能计算库（High Performance Library）、 自定义算子插件开发。

  https://doc.oe.horizon.auto/static/image/diagram_arch.c7232816f1.png<img width="2021" height="1209" alt="image" src="https://github.com/user-attachments/assets/a7b1f3a3-6a6b-4615-9191-0b1086f3c8dd" />


- UCP 优势：
  - 高度抽象，对于单算子的功能，不会收到硬件带来的干扰，可以通过制定backend选择需要执行的硬件，降低硬件部署难度。
  - 集成度高，一套接口完成所有需求开发。
  
# 工作模式

- 两种工作模式
  - 直连模式
  - 中继模式，中继模式下，支持多进程任务的统一调度，使用中继模式前，首先启动 `ucp_service`，service文件位于 `deps_aarch64/ucp/bin/service/` 路径下， 并通过设置环境变量 `HB_UCP_ENABLE_RELAY_MODE=true` 来启用Relay模式，使得用户进程可以通过中继服务进行通信。 无论是直连模式还是中继模式，UCP接口的调用方式保持一致，不会对编程逻辑产生影响。您可以根据实际需求灵活选择这两种模式，以满足系统在性能和灵活性方面的要求。

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

创建任务时，需要提前将`taskHandle`参数初始化为`nullptr`； 提交UCP任务（hbUCPSubmitTask）后，在线程指定位置执行接口（hbUCPWaitTaskDone），等待任务完成。

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
  
