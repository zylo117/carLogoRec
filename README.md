# carLogoRec

2018.4.18更新逻辑回归模块  

2018.4.20更新OpenCV自带BP网路模块    

2018.4.26更新手撸BP模块  

2018.8.08更新LeNet-5-based CNN模块(使用Pytorch)



## 模块列表

- Opencv自带BP网络+HOG  


- Logistic Regression  content/LR  
- 全局函数  
- CNN_pytorch

## 1. opencv自带BP网络+Hog特征识别车标

opencv2.4.11：使用HOG特征进行车标分类识别

>工程基于本人早期课程设计，记录在博客园博客中---[使用OpenCV实现车标识别](http://logwhen.cn/2018/01/16/%E4%BD%BF%E7%94%A8OpenCV%E5%AE%9E%E7%8E%B0%E8%BD%A6%E6%A0%87%E8%AF%86%E5%88%AB.html)。 Github上的版本是重构后的代码。

### 环境要求
测试环境：win10+VS2017+OpenCV2.4.11  
测试结果：100%识别  
opencv的配置这里就不再赘述了。简单讲下工程的情况。  

### 简单思路
其实直接将图像灰度值作为网络输入也可以，本工程使用了人工提取的图像的`HOG`特征作为网络输入，输出为车标类别。  

### 训练集&测试集准备
1. 先将数据集手动划分成训练集和测试集，并分好类，比如第一类就放在文件夹名为`0`的文件夹下，第二类就是`1`，如此类推。
>当前程序只能处理10类以下车标，因为当前程序逻辑不支持10以上的数字识别（具体可以仔细看下代码）

 所有训练集的图片放在`train`文件夹中，测试集放在`test`文件夹下。最终的文件树如下：
![捕获.PNG](https://i.loli.net/2018/03/02/5a982f6d2826a.png)
> 1. `reCarlog`是工程名，即`Cardata`和`main.cpp`同目录。
> 2. 测试集的类别数字和训练集的要一一对应。因为程序将要用它们作为分类依据。

2. 在 `main.cpp`目录下准备两个文件，`trainpath.txt`和`testpath.txt`，用以保存所有训练集和测试集图片的路径。程序需要这两个文件来读取训练集和测试集的图片。举例如下(`trainpath.txt`)
> ./Cardata/train/0/train_citroen1.jpg  
> ./Cardata/train/0/train_citroen10.jpg
>./Cardata/train/0/train_citroen100.jpg
>./Cardata/train/0/train_citroen101.jpg

  建议使用相对路径。  
  这样，当我们读取一张图片的时候，可以利用图片的路径名称，通过`split`调用确定该车标的类别（使用切分字符`'/'`，第4个值即类别（0，1，2，3，4...））

### 代码概览
代码很简单，就一个`main.cpp`文件。大致分为以下3块
- **全局变量**：比如整理好的数据集，标签集，HOG特征向量 
- **全局函数**：模块划分，使得`main`函数不显得臃肿。
- **自定义网络类**：`myNeraulNetwork`用于搭建简单BP网络和训练预测

### 运行流程
分3步：
1. 训练集装载
2. 定义网络+训练网络
3. 测试网络

#### 1.训练集装载
全局变量设定：

	#define N_SAMPLE 1000
	#define F_NUM   1764 
	#define CLASSNUM 5
	float Data[N_SAMPLE][F_NUM];      // 数据存放
	float Label[N_SAMPLE][CLASSNUM]   // 标签存放

训练网络输入是两个二维矩阵，第一个矩阵是数据矩阵（第一维是样本个数`N_SAMPLE`,第二维是每个样本的特征向量是，宽度为`F_NUM`），第二个矩阵是标签矩阵，对应每个样本，都有一个类别标签，如果是第一类，则它的标签向量为`1,0,0,0,0`(本例是5维)。  
这里主要提一下数据矩阵的第二维是怎么确定的。  
>每个样本的特征向量即每张图片的HOG特征。HOG特征是一个一维向量。

##### HOG特征维度确定
对于一张图片，使用一个滑动窗口以一定的步进滑动，分别获取每个窗口的特征值，是一般的人工图像特征提取方式。简单说下HOG特征的提取。  
假设一张图片的维度是`img_size=64x64`，我们使用的滑动窗口大小为`block_size=16x16`,滑动步进`stride=8x8`，那么对一个这样的图像，能得到`(64-8)/8 x (64-8)/8=7x7=49`个窗口，对于每个窗口`block`，HOG特征细分为胞元`cell_size=8x8`。于是一个`block`就有`2x2=4`个胞元，每个胞元默认有`9`个特征值，所以在上述参数的情况下，HOG特征的维度为`49x4x9=1764`，这也是本工程的默认参数。
>opencv自带HOG特征提取，`img_size`、`block_size`、`stride`和`cell_size`都需要自行设定，因此需要事先计算好特征维度，才能确定数据矩阵第二维的宽度。

##### 装载过程
	read trainpath.txt;   // 读取路径文件
	for each trainImg in trainpath.txt :
		getHOG(trainImg)   // 获取HOG特征
	    getLabel according to its path 
	    put its hog into Data[][]
	    put its label into Label[][]

#### 2.定义网络+训练网络
对opencv自带网络类进行了简单的封装，如下：
![捕获.PNG](https://i.loli.net/2018/03/02/5a98dbd8c07a5.png)
定义和使用代码里说的很清楚了，这里再提下两个构造函数：
带参数的构造函数使用网络参数文件名作为参数。可以直接使用训练好的网络参数文件直接初始化网络，而不需要`initialNN`。
#### 3.测试网络
读取测试文件，输入网络，获得输出。
>输入为每次一个图片，所以输入的二维矩阵为`test[1][F_NUM]`。`myNerualNetwork().predict(img)`获得一个预测值，可以跟实际值(分析文件路径名获得)做对比，得到分类正确率。  

## 2. Logistic Regression

#### 1. 思想

逻辑回归基于简单的代数思想：
$$
\notag \hat y=\sigma(W^TX+b)
$$
对于多分类，运用 one-vs-rest的思想，可以看成多个二分类问题，例如，对于一个5分类问题，训练5个分类器，对于每个分类器，将1个分类作为正样本，剩余类作为负样本，这样，对于每个分类器来说，就是计算一个属于当前分类的概率。5个分类器的最终结果是：
$$
\notag [\hat y_0,\hat y_1,\hat y_2,\hat y_3 \hat y_4]
$$
则最终分类为 max_value_index。

#### 2. 代码

代码使用python完成。

> datasets_utils.py

完成两个任务.

- 根据正样本索引号完成训练集读取`load_train_data`
- 读取测试集 `load_test_set`

> lr.py

LR分类器构建与识别，函数层级如下

>lr_model() #构建1个lr分类器
>
>>initial_parameters() 	# 初始化模型参数
>>
>>optimize() 			#优化
>>
>>> propagate() 		   # 梯度下降
>
>estimate()	# 评估模型
>
>> predict()   # 测试测试集



## 全局函数

> dataModel.py

根据`Cardata/`生成 `h5`数据集，该数据集包含4个部分

- `data['train_X']`：训练集
- `data['train_Y']`：训练集标签
- `data['test_X']`：测试集
- `data['test_Y']`：测试集标签



## 3. CNN_LeNet5

### 3.1 模型结构

首先看下LeNet5的结构：

<img src='https://markdownfoto-1252952266.cos.ap-guangzhou.myqcloud.com/%E7%BD%91%E7%BB%9C%E6%A8%A1%E5%9E%8B/lenet5.jpg' width=100%>

最初这个模型是Yann LeCun 用来做一个手写体识别的task。由于模型简单计算量小，这里非常适合我们的车标数据集。

### 3.2 数据集处理

使用python 模块 h5py 处理了下数据集，使得数据集由车标的灰度图像素组成。  

使用下面的代码可以查看任意一张训练集或者测试集的图(调用`data_utils`里的函数)

```python
from data_utils import showImage
import h5py
dataset=h5py.File('carDatasets.h5','r')
showImage(dataset=dataset,label='train_X',index=10)
```

如果是查看测试集，则`label='test_X'` ，另外确保index不超出范围(训练集1000图，测试集500图)

下面是结果示例：

<img src='https://markdownfoto-1252952266.cos.ap-guangzhou.myqcloud.com/Github/readme1.PNG'>

**注：** 从左到右依次是 雪铁龙、大众、一汽、福田、本田



### 3.3 快速开始

**训练模型**:

```python
from model.cnn_lenet5 import *
model =LeNet5()
train(model,num_epoch=10,batch_size=16,learning_rate=0.01,save=False)
```

save参数用来决定模型参数是否存储到`./ModelParams`文件夹下，该文件夹下有已训练好的参数文件

**加载模型**:

```python
model=LeNet5()
model=loadModel(model)
```

则会自动加载文件夹下的参数。

**预测一张图片**:

`imgSample`文件夹下有几种车标的样例，根据输入的文件名可以预测图片所属车标：

```python
predict(model,img_path='../imgSample/Honda_sample.jpg')
```



最后感谢大家的耐心，能看完这个简单的document。如果这个简单的工程对你有帮助，还望大家不吝惜右上角的star喔

