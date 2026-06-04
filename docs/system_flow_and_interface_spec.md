# 课前拍照打卡系统总流程与接口规范

## 1. 文档目的

本文档用于说明课前拍照打卡系统的整体实现流程、模块拆分、模块内部职责、模块之间的接口规范、数据结构、错误码、数据库读写约定和后续扩展方向。

当前阶段系统按单机板端方案设计，优先实现两个基础能力：

1. 大图人头数统计：输入课堂大图，检测人头或人脸位置，输出人数统计结果。
2. 单张人脸匹配：输入单张清晰人脸，和本地已注册人脸库匹配，输出识别结果。

完整课堂大图多人脸识别打卡属于后续扩展功能。当前文档会预留接口，但不会把第一阶段目标复杂化。

## 2. 设计原则

1. 单机优先：程序、模型、配置、数据库都部署在 IMX6ULL_ALPHA 板端。
2. 先命令行，后界面：第一阶段使用命令行运行，方便调试、答辩和移植。
3. 模块解耦：摄像头、模型推理、后处理、数据库、业务流程分离。
4. 模型可替换：接口层不绑定具体模型，ncnn 和 TFLite 通过适配层切换。
5. 数据可追溯：每次检测、匹配都保存输入图片路径、结果、阈值、模型版本和时间。
6. 板端友好：避免重复加载模型，避免保存过多大文件，避免不必要的内存拷贝。

## 3. 默认技术决策

当前先采用以下默认方案，后续如果实测性能不满足，再做调整。

| 项目 | 默认方案 | 原因 |
| --- | --- | --- |
| 开发语言 | C++17 | 适合嵌入式 Linux，便于调用 OpenCV、SQLite、ncnn |
| 构建系统 | CMake | 方便 PC 编译和交叉编译 |
| 图像处理 | OpenCV | 摄像头采集、图片读写、resize、画框方便 |
| 推理框架 | ncnn 优先，TFLite 预留 | ncnn 对 ARM CPU 友好，部署文件简单 |
| 本地数据库 | SQLite | 单机部署简单，不需要数据库服务 |
| 配置格式 | JSON | 结构清晰，便于维护 |
| 第一阶段运行方式 | 命令行工具 | 降低 UI 干扰，先跑通核心功能 |
| 时间格式 | 本地时间字符串 `YYYY-MM-DD HH:MM:SS` | 便于查看数据库记录 |
| 图片路径 | 相对项目运行目录保存 | 方便板端迁移 |

## 4. 系统总体架构

```text
                 +-----------------------------+
                 |        attendance_app       |
                 |  command line entry point   |
                 +--------------+--------------+
                                |
                                v
                 +-----------------------------+
                 |          AppContext         |
                 | config / db / model paths   |
                 +--+-----------+-----------+--+
                    |           |           |
        +-----------+           |           +----------------+
        v                       v                            v
+---------------+       +---------------+            +----------------+
| CameraCapture |       | HeadCounter   |            | FaceMatchFlow  |
| V4L2/OpenCV   |       | large image   |            | register/match |
+-------+-------+       +-------+-------+            +--------+-------+
        |                       |                             |
        v                       v                             v
+---------------+       +---------------+            +----------------+
| ImageUtils    |       | HeadDetector  |            | FaceDetector   |
| read/save     |       | model infer   |            | single face    |
| draw/resize   |       +-------+-------+            +--------+-------+
+---------------+               |                             |
                                v                             v
                        +---------------+            +----------------+
                        | Postprocess   |            | FaceAligner    |
                        | decode / NMS  |            | crop / align   |
                        +-------+-------+            +--------+-------+
                                |                             |
                                v                             v
                        +---------------+            +----------------+
                        | AttendanceDB  |<-----------| FeatureExtract |
                        | SQLite write  |            | embedding      |
                        +---------------+            +--------+-------+
                                                              |
                                                              v
                                                     +----------------+
                                                     | FaceMatcher    |
                                                     | cosine search  |
                                                     +----------------+
```

## 5. 程序运行模式

主程序 `attendance_app` 负责解析命令行参数，并根据 `--mode` 调用不同业务流程。

| 模式 | 功能 | 当前阶段 |
| --- | --- | --- |
| `capture` | 摄像头拍照并保存图片 | 可选实现 |
| `head_count` | 大图人头数统计 | 必须实现 |
| `register_face` | 注册学生人脸特征 | 必须实现 |
| `face_match` | 单张人脸匹配 | 必须实现 |
| `init_db` | 初始化数据库 | 必须实现 |
| `full_attendance` | 大图多人脸识别打卡 | 后续扩展 |

### 5.1 命令行规范

```bash
./attendance_app --mode init_db
./attendance_app --mode capture --output data/images/capture_001.jpg
./attendance_app --mode head_count --image data/images/classroom.jpg
./attendance_app --mode register_face --student-no 20240001 --name zhangsan --class-name class01 --image data/faces/zhangsan.jpg
./attendance_app --mode face_match --image data/images/test_face.jpg
```

### 5.2 通用命令行参数

