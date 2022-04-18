# [目录](#目录)
1. [音视频基础知识](#音视频基础知识)
	1. [音视频原始数据格式](#音视频原始数据格式)
	2. [封装格式](#封装格式)
	3. [流媒体协议](#流媒体协议)
	4. [ffmpeg](#ffmepg)
	5. [ffplay](#ffplay)
2. [srs](#srs)
3. [webrtc](#webrtc)
4. [opengl](#opengl)
5. [计算机网络](#计算机网络)
6. [c++](#c++)
7. [数据结构与算法](#数据结构与算法)
8. [项目](#项目)
	1. [执法项目](#执法项目)
	2. [投屏项目](#投屏项目)
	3. [webrtc多人视频通过话](#webrtc多人视频通过话)
	4. [ffmepg播放器](#ffmepg播放器)
	5. [opengl美颜特效](#opengl美颜特效)
	6. [ai车牌识别](#ai车牌识别)


[//]: -------------------------------------参考式目录跳转链接-------------------------------------------
[目录]: #目录
[音视频基础知识]: #音视频基础知识
[音视频原始数据格式]: #音视频原始数据格式
[封装格式]: #封装格式
[流媒体协议]: #流媒体协议
[ffmpeg]: #ffmepg
[ffplay]: #ffplay
[srs]: #srs
[webrtc]: #webrtc
[opengl]: #opengl
[计算机网络]: #计算机网络
[c++]: #c++
[数据结构与算法]: #数据结构与算法
[项目]: #项目
[执法项目]: #执法项目
[投屏项目]: #投屏项目
[webrtc多人视频通过话]: #webrtc多人视频通过话
[ffmepg播放器]: #ffmepg播放器
[opengl美颜特效]: #opengl美颜特效
[ai车牌识别]: #ai车牌识别
[//]: -------------------------------------参考式目录跳转链接-------------------------------------------


--------------------------------------------------------------------------------------------------------


## 交科院-执法项目
* **职责**:
	* 正常开发
	* 封装通用控件，减少重复代码
	* 研究技术，做技术积累
* **参与的工作**
	* 基本信息录入，查询信息展示
	* 代码很乱，赋值粘贴，自告奋勇，通过自定义view自己实现，封装成class，替换旧代码
	* 自定义相机，满足设计要求，系统api无法实现
* **遇到的问题**
	* 读卡api: 不同省份提供，每次打包需要修改代码。使用apt动态生成class代理类，对外提供统一接口，屏蔽继承与内部实现,实现一键打包，得到表扬
	* 上线后统计http信息: 传统思路插入函数，新思路使用apo-aspectJ,用注解，避免代码入侵,去掉注解对编译无任何影响

## 视博云-projection投屏
* **知识点**
	1. mediaprojection采集
		* MediaProjection.createVirtualDisplay(Surface(SurfaceTexture(texture)))
		* SurfaceTexture获取到数据，updateTexImage更新纹理
	2. opengl渲染
		* 创建HandlerThread和Handler
		* 创建egl环境,其中surface有编码器mediaCodec提供
		* shader修改纹理坐标，进行图像裁剪
		* 按照帧率通过handler消息循环渲染,输入是采集端纹理id
	3. MediaCodec编码
		* 创建编码器createEncoderByType
		* 设置参数: 编码格式、码率、帧率、关键帧间隔、图像宽高、profileLevel
		* 设置参数: 编码格式、采样率、通道数、码率、每帧大小、buffer帧数
		* 对于音频手动输入数据编码: dequeueInputBuffer、getInputBuffer、填充数据、queueInputBuffer
		* 开启线程读取编码数据: dequeueOutputBuffer、getOutputBuffer,回调
	4. 打包TS
		* 音频添加ADTS
		* 视频I帧添加SPS、PPS
		* 打包TS发送
	5. 如何解决延迟问题
		* 采集环节: 不要缓存很多帧，音频bufferSize不要太大;AAC每帧4096字节，25帧，大概23ms
		* 渲染环节: 比较复杂耗时的操作如矩阵变换，颜色转换等，放到GPU处理，也就是shader
		* 编码环节: 去除B帧；质量与编码速度矛盾，不是码率越高越好有公式,profileLeve base；preset作用于一组参数，fast，veryfast;tune zerolatency
		* 发送环节: 使用阻塞队列而不是sleep，尽快通知；网络不好发送速率小于编码有堆积，检测发送速率和队列长度，反馈给编码器，降低帧率、码率，同时做对帧处理，注意gop IDR帧
		* 接收渲染端: 队列不要太大，在发送端网络不好突然变好可能导致接收端队列突然变大

* **遇到的问题**
	* 录流: 开线程专门写文件，发送前先放到录流队列，使用释放的内存，二次delete程序崩溃，gdb调试，使用智能指针
	* opengl渲染没画面: 无法定位问题
		* 直接clearColor刷颜色
		* shader刷颜色
		* 录原始解码流
		* 直接渲染解码流
		* 一点点放开注释
		* 定位到问题glDrawElement es2.0 unsigned_int 改为 unsigned short
	* 从零开始

## opengl美颜特效
* **功能**
	1. 大眼滤镜
	2. 美颜
	3. 贴图-猫耳朵
	4. 录制
* **知识点**
	1. 相机数据流
		* cameraId = CameraInfo.前摄/后摄
		* Camera.Open(cameraId)
		* setPreviewFormat(NV21)
		* setPreviewCallback获取预览yuv数据,送入opencv进行人脸检测
		* setPreviewTexture更新纹理到SurfaceTexture,渲染到FBO帧缓冲
	2. opencv人脸检测
		* 导入模型，创建级联分类器CascadeClassifier
		* 创建检测追踪器DetectionBasedTracker
		* 传入yuv byte数组，调用process进行人脸检测，得到人脸框
	3. 中科院人脸5点定位
		* 加载中科院5点人脸检测模型
		* 根据原图像和人脸框信息检测人脸5各关键点的坐标: 两个眼睛、鼻尖，左右嘴角
		* 保存信息，通过jni创建class对象，返回检测数据
	4. opengl渲染
		* surfaceTexture收到camera预览数据回调
		* glSurfaceView.requestRender请求渲染
		* updateTextImage更新纹理
		* 渲染到FBO帧缓冲
	5. 大眼睛算法
		* uniform传递眼睛坐标
		* 公式fs(r) = (1-(r/Rmax-1)^2)*a*r
		* r: 当前像素到眼睛的距离
		* Rmax: 最大作用半径
		* a: 放大系数
		* fs: 计算后得到的新的像素距离眼睛的距离
		* 通过这个公式: 输入当前像素坐标，得到一个转换后的像素坐标，对它进行采样输出颜色
	6. 美颜算法
		* 高斯模糊
		* 高反差 = 源图-高斯模糊
		* 磨皮
		* mix线性混合
	7. 贴图原理
		* 创建纹理Id
		* texImage2D将图片转成纹理
		* 开启图层混合glBendFunc
		* 根据人脸框位置设置贴图区域，glViewport
		* 渲染到屏幕上
	8. 录制原理
		* 创建egl环境
		* 创建MediaCodec，并将surface设置给opengl
		* 渲染编码
		* 通过MediaMuxer封装成mp4


## 车牌识别
* **知识点**
	1. 特征
		* HAAR: 由黑白区域构成的模板，黑白区域像素和之差
		* LBP: 局部二值化，描述图像局部纹理，旋转不变性和灰度不变性	
		* HOG: 方向梯度直方图，图像重叠区域密集描述符，关注图像结构或形状，适合轮廓检测
	2. svm支持向量机
		* 定义在特征空间上的间隔最大的线性分类器。
		* 目标: 找出基于特征的分解函数
		* 核函数: 对于线性不可分，把低维向量映射到高维空间，从而找出一个分解超平面	
	3. ann人工神经网络-MLP前馈多层感知机
		* 输入层、隐含层、输出层，每层由多个神经元组成
		* 神经元: 输入、激活函数、输出
		* 对于基于HOG特征的神经网络，通常输入层的神经元数量等于HOG特征的维度，输出层的输出等于时别目标数，比如识别26个字母
	4. 模型训练
		* 基于HOG特征的车牌模型
		* 基于HOG特征的数字-字母-省份简称汉字模型
	5. 识别流程
		* 加载模型，SVM::load,ANN_MLP::load
		* 高斯模糊去噪
		* 灰度化cvtColor
		* Sobel边缘检测
		* 二值化
		* 腐蚀膨胀得到连通区域
		* 求轮廓得到外接矩形
		* 尺寸调整、角度旋转得到候选车牌列表 
		* 使用颜色空间HSV处理得到候选车牌列表2
		* 合并候选车牌列表
		* 提取候选车牌HOG特征送入SVM模型，选出可信度最高的
		* 抠出字符提取HOG特征，送入ANN模型，识别出字符

