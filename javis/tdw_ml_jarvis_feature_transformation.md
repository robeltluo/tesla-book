## 特征转换

 * [Dummy](#2-dummy)
 * [PCA](#3-pca)
 * [Normalizer](#4-normalizer)
 * [SVD](#5-svd)
 * [Scaler](#6-scaler)
 * [Discrete](#7-discrete)
 * [RandomizedSVD](#9-randomizedsvd)

### 1. Dummy

- **算法说明**

  模块将低维的特征做**特征交叉** ，并将特征转换成高维的One-Hot特征。Dummy模块中包含两个阶段，**特征交叉** 和 **特征One-Hot**。**特征交叉** 根据json配置文件，对指定的特征字段做交叉，生成feature name组成的特征； **特征One-Hot** 将feature name编码成全局统一、连续的index。

- **输入**
  - 数据形式：[Dense或libsvm](./tdw_ml_jarvis_dataformat.md#2-数据格式要求)
  - 格式：| features |
  - 说明：数据中的 0 和 -1是两个特殊值，0表示默认值，-1 表示非法值；读取数据过程中将会过滤掉这两个特殊值；因此特征的数值表示应该避开0和-1。Dense数据支持多值特征，顾名思义，多值特征是指该特征可以包含多个value，每个value以“|”分割。例如，某特征是“喜欢的游戏”，该特征对应的值应该是多个游戏名，即游戏列表。必须包含target字段。训练数据的target字段是0，1的label值，预测数据的target字段是每条数据的标识id。
- **输出**

  - 说明：包含三个目录，featureCount、featureIndexs、instances
  - featureCount：dummy后的特征空间维度，一个Int值
  - featureIndexs：feature name到feature index的映射关系，中间用“:”分隔
  - instances：One-Hot后的dummmy格式的样本数据

	
- **参数**：
  - 并行数：训练数据的分区数、spark的并行数
  - 特征生成配置：特征交叉的配置文件有两个对象“fields”和“feature_cross”，"fields"对象保存着输入数据每个字段对应的name和index；“feature_cross”对象是生成特征的配置，其中“id_features”是指生成单个特征，“comb_features”是指交叉的特征，dependencies是指交叉的特征，用逗号分隔，可以指定多个。
  - 说明：必须包含target字段。若某个特征是多值特征，"id_features"会生成多个特征，"comb_features"也会多次与其他的dependencies做交叉。
  - 样例:以下就是样例数据的配置和特征交叉阶段生成的中间结果。

```json

	{
  	"fields": [
  	  {
  	    "name": "target",
  	    "index": "0"
  	  },
  	  {
  	    "name": "f1",
  	    "index": "1"
  	  },
  	  {
   	    "name": "f2",
  	    "index": "2"
 	   },
 	   {
   	    "name": "f3",
  	    "index": "3"
 	   }
 	 ],
        "feature_cross": {
 	 "id_features": [
           {
            "name": "f1"
           },
           {
            "name": "f3"
           }
   	  ],
  	 "comb_features": [
          {
            "name": "f1_f2",
            "dependencies": "f1,f2"
          },
          {
            "name": "f2_f3",
            "dependencies": "f2,f3"
          }
        ]
     }
  }

```
   - featureIndex：指定了特征One-Hot时，使用的feature name到index之间的映射。属于选填参数当需要基于上次特征Dummy的结果做增量更新或者预测数据的dummy，需要指定该参数，保证两次dummy的index是一致的。
  - 负样本抽样率：大部分应用场景中，负样本比例太大，可通过该参数对负样本采样
 

### 2. PCA

- **算法说明**

  主成分分析一种统计学的特征降维方法，将数据从原来的坐标系投影到新的坐标系，通过每个维度的方差大小来衡量该维度的重要性。从中选取重要性排在前K个的特征作为新的特征，达到数据降维的目的。

- **输入**
  - 数据形式：[Dense或Libsvm](./tdw_ml_jarvis_dataformat.md#2-数据格式要求)
  - 格式：| 参与计算的features | 不参与计算的features | 
  - 参与计算的features：可通过**选择特征列**指定
  - 不参与计算的features：可包括不参与计算的特征，如果存在则保留在输出中，对于Libsvm格式的数据，也包括了label列，该label仅用于标识样本ID

- **输出**
  - 格式：| 不参与计算的features | PCA结果列 |
  - PCA结果列：选取的前K个新特征

- **参数**：
  - 选择特征列：表示需要计算的特征所在列，例如“1-12,15”，表示取特征在表中的1到12列，15列，从0开始计数
  - 提取主成份维度：降维后的特征维度
  - 并行数：训练数据的分区数、spark的并行数
  - 抽样率：输入数据的采样率

### 3. Normalizer

- **算法说明**

  该模块提供了特征正则化，将各维特征正则化成[0, 1]阈值，可以避免不同特征值的分布域不一致而影响算法的决策。算法的计算过程为首先计算每行所选特征的p阶范数，然后将该行所选的每个特征值与该范数相除的结果作为最终的正则化结果。注意，当某行特征的p阶范数为0.0时，该行特征的正则化结果为该行特征的原始值。

- **输入**
  - 数据形式：[Dense或Libsvm](./tdw_ml_jarvis_dataformat.md#2-数据格式要求)
  - 格式：| 参与计算的features | 不参与计算的features | 
  - 参与计算的features：可通过**选择特征列**指定
  - 不参与计算的features：可包括不参与计算的特征，如果存在则保留在输出中，对于Libsvm格式的数据，也包括了label列，该label仅用于标识样本ID

- **输出**
  - 格式：| 不参与计算的features | 正则化的结果 |
  - 正则化的结果：被选取的特征的正则化结果


- **参数**:
  - 选择特征列：表示需要计算的特征所在列，例如“1-12,15”，表示取特征在表中的1到12列，15列，从0开始计数
  - p： p-norm参数，默认为2，表示用L2-Norm做正则化
  - 并行数：训练数据的分区数、spark的并行数
  - 抽样率：输入数据的采样率
  
### 4. SVD

- **算法说明**

  SVD（Singular Value Decomposition)奇异值分解法是一种对矩阵的分解方法。通常，对于大型稀疏矩阵而言，我们往往采用矩阵分解技术将矩阵表示成新的易于处理的形式，从而简化计算。一个矩阵A可以通过SVD算法分解为：S、V和U三个矩阵。其中，S为奇异值矩阵，U称为左奇异向量，V为右奇异向量(注:V^T不是右奇异向量)。SVD算法原理可参考[《矩阵奇异值分解法SVD介绍》](http://km.oa.com/articles/show/256138)。

- **输入**
  - 数据形式：[Dense或Libsvm](./tdw_ml_jarvis_dataformat.md#2-数据格式要求)
  - 格式：| label | 参与计算的features | 不参与计算的features | 
  - label: 每行矩阵的行索引，通过**算法参数**中的**矩阵行索引**指定。
  - 参与计算的features：与行索引对应的矩阵中的行向量，可通过**算法参数**的**特征起始列**和**特征终止列**指定
  - 不参与计算的features：对于Dense形式的数据可包括不参与计算的特征

- **输出**

  - outputS：奇异值矩阵<img src="http://chart.googleapis.com/chart?cht=tx&chl= \Sigma" style="border:none;">
    - 格式：|S1 S2 ...|  
    - 说明：奇异值（如S1、S2等），以降序排列组成的行向量，以空格连接
  - outputV：右奇异向量V
    - 格式：| index | vector |
    - 说明：以空格连接各字段
    - index:对应输入数据的行索引
    - vector:右奇异向量V矩阵

  - outputU：左奇异向量U
    - 格式：| index | vector |
    - 说明：以空格连接各字段
    - index:对应输入数据的行索引
    - vector:左奇异向量U矩阵

- **参数**：
  - 矩阵行索引：矩阵行索引所在的列号，默认为0，libsvm数据填0
  - 特征起始列：待分解矩阵的第一列所在的列号，从0开始计数
  - 特征终止列：待分解矩阵的最后一列所在的列号，从0开始计数
  - 并行数：训练数据的分区数、spark的并行数
  - 抽样率：输入数据的采样率
  - k：奇异值的个数
  - rCond：条件数倒数

### 5. Scaler

- **算法说明**

  Scaler模块集成了最大最小值归一化（MinMaxScaler）和标准化（StandardScaler）两种方式，用户可通过特征配置文件来指定某个特征的归一化方法。下面分别介绍这两种方法：

  MinMaxScaler算法对特征数据进行统一的归一化处理，默认归一化后的特征数值范围在[0,1]，用户也可以指定归一化后的取值范围为[min,max]。
  该归一化的计算公式为：

	((x-EMin)/(EMax-EMin))*(max-min)+min

  其中x表示需要归一化的特征值，EMin表示该特征下的最小值，EMax表示该特征下的最大值，min与max为用户设定的归一化后的数值范围。注意，当某列特征的最大最小值相等时，该列所有数值归一化为0.5*(max - min) + min。

  StandardScaler算法主要对特征进行标准化处理，原始特征数据通过该算法的转化将成为方差为1，均值为0的新的特征。计算公式为：

	((x-Mean)/Var)

  其中，Mean代表该特征的平均值，Var表示该特征的样本标准差。以下的特殊情况应该**注意**:

  (1)如果Var为0，则x标准化后的结果直接为0.0

  (2)不需要均值化处理：此时算法只做方差处理，即：x/Var

  (3)不需要方差处理:x直接取值(x-Mean)

- **输入**
  - 数据形式：[Dense](./tdw_ml_jarvis_dataformat.md#21-dense数据格式)
  - 格式：| 参与计算的features | 不参与计算的features | 
  - 参与计算的features：通过**特征配置文件**指定
  - 不参与计算的features：可包括不参与计算的特征，如果存在则保留在输出中

- **输出**
  - 格式：与输入数据格式一致
  - 说明：特征排列顺序与输入数据的一致，根据特征配置对数据进行归一化或标准化处理，没有配置的特征值保持不变

- **参数**：
  - 并行数：训练数据的分区数、spark的并行数
  - 抽样率：输入数据的采样率
  - 特征配置文件名：从TDInsight页面上传配置文件。下面是一个该模块的JSON格式的特征配置文件样例

```json
{
    "minmax":[
      {
        "colStr": "1-2",
        "min": "0",
		"max":"1"
      },
      {
		"colStr":"5",
        "min":"-1",
        "max": "1"
       }
       ],
		"standard":[
       {
        "colStr": "3,6-7",
        "std": "true",
		"mean":"false"
       },
       {
        "colStr": "8,9",
		"std":"true",
		"mean":"true"
       }
       ]
}

```

- **特征配置文件参数**:
  - "minmax"和"standard":分别代表了相应的归一化模块MinMaxScaler和StandardScaler
  - "colStr":需要做相应处理的特征id,取值根据该特征在原始表的所在列，从0开始计数。多个特征可用","隔开，还可用"-"标识特征的起始到结束列，例如"1-20"表示第1列到第20列。
  - "min":归一化后的最小值
  - "max":归一化后的最大值
  - "std":是否需要做方差的标准化
  - "mean":是否需要做均值的标准化
	
	
### 6. Discrete

- **算法说明**

  Discrete算法对特征数据进行离散化处理。离散方法包括等频和等值离散方式，等频离散法按照特征值从小到大的顺序将特征值划分到对应的桶中，每个桶中容纳的元素个数是相同的；等值离散法根据特征值中的最小值和最大值确定每个桶的划分边界，从而保证每个桶的划分宽度在数值上相等。

- **输入**：
  - 数据形式：[Dense](./tdw_ml_jarvis_dataformat.md#21-dense数据格式)
  - 格式：| 参与计算的features | 不参与计算的features | 
  - 参与计算的features：可通过**特征配置文件**指定
  - 不参与计算的features：可包括不参与计算的特征，如果存在则保留在输出中

- **输出**：
  - 离散化结果：
    - 格式：与输入数据格式一致
    - 说明：特征的排列顺序与输入数据一致，根据特征配置对数据进行离散化处理的结果，没有配置的特征值保持不变

  - 离散特征边界：每个特征进行离散化时使用的边界值
    - 格式：| index | bound1 bound2 bound3 ...|
    - 说明：以空格连接各字段。
    - index：离散化的特征Id，从0开始计数
    - boundx：每个特征对应的离散边界值，各边界值以空格连接

- **参数**：
  - 并行数：训练数据的分区数、spark的并行数
  - 抽样率：输入数据的采样率
  - 特征配置文件名：从TDInsight页面上传配置文件。下面是一个该模块的JSON格式的特征配置文件样例，特征配置之间的顺序没有要求，如果某些特征不需要离散配置，则不写在配置中。该配置文件中的"feature"不可更改，用户只需对如下参数进行修改即可。

```json
{
    "feature": [
      {
        "id": "2",
        "discreteType": "equFre",
        "numBin": "3"
      },
      {
		"id": "5",
        "discreteType": "equVal",
        "numBin": "3",
		"min": "0",
		"max": "100"
       },
       {
        "id": "0",
        "discreteType": "equVal",
        "numBin": "2",
		"min":"-1"
      }
    ]
}

```

- **特征配置文件参数**:
  - "id":表示特征Id,注意该Id是输入数据中特征所在的列号，从0开始计数
  - "discreteType":离散化的类型，"equFre"表示等频离散，"equVal"表示等值离散
  - "numBin":离散化的桶数，需注意，在等频离散方法中，如果桶数设置过大，每个桶中的元素个数过少，导致离散边界中有重复的点，则出错
  - "min":针对等值离散的配置，限定了特征值的最小值，如果特征值中有比这个值还小的则出错。如果没有需要可不填，同下面的"max"
  - "max":针对等值离散的配置，限定了特征值的最大值，如果特征值中有比这个值还大的则出错，没有需要可不填


### 7. RandomizedSVD

- **算法说明**

  RandomizedSVD算法是根据论文[《Finding structure with randomness: Probabilistic algorithms for constructing approximate matrix decompositions》](https://arxiv.org/pdf/0909.4061.pdf)原理以及spark平台实现的矩阵svd分解算法。与TDInsight平台提供的另外一个矩阵分解SVD算法在原理上有所不同，具体内容请参考论文，此处不再详述。

- **输入**
  - 数据形式：[Dense或Libsvm](./tdw_ml_jarvis_dataformat.md#2-数据格式要求)
  - 格式：| label | 参与计算的features | 不参与计算的features | 
  - label: 每行矩阵的行索引，通过**算法参数**中的**标签列**指定
  - 参与计算的features：与行索引对应的矩阵中的行向量，可通过**算法参数**的**选择特征列**指定
  - 不参与计算的features：对于Dense形式的数据可包括不参与计算的特征

- **输出**
  - outputS：奇异值矩阵sigma
    - 格式：|S1 S2 ...|  
    - 说明：奇异值（如S1、S2等），以降序排列组成的行向量，以空格连接
		
  - outputV：右奇异向量V
    - 格式：| index | vector |
    - 说明：以空格连接各字段
    - index：对应输入数据的行索引
    - vector：右奇异向量V矩阵

  - outputU：左奇异向量U
    - 格式：| index | vector |
    - 说明：以空格连接各字段
    - index：对应输入数据的行索引
    - vector：左奇异向量U矩阵
 
	
- **参数**

  - 标签列：label所在列，从0开始计数，默认为0，libsvm数据填0
  - 选择特征列：例如“1-12,15”，其说明取特征在表中的1到12列，15列，从0开始计数
  - 并行数：训练数据的分区数、spark的并行数
  - 抽样率：输入数据的采样率
  - 矩阵乘积的迭代类型:分为两种："QR"与"none","QR"方式表示每轮迭代都需要进行QR分解，"none"方式表示每轮迭代仅进行左乘矩阵A以及A的转置，中间无"QR"分解过程
  - k:svd分解的奇异值个数
  - 过采样参数q:randomized svd的过采样个数
  - 迭代轮数:矩阵乘积的迭代轮数
  - rCond:条件数倒数