| 参数 | 必填 | 说明 |
| --- | --- | --- |
| `--mode` | 是 | 运行模式 |
| `--config` | 否 | 主配置文件路径，默认 `config/app_config.json` |
| `--image` | 部分模式必填 | 输入图片路径 |
| `--output` | 否 | 输出图片路径 |
| `--save-result-image` | 否 | 是否保存带框结果图 |
| `--verbose` | 否 | 输出详细日志 |

## 6. 总流程设计

## 6.1 程序启动流程

所有模式共用启动流程。

```text
main()
  -> parse command line
  -> load app_config.json
  -> load camera_config.json
  -> load model_config.json
  -> create AppContext
  -> init log module
  -> ensure data directories exist
  -> open SQLite database
  -> dispatch by mode
```

### 6.1.1 启动阶段检查项

| 检查项 | 失败处理 |
| --- | --- |
| 配置文件存在且可解析 | 返回 `CONFIG_ERROR` |
| 数据目录可写 | 返回 `FILE_IO_ERROR` |
| 数据库可打开 | 返回 `DATABASE_ERROR` |
| 模型文件存在 | 进入对应模式时返回 `MODEL_LOAD_ERROR` |
| 输入图片存在 | 返回 `INVALID_ARGUMENT` 或 `FILE_IO_ERROR` |

## 6.2 大图人头数统计流程

### 6.2.1 流程说明

```text
输入图片路径或摄像头采集
  -> 读取原图 cv::Mat
  -> 检查图片是否为空
  -> resize / letterbox 到模型输入尺寸
  -> 模型推理
  -> 解码模型输出
  -> 置信度过滤
  -> NMS 去重
  -> 坐标映射回原图
  -> 统计有效目标数量
  -> 绘制检测框
  -> 保存结果图片
  -> 写入 head_count_records
  -> 输出 JSON 或终端结果
```

### 6.2.2 顺序调用关系

```text
HeadCountFlow
  -> ImageUtils::ReadImage()
  -> HeadCounter::Count()
     -> HeadDetector::Detect()
        -> HeadPreprocessor::Preprocess()
        -> ModelEngine::Forward()
        -> DetectPostprocess::Decode()
        -> DetectPostprocess::Nms()
     -> ImageUtils::DrawDetections()
  -> AttendanceDB::InsertHeadCountRecord()
```

### 6.2.3 输入

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `image_path` | string | 待检测课堂大图 |
| `save_result_image` | bool | 是否保存带框图片 |
| `result_image_path` | string | 结果图片路径，可为空 |
| `confidence_threshold` | float | 检测置信度阈值 |
| `nms_threshold` | float | NMS 阈值 |

### 6.2.4 输出

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `head_count` | int | 最终统计人数 |
| `detections` | array | 检测框列表 |
| `result_image_path` | string | 带框图片路径 |
| `elapsed_ms` | int | 处理耗时 |
| `record_id` | int64 | 数据库记录 ID |

## 6.3 单张人脸注册流程

### 6.3.1 流程说明

```text
输入学生信息和人脸图片
  -> 检查 student_no 是否已存在
  -> 读取图片
  -> 人脸检测
  -> 如果多张人脸，选择最大人脸
  -> 人脸关键点定位
  -> 人脸对齐为固定尺寸
  -> 特征提取
  -> 特征向量 L2 归一化
  -> 保存学生信息
  -> 保存人脸图片和特征向量
  -> 输出注册结果
```

### 6.3.2 顺序调用关系

```text
FaceRegisterFlow
  -> FaceRepository::FindStudentByNo()
  -> ImageUtils::ReadImage()
  -> FaceDetector::Detect()
  -> FaceAligner::Align()
  -> FaceFeatureExtractor::Extract()
  -> FaceMatcher::Normalize()
  -> FaceRepository::InsertStudent()
  -> FaceRepository::InsertFaceTemplate()
```

### 6.3.3 输入

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `student_no` | string | 学号，唯一 |
| `name` | string | 姓名 |
| `class_name` | string | 班级 |
| `image_path` | string | 注册人脸图片 |
| `overwrite` | bool | 学号已存在时是否覆盖 |

### 6.3.4 输出

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `student_id` | int64 | 数据库学生 ID |
| `template_id` | int64 | 人脸模板 ID |
| `face_image_path` | string | 保存的人脸图片路径 |
| `feature_dim` | int | 特征维度 |
| `message` | string | 注册说明 |

## 6.4 单张人脸匹配流程

### 6.4.1 流程说明

```text
输入测试人脸图片
  -> 读取图片
  -> 人脸检测
  -> 如果多张人脸，选择最大人脸
  -> 人脸对齐
  -> 特征提取
  -> 特征向量归一化
  -> 从数据库读取所有注册特征
  -> 逐个计算余弦相似度
  -> 选择最高分
  -> 根据阈值判断 matched / unknown
  -> 写入 face_match_records
  -> 输出匹配结果
```

### 6.4.2 顺序调用关系

