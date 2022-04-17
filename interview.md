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

## projection投屏
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
