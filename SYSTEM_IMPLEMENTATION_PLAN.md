# 课前拍照打卡系统实现规划

## 1. 系统目标

本系统基于 IMX6ULL_ALPHA 板端单机运行，优先拆分为两个相对独立的功能：

1. 大图人头数统计功能：对一张课堂大图进行检测，统计图片中出现的人数或人头数。
2. 单张人脸匹配功能：对单个清晰人脸进行识别匹配，判断该人脸是否属于已注册人员。

当前阶段不直接实现“课堂大图中所有人脸逐一识别”的完整流程，而是先完成两个基础能力：

- 证明板端能够完成轻量目标检测。
- 证明板端能够完成单人脸特征提取和数据库匹配。
- 为后续合并成完整课前拍照打卡流程做准备。

## 2. 技术路线

### 2.1 推荐技术栈

- 操作系统：Linux，运行在 IMX6ULL_ALPHA 开发板。
- 图像采集：V4L2 摄像头接口。
- 图像处理：OpenCV。
- 模型推理：ncnn 或 TFLite Micro/TFLite。
- 数据存储：SQLite。
- 配置文件：JSON 或 INI。
- 日志：本地文本日志。

### 2.2 模型选择建议

大图人头数统计可以选择：

- YOLOv5n/YOLOv8n 的轻量化人头检测模型。
- YOLO-Fastest 人头检测模型。
- NanoDet 或其他轻量目标检测模型。
- 如果只做人脸计数，也可以使用 YuNet 等轻量人脸检测模型。

单张人脸匹配可以选择：

- MobileFaceNet。
- SFace。
- 其他轻量 ArcFace 结构模型。

模型需要尽量转换为适合 ARM CPU 推理的格式：

- ncnn：`.param` + `.bin`
- TFLite：`.tflite`

## 3. 功能拆分

## 3.1 大图人头数统计功能

### 3.1.1 功能目标

输入一张课堂大图，系统检测图中所有人头或人脸位置，输出人数统计结果，并保存检测信息。

### 3.1.2 需要实现的功能

1. 摄像头拍照
   - 从 USB 摄像头或 DVP 摄像头采集一帧图片。
   - 保存原始图片到本地。
   - 支持从本地图片文件读取，方便调试。

2. 图片预处理
   - 调整图片尺寸。
   - 格式转换，如 BGR/RGB。
   - 归一化处理。
   - 根据模型输入要求进行 padding 或 resize。

3. 人头检测推理
   - 加载轻量检测模型。
   - 对课堂大图执行推理。
   - 输出候选框、置信度、类别。

4. 后处理
   - 解析模型输出。
   - 执行 NMS 去重。
   - 过滤低置信度目标。
   - 统计最终人头数量。

5. 结果保存
   - 保存检测框坐标。
   - 保存统计人数。
   - 保存检测时间。
   - 保存带框的结果图片。
   - 将结果写入 SQLite 数据库。

6. 结果显示
   - 在终端输出统计结果。
   - 后续可扩展到 LCD 或 Web 页面展示。

### 3.1.3 输出结果示例

```json
{
  "image_path": "data/images/session_001.jpg",
  "result_image_path": "data/results/session_001_detected.jpg",
  "head_count": 36,
  "detections": [
    {
      "x": 120,
      "y": 85,
      "w": 32,
      "h": 36,
      "score": 0.91
    }
  ],
  "created_at": "2026-06-04 09:00:00"
}
```

## 3.2 单张人脸匹配功能

### 3.2.1 功能目标

输入一张单人脸图片，系统提取人脸特征，与本地数据库中的注册人脸特征进行匹配，输出识别结果。

### 3.2.2 需要实现的功能

1. 人员注册
   - 输入学生姓名、学号、班级等信息。
   - 采集或导入单张人脸图片。
   - 检测图片中的人脸。
   - 裁剪并对齐人脸。
   - 提取人脸特征向量。
   - 保存人员信息和特征向量到 SQLite。