```text
FaceMatchFlow
  -> ImageUtils::ReadImage()
  -> FaceDetector::Detect()
  -> FaceAligner::Align()
  -> FaceFeatureExtractor::Extract()
  -> FaceRepository::LoadAllTemplates()
  -> FaceMatcher::Match()
  -> FaceRepository::InsertFaceMatchRecord()
```

### 6.4.3 输入

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `image_path` | string | 待匹配人脸图片 |
| `threshold` | float | 匹配阈值 |
| `save_aligned_face` | bool | 是否保存对齐后人脸 |

### 6.4.4 输出

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `matched` | bool | 是否匹配成功 |
| `student_id` | int64 | 匹配到的学生 ID |
| `student_no` | string | 学号 |
| `name` | string | 姓名 |
| `score` | float | 最高相似度 |
| `threshold` | float | 使用阈值 |
| `status` | string | `matched`、`unknown`、`failed` |
| `record_id` | int64 | 识别记录 ID |

## 6.5 后续完整打卡流程

完整流程在第一阶段之后实现。

```text
课堂大图拍照
  -> 大图人脸检测
  -> 裁剪每个人脸
  -> 对每个人脸做对齐和特征提取
  -> 对每个人脸做数据库匹配
  -> 同一学生结果去重
  -> 生成一次课堂 session
  -> 保存每个学生的到课状态
  -> 保存未知人脸和低置信度人脸
```

为避免第一阶段过度复杂，当前仅预留 `full_attendance` 模式和数据库扩展方向。

## 7. 模块详细设计

## 7.1 App 模块

### 7.1.1 职责

| 文件 | 职责 |
| --- | --- |
| `include/app/app_config.h` | 配置结构定义 |
| `src/app/app_config.cpp` | JSON 配置读取、默认值填充 |
| `include/app/app_context.h` | 全局运行上下文 |
| `src/app/app_context.cpp` | 初始化目录、数据库、公共资源 |
| `src/main.cpp` | 命令行入口和模式分发 |

### 7.1.2 AppContext 内容

```cpp
struct AppContext {
    AppConfig app_config;
    CameraConfig camera_config;
    ModelConfig model_config;
    std::shared_ptr<AttendanceDB> db;
    std::shared_ptr<Logger> logger;
};
```

### 7.1.3 约束

1. `AppContext` 只保存公共依赖，不直接实现业务逻辑。
2. 模型对象由具体业务模块懒加载或在流程开始时加载。
3. 数据库连接在程序启动时打开，程序退出时关闭。

## 7.2 Config 模块

### 7.2.1 app_config.json

```json
{
  "data_root": "data",
  "database_path": "data/database/attendance.db",
  "log_path": "data/logs/attendance.log",
  "save_result_image": true,
  "timezone": "Asia/Shanghai"
}
```

### 7.2.2 camera_config.json

```json
{
  "device": "/dev/video0",
  "width": 1280,
  "height": 720,
  "fps": 15,
  "format": "MJPEG",
  "warmup_frames": 5
}
```

### 7.2.3 model_config.json

```json
{
  "engine": "ncnn",
  "head_detect": {
    "param": "models/head_detect/head_detect.param",
    "bin": "models/head_detect/head_detect.bin",
    "input_width": 416,
    "input_height": 416,
    "confidence_threshold": 0.4,
    "nms_threshold": 0.45,
    "model_version": "head_detect_v1"
  },
  "face_detect": {
    "param": "models/face_recognition/face_detect.param",
    "bin": "models/face_recognition/face_detect.bin",
    "input_width": 320,
    "input_height": 320,
    "confidence_threshold": 0.6,
    "nms_threshold": 0.4,
    "model_version": "face_detect_v1"
  },
  "face_feature": {
    "param": "models/face_recognition/face_feature.param",
    "bin": "models/face_recognition/face_feature.bin",
    "input_width": 112,
    "input_height": 112,
    "feature_dim": 128,
    "match_threshold": 0.75,
    "model_version": "face_feature_v1"
  }
}
```

### 7.2.4 配置读取规则

1. 配置缺失时使用默认值，但模型路径、数据库路径不能缺失。
2. 命令行参数优先级高于配置文件。
3. 阈值必须在 `[0, 1]` 范围内，否则返回 `CONFIG_ERROR`。
4. 文件路径统一使用 `/`，在 Windows 调试时由 C++ 标准库兼容处理。

## 7.3 Camera 模块

### 7.3.1 职责

1. 打开摄像头设备。
2. 设置分辨率、帧率、格式。
3. 采集一帧图片。
4. 保存拍照图片。
5. 出错时返回明确错误码。

### 7.3.2 接口定义

```cpp
struct CameraConfig {
    std::string device;
    int width;
    int height;
    int fps;
    std::string format;
    int warmup_frames;
};

struct CaptureResult {
    cv::Mat image;
    std::string saved_path;
    int width;
    int height;
    std::string captured_at;
};

class CameraCapture {
public:
    Status Open(const CameraConfig& config);
    Result<CaptureResult> CaptureOne(const std::string& save_path);
    void Close();
};
```

### 7.3.3 实现约定

