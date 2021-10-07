- [《MIT - 线性代数》笔记](#mit---线性代数笔记)
  - [一、Lesson 1](#一lesson-1)
    - [1.1 方程组的几何解释](#11-方程组的几何解释)
- [\end{bmatrix}](#endbmatrix)
      - [1.1.1 行图像](#111-行图像)
      - [1.1.2 列图像](#112-列图像)
    - [1.2 方程组的几何形式推广](#12-方程组的几何形式推广)
      - [1.2.1 高维行图像](#121-高维行图像)
- [\end{bmatrix}](#endbmatrix-1)
      - [1.2.2 高维列图像](#122-高维列图像)
      - [1.2.3 不能求解的场景](#123-不能求解的场景)
    - [1.3 矩阵乘法](#13-矩阵乘法)
  - [Lesson 2](#lesson-2)
    - [2.1 消元矩阵](#21-消元矩阵)
- [\end{bmatrix}](#endbmatrix-2)
    - [2.2 单位阵](#22-单位阵)
- [\end{bmatrix}](#endbmatrix-3)
- [\end{bmatrix}](#endbmatrix-4)
- [\end{bmatrix}](#endbmatrix-5)
- [\end{bmatrix}](#endbmatrix-6)
    - [2.3 行列变换](#23-行列变换)
- [\end{bmatrix}](#endbmatrix-7)
- [\end{bmatrix}](#endbmatrix-8)
    - [2.4 逆矩阵](#24-逆矩阵)
- [\end{bmatrix}](#endbmatrix-9)
  - [Lesson 3](#lesson-3)
    - [3.1 矩阵乘法](#31-矩阵乘法)
    - [3.2 列组合](#32-列组合)
- [\end{bmatrix}](#endbmatrix-10)
    - [3.3 行组合](#33-行组合)
    - [3.4 逆矩阵](#34-逆矩阵)
    - [3.5 高斯消元求逆矩阵](#35-高斯消元求逆矩阵)
# 《MIT - 线性代数》笔记

## 一、Lesson 1

### 1.1 方程组的几何解释

$$
\begin{cases}
2x - y = 0 \\
-x + 2y = 3\\
\end{cases}
$$

上面方程组我们可以写成矩阵形式

 $$
\begin{bmatrix}
2 & -1\\
-1 & 2\\
\end{bmatrix}
\begin{bmatrix}
x\\
y\\
\end{bmatrix}
= 
\begin{bmatrix}
0\\
3\\
\end{bmatrix}
$$

上面的矩阵可以看成 `Ax = b`的形式 :

1. 系数矩阵(A)：将方程系数按行提取出来，构成一个矩阵
2. 未知向量(x)：将方程未知数提取出来，按列构成一个向量。
3. 向量(b) ：将等号右侧结果按列提取，构成一个向量

#### 1.1.1 行图像

在坐标系上画出“行图像”，可以知两个线交点就是我们要求的解

![image.png](https://upload-images.jianshu.io/upload_images/12321605-daf0b19e00a4a95f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 1.1.2 列图像

从列图像的角度，我们再求这个方程可以看成矩阵：

$$ x\left[\begin{matrix}
 2  \\ -1  \end{matrix} \right]
 +y\left[\begin{matrix}
 -1\\ 2\end{matrix} \right]
=\left[\begin{matrix}
 0 \\ 3 \end{matrix} \right]$$ 

$$
如上，我们用列向量构成系数矩阵，将问题化为：由向量
\begin{bmatrix}
2\\
-1\\
\end{bmatrix}与向量
\begin{bmatrix}
-1\\
2\\
\end{bmatrix}正确组合，使其结果的到\begin{bmatrix}
0\\
3\\
\end{bmatrix}
$$


![image.png](https://upload-images.jianshu.io/upload_images/12321605-f9002c2daa374179.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 1.2 方程组的几何形式推广

#### 1.2.1 高维行图像

我们将方程维数推广，从三维开始，如果我们继续做行图像求解，那么会的到一个很复杂的图像。

$$
\begin{cases}
2x - y = 0 \\
-x + 2y -z = -1\\
-3y +4z= 4\\
\end{cases} //(0,0,1)
$$

矩阵如下：
$$
A = \begin{bmatrix}
2 & -1 & 0\\
-1 & 2 & -1 \\
0 & -3 & 4 \\
\end{bmatrix}, b =
\begin{bmatrix}
0\\
-1\\
4 \\
\end{bmatrix}，方程 Ax = b
$$

$$
\begin{bmatrix}
2 & -1 & 0\\
-1 & 2 & -1 \\
0 & -3 & 4 \\
\end{bmatrix} 
\begin{bmatrix}
x\\
y\\
z \\
\end{bmatrix}
= 
\begin{bmatrix}
0\\
-1\\
4 \\
\end{bmatrix}
$$

如果绘制行图像，很明显这是一个三个平面相交得到一点，我们想直接看出 这个点的性质可谓是难上加难，比较靠谱的思路是先联立其中两个平面，使其相 交于一条直线，在研究这条直线与平面相交于哪个点，最后得到点坐标即为方程 的解。

**这个求解过程对于三维来说或许还算合理，那四维呢？五维甚至更高维数 呢？直观上很难直接绘制更高维数的图像，这种行图像受到的限制也越来越多。**

#### 1.2.2 高维列图像

我们使用列图像的思路进行计算，那矩阵形式就变为：

$$
x\begin{bmatrix}
2 \\
-1 \\
0 \\
\end{bmatrix} 
+y\begin{bmatrix}
-1 \\
2 \\
-3 \\
\end{bmatrix} 
+z\begin{bmatrix}
0 \\
-1 \\
4 \\
\end{bmatrix} 
=\begin{bmatrix}
0 \\
-1 \\
4 \\
\end{bmatrix} 
$$

左侧是线性组合，右侧是合适的线性组合组成的结果，这样一来思路就清晰多 了，“寻找线性组合”成为了解题关键。

![image.png](https://upload-images.jianshu.io/upload_images/12321605-8d2264df832779b5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 1.2.3 不能求解的场景

另外，还要注意的一点是对任意的 b 是不是都能求解 Ax = b 这个矩阵方程呢？ 也就是对 3*3 的系数矩阵 A，其列的线性组合是不是都可以覆盖整个三维空间呢？ 对于我们举的这个例子来说，一定可以，还有我们上面 2*2 的那个例子，也可以 覆盖整个平面，但是有一些矩阵就是不行的，比如三个列向量本身就构成了一个 平面，那么这样的三个向量组合成的向量只能活动在这个平面上，肯定无法覆盖 2 −1 1 一个三维空间，

比如三个列向量分别为

$$
\begin{bmatrix}
2 \\
-1 \\
0 \\
\end{bmatrix} 
\begin{bmatrix}
-1 \\
2 \\
-3 \\
\end{bmatrix} 
\begin{bmatrix}
1 \\
1 \\
-3 \\
\end{bmatrix} 
$$

由于第一列+第二列=第三列，第三列向量并没有任何作用。
$$
\begin{bmatrix}
2 \\
-1 \\
0 \\
\end{bmatrix} 
+\begin{bmatrix}
-1 \\
2 \\
-3 \\
\end{bmatrix} 
= \begin{bmatrix}
1 \\
1 \\
-3 \\
\end{bmatrix} 
$$

上面的矩阵，只能构成一个平面，这样的矩阵构成的方程`Ax = b`,其中的b就无法覆盖整个三维空间，也就无法实现：对任意的b，都能求解 `Ax = b`这个方程。

### 1.3 矩阵乘法

行列式乘法，C_1_1 = A(Row 1) * B(Col 1)

$$
A=\begin{bmatrix}
a_{00}&a_{01}&a_{02}\\
a_{10}&a_{11}&a_{13}\\
\end{bmatrix}
$$

$$
B=\begin{bmatrix}
b_{00}&b_{01}\\
b_{10}&b_{11}\\
b_{20}&b_{21}\\
\end{bmatrix}
$$

$$
C=AB=\begin{bmatrix}
a_{00}b_{00}+a_{01}b_{10}+a_{02}b_{20}& a_{00}b_{01}+a_{01}b_{11}+a_{02}b_{21}\\
a_{10}b_{00}+a_{11}b_{10}+a_{12}b_{20}& a_{10}b_{01}+a_{11}b_{11}+a_{12}b_{21}\\
\end{bmatrix}
$$

## Lesson 2

本节主要内容是矩阵消元和逆矩阵。

### 2.1 消元矩阵


$$
求解方程：\begin{cases}
x + 2y + z = 2 \\
3x + 8y + z = 12\\
4y + z = 4\\
\end{cases} //(2,1,-2)
$$

写成矩阵形式 `Ax = b`

$$
\begin{bmatrix}
1 & 2 & 1 \\
3 & 8 & 1 \\
0 & 4 & 1 \\
\end{bmatrix} 
\begin{bmatrix}
x\\
y\\
z \\
\end{bmatrix}
= 
\begin{bmatrix}
2\\
12\\
2 \\
\end{bmatrix}
$$

矩阵消元其实跟方程消元差不多，不过矩阵消元得到的结果是最终是一个下三角都是0的矩阵（上三角矩阵）。

$$
\begin{bmatrix}
1 & 2 & 1\\
3 & 8 & 1 \\
0 & 4 & 1 \\
\end{bmatrix} 
\overrightarrow {(2,1)=0}
\begin{bmatrix}
1 & 2 & 1\\
0 & 2 & -2 \\
0 & 4 & 1 \\
\end{bmatrix} 
\overrightarrow {(3,2)=0}
\begin{bmatrix}
1 & 2 & 1\\
0 & 2 & -2 \\
0 & 0 & 5 \\
\end{bmatrix} (这个上三角矩阵就是我们要的结果)
$$

主元：U(1,1),U(2,2),U(3,3)  我们视为主元(pivot)

$$
U = 
\begin{bmatrix}
1 & 2 & 1\\
0 & 2 & -2 \\
0 & 0 & 5 \\
\end{bmatrix} 
$$


注： 并不是所有的 A 矩阵都可消元处理，需要注意在我们消元过程中，如果 主元位置（左上角）为 0，那么意味着这个主元不可取，需要进行 “换行”处理， 首先看它的下一行对应位置是不是 0，如果不是，就将这两行位置互换，将非零数视为主元。

如果是，就再看下下行，以此类推。若其下面每一行都看到了，仍然没有非零数的话，那就意味着这个矩阵不可逆，消元法求出的解不唯一(其实就是少了一个变量，求解不了方程)。下面是三个例子：


$$
\begin{bmatrix}
0 & 2 & 1\\
0 & 2 & -2 \\
0 & 0 & 5 \\
\end{bmatrix} 
\begin{bmatrix}
1 & 2 & 1\\
0 & 0 & -2 \\
0 & 0 & 5 \\
\end{bmatrix} 
\begin{bmatrix}
1 & 2 & 1\\
0 & 2 & -2 \\
0 & 0 & 0 \\
\end{bmatrix} 
$$


我们把上面的U 带回方程`Ax = b`

$$
求解方程：\begin{cases}
x + 2y + z = 2 \\
2y - 2z = 6\\
5z = -10\\
\end{cases} //(2,1,-2)
$$

### 2.2 单位阵

$$
\begin{bmatrix}
1 & 0 & 0\\
\end{bmatrix} 
\begin{bmatrix}
1 & 1 & 1\\
? & ? & ? \\
? & ? & ? \\
\end{bmatrix} 
=
\begin{bmatrix}
1 & 1 & 1\\
\end{bmatrix} 
$$

$$
\begin{bmatrix}
0 & 1 & 0\\
\end{bmatrix} 
\begin{bmatrix}
? & ? & ? \\
1 & 1 & 1\\
? & ? & ? \\
\end{bmatrix} 
=
\begin{bmatrix}
1 & 1 & 1\\
\end{bmatrix} 
$$

$$
\begin{bmatrix}
0 & 0 & 1\\
\end{bmatrix} 
\begin{bmatrix}
? & ? & ? \\
? & ? & ? \\
1 & 1 & 1\\
\end{bmatrix} 
=
\begin{bmatrix}
1 & 1 & 1\\
\end{bmatrix} 
$$


$$
由此
\begin{bmatrix}
1 & 0 & 0\\
\end{bmatrix} 
\begin{bmatrix}
0 & 1 & 0\\
\end{bmatrix} 
\begin{bmatrix}
0 & 0 & 1\\
\end{bmatrix} 
 构成一个单位矩阵：
\begin{bmatrix}
1 & 0 & 0\\
0 & 1 & 0\\
0 & 0 & 1\\
\end{bmatrix} 
$$

我们很显然验证单位阵与任意矩阵相乘，不改变矩阵。例如：

$$
\begin{bmatrix}
1 & 0 & 0 \\
0 & 1 & 0 \\
0 & 0 & 1 \\
\end{bmatrix} 
\begin{bmatrix}
1 & 2 & 1 \\
3 & 8 & 1 \\
0 & 4 & 1 \\
\end{bmatrix}  
=
\begin{bmatrix}
1 & 2 & 1 \\
3 & 8 & 1 \\
0 & 4 & 1 \\
\end{bmatrix}  
$$

再看下上面的消元过程

$$
\begin{bmatrix}
1 & 2 & 1\\
3 & 8 & 1 \\
0 & 4 & 1 \\
\end{bmatrix} 
\overrightarrow {(2,1)=0}
\begin{bmatrix}
1 & 2 & 1\\
0 & 2 & -2 \\
0 & 4 & 1 \\
\end{bmatrix} 
\overrightarrow {(3,2)=0}
\begin{bmatrix}
1 & 2 & 1\\
0 & 2 & -2 \\
0 & 0 & 5 \\
\end{bmatrix} 
$$

第一步是把第一行乘以 -3 然后加上第二行。所以有

$$
\begin{bmatrix}
1 & 0 & 0\\
0 & 1 & 0\\
0 & 0 & 1\\
\end{bmatrix} 
\overrightarrow {第一行乘以-3然后加上第二行}
\begin{bmatrix}
1 & 0 & 0\\
-3 & 1 & 0\\
0 & 0 & 1\\
\end{bmatrix} 
$$

$$
单独看第二行
\begin{bmatrix}
-3 & 1 & 0\\
\end{bmatrix} 
\begin{bmatrix}
1 & 2 & 1\\
3 & 8 & 1 \\
0 & 4 & 1 \\
\end{bmatrix}  = 
\begin{bmatrix}
0 & 2 & -2 \\
\end{bmatrix} 
$$

所以第一步消元矩阵就是

$$
E_{21} = 
\begin{bmatrix}
1 & 0 & 0\\
-3 & 1 & 0\\
0 & 0 & 1\\
\end{bmatrix} 
表示将矩阵A中第二行第一列(2,1)位置变0的消元矩阵
$$


$$
同理
E_{32} = 
\begin{bmatrix}
1 & 0 & 0 \\
0 & 1 & 0 \\
0 & -2 & 1 \\
\end{bmatrix} 
得到
E_{32}E_{32}A(系数矩阵) = U(上三角矩阵)
$$

结论： 求消元矩阵，其实就是从“单位阵”开始，按照A每次变化消元的步骤操作 I 矩阵，分别能得到E(row,clo),最后累积得到E即可。


### 2.3 行列变换

由上面的“单位阵”起发，不难得到交换2 x 2矩阵行列矩阵为：

$$
\begin{bmatrix}
0 & 1 \\
1 & 0 \\
\end{bmatrix} 
\begin{bmatrix}
a & b \\
c & d \\
\end{bmatrix} 
=
\begin{bmatrix}
c & d \\
a & b \\
\end{bmatrix} 
$$


$$
\begin{bmatrix}
a & b \\
c & d \\
\end{bmatrix} 
\begin{bmatrix}
0 & 1 \\
1 & 0 \\
\end{bmatrix} 
=
\begin{bmatrix}
_b & a \\
d & c \\
\end{bmatrix} 
$$


**所以左乘同行交换，右乘同列交换**

### 2.4 逆矩阵

E(2,1) 是基于“单位阵 I” 第一行*(-3)加第二行得到：

$$
E_{21} = 
\begin{bmatrix}
1 & 0 & 0 \\
-3 & 1 & 0 \\
0 & 0 & 1 \\
\end{bmatrix} 
$$

反之，我们在第二行上加上第一行乘以3可以复原这一运算过程：

$$
\begin{bmatrix}
1 & 0 & 0 \\
3 & 1 & 0 \\
0 & 0 & 1 \\
\end{bmatrix}
\begin{bmatrix}
1 & 0 & 0 \\
-3 & 1 & 0 \\
0 & 0 & 1 \\
\end{bmatrix} 
=
\begin{bmatrix}
1 & 0 & 0 \\
0 & 1 & 0 \\
0 & 0 & 1 \\
\end{bmatrix}  
= I
$$

$$
此时 
\begin{bmatrix}
1 & 0 & 0\\
3 & 1 & 0\\
0 & 0 & 1\\
\end{bmatrix}
就是E_{21} ^ -1
$$

$$
E_{21} ^ -1 E_{21} = I
或者
E_{21}E_{21} ^ -1  = I
此时E_{21} ^ -1 就是 E_{21}的逆矩阵
$$


## Lesson 3

### 3.1 矩阵乘法


$$
A=\begin{bmatrix}
a_{00}&a_{01}&a_{02}\\
a_{10}&a_{11}&a_{13}\\
\end{bmatrix}
$$

$$
B=\begin{bmatrix}
b_{00}&b_{01}\\
b_{10}&b_{11}\\
b_{20}&b_{21}\\
\end{bmatrix}
$$

$$
C=AB=\begin{bmatrix}
a_{00}b_{00}+a_{01}b_{10}+a_{02}b_{20}& a_{00}b_{01}+a_{01}b_{11}+a_{02}b_{21}\\
a_{10}b_{00}+a_{11}b_{10}+a_{12}b_{20}& a_{10}b_{01}+a_{11}b_{11}+a_{12}b_{21}\\
\end{bmatrix}
$$

### 3.2 列组合

$$
\begin{bmatrix}
a_{1} & a_{2} & a_{3} \\
b_{1} & b_{2} & b_{3} \\
c_{1} & c_{2} & c_{3} \\
\end{bmatrix}
\begin{bmatrix}
3 \\
4 \\
5 \\
\end{bmatrix}= 3
\begin{bmatrix}
a_{1}  \\
b_{1} \\
c_{1}  \\
\end{bmatrix}+ 4
\begin{bmatrix}
a_{2} \\
b_{2}  \\
c_{2}  \\
\end{bmatrix}
+5
\begin{bmatrix}
a_{3} \\
b_{3}  \\
c_{3}  \\
\end{bmatrix}
= 
\begin{bmatrix}
3a_{1} + 4b_{1} + 5c_{1}  \\
3a_{2} + 4b_{2} + 5c_{2}  \\
3a_{3} + 4b_{3} + 5c_{3}  \\
\end{bmatrix}
$$

矩阵乘法也可以拆解成矩阵和向量乘法


$$
A=\begin{bmatrix}
a_{11}&a_{12}&a_{13}\\
a_{21}&a_{22}&a_{23}\\
a_{31}&a_{33}&a_{33}\\
\end{bmatrix}
B=\begin{bmatrix}
b_{11}&b_{12}&b_{13}\\
b_{21}&b_{22}&b_{23}\\
b_{31}&b_{32}&b_{33}\\
\end{bmatrix}
$$

$$
AB = 
\begin{bmatrix}
A \begin{bmatrix}
b_{11}\\
b_{21}\\
b_{31}\\
\end{bmatrix} & A 
\begin{bmatrix}
b_{12} \\
b_{22} \\
b_{32} \\
\end{bmatrix} & A 
\begin{bmatrix}
b_{13} \\
b_{23} \\
b_{33} \\
\end{bmatrix}
\end{bmatrix}= C
$$

$$
A \begin{bmatrix}
b_{11}\\
b_{21}\\
b_{31}\\
\end{bmatrix} 
这样矩阵和向量的乘积，都可以转化为矩阵A的列向量的线性组合。
$$

这种方法的关键就是将右侧矩阵 B 看做列向量组合，将问题转化为矩阵与向量的乘法问题。也表明了矩阵 C 就是矩阵 A 中各列向量的线性组合，而 B 其实是在告诉我们，要以什么样的方式组合 A 中的列向量。

### 3.3 行组合

与上面列组合有点相似

$$
AB = 
\begin{bmatrix}
\begin{bmatrix}
a_{11} && a_{12} && a_{13}\\
\end{bmatrix} 
B \\  
\begin{bmatrix}
a_{21} && a_{22} && a_{23}\\
\end{bmatrix}
B \\
\begin{bmatrix}
a_{31} && a_{32} && a_{33}\\
\end{bmatrix}
B
\end{bmatrix}
= C
$$

### 3.4 逆矩阵

对于一个方阵A，如果A可逆，就有这样一个A<sup>-1</sup>使得
$$
AA^-1 = I = A^-1A
$$

如果A不是方阵，左侧的A<sup>-1</sup>和右侧的A<sup>-1</sup>肯定不相等。违背了我们说的有唯一的一个A<sup>-1</sup>


$$
在比如 
\begin{bmatrix}
1 & 3 \\
2 & 6 \\
\end{bmatrix} 
也是一个不可逆的矩阵
因为要使 AA^-1 = I = A^-1A成立
I 需要是一个全0行列式
$$

或者换个看法，我们看到这个矩阵中两个列向量[1,2] ,[3,6],他们是线性相关的，他们之前互为倍数，也就是说这两个向量之一对其线性组合无意义，那么A不可能有逆。所有推出：

**若存在非零向量x，使得 Ax = 0， 那么A就不可能有逆矩阵。**

因为如果 A 有逆，在 `Ax=0`这个等式两端同时乘上A<sup>-1</sup>就有：

$$A^-1Ax = Ix = 零向量 $$

而 I 是单位矩阵，x 又是一个非零0的向量所以不可能是零向量。自相矛盾，所以此时A没有逆矩阵。


### 3.5 高斯消元求逆矩阵

$$
求 \begin{bmatrix}
1 & 3 \\
2 & 7 \\
\end{bmatrix} 的逆矩阵
$$

$$
\begin{bmatrix}
1 & 3 \\
2 & 7 \\
\end{bmatrix}
\begin{bmatrix}
a & b \\
c & d \\
\end{bmatrix}
= I = 
\begin{bmatrix}
1 & 0 \\
0 & 1 \\
\end{bmatrix}
$$


$$
可以用解方程思想来解：\begin{bmatrix}
1 & 3 \\
2 & 7 \\
\end{bmatrix}
\begin{bmatrix}
a \\
c \\
\end{bmatrix}=
\begin{bmatrix}
1  \\
0 \\
\end{bmatrix},
\begin{bmatrix}
1 & 3 \\
2 & 7 \\
\end{bmatrix}
\begin{bmatrix}
b \\
d \\
\end{bmatrix}=
\begin{bmatrix}
0  \\
1 \\
\end{bmatrix}
$$

高斯消元来求逆矩阵

$$
\begin{bmatrix}
1 & 3 | & 1 & 0 \\
2 & 7 | & 0 & 1 \\
\end{bmatrix}
\overrightarrow{3row2 - 2row1}
\begin{bmatrix}
1 & 3 | & 1 & 0 \\
0 & 1 | & -2 & 1 \\
\end{bmatrix}
\overrightarrow{row1- 3row2}
\begin{bmatrix}
1 & 0 | & 7 & -3 \\
0 & 1 | & -2 & 1 \\
\end{bmatrix}
$$

$$
所以逆矩阵为
\begin{bmatrix}
  7 & -3 \\
 -2 & 1 \\
\end{bmatrix}
$$