2. 单张人脸检测
   - 从图片中检测单个人脸。
   - 如果检测到多张人脸，默认选择最大或最清晰的人脸。
   - 如果未检测到人脸，返回失败信息。

3. 人脸对齐
   - 根据眼睛、鼻尖、嘴角等关键点进行对齐。
   - 输出统一尺寸的人脸图，例如 `112x112`。

4. 特征提取
   - 加载轻量人脸识别模型。
   - 输出固定长度特征向量。
   - 对特征向量进行归一化。

5. 特征匹配
   - 从 SQLite 读取已注册人员特征。
   - 使用余弦相似度或欧氏距离进行比较。
   - 找到最高分人员。
   - 根据阈值判断是否识别成功。

6. 识别记录保存
   - 保存识别图片路径。
   - 保存匹配到的学生 ID。
   - 保存相似度分数。
   - 保存识别状态。
   - 保存识别时间。

### 3.2.3 输出结果示例

```json
{
  "input_image": "data/images/face_test_001.jpg",
  "matched": true,
  "student_id": "20240001",
  "name": "张三",
  "score": 0.82,
  "threshold": 0.75,
  "created_at": "2026-06-04 09:05:00"
}
```

## 4. 建议目录结构

```text
attendance-system/
├── README.md
├── SYSTEM_IMPLEMENTATION_PLAN.md
├── CMakeLists.txt
├── config/
│   ├── app_config.json
│   ├── camera_config.json
│   └── model_config.json
├── models/
│   ├── head_detect/
│   │   ├── head_detect.param
│   │   └── head_detect.bin
│   └── face_recognition/
│       ├── face_detect.param
│       ├── face_detect.bin
│       ├── face_feature.param
│       └── face_feature.bin
├── data/
│   ├── images/
│   ├── faces/
│   ├── results/
│   └── database/
│       └── attendance.db
├── include/
│   ├── app/
│   │   ├── app_context.h
│   │   └── app_config.h
│   ├── camera/
│   │   └── camera_capture.h
│   ├── common/
│   │   ├── image_utils.h
│   │   ├── logger.h
│   │   └── time_utils.h
│   ├── database/
│   │   ├── attendance_db.h
│   │   └── face_repository.h
│   ├── head_count/
│   │   ├── head_counter.h
│   │   ├── head_detector.h
│   │   └── detect_postprocess.h
│   └── face_match/
│       ├── face_detector.h
│       ├── face_aligner.h
│       ├── face_feature_extractor.h
│       └── face_matcher.h
├── src/
│   ├── main.cpp
│   ├── app/
│   │   ├── app_context.cpp
│   │   └── app_config.cpp
│   ├── camera/
│   │   └── camera_capture.cpp
│   ├── common/
│   │   ├── image_utils.cpp
│   │   ├── logger.cpp
│   │   └── time_utils.cpp
│   ├── database/
│   │   ├── attendance_db.cpp
│   │   └── face_repository.cpp
│   ├── head_count/
│   │   ├── head_counter.cpp
│   │   ├── head_detector.cpp
│   │   └── detect_postprocess.cpp
│   └── face_match/
│       ├── face_detector.cpp
│       ├── face_aligner.cpp
│       ├── face_feature_extractor.cpp
│       └── face_matcher.cpp
├── tools/
│   ├── register_face.cpp
│   ├── test_head_count.cpp
│   ├── test_face_match.cpp
│   └── init_database.cpp
├── scripts/
│   ├── build_arm.sh
│   ├── build_pc.sh
│   └── convert_model.md
├── tests/
│   ├── test_head_postprocess.cpp
│   └── test_face_similarity.cpp
└── docs/
    ├── database_design.md
    ├── model_deployment.md
    └── board_deployment.md
```

## 5. 主要文件功能说明

### 5.1 根目录文件

| 文件 | 功能 |
| --- | --- |
| `README.md` | 项目说明、编译方式、运行方式 |
| `SYSTEM_IMPLEMENTATION_PLAN.md` | 当前系统实现规划 |
| `CMakeLists.txt` | C++ 工程编译配置 |

### 5.2 配置目录