1. PC 调试阶段可使用 OpenCV `VideoCapture`。
2. 板端如果 OpenCV 摄像头采集不稳定，再切换为 V4L2 实现。
3. 摄像头打开后先丢弃 `warmup_frames` 帧，降低曝光不稳定影响。
4. 保存图片默认使用 JPEG，质量参数建议 `85`。

## 7.4 ImageUtils 模块

### 7.4.1 职责

1. 图片读取和保存。
2. 图片尺寸检查。
3. resize 和 letterbox。
4. 检测框绘制。
5. 图片路径生成。

### 7.4.2 接口定义

```cpp
struct LetterboxInfo {
    float scale;
    int pad_x;
    int pad_y;
    int resized_width;
    int resized_height;
};

Result<cv::Mat> ReadImage(const std::string& path);
Status SaveImage(const std::string& path, const cv::Mat& image);
Result<cv::Mat> ResizeWithLetterbox(
    const cv::Mat& image,
    int target_width,
    int target_height,
    LetterboxInfo* info
);
cv::Rect MapBoxToOriginal(
    const cv::Rect2f& model_box,
    const LetterboxInfo& info,
    int original_width,
    int original_height
);
void DrawDetections(cv::Mat* image, const std::vector<Detection>& detections);
```

### 7.4.3 图片坐标约定

1. 原点在左上角。
2. `x` 向右增大，`y` 向下增大。
3. 检测框格式默认使用 `x, y, width, height`。
4. 数据库存储整数坐标，模型内部可使用浮点坐标。

## 7.5 ModelEngine 模块

### 7.5.1 职责

为 ncnn 或 TFLite 提供统一推理接口，让业务模块不直接依赖具体推理框架。

### 7.5.2 接口定义

```cpp
struct Tensor {
    std::vector<int> shape;
    std::vector<float> data;
};

struct ModelInput {
    std::string name;
    Tensor tensor;
};

struct ModelOutput {
    std::string name;
    Tensor tensor;
};

class ModelEngine {
public:
    virtual ~ModelEngine() = default;
    virtual Status Load(const ModelLoadOptions& options) = 0;
    virtual Result<std::vector<ModelOutput>> Forward(
        const std::vector<ModelInput>& inputs
    ) = 0;
};
```

### 7.5.3 第一阶段简化方案

如果时间紧，可以先不抽象 `ModelEngine`，直接在 `HeadDetector`、`FaceDetector`、`FaceFeatureExtractor` 中调用 ncnn。  
但头文件接口仍然建议保持稳定，避免后续切换模型时影响业务流程。

## 7.6 HeadCount 模块

### 7.6.1 文件职责

| 文件 | 职责 |
| --- | --- |
| `include/head_count/head_detector.h` | 人头检测模型接口 |
| `src/head_count/head_detector.cpp` | 模型加载、预处理、推理 |
| `include/head_count/detect_postprocess.h` | 检测后处理接口 |
| `src/head_count/detect_postprocess.cpp` | 输出解码、阈值过滤、NMS |
| `include/head_count/head_counter.h` | 人头统计业务接口 |
| `src/head_count/head_counter.cpp` | 串联检测、统计、保存结果 |

### 7.6.2 核心数据结构

```cpp
struct Detection {
    int class_id;
    std::string class_name;
    float score;
    float x;
    float y;
    float width;
    float height;
};

struct HeadCountRequest {
    std::string image_path;
    bool save_result_image;
    std::string result_image_path;
    float confidence_threshold;
    float nms_threshold;
};

struct HeadCountResult {
    int head_count;
    std::vector<Detection> detections;
    std::string image_path;
    std::string result_image_path;
    int elapsed_ms;
    int64_t record_id;
};
```

### 7.6.3 HeadDetector 接口

```cpp
class HeadDetector {
public:
    Status Load(const HeadDetectConfig& config);
    Result<std::vector<Detection>> Detect(const cv::Mat& image);
};
```

### 7.6.4 HeadCounter 接口

```cpp
class HeadCounter {
public:
    HeadCounter(HeadDetector* detector, AttendanceDB* db);
    Result<HeadCountResult> Count(const HeadCountRequest& request);
};
```

### 7.6.5 后处理规则

1. 先按置信度过滤。
2. 再按类别过滤，只保留 `head` 或 `face` 类别。
3. 执行 NMS。
4. 将检测框映射回原图。
5. 裁剪超出图片边界的坐标。
6. 最终 `head_count = detections.size()`。

### 7.6.6 NMS 接口

```cpp
std::vector<Detection> Nms(
    const std::vector<Detection>& input,
    float iou_threshold
);
```

## 7.7 FaceMatch 模块

### 7.7.1 文件职责

| 文件 | 职责 |
| --- | --- |
| `include/face_match/face_detector.h` | 单张图片人脸检测接口 |
| `src/face_match/face_detector.cpp` | 人脸检测模型推理 |
| `include/face_match/face_aligner.h` | 人脸对齐接口 |
| `src/face_match/face_aligner.cpp` | 关键点对齐和裁剪 |
| `include/face_match/face_feature_extractor.h` | 特征提取接口 |
| `src/face_match/face_feature_extractor.cpp` | 人脸特征模型推理 |
| `include/face_match/face_matcher.h` | 特征匹配接口 |
| `src/face_match/face_matcher.cpp` | 余弦相似度匹配 |

### 7.7.2 核心数据结构

```cpp
struct FaceLandmarks {
    cv::Point2f left_eye;
    cv::Point2f right_eye;
    cv::Point2f nose;
    cv::Point2f left_mouth;
    cv::Point2f right_mouth;
};

struct FaceDetection {
    float score;
    cv::Rect2f box;
    FaceLandmarks landmarks;
};

struct FaceFeature {
    std::vector<float> values;
    int dim;
    std::string model_version;
};

struct Student {
    int64_t id;
    std::string student_no;
    std::string name;
    std::string class_name;
};

struct FaceTemplate {
    int64_t id;
    Student student;
    FaceFeature feature;
    std::string face_image_path;
};
```

### 7.7.3 FaceDetector 接口

```cpp
class FaceDetector {
public:
    Status Load(const FaceDetectConfig& config);
    Result<std::vector<FaceDetection>> Detect(const cv::Mat& image);
};
```

### 7.7.4 FaceAligner 接口

```cpp
class FaceAligner {
public:
    Result<cv::Mat> Align(
        const cv::Mat& image,
        const FaceDetection& face,
        int output_width,
        int output_height
    );
};
```

### 7.7.5 FaceFeatureExtractor 接口

```cpp
class FaceFeatureExtractor {
public:
    Status Load(const FaceFeatureConfig& config);
    Result<FaceFeature> Extract(const cv::Mat& aligned_face);
};
```

### 7.7.6 FaceMatcher 接口

```cpp
struct FaceMatchCandidate {
    int64_t student_id;
    std::string student_no;
    std::string name;
    float score;
};

struct FaceMatchRequest {
    std::string image_path;
    float threshold;
    bool save_aligned_face;
};

struct FaceMatchResult {
    bool matched;
    std::string status;
    FaceMatchCandidate best;
    float threshold;
    std::string input_image_path;
    std::string aligned_face_path;
    int elapsed_ms;
    int64_t record_id;
};

class FaceMatcher {
public:
    static void Normalize(FaceFeature* feature);
    static float CosineSimilarity(
        const std::vector<float>& a,
        const std::vector<float>& b
    );
    Result<FaceMatchResult> Match(
        const FaceFeature& query,
        const std::vector<FaceTemplate>& templates,
        float threshold
    );
};
```

### 7.7.7 多人脸处理规则

单张人脸匹配功能原则上要求输入图片只有一个人脸。若检测到多张人脸，第一阶段默认选择面积最大的人脸。

后续可以增加策略：

1. 选择置信度最高的人脸。
2. 选择离图片中心最近的人脸。
3. 返回错误，要求重新拍摄。

当前推荐：注册和单张匹配都选择面积最大人脸，并在日志中记录检测到的人脸数量。

## 7.8 Database 模块

### 7.8.1 文件职责

| 文件 | 职责 |
| --- | --- |
| `include/database/attendance_db.h` | 数据库连接和表初始化 |
| `src/database/attendance_db.cpp` | SQLite 执行 SQL、事务封装 |
| `include/database/face_repository.h` | 学生、人脸模板、识别记录的数据访问 |
| `src/database/face_repository.cpp` | 业务数据读写实现 |

### 7.8.2 AttendanceDB 接口

```cpp
class AttendanceDB {
public:
    Status Open(const std::string& db_path);
    Status InitSchema();
    Status Execute(const std::string& sql);
    sqlite3* RawHandle();
    void Close();
};
```

### 7.8.3 FaceRepository 接口

```cpp
class FaceRepository {
public:
    explicit FaceRepository(AttendanceDB* db);

    Result<int64_t> InsertStudent(const Student& student);
    Result<Student> FindStudentByNo(const std::string& student_no);
    Result<int64_t> InsertFaceTemplate(
        int64_t student_id,
        const std::string& image_path,
        const FaceFeature& feature
    );
    Result<std::vector<FaceTemplate>> LoadAllTemplates();
    Result<int64_t> InsertFaceMatchRecord(const FaceMatchResult& result);
    Result<int64_t> InsertHeadCountRecord(const HeadCountResult& result);
};
```

### 7.8.4 事务约定

1. 注册学生时，`students` 和 `face_templates` 必须在同一事务内写入。
2. 写入检测记录时，如果结果图片保存失败，不写数据库记录。
3. 数据库写失败时，业务流程返回失败，但不删除原始输入图片。
4. 特征向量以 float32 二进制 BLOB 保存。

## 8. 通用接口规范

## 8.1 命名空间

所有业务代码建议放入统一命名空间：

```cpp
namespace attendance {
// ...
}
```

## 8.2 返回值规范

推荐使用 `Status` 和 `Result<T>` 统一表达成功、失败和错误信息。