| 文件 | 功能 |
| --- | --- |
| `config/app_config.json` | 系统运行参数，如日志级别、数据目录、是否保存结果图 |
| `config/camera_config.json` | 摄像头设备号、分辨率、帧率、图片格式 |
| `config/model_config.json` | 模型路径、输入尺寸、置信度阈值、匹配阈值 |

### 5.3 模型目录

| 文件或目录 | 功能 |
| --- | --- |
| `models/head_detect/` | 大图人头检测模型 |
| `models/face_recognition/face_detect.*` | 单张人脸检测模型 |
| `models/face_recognition/face_feature.*` | 人脸特征提取模型 |

### 5.4 数据目录

| 目录或文件 | 功能 |
| --- | --- |
| `data/images/` | 原始拍照图片和测试图片 |
| `data/faces/` | 注册时裁剪后的人脸图片 |
| `data/results/` | 检测后带框图片、识别结果图片 |
| `data/database/attendance.db` | SQLite 数据库文件 |

### 5.5 摄像头模块

| 文件 | 功能 |
| --- | --- |
| `include/camera/camera_capture.h` | 摄像头采集接口定义 |
| `src/camera/camera_capture.cpp` | V4L2 或 OpenCV 摄像头采集实现 |

### 5.6 大图人头数统计模块

| 文件 | 功能 |
| --- | --- |
| `include/head_count/head_detector.h` | 人头检测模型接口定义 |
| `src/head_count/head_detector.cpp` | 加载模型并执行人头检测 |
| `include/head_count/detect_postprocess.h` | 检测后处理接口定义 |
| `src/head_count/detect_postprocess.cpp` | 解码模型输出、NMS、置信度过滤 |
| `include/head_count/head_counter.h` | 人头统计主流程接口 |
| `src/head_count/head_counter.cpp` | 调用预处理、检测、后处理并输出人数 |

### 5.7 单张人脸匹配模块

| 文件 | 功能 |
| --- | --- |
| `include/face_match/face_detector.h` | 人脸检测接口定义 |
| `src/face_match/face_detector.cpp` | 检测输入图片中的人脸 |
| `include/face_match/face_aligner.h` | 人脸对齐接口定义 |
| `src/face_match/face_aligner.cpp` | 根据关键点裁剪并对齐人脸 |
| `include/face_match/face_feature_extractor.h` | 人脸特征提取接口定义 |
| `src/face_match/face_feature_extractor.cpp` | 调用模型生成人脸特征向量 |
| `include/face_match/face_matcher.h` | 人脸匹配接口定义 |
| `src/face_match/face_matcher.cpp` | 计算相似度并返回最佳匹配人员 |

### 5.8 数据库模块

| 文件 | 功能 |
| --- | --- |
| `include/database/attendance_db.h` | 数据库初始化、连接、通用操作接口 |
| `src/database/attendance_db.cpp` | SQLite 数据库操作实现 |
| `include/database/face_repository.h` | 人员信息和人脸特征数据访问接口 |
| `src/database/face_repository.cpp` | 注册人员、读取特征、保存识别记录 |

### 5.9 公共工具模块

| 文件 | 功能 |
| --- | --- |
| `include/common/image_utils.h` | 图片读取、保存、缩放、画框等工具函数 |
| `src/common/image_utils.cpp` | 图片工具函数实现 |
| `include/common/logger.h` | 日志接口定义 |
| `src/common/logger.cpp` | 日志输出实现 |
| `include/common/time_utils.h` | 时间格式化接口 |
| `src/common/time_utils.cpp` | 时间工具实现 |

### 5.10 工具程序

| 文件 | 功能 |
| --- | --- |
| `tools/init_database.cpp` | 初始化 SQLite 表结构 |
| `tools/register_face.cpp` | 注册学生人脸信息 |
| `tools/test_head_count.cpp` | 输入大图，测试人头统计功能 |
| `tools/test_face_match.cpp` | 输入单人脸图，测试人脸匹配功能 |

## 6. 数据库初步设计

### 6.1 students 表