```cpp
enum class StatusCode {
    OK = 0,
    INVALID_ARGUMENT,
    CONFIG_ERROR,
    FILE_IO_ERROR,
    CAMERA_ERROR,
    MODEL_LOAD_ERROR,
    MODEL_INFER_ERROR,
    IMAGE_PROCESS_ERROR,
    FACE_NOT_FOUND,
    MULTIPLE_FACES_FOUND,
    FEATURE_ERROR,
    DATABASE_ERROR,
    NOT_FOUND,
    UNKNOWN_ERROR
};

struct Status {
    StatusCode code;
    std::string message;

    bool ok() const {
        return code == StatusCode::OK;
    }
};

template <typename T>
struct Result {
    Status status;
    T value;

    bool ok() const {
        return status.ok();
    }
};
```

### 8.2.1 错误码说明

| 错误码 | 说明 |
| --- | --- |
| `OK` | 成功 |
| `INVALID_ARGUMENT` | 命令行参数或函数入参不合法 |
| `CONFIG_ERROR` | 配置文件缺失、格式错误、字段非法 |
| `FILE_IO_ERROR` | 图片、模型、目录读写失败 |
| `CAMERA_ERROR` | 摄像头打开或采集失败 |
| `MODEL_LOAD_ERROR` | 模型文件加载失败 |
| `MODEL_INFER_ERROR` | 模型推理失败 |
| `IMAGE_PROCESS_ERROR` | 图片 resize、转换、裁剪失败 |
| `FACE_NOT_FOUND` | 未检测到人脸 |
| `MULTIPLE_FACES_FOUND` | 检测到多张人脸且当前策略不允许 |
| `FEATURE_ERROR` | 特征维度错误或特征提取失败 |
| `DATABASE_ERROR` | SQLite 执行失败 |
| `NOT_FOUND` | 数据不存在 |
| `UNKNOWN_ERROR` | 未分类错误 |

## 8.3 日志规范

日志格式：

```text
[2026-06-04 09:00:00] [INFO] [HeadCounter] image=data/images/classroom.jpg head_count=36 elapsed_ms=2530
```

日志级别：

| 级别 | 使用场景 |
| --- | --- |
| `DEBUG` | 模型输入尺寸、检测框数量、阈值等调试信息 |
| `INFO` | 主要流程开始和结束 |
| `WARN` | 非致命问题，例如检测到多张人脸但选择最大人脸 |
| `ERROR` | 流程失败，例如模型加载失败、数据库写失败 |

## 8.4 JSON 输出规范

命令行工具建议支持 `--json` 参数，便于后续 UI 或脚本调用。

### 8.4.1 head_count 输出

```json
{
  "status": "ok",
  "mode": "head_count",
  "image_path": "data/images/classroom.jpg",
  "result_image_path": "data/results/classroom_detected.jpg",
  "head_count": 36,
  "detections": [
    {
      "class_id": 0,
      "class_name": "head",
      "score": 0.91,
      "x": 120,
      "y": 85,
      "width": 32,
      "height": 36
    }
  ],
  "elapsed_ms": 2530,
  "record_id": 1
}
```

### 8.4.2 face_match 输出

```json
{
  "status": "matched",
  "mode": "face_match",
  "input_image_path": "data/images/test_face.jpg",
  "matched": true,
  "student_id": 1,
  "student_no": "20240001",
  "name": "zhangsan",
  "score": 0.82,
  "threshold": 0.75,
  "elapsed_ms": 840,
  "record_id": 12
}
```

### 8.4.3 失败输出

```json
{
  "status": "error",
  "code": "FACE_NOT_FOUND",
  "message": "no face detected in image",
  "mode": "face_match"
}
```

## 9. 数据库设计

## 9.1 students

```sql
CREATE TABLE IF NOT EXISTS students (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    student_no TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    class_name TEXT,
    created_at TEXT NOT NULL,
    updated_at TEXT
);
```

## 9.2 face_templates

```sql
CREATE TABLE IF NOT EXISTS face_templates (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    student_id INTEGER NOT NULL,
    face_image_path TEXT NOT NULL,
    feature BLOB NOT NULL,
    feature_dim INTEGER NOT NULL,
    model_version TEXT NOT NULL,
    created_at TEXT NOT NULL,
    FOREIGN KEY(student_id) REFERENCES students(id)
);
```

## 9.3 head_count_records

```sql
CREATE TABLE IF NOT EXISTS head_count_records (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    image_path TEXT NOT NULL,
    result_image_path TEXT,
    head_count INTEGER NOT NULL,
    detections_json TEXT NOT NULL,
    confidence_threshold REAL NOT NULL,
    nms_threshold REAL NOT NULL,
    model_version TEXT NOT NULL,
    elapsed_ms INTEGER,
    created_at TEXT NOT NULL
);
```

## 9.4 face_match_records

```sql
CREATE TABLE IF NOT EXISTS face_match_records (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    input_image_path TEXT NOT NULL,
    aligned_face_path TEXT,
    matched_student_id INTEGER,
    score REAL,
    threshold REAL NOT NULL,
    status TEXT NOT NULL,
    model_version TEXT NOT NULL,
    elapsed_ms INTEGER,
    created_at TEXT NOT NULL,
    FOREIGN KEY(matched_student_id) REFERENCES students(id)
);
```