保存学生基础信息。

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | INTEGER | 主键 |
| `student_no` | TEXT | 学号 |
| `name` | TEXT | 姓名 |
| `class_name` | TEXT | 班级 |
| `created_at` | TEXT | 创建时间 |

### 6.2 face_templates 表

保存注册人脸特征。

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | INTEGER | 主键 |
| `student_id` | INTEGER | 对应学生 ID |
| `face_image_path` | TEXT | 注册人脸图片路径 |
| `feature` | BLOB | 人脸特征向量 |
| `feature_dim` | INTEGER | 特征维度 |
| `model_version` | TEXT | 特征模型版本 |
| `created_at` | TEXT | 创建时间 |

### 6.3 head_count_records 表

保存大图人头统计结果。

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | INTEGER | 主键 |
| `image_path` | TEXT | 原图路径 |
| `result_image_path` | TEXT | 带框结果图路径 |
| `head_count` | INTEGER | 统计人数 |
| `detections_json` | TEXT | 检测框 JSON |
| `created_at` | TEXT | 创建时间 |

### 6.4 face_match_records 表

保存单张人脸识别记录。

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | INTEGER | 主键 |
| `input_image_path` | TEXT | 输入图片路径 |
| `matched_student_id` | INTEGER | 匹配学生 ID，未匹配时为空 |
| `score` | REAL | 相似度分数 |
| `threshold` | REAL | 使用的匹配阈值 |
| `status` | TEXT | `matched`、`unknown`、`failed` |
| `created_at` | TEXT | 创建时间 |

## 7. 主程序运行模式

主程序可以先设计为命令行方式，方便调试和答辩展示。

### 7.1 大图人头统计

```bash
./attendance_app --mode head_count --image data/images/classroom.jpg
```

输出：

```text
head count: 36
result image: data/results/classroom_detected.jpg
```

### 7.2 注册人脸

```bash
./register_face --student-no 20240001 --name zhangsan --class-name class01 --image data/faces/zhangsan.jpg
```

输出：

```text
register success: 20240001 zhangsan
```

### 7.3 单张人脸匹配

```bash
./attendance_app --mode face_match --image data/images/test_face.jpg
```

输出：

```text
matched: true
student_no: 20240001
name: zhangsan
score: 0.82
```

## 8. 实施顺序

建议按以下顺序实现：

1. 搭建 CMake 工程目录。
2. 实现图片读取、保存、画框等基础工具。
3. 实现 SQLite 数据库初始化。
4. 在 PC 上跑通大图人头统计模型。
5. 在 PC 上跑通单张人脸匹配模型。
6. 将模型转换为 ncnn 或 TFLite 格式。
7. 交叉编译 OpenCV、SQLite、ncnn/TFLite 到 IMX6ULL。
8. 将程序部署到 IMX6ULL_ALPHA 板端。
9. 测试摄像头拍照。
10. 测试板端人头统计速度和准确率。
11. 测试板端单张人脸匹配速度和准确率。
12. 根据实际性能调整模型大小、输入分辨率和阈值。

## 9. 当前阶段验收标准

### 9.1 大图人头统计

- 能读取一张课堂大图。
- 能检测出主要人头或人脸位置。
- 能输出人数统计值。
- 能保存带检测框的结果图片。
- 能把统计结果写入 SQLite。

### 9.2 单张人脸匹配

- 能注册至少 5 名学生的人脸。
- 能对单张清晰人脸进行匹配。
- 能输出姓名、学号、相似度。
- 能识别未知人员。
- 能把识别记录写入 SQLite。

## 10. 后续扩展方向

完成两个基础功能后，可以继续扩展为完整课前拍照打卡系统：

1. 将大图中的每个人脸裁剪出来。
2. 对每个人脸分别执行特征提取。
3. 与数据库中所有学生特征进行匹配。
4. 对同一学生的多个匹配结果进行去重。
5. 生成一次课堂考勤记录。
6. 添加 LCD 显示界面或 Web 管理页面。
7. 添加人工复核功能，修正误识别和漏识别结果。