## 9.5 后续扩展表

完整课堂打卡时建议增加：

```sql
CREATE TABLE IF NOT EXISTS attendance_sessions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    class_name TEXT,
    course_name TEXT,
    image_path TEXT NOT NULL,
    result_image_path TEXT,
    expected_count INTEGER,
    detected_count INTEGER,
    recognized_count INTEGER,
    unknown_count INTEGER,
    created_at TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS attendance_records (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id INTEGER NOT NULL,
    student_id INTEGER,
    face_image_path TEXT,
    bbox_json TEXT,
    score REAL,
    status TEXT NOT NULL,
    created_at TEXT NOT NULL,
    FOREIGN KEY(session_id) REFERENCES attendance_sessions(id),
    FOREIGN KEY(student_id) REFERENCES students(id)
);
```

## 10. 文件和目录规范

## 10.1 目录用途

| 目录 | 用途 |
| --- | --- |
| `config/` | 配置文件 |
| `models/head_detect/` | 大图人头检测模型 |
| `models/face_recognition/` | 人脸检测和特征模型 |
| `data/images/` | 原始图片 |
| `data/faces/` | 注册人脸、对齐人脸、裁剪人脸 |
| `data/results/` | 带框结果图 |
| `data/database/` | SQLite 数据库 |
| `data/logs/` | 日志文件 |
| `include/` | 头文件 |
| `src/` | 源码 |
| `tools/` | 独立工具程序 |
| `tests/` | 单元测试 |
| `docs/` | 设计文档 |

## 10.2 文件命名

拍照图片：

```text
data/images/capture_YYYYMMDD_HHMMSS.jpg
```

人头统计结果图：

```text
data/results/head_count_YYYYMMDD_HHMMSS.jpg
```

注册人脸图：

```text
data/faces/register_{student_no}_YYYYMMDD_HHMMSS.jpg
```

单张匹配对齐图：

```text
data/faces/match_aligned_YYYYMMDD_HHMMSS.jpg
```

## 11. 模块间数据流

## 11.1 head_count 数据流

| 阶段 | 输入 | 输出 |
| --- | --- | --- |
| 图片读取 | `image_path` | `cv::Mat original` |
| 预处理 | `original` | `model_input`, `LetterboxInfo` |
| 推理 | `model_input` | raw tensors |
| 后处理 | raw tensors, `LetterboxInfo` | `vector<Detection>` |
| 统计 | `vector<Detection>` | `head_count` |
| 画图 | `original`, detections | result image |
| 入库 | result object | record ID |

## 11.2 face_match 数据流

| 阶段 | 输入 | 输出 |
| --- | --- | --- |
| 图片读取 | `image_path` | `cv::Mat image` |
| 人脸检测 | `image` | `vector<FaceDetection>` |
| 人脸选择 | detections | `FaceDetection selected` |
| 对齐 | image, selected | `cv::Mat aligned_face` |
| 特征提取 | aligned_face | `FaceFeature query` |
| 模板读取 | database | `vector<FaceTemplate>` |
| 匹配 | query, templates | `FaceMatchResult` |
| 入库 | result | record ID |

## 12. 性能和资源约束

IMX6ULL_ALPHA 板端性能有限，代码实现时需要遵守以下约束：

1. 模型只加载一次，不能每张图片重新加载。
2. 尽量使用较小输入尺寸，例如 `320x320`、`416x416`。
3. 大图检测优先使用 letterbox，不直接拉伸变形。
4. 数据库特征模板加载后可以缓存，注册新人后再刷新缓存。
5. 图片保存默认 JPEG，避免保存 BMP/PNG 大文件。
6. 日志不要频繁写大量检测框细节，调试时再打开 `DEBUG`。
7. 避免在循环中频繁创建大 `cv::Mat`，可复用缓冲区。
8. 第一阶段采用单线程顺序流程，确保稳定后再考虑多线程。

## 13. 测试计划

## 13.1 单元测试

| 测试文件 | 测试内容 |
| --- | --- |
| `tests/test_head_postprocess.cpp` | NMS、坐标映射、置信度过滤 |
| `tests/test_face_similarity.cpp` | 余弦相似度、特征归一化 |
| `tests/test_config.cpp` | 配置读取和默认值 |
| `tests/test_database.cpp` | 数据库初始化、插入、查询 |

## 13.2 PC 端集成测试

1. 使用本地图片测试 `head_count`。
2. 注册 5 名学生。
3. 用同一学生不同图片测试 `face_match`。
4. 用未注册人员图片测试 `unknown`。
5. 删除模型文件，确认返回 `MODEL_LOAD_ERROR`。
6. 输入不存在图片，确认返回 `FILE_IO_ERROR`。

## 13.3 板端测试

1. 摄像头能正常拍照并保存。
2. `head_count` 能在板端完成一次推理。
3. `face_match` 能在板端完成一次推理。
4. 连续运行 20 次不崩溃。
5. 断电重启后数据库仍可读取。
6. 存储空间不足时能返回明确错误。

## 14. 第一阶段验收标准

## 14.1 大图人头数统计

1. 能读取本地课堂大图。
2. 能输出检测到的人头数量。
3. 能保存带框结果图。
4. 能将检测记录写入 SQLite。
5. 命令行能输出文本结果和 JSON 结果。

## 14.2 单张人脸匹配

1. 能注册至少 5 名学生。
2. 能保存学生信息和特征向量。
3. 能识别已注册学生。
4. 能将未注册人员识别为 `unknown`。
5. 能保存识别记录。

## 15. 待确认决策项

以下问题建议在开始编码核心模型模块前确认。

| 编号 | 决策项 | 推荐默认值 | 影响 |
| --- | --- | --- | --- |
| D1 | 推理框架选 ncnn 还是 TFLite | ncnn | 影响模型转换和 C++ 调用方式 |
| D2 | 大图统计检测对象是人头还是人脸 | 人头优先，脸部可备选 | 人头对低头、侧脸更稳；人脸方便后续识别 |
| D3 | 摄像头类型 | 先 USB 摄像头 | OpenCV 采集更容易调试 |
| D4 | 人脸特征维度 | 按模型确定，优先 128 或 512 | 影响数据库 BLOB 大小和匹配速度 |
| D5 | 注册时一人保存几个模板 | 第一阶段 1 个 | 多模板准确率更高，但管理更复杂 |
| D6 | 多人脸输入单张匹配时如何处理 | 选择最大人脸 | 简单可演示，但可能误选背景人脸 |
| D7 | 匹配阈值 | 初始 0.75 | 需要通过测试集调参 |
| D8 | 是否加入 LCD/Web 界面 | 第一阶段不加 | 先保证算法和数据链路稳定 |
| D9 | 是否做完整大图多人脸识别 | 第二阶段做 | 第一阶段先验证两个基础能力 |

## 16. 建议实施顺序

1. 建立基础 CMake 工程。
2. 实现 `Status`、`Result<T>`、日志、时间工具。
3. 实现配置读取。
4. 实现 SQLite 初始化和基础表结构。
5. 实现图片读写、画框、letterbox、坐标映射。
6. 实现 `head_count` 的后处理和假数据测试。
7. 接入真实人头检测模型。
8. 实现人脸特征相似度和数据库模板读写。
9. 接入人脸检测、人脸对齐、特征提取模型。
10. 打通 `register_face`。
11. 打通 `face_match`。
12. 在 PC 端完整测试。
13. 交叉编译部署到 IMX6ULL_ALPHA。
14. 根据板端耗时和准确率调整模型和阈值。

## 17. 后续扩展接口预留

完整课堂打卡可以新增以下业务接口。

```cpp
struct AttendanceSessionRequest {
    std::string image_path;
    std::string class_name;
    std::string course_name;
    bool save_result_image;
};

struct AttendanceRecord {
    int64_t student_id;
    std::string student_no;
    std::string name;
    Detection face_box;
    float score;
    std::string status;
    std::string face_image_path;
};

struct AttendanceSessionResult {
    int64_t session_id;
    int detected_count;
    int recognized_count;
    int unknown_count;
    std::vector<AttendanceRecord> records;
    std::string image_path;
    std::string result_image_path;
};

class AttendanceFlow {
public:
    Result<AttendanceSessionResult> Run(
        const AttendanceSessionRequest& request
    );
};
```

后续完整打卡流程可以复用第一阶段已经实现的：

1. 图片读取和保存。
2. 人脸或人头检测。
3. 人脸对齐。
4. 特征提取。
5. 特征匹配。
6. SQLite 入库。

## 18. 风险点和规避方案

| 风险 | 表现 | 规避方案 |
| --- | --- | --- |
| 板端推理慢 | 一张图处理十几秒以上 | 降低输入尺寸、换更小模型、只做演示级检测 |
| 后排人脸太小 | 漏检严重 | 使用更高分辨率拍照、调整摄像头位置、使用人头检测模型 |
| 光照差 | 检测和识别不稳定 | 固定摄像头曝光，增加补光，加入模糊/亮度检查 |
| 阈值不合适 | 误识别或大量 unknown | 用真实班级图片调参 |
| 数据库特征模型版本混乱 | 新旧特征不可比 | `face_templates` 必须保存 `model_version` |
| 中文路径或姓名乱码 | 数据显示异常 | 源码和数据库统一 UTF-8 |
| 摄像头驱动不稳定 | 拍照失败 | 先用 USB 摄像头，必要时改 V4L2 |

## 19. 当前推荐结论

第一阶段按以下路线推进最稳：

1. 使用 C++17 + OpenCV + SQLite + ncnn。
2. 先实现命令行程序，不做界面。
3. 大图人头数统计和单张人脸匹配分开实现。
4. 人头统计先只输出人数，不做人脸身份识别。
5. 单张人脸匹配先要求输入清晰单人脸。
6. 所有结果都保存到 SQLite，便于答辩展示和后续扩展。

