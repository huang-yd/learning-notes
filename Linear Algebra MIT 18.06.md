[toc]
# Linear Algebra MIT 18.06

[https://www.bilibili.com/video/bv1bb411H7JN/?p=8](https://www.bilibili.com/video/bv1bb411H7JN/?p=8)

## Lecture 1

### 矩阵乘列向量代表矩阵各列的线性组合

$$
2x - y = 0\\
-x + y = 3\\
$$

$$
A = \left[
\begin{array}{cc}
  2 & -1\\
  -1 & 2 
\end{array}
\right]
X = \left[
\begin{array}{c}
x\\
y
\end{array}
\right]
b = \left[
\begin{array}{c}
0\\
3
\end{array}
\right]\\
AX=b\\
x\left[\begin{array}{c}2\\-1\end{array}\right] + y\left[\begin{array}{c}-1\\2\end{array}\right] = \left[\begin{array}{c}0\\3\end{array}\right]\\
$$

将各列视作向量，做向量加法，得到右侧向量

## Lecture 2

### 高斯消元法

$$
\left[A | b\right]是增广矩阵
$$

核心是矩阵行变换，将系数矩阵变为上三角阵（U矩阵），每行第一个非零元素称为主元

1. 交换行
2. scaling（乘标量）
3. 各行相加

$$
AX = b \Rightarrow UX = c
$$

### 高斯消元法的矩阵形式

行向量乘矩阵代表矩阵各行的线性组合
$$
消元矩阵E（初等矩阵）左乘系数矩阵\\
E_{21}表示将系数矩阵2行1列位置变为0的初等矩阵\\
E=\left[\begin{array}{cc}a_{11} && a_{12}\\
a_{21} && a_{22}\\
a_{31} && a_{32}\end{array}\right],
A=\left[\begin{array}{c}row_1\\row_2\end{array}\right]\\
EA=\left[\begin{array}{c}
a_{11} * row_1 + a_{12} * row_2\\
a_{21} * row_1 + a_{22} * row_2\\
a_{31} * row_1 + a_{32} * row_2\end{array}\right]
$$
矩阵满足结合律（暴力运算可证）

左乘初等矩阵交换行，右乘初等矩阵交换列（每一列都是将被乘矩阵的列进行线性组合）

利用左乘矩阵进行消元得到主元

### 可逆矩阵

$$
E^{-1}称为E的逆矩阵，E^{-1}E = I
$$

## Lecture 3

### 矩阵乘法的求解

1. $$
   AB = C\\
   c_{ij} = \sum_k{a_{ik}b_{kj}}\\
   $$

2. $$
   C_{column\_1} = AB_{column\_1}\\
   C中每一列等于A乘以B中对应列\\
   等于A中各列的线性组合\\
   $$

3. $$
   C_{row\_1} = A_{row\_1}B\\
   C中每一行等于A中对应行乘以B\\
   等于B中各行的线性组合\\
   $$

4. $$
   C=\sum{(col\ of \ A)}(row\ of \ B)\\
   c_{ij}=a_{第1列第i行}b_{第1行第j列}+a_{第2列第i行}b_{第2行第j列}+\cdots+a_{第n列第i行}b_{第n行第j列}\\
   =a_{i1}b_{1j}+a_{i2}b_{2j}+\cdots+a_{in}b_{nj}\\
   =\sum_k{a_{ik}b_{kj}}\\
   $$

5. 分块计算

### 矩阵的逆

方阵如果有逆，左逆等于右逆，即
$$
A^{-1}A = AA^{-1} = I\\
设左逆为A_1,右逆为A_2\\
A_1A = I\\
A_1AA_2 = A_2\\
由结合律A_1AA_2 = A_1(AA_2) = A_1I = A_1 = A_2
$$
存在非零向量X使得方阵AX=0，则A为不可逆矩阵（奇异矩阵），否则若A可逆，左乘A逆可得X=0，矛盾

#### Gauss-Jordan消元法

方阵A若有逆，则可通过如下变换求得
$$
[A | I] \Rightarrow [I | A^{-1}]\\
$$
求解 AX=I 即通过左乘初等矩阵将左式变为 IX，这些初等矩阵的乘积即为右式，即A的逆

或者用分块的思想看，消元的过程即做行变换，将初等矩阵的乘积记作E，则$E\left[\begin{array}{cc}A&I\end{array}\right] = \left[\begin{array}{cc}I&E\end{array}\right]，则E = A^{-1}$

## Lecture 4

$$
AA^{-1} = I\\
(AA^{-1})^T = I\\
(A^{-1})^TA^T = I\\
(A^T)^{-1} = (A^{-1})^T
$$

### A=LU分解(假设没有行变换)

下三角阵（L矩阵，右上角是0）相乘仍是下三角阵，上三角阵（U矩阵，左下角是0）相乘仍是上三角阵。

L矩阵可以直接观察得到，消元所需要乘的系数(各行主元系数的比值)可以直接写入L中
$$
n阶置换矩阵有 n! 个（左乘可以互换行的矩阵），乘法运算封闭，其逆和其转置相等\\
P^{-1} = P^T
$$

## Lecture 5

### PA=LU（存在行变换，令主元位置不为0）

对任意可逆矩阵，有PA=LU的分解

### 对称矩阵

$$
定义：A^T = A即(A)_{ij} = (A)_{ji}
$$

$$
对任意矩阵R，R^TR为对称矩阵\\
(R^TR)^T = R^T(R^T)^T = R^TR
$$

### 向量空间

空间中的向量对于加法和数乘（线性组合）封闭

必须包含零向量
$$
R^2的3个子空间：\\
1.R^2\\
2.穿过原点的直线\\
3.\left\{\left[\begin{array}{c}0 \\ 0\end{array}\right]\right\}
$$
任意两个子空间S和T，则S和T的交集仍然是子空间

## Lecture 6

### 矩阵列空间

矩阵A的列空间C(A)为A中各列的线性组合

Ax=b有解当且仅当b属于A的列空间

矩阵A的零空间为Ax=0的所有向量x的集合

## Lecture 7

### 矩阵的秩

rank(A)=消元之后主元的个数=有主元的行的个数=线性无关的行的个数=线性无关的列的个数（将A化作RREF形式可知这些定义等价），而主元各占一列，因此rank(A)=有主元的列的个数，行秩=列秩

主元所在的列叫主列(pivot column)，其他列称自由列(free column)

自由列对应的变量为自由变量，可任意取值，主元的值由自由变量表示

### 零空间

零空间是Ax=0的特解的线性组合(数乘系数c并相加)，特解的数量等于自由变量的个数(自由变量分别取1，其余取0，通过线性组合可以得到所有可能值)。以下针对Ax=0

RREF(reduced row echelon form) 简化行阶梯形式，即主元系数化为1，且使得其他行的该主元系数为0

将RREF的主列提前，自由列往后，得到
$$
R=\left[\begin{array}{cc}I&F\\0&0\end{array}\right]\\
F为自由变量所在的列，A为m行n列，秩为r，则I为r阶，F为n-r阶
$$
零空间矩阵N(null space matrix)，它的各列由Rx=0的特解组成，RN=0
$$
Rx=0\\
[I \ F]\left[\begin{array}{c}x_{pivot}\\x_{free}\end{array}\right]=0\\
x_{pivot} + Fx_{free} = 0\\
x_{pivot} = -Fx_{free}\\
用消元法回代，得到方程的解最终会呈现这种形式\\
将x_{free}的n-r个元素轮流取1，其他取0，即得到Rx=0的各个特解，即-F中的各个列（第1个元素取1对应第1列）\\
即N_{n*(n-r)}=\left[\begin{array}{c}-F\\I\end{array}\right]，注意此处每一列中变量的位置跟原始的[x_1,x_2,…,x_n]^T位置不同\\
$$

## Lecture 8

### Ax=b有解时b应满足的条件

仅当b属于A的列空间C(A)时，Ax=b有解，即b为A各列的线性组合，不会出现零行左侧为0右侧不为0的情况。

### 求Ax=b的所有解

1. 求$ Ax=b $ 的特解，令自由变量为0，求解主元，得到特解$x_p$ 
2. 取零空间中的向量$x_1,x_2,\cdots,x_n$，则所有解$x=x_p+c_1x_1+c_2x_2+\cdots+c_nx_n$ 

$Ax=b$的特解只需要求一个
$$
假设有不同的特解x_{p_1},x_{p_2},Ax_{p_1}=b,Ax_{p_2}=b\\
A(x_{p_1} - x_{p_2}) = 0\\
(x_{p_1} - x_{p_2}) \in N\\
即x_{p_2}可通过x_{p_1}与N中向量的线性组合得到，所有解（通解）的形式不变
$$

### 列满秩时Ax=b的解

每一列都有主元（意味着RREF必定可以化为$\left[\begin{array}{c}I\\O\end{array}\right]$的形式，若行满秩则没有O），没有自由变量可以赋值，N中只有零向量，Ax=b若有解则有唯一解（只有0或1个解）。

### 行满秩时Ax=b的解

意味着消元时不会出现零行（$0=b_i，b_i≠0$），因此对任意b，Ax=b有解。主元有$m=r$个，自由变量有$n-r=n-m$个。

对于$r=m<n$，RREF必定可与化为$\left[\begin{array}{cc}I & F\end{array}\right]$的形式（可能需要交换行），F将构成零空间的特殊解。

对于$r=m=n,RREF(A)=I,A可逆,Ax=b$ 必定有唯一解。N中只有零向量。

###  r小于m且r小于n时Ax=b的解

$RREF(A)=\left[\begin{array}{cc}I&F\\O&O\end{array}\right]$（可能需要行交换），要么无解$(0=b_i,b_i≠0)$，要么有无穷多解（若$x_1$为$Ax=b$的解，设$x_n$是$Ax=0$的非零解（$x_n$必定存在否则没有自由变量），则$x=x_1+cx_n$均为$Ax=b$的解）。

### 矩阵的秩决定了方程组解的数目

## Lecture 9

### 线性相关性

向量组$x_1,x_2,\cdots,x_n,c_1x_1+c_2x_2+\cdots+c_nx_n=0$，仅当$c_i$全为$0$时成立，则向量组线性无关，否则线性相关。

向量组里有零向量则必定线性相关。  

在一个m维空间里，n>m个向量必定相关。将它们放进矩阵里，Ac=0有非零解，由定义得证。

A的零空间中存在非零向量，则A中各列线性相关，否则线性无关。

从自由变量的角度考虑，则rank(A)=n$\Leftrightarrow$列向量线性无关；rank(A)<n$\Leftrightarrow$线性相关。

### 基

向量空间的基是指一组向量满足2个性质：1、线性无关；2、生成整个空间。

方阵的列能组成$R^n$的基$\Leftrightarrow$方阵可逆。

对于给定空间，基向量的数量相等。

基向量的个数称为空间的维数。

rank(A)=线性无关的列的个数=主列的个数=列空间的维数

任意n个n维线性无关的向量构成一组基，生成同样的n维空间（若向量的维度大于n，则生成的空间不同；若向量维度小于n，则代入Ax=b，不是线性无关；取n维空间任意b，Ax=b有解，所以生成的空间一定相同。）

零空间的维数=自由变量的数量=n-r

## Lecture 10

### 矩阵的4个基本子空间

列空间C(A)，零空间N(A)，行空间$C(A^T)$，A的左零空间（$A^T$的零空间）$N(A^T)$

C(A)的一组基是主列，维数为r，是$R^m$的子空间

主元所在行数=主元所在列数，因此$C(A^T)$的维数是r，基是RREF的前r行，是$R^n$的子空间

N(A)的一组基是Ax=0的特解，维数是n-r，是$R^n$的子空间

$N(A^T)$的维数是m-r，是$R^m$的子空间。求A的左零空间就是试着寻找一个产生零行向量的行组合，行初等变化可以做到这一点。对于$A_{m*n}$，构造$\left[\begin{array}{cc}A_{m*n}&I_{m*m}\end{array}\right]$，用Gauss-Jordan消元法变为$\left[\begin{array}{cc}R_{m*n}&E_{m*m}\end{array}\right]$，即通过左乘E进行行变换，EA=R，A和R的秩为r，R中的零行有m-r个（注意$N(A^T)$的维数是m-r），此时R中的零行对应的E中的行就是要找的$N(A^T)$的基。（行初等变换不改变秩，I的秩为m，所以E的秩为m，E中各行线性无关，所以R中的零行对应的E中的行可以作为基。）

## Lecture 11

### “向量”空间

$dim(S) + dim(U) = dim(S \cap U) + dim(S + U)$

此处空间（集合）的加法定义是从S任取一个元素与U任一个元素相加

### 秩的性质

秩为1的矩阵全部可以表示为一个列向量乘一个行向量

rank(A + B) <= rank(A) + rank(B)
$$
设A(a_1,a_2,\cdots,a_n),B(b_1,b_2,\cdots,b_n),A的秩为s，B的秩为t\\
A的极大线性无关组（最多的线性无关的列）为(a_{i_1},a_{i_2},\cdots,a_{i_s})\\
B的极大线性无关组为(b_{j_1},b_{j_2},\cdots,b_{j_t})\\
构造C(a_{i_1},a_{i_2},\cdots,a_{i_s},b_{j_1},b_{j_2},\cdots,b_{j_t})\\
则A+B可以由C中的列线性组合得到\\
所以rank(A+B) \le rank(C) \le s+t = rank(A) + rank(B)\\
$$
秩为 $r$ 的矩阵可以分解成 $r$ 个秩为$1$的矩阵（每个一主列）

$rank(A^TA)≤rank(A),rank(AA^T)≤rank(A)$,第一个左边每一行是$A$的行的线性组合，第二个左边每一列是$A$的列的线性组合，得证。

## Lecture 12

不说了，牛逼

## Lecture 13

复习课

## Lecture 14

### 正交

向量正交的定义$x^Ty=0$

空间S和T正交的定义是每个S中的向量都与T中的每个向量正交

行空间正交于零空间。x为零空间向量，A中每一行与x点乘都为0，而行空间是由A的各行生成的。

列空间正交于A转置的零空间。

零空间是行空间的正交补，即零空间包含所有与行空间垂直的向量。
$$
任意x \in N(A)有Ax=0\\
A^TAx=0\\
所以x \in N(A^TA)\\
所以N(A) \subset N(A^TA)\\
对任意 x \in N(A^TA)有\\
\begin{aligned}
0 &= x^T0\\
&=x^TA^TAx\\
&=(Ax)^TAx\\
\end{aligned}\\
注意Ax为列向量，所以Ax=0\\
所以x \in N(A)\\
所以N(A^TA) \subset N(A)\\
得证N(A^TA)=N(A)\\
$$
$N(A^TA)=N(A),rank(A^TA)=rank(A)$

$A^TA$可逆$\Leftrightarrow$A的零空间内只有零向量$\Leftrightarrow$A的各列线性无关

## Lecture 15

### 投影

#### 向量到向量的投影

$n$维空间中向量$b$投影到向量$a$，投影记作$p,p=xa,e=b-p垂直于a,a^T(b-xa)=0,x=\frac{a^Tb}{a^Ta},p=a\frac{a^Tb}{a^Ta}$，投影矩阵$P=\frac{aa^T}{a^Ta}$，投影为$p=Pb$

$rank(P)=1$，P的列空间是通过a的一条线，基是a

$P^T=P$

$P^2=P$，对b连续投影2次，和投影1次结果一样

#### 为什么要投影

$Ax=b$可能无解，而$Ax$总在$A$的列空间中，找到列空间中与$b$最接近的向量，即$b$的投影$p$，求解$A\widehat{x}=p,\widehat{x}$就是最接近原问题的解。

#### 向量到空间的投影

对于$m$维空间的$n$维子空间，有基向量$a_1,a_2,\cdots,a_n$，空间外有向量$b$，b在空间中的投影记作$p$，$e=b-p$为b中垂直于空间的分量，$p$为基的线性组合，即$p=a_1\hat{x_1}+a_2\hat{x_2}+\cdots+a_n\hat{x_n}$，令$A=\left[\begin{array}{cccc}a_1 & a_2 & \cdots & a_n\end{array}\right],\hat{x}=\left[\begin{array}{c}\hat{x_1}\\\hat{x_2}\\ \cdots \\\hat{x_n}\end{array}\right],A\hat{x}=p$，由$a_i \perp e$，得以下方程：
$$
\left\{
\begin{aligned}
&{a_1}^T(b-A\hat{x})=0\\
&{a_2}^T(b-A\hat{x})=0\\
&\cdots \\
&{a_n}^T(b-A\hat{x})=0\\
\end{aligned}
\right.\\
即\left[\begin{array}{c}{a_1}^T \\ {a_2}^T \\ \cdots \\ {a_n}^T\end{array}\right](b-A\hat{x})=\left[\begin{array}{c}0 \\ 0 \\ \cdots \\ 0\end{array}\right]\\
即A^T(b-A\hat{x})=0\\
$$
$e=(b-A\hat{x}),e \in N(A^T),N(A^T)与A的列空间正交，所以e \perp C(A)$

解上述方程，由$rank(A^TA)=rank(A),A^TA$列满秩可逆，得$\hat{x}=(A^TA)^{-1}A^Tb,p=A(A^TA)^{-1}A^Tb,投影矩阵P=A(A^TA)^{-1}A^T$，此处注意A不是方阵，所以逆不能化简。A如果是方阵，则$C(A)=R^n,P=I$，b就不在空间外，b的投影就是它本身。
$$
对于可逆矩阵M,(MM^{-1})^T=(M^{-1})^TM^T=I，得到\\
(M^T)^{-1}=(M^{-1})^T\\
\begin{aligned}
P^T&=(A(A^TA)^{-1}A^T)^T\\
&=A((A^TA)^{-1})^TA^T\\
&=A((A^TA)^T)^{-1}A^T\\
&=A(A^TA)^{-1}A^T\\
&=P\\
P^2&=A(A^TA)^{-1}A^TA(A^TA)^{-1}A^T\\
&=A(A^TA)^{-1}A^T\\
&=P
\end{aligned}
$$

## Lecture 16

### 投影的几何和算术意义

$P=A(A^TA)^{-1}A^T$，$b$在$C(A)$中时，投影为本身，此时$b=Ax$，它的投影$p=Pb=A(A^TA)^{-1}A^TAx=Ax=b$。

$b \perp C(A)$时，投影到空间为一个点，即为$0$。由于$N(A^T)$是列空间的正交补，所以$b \in N(A^T)$，投影$p=Pb=A(A^TA)^{-1}A^Tb=0$

$b$分成投影分量$p$和垂直于$C(A)$的分量$e$，$b=p+e=Pb+(I-P)b$，因此$e=(I-P)b,I-P$是投影到$N(A^T)$的投影矩阵，与$P$具有相同性质。

### 最小二乘法

在$ty$平面上有三个点$(1,1),(2,2),(3,2)$，寻找一条直线，最接近这三个点

设直线为$y=C+Dt$，则直线应该满足$\left\{\begin{aligned}C+D=1\\C+2D=2\\C+3D=2\end{aligned}\right.$

写成矩阵形式$\left[\begin{array}{cc}1&1\\1&2\\1&3\end{array}\right]\left[\begin{array}{c}C\\D\end{array}\right]=\left[\begin{array}{c}1\\2\\2\end{array}\right],Ax=b$

上述方程无解，$b$不在$A$的列空间中，但可以有最优解，设最优解的直线为$y=\hat{C}+\hat{D}t$，矩阵为$\left[\begin{array}{cc}1&1\\1&2\\1&3\end{array}\right]\left[\begin{array}{c}\hat{C}\\\hat{D}\end{array}\right]=\left[\begin{array}{c}1\\2\\2\end{array}\right],A\hat{x}=b$

定义最接近为误差最小，误差为$||e||=||A\hat{x}-b||$，为计算方便，一般取模长的平方，即$||e||^2$，这就是最小二乘法

通过$\hat{C},\hat{D}$得到新的直线，代入$t$得到新的点$p_1,p_2,p_3，则e=[p_1-b_1,e_2-p_2,e_3-p_3]^T$，在$ty$平面上，$e$表示原先的点和新的点的竖直距离，这是方程组的图和视角

在矩阵空间上，$p$是$b$在$A$的列空间中的投影，$e$是$b$垂直于列空间的分量，这是矩阵的图和视角（实际上到这一步没有对原矩阵方程做任何处理，从空间的角度看，$b$不在$A$的列空间中，要求解就只能求列空间中最接近$b$的向量的线性组合，这个线性组合就是$\hat{x}$，最接近$b$的向量就是$b$的投影，误差的表达式恰好与$ty$平面的误差表达式吻合，所以$ty$平面的新点$p$恰好就是投影向量，误差$e$的式子不变）

$A\hat{x}=b$无解，要有解需先将$b$投影成$p$，求$p$在$A$列空间中的线性组合，于是方程变为$A^TA\hat{x}=A^Tb$（向量到空间的投影一节中 推导的 求投影在列空间中的线性组合的方程）（微积分的方法，将$||e||^2$的表达式列出，$C,D$作为变量，求表达式的最小值，对$C,D$分别求偏导数令其为$0$，得到的方程组与$A^TA\hat{x}=A^Tb$一样），解得$\hat{C}=\frac{2}{3},\hat{D}=\frac{1}{2},y=\frac{2}{3}+\frac{1}{2}t$就是最接近原3个点的直线。

#### 习题

有方程组$\left[\begin{array}{cc}1&0\\1&1\\1&2\end{array}\right]\left[\begin{array}{c}c\\d\end{array}\right]=\left[\begin{array}{c}3\\4\\1\end{array}\right],即Ax=b$的形式，而最小二乘的答案为$\hat{c}=11/3,\hat{d}=-1$，求$b$到矩阵列空间的投影$p$

$p=\frac{11}{3}*col1-1*col2$，因为最小二乘的意义就是找到列空间中最接近$b$的向量的线性组合，即$b$在列空间中的投影$p$

画出拟合的最佳直线

方程组是$\left\{\begin{aligned}c+0d=3\\c+1d=4\\c+2d=1\end{aligned}\right.$，变量乘在$d$上，因此直线为$y=c+dt$，三个点为$(0,3),(1,4),(2,1)$，最小二乘法求出的$\hat{c},\hat{d}$解决的就是拟合最佳直线的问题，截距是$11/3$，斜率$-1$，因此直接画出来即可

找出另一个非零向量$b$，使得最小二乘的结果为$0$

最小二乘的解是$b$在列空间中的投影的线性组合，线性组合为$0$说明投影为零向量，因此$b$垂直于列空间即可满足投影为零向量的条件。观察一下得到$\left[\begin{array}{c}1\\-2\\1\end{array}\right]$满足条件。

### A各列线性无关则$A^TA$可逆

$$
A各列线性无关则Ax=0当且仅当x=0成立\\
假设A^TAx=0\\
x^TA^TAx=0\\
(Ax)^TAx=0\\
Ax=0\\
x=0\\
所以A^TAx=0仅当x=0成立，所以A^TA各列线性无关，可逆\\
$$

### 互相垂直的向量一定线性无关

互相垂直的单位向量（长度为1）称为标准正交向量组

## Lecture 17

### 标准正交矩阵

若Q的每一列都是标准正交基则Q为标准正交矩阵，$Q^TQ=I$。

标准正交方阵称为正交矩阵，此时方阵有逆，$Q^T=Q^{-1}$。

列向量投影到标准正交矩阵$Q$的列空间的投影矩阵$P=Q(Q^TQ)^{-1}Q^T=QQ^T$。如果Q为方阵，$P=I$。

对于正规方程$A^TA\hat{x}=A^Tb$,将Q代入，$\hat{x}=Q^Tb,意味着\hat{x_i}={q_i}^Tb$，第$i$个分量等于第$i$个正交基和b的内积，等于b在第$i$个基方向上的投影大小。

### Gram-Schmidt标准正交化法

以3维为例，有线性无关的向量组$a,b,c$，取$A=a$，$b$在$A$上的投影为$\frac{A^Tb}{A^TA}A$，则在$b$中减去$A$的投影方向分量，得到的$B$与$A$垂直,$B=b-\frac{A^Tb}{A^TA}A$，同理在$c$中减去$A$和$B$的投影方向分量，得到的$C$与$A、B$垂直，$C=c-\frac{A^Tc}{A^TA}A-\frac{B^Tc}{B^TB}B$，标准正交向量组为$q_1=\frac{A}{||A||},q_2=\frac{B}{||B||},q_3=\frac{C}{||C||}$。（至此$|a|$和$||a||$都只表示向量$a$的长度）

矩阵$M=\left[\begin{array}{ccc}a&b&c\end{array}\right]$有列空间$C(M)$，用以上方法得到标准正交矩阵$Q$，从以上计算知道$Q$中各列都是$M$中各列线性组合得到，所以$C(Q)=C(M)$，所做的仅是在原列空间中找到互相垂直的一组标准正交基。

#### 原矩阵M与标准正交矩阵Q的联系

$M=QR$，$R$是一个上三角矩阵（左下角为0）

以以上正交化过程为例
$$
\left\{
\begin{aligned}
a&=A\\
b&=\frac{A^Tb}{A^TA}A+B\\
c&=\frac{A^Tc}{A^TA}A+\frac{B^Tc}{B^TB}B+C\\
\end{aligned}\\
\right.
于是\left[\begin{array}{ccc}a&b&c\end{array}\right]可以写成A、B、C的线性组合\\
M=\left[\begin{array}{ccc}A&B&C\end{array}\right]R_1=Q_1R_1\\
并且M的第i列仅由Q_1的前i列组合得到\\
因此R_1的第i列的第i行后全为0，R_1为上三角阵\\
$$
由上述方程组和
$$
\left\{
\begin{aligned}
A=||A||q_1\\
B=||B||q_2\\
C=||C||q_3\\
\end{aligned}\\
\right.
$$
可以得到$M=\left[\begin{array}{ccc}q_1&q_2&q_3\end{array}\right]R=QR$，将$A、B、C$替换为$q_1,q_2,q_3$时仅仅是等式的系数发生变化，因此$R$为上三角阵。

## Lecture 18

### 行列式

行列式仅针对方阵。

奇异矩阵的定义：不满秩的方阵。

$\det(A)=|A|$意思是矩阵的行列式

矩阵可逆等价于行列式非零。行列式为零时矩阵是奇异的。

行列式的3个性质：

1. $\det(I)=1$

2. 交换行，行列式的值的符号会相反。置换矩阵的行列式为$1$或$-1$，由从$I$交换了多少次决定。

3. $$
   \begin{aligned}
   &(a).\ \left|\begin{array}{cc}ta&tb\\c&d\end{array}\right|=t\left|\begin{array}{cc}a&b\\c&d\end{array}\right|\\
   &(b).\ \left|\begin{array}{cc}a+a'&b+b'\\c&d\end{array}\right|=\left|\begin{array}{cc}a&b\\c&d\end{array}\right|+\left|\begin{array}{cc}a'&b'\\c&d\end{array}\right|\\
   &以上性质针对只改变一行的情况\\
   &推论：t \in R, \det(tA_{n*n})=t^n\det(A)
   \end{aligned}\\
   $$

   
   
4. 两行相等，行列式为$0$。（交换相等的两行，由性质2证得）

5. 从行$k$减去行$j$的$i$倍，行列式不变。（利用3b性质，拆开减去部分，再利用性质4，拆出部分行列式为0）

6. 若有零行则行列式为$0$。（用性质3a，令$t=0$，得证）

7. 上三角阵$U$（左下角为0）的行列式等于对角线上元素的乘积。（若对角线有元素为0，通过消元得到零行，由性质6得证；若对角线没有$0$，通过消元可以将U化为对角阵，由性质3a，提出每一个对角元素，最终化为单位阵，得证。消元时若有行交换，由性质2，注意在最终U的行列式前加上正负号。）

8. $\det(A)=0 \Leftrightarrow A$是奇异矩阵。（左推右，将A化为U，$\det(A)=0$说明对角线有$0$，通过消元得到零行，A不满秩，奇异；右推左，A不满秩，化为U必定有零行，得证。）$\det(A)≠0 \Leftrightarrow A$可逆。（证明同上同理。）

9. $\det(AB)=\det(A)*\det(B)$。（若AB中有奇异矩阵，用行变换和列变换的角度看，AB化为U始终有零行，显然成立；AB都不是奇异矩阵，则A可以视作$I$进行初等行变换得到，该变换使得$\det(I)$放大了$\det(A)$倍，AB视作对B进行同样的行变换，结果是使$\det(B)$放大$\det(A)$倍）。推论有$\det(A^{-1})=\frac{1}{\det(A)},\det(A^2)=(\det(A))^2$。

10. $\det(A^T)=\det(A)$.
    $$
    对于置换矩阵P\\
    |P||P^{-1}|=|P||P^T|=1,P^{-1}=P^T,(P^{-1})^T=P\\
    由性质2，得到|P|=|P^{-1}|=|P^T|\\
    \begin{aligned}
    |A|&=|P^{-1}LU|\\
    &=|P^{-1}||L||U|\\
    &=|U^T||L^T|P^{-1}|\\
    &=|U^T||L^T||P|\\
    &=|U^T||L^T||(P^{-1})^T|\\
    &=|(P^{-1}LU)^T|\\
    &=|A^T|\\
    \end{aligned}\\
    因此，以上所有对于行的性质对列同样适用。\\
    $$

性质1、2、3由行列式的定义可以得到，其他由1、2、3推出。

## Lecture 19

### 行列式的计算

这门课是先定义行列式满足的性质，再从性质推导计算式

从性质可以推出计算式，从计算式也可以推出性质，定义性质或定义计算式是等价的。
$$
A=
\left[\begin{array}{cccc}
a_{11}&a_{12}&\cdots&a_{1n}\\
a_{21}&a_{22}&\cdots&a_{2n}\\
\vdots&\vdots&\cdots&\vdots\\
a_{n1}&a_{n2}&\cdots&a_{nn}\\
\end{array}\right]\\
|A|=\sum{(-1)^ka_{1k_1}a_{2k_2}\cdots a_{nk_n}}\\
k_1,k_2,\cdots,k_n为1,2,\cdots,n的全排列，上式共有n!项\\
k为
k_1,k_2,\cdots,k_n通过交换变为1,2,\cdots,n的次数\\
$$
计算式可以通过将行列式分解成每一行只有一个元素，此时仅有每一行每一列都有1个元素的行列式不为0，这样的共有$n!$项，通过交换列标为正序可以确定符号并得到对角阵求值。

### 代数余子式

$a_{ij}$的代数余子式(Cofactor)$C_{ij}$是大公式里所有含$a_{ij}$项的和，等于原行列式抹去$a_{ij}$所在行和列后得到的行列式，行列式的符号在$i+j$为偶数时取正，奇数时取负。

$|A|=a_{i1}C_{i1}+a_{i2}C_{i2}+\cdots+a_{in}C_{in}$，$i$表示沿第$i$行展开。

因为矩阵转置之后行列式不变，所以沿行展开和沿列展开相等。

#### 三对角线矩阵的行列式

$|A_1|=|1|,|A_2|=\left|\begin{array}{cc}1&1\\1&1\end{array}\right|,|A_3|=\left|\begin{array}{ccc}1&1&0\\1&1&1\\0&1&1\end{array}\right|\cdots$，$A_n$为$A_{n-1}$右下角的元素分别向右、下、右下方向填$1$，其余填$0$得到。

$|A_1|=1,|A_2|=0,|A_3|=-1,|A_4|=-1,|A_5|=0,|A_6|=1,|A_7|=1\cdots$，6个元素为一个周期循环。$|A_n|=|A_{n-1}|-|A_{n-2}|$，将$|A_n|$沿第一列展开，得到两个代数余子式，第一个即$|A_{n-1}|$，将第二个沿它的第一行展开，得到$|A_{n-2}|$。

## Lecture 20

### 逆矩阵公式

$A$为满秩方阵，$A^{-1}=\frac{1}{|A|}C^T$，其中$C$是$A$的伴随矩阵（$C$中第$i$行第$j$列的元素是$a_{ij}$的代数余子式）。
$$
\begin{aligned}
&A^{-1}=\frac{1}{|A|}C^T \Leftrightarrow AC^T=|A|I\\
&AC^T\\
&=\left[
\begin{array}{cccc}
a_{11}&a_{12}&\cdots&a_{1n}\\
a_{21}&a_{22}&\cdots&a_{2n}\\
\vdots&\vdots&\cdots&\vdots\\
a_{n1}&a_{n2}&\cdots&a_{nn}
\end{array}
\right]
\left[
\begin{array}{cccc}
C_{11}&C_{21}&\cdots&C_{n1}\\
C_{12}&C_{22}&\cdots&C_{n2}\\
\vdots&\vdots&\cdots&\vdots\\
C_{1n}&C_{2n}&\cdots&C_{nn}
\end{array}
\right]\\
&=\left[
\begin{array}{cccc}
\sum{a_{1k}C_{1k}}&\sum{a_{1k}C_{2k}}&\cdots&\sum{a_{1k}C_{nk}}\\
\sum{a_{2k}C_{1k}}&\sum{a_{2k}C_{2k}}&\cdots&\sum{a_{2k}C_{nk}}\\
\vdots&\vdots&\cdots&\vdots\\
\sum{a_{nk}C_{1k}}&\sum{a_{nk}C_{2k}}&\cdots&\sum{a_{nk}C_{nk}}
\end{array}
\right]\\\\
&\sum{a_{ik}C_{ik}}=a_{i1}C_{i1}+a_{i2}C_{i2}+\cdots+a_{in}C_{in}即|A|按第i行展开\\\\
&对\sum{a_{ik}C_{jk}}(i≠j)，将A中的第j行替换成第i行，再按第j行展开即可得到\\\\
&由于替换后第i行和第j行相同，所以\sum{a_{ik}C_{jk}}(i≠j)=0\\\\
&AC^T=\left[
\begin{array}{cccc}
|A|&0&\cdots&0\\
0&|A|&\cdots&0\\
\vdots&\vdots&\cdots&\vdots\\
0&0&\cdots&|A|\\
\end{array}
\right]
=|A|I\\
\end{aligned}\\
$$

### 克莱姆(Cramer)法则

$$
Ax=b\\
x=A^{-1}b=\frac{1}{|A|}C^Tb\\
x的第j个分量x_j由C^T的第j行即C的第j列与b的内积得到\\
而C的第j列是A的第j列的代数余子式\\
数字乘以代数余子式联想到行列式按列展开\\
A=\left[\begin{array}{cccccc}a_1&a_2&\cdots&a_j&\cdots&a_n\end{array}\right]\\
将A的第j列替换为b，得到B_j=\left[\begin{array}{cccccc}a_1&a_2&\cdots&b&\cdots&a_n\end{array}\right]\\
x_{j}=\frac{|B_j|}{|A|}\\
$$

### 行列式的应用

行列式的绝对值等于各列向量组成的物体的体积。

考虑二维情况，平行四边形的一半为三角形，$a,b$两个向量为两条边，固定$a$，将$b$分解成$a$方向和垂直的两个方向，任意增加$a$方向的分量，三角形同底等高，面积不变。**决定面积的仅有相互正交的分量的长度**。即在n维空间内，将n个向量投影到各个正交基的方向，即得到各个向量相互正交的分量，组成一个“立方体”，与原物体体积相等。

以上得到结论，方阵$A$经过Gram-Schmidt正交化（不做标准化）后行列式（体积）不变。

因此，证明行列式的绝对值等于体积仅需要考虑立方体的情况。

从行列式的3条性质考虑，行列式可以由3条性质定义，若体积满足3条性质，则两者等价。

性质1，各边为1的立方体体积为1，显然成立。

性质2，交换矩阵的两行(两列)，行列式绝对值不变，对立方体而言仅是换个角度观察，不变，得证。

性质3a，某行（列）数乘$t$，立方体某条边长变为$t$倍，等于$t$个同样的立方体摆在一起，体积变为$t$倍，行列式也变为$t$倍，得证。

性质3b，在立方体中显然。

另一种思路，3条性质对于$I$成立，空间内的所有矩阵可以由$I$经过行变换得到（所有物体可以由立方体某条边与其他边的线性组合得到，且这种组合同底等高，不改变体积），行变换后行列式的值不变，所以体积与行列式仍然相等，得证。

## Lecture 21

### 特征值和特征向量

给定方阵$A$，向量$x$使得$Ax=\lambda x,\lambda$属于复数，则$x$称为$A$的特征向量，$\lambda$称为特征值。

如果$A$是奇异矩阵，零空间内任意向量都是特征向量，零是特征值。

对于$A$的列空间的投影矩阵$P$，任意空间内的向量$x$都是特征向量，$Px=x$，特征值为$1$。任意垂直与空间的向量也是特征向量，特征值为0。

$\sum\lambda=tr(A)=\sum{a_{ii}}$，特征值的和为方阵的迹。

$\Pi\lambda=|A|$，特征值的积为方阵的行列式（韦达定理，或将行列式看成$\lambda$的函数，按不同次方项展开看系数）。
$$
Ax=\lambda x\\
(A-\lambda I)x=0\\
$$
考虑$x$不是零向量的情况，则意味着$A-\lambda I$的零空间有非零向量，它是奇异的，则$|A-\lambda I|=0$，叫做特征方程或特征值方程。
$$
在复数域上，A为n阶方阵，上述方程有n个根\\
\begin{aligned}
&\ \ \ \ \ \ \ |\lambda I-A|=0\\
&\Rightarrow(\lambda-\lambda_1)(\lambda-\lambda_2)\cdots(\lambda-\lambda_n)=0\\
&\Rightarrow\lambda^n+(-(\lambda_1+\lambda_2+\cdots+\lambda_n))\lambda^{n-1}+\cdots=0\\
&\Rightarrow\left|\begin{array}{cccc}
\lambda-a_{11}&-a_{12}&\cdots&-a_{1n}\\
-a_{21}&\lambda-a_{22}&\cdots&-a_{2n}\\
\vdots&\vdots&\cdots&\vdots\\
-a_{n1}&-a_{n2}&\cdots&\lambda-a_{nn}\\
\end{array}\right|=0\\
&\Rightarrow\lambda^n+(-(a_{11}+a_{22}+\cdots+a_{nn}))\lambda^{n-1}+\cdots=0\\
\end{aligned}\\
将行列式展开，\lambda^{n-1}仅能从(\lambda-a_{11})(\lambda-a_{22})\cdots(\lambda-a_{nn})得出\\
（因为在其中挑了n-1项则最后一项也必定在其中否则为0）\\
由此得到上式最后一步推导\\
因此\lambda_1+\lambda_2+\cdots+\lambda_n=a_{11}+a_{22}+\cdots+a_{nn}\\
特征值的和为方阵的迹
$$


## Lecture 22

### 特征值和特征向量

假设$A$有$n$个线性无关的特征向量，将它们按列排成特征向量矩阵$S$。
$$
\begin{aligned}
AS&=A\left[\begin{array}{cccc}x_1&x_2&\cdots&x_n\end{array}\right]\\
&=\left[\begin{array}{cccc}\lambda_1x_1&\lambda_2x_2&\cdots&\lambda_nx_n\end{array}\right]\\
&=\left[\begin{array}{cccc}x_1&x_2&\cdots&x_n\end{array}\right]
\left[\begin{array}{cccc}
\lambda_1&0&\cdots&0\\
0&\lambda_2&\cdots&0\\
\vdots&\vdots&\cdots&\vdots\\
0&0&\cdots&\lambda_n
\end{array}\right]\\
&=S\Lambda \\
\end{aligned}\\
$$
$S^{-1}AS=\Lambda$，$A=S\Lambda S^{-1}$

$if Ax=\lambda x,A^2x=\lambda ^2x$

$A^2$的特征向量与$A$相同，特征值平方

上述的矩阵形式$A^2=S\Lambda S^{-1}S\Lambda S^{-1}=S\Lambda ^2S^{-1}$，$S$不变，说明特征向量不变，$\Lambda$为特征值组成的对角阵，说明特征值变平方。

可推得对$k$次方成立。

如果$A$的特征值均不相同，则$A$有$n$个线性无关的特征向量，可用上述方式对角化（不考虑特征向量为零向量）。如果存在重复特征值，可能但不一定存在$n$个线性无关的特征向量（比如单位阵）。
$$
设A有特征值\lambda_1,\lambda_2,\cdots,\lambda_n且互不相等，对应的特征向量为x_i\\
如果x_1,x_2,\cdots,x_n线性相关，存在不全为0的c_i使得\\
c_1x_1+c_2x_2+\cdots+c_nx_n=0\\
A(c_1x_1+c_2x_2+\cdots+c_nx_n)=0\\
c_1\lambda_1x_1+c_2\lambda_2x_2+\cdots+c_n\lambda_nx_n=0\\
而c_nx_n=-c_1x_1-\cdots-c_{n-1}x_{n-1}\\
代入得\\
c_1(\lambda_1-\lambda_n)x_1+c_2(\lambda_2-\lambda_n)x_2+\cdots+c_{n-1}(\lambda_{n-1}-\lambda_n)x_{n-1}=0\\
写成d_1x_1+d_2x_2+\cdots+d_nx_n=0,d_i不全为0\\
重复上述过程，最后得到m_1x_1+m_2x_2=0\\
m_1\lambda_1x_1+m_2\lambda_2x_2=0\\
m_2(\lambda_2-\lambda_1)x_2=0\\
上式成立只能\lambda_1=\lambda_2，矛盾\\
因此x_i线性无关
$$

### 数列与特征方程与特征值的联系

特征值决定数列增长的趋势（大小\快慢）

给出向量$u_0$，$u_{k+1}=Au_k,u_{k}=A^ku_0$（在数学上，**递推关系（recurrence relation）**，也就是**差分方程（difference equation）**，是一种[递推](https://baike.baidu.com/item/递推)地定义一个序列的方程式：序列的每一项目是定义为前一项的函数。）。

上述形式是一阶差分方程组，因为只含有一阶差分($u_{k+1}$的表达式只包含前一项)。

假设$A$有n个不同的特征向量$x_1,x_2,\cdots,x_n$，那么$u_0=c_1x_1+c_2x_2+\cdots+c_nx_n=Sc$，$u_k=A^ku_0=c_1\lambda_1^kx_1+c_2\lambda_2^kx_2+\cdots+c_n\lambda_n^kx_n=S\Lambda^kc$，$S$是特征向量构成的矩阵。

#### 斐波那契数列

$0,1,1,2,3,5,8,13\cdots,F_{k+2}=F_{k+1}+F_k$，是二阶差分方程，希望将它写成一阶差分方程组的形式。

追加方程$F_{k+1}=F_{k+1}$，定义$u_k=\left[\begin{array}{c}F_{k+1}\\F_k\end{array}\right],u_{k+1}=Au_k=\left[\begin{array}{cc}1&1\\1&0\end{array}\right]u_k$

求解$A$的特征值，解$|A-\lambda I|=0,\left|\begin{array}{cc}1-\lambda&1\\1&-\lambda\end{array}\right|=0,\lambda^2-\lambda-1=0$。注意$\lambda^2=\lambda+1$与$F_{k+2}=F_{k+1}+F_k$形式一致。

求得特征值$\lambda_1=\frac{1}{2}(1+\sqrt5)≈1.618,\lambda_2=\frac{1}{2}(1-\sqrt5)≈0.618$，因此$A$可以对角化，有两个线性无关的特征向量$x_1=\left[\begin{array}{c}\lambda_1\\1\end{array}\right]x_2=\left[\begin{array}{c}\lambda_2\\1\end{array}\right]$

$u_0=\left[\begin{array}{c}1\\0\end{array}\right]$，$u_0$可以写成$u_0=c_1x_1+c_2x_2$，联立求出$c_1,c_2$就可以得到$u_k$表达式，进而得到$F_k=c_1\lambda_1^k+c_2\lambda_2^k$。

将上述二阶差分方程推广到一般形式$F_{k+2}=aF_{k+1}+bF_k$，则可以得到$A=\left[\begin{array}{cc}a&b\\1&0\end{array}\right],|A-\lambda I|=\left|\begin{array}{cc}a-\lambda&b\\1&-\lambda\end{array}\right|=\lambda^2-a\lambda-b=0$

与$F_{k+2}-aF_{k+1}-F_k=0$形式一致。$x_1,x_2$表达式不变，用同样的方法求得$u_k$表达式和$F_k$表达式不变。

因此二阶差分方程通式为$F_k=c_1\lambda_1^k+c_2\lambda_2^k$，$\lambda$解方程求得，$c_1,c_2$将$F_0,F_1$代入通式确定（如果方程的解有虚数，那么由通式可以看出数列为周期数列）。

因此数列增长的快慢由特征值决定，斐波那契数列$F_{100}≈c_1(1.618)^{100}$。

## Lecture 23

### 微分方程组的矩阵表示

$$
\newcommand{\dif}{\mathop{}\!\mathrm{d}}\\
\newcommand{\e}{\mathop{}\!\mathrm{e}}\\
对互相耦合的微分方程组\frac{\mathrm{d}u}{\mathrm{d}t}=Au\\
eg:
\left\{
\begin{aligned}
\frac{\mathrm{d}u}{\mathrm{d}t}&=-u_1+2u_2\\
\frac{\dif{u}}{\dif{t}}&=u_1-2u_2\\
\end{aligned}
\right.\\
$$

若$A$有$n$个线性无关的特征向量，则可以通过特征值和特征向量对方程组进行解耦，又称对角化。

令$u=Sv,S$是特征向量为列构成的矩阵，将$u$代入原方程
$$
\begin{aligned}
S\frac{\mathrm{d}{v}}{\mathrm{d}{t}}&=ASv\\
\frac{\mathrm{d}{v}}{\mathrm{d}{t}}&=S^{-1}ASv\\
\frac{\mathrm{d}{v}}{\mathrm{d}{t}}&=\Lambda v\\
\end{aligned}\\
$$
新方程组不存在耦合。
$$
\left\{
\begin{aligned}
\frac{\mathrm{d}{v_1}}{\mathrm{d}{t}}&=\lambda_1v_1\\
\frac{\mathrm{d}{v_2}}{\mathrm{d}{t}}&=\lambda_2v_2\\
\end{aligned}
\right.\\
v_1=c_1\mathrm{e}^{\lambda_1t}\\
惯用符号表示\\
v(t)=\mathrm{e}^{\Lambda t}v(0)\\
u(t)=S\mathrm{e}^{\Lambda t}S^{-1}u(0)\\
$$
原方程的解为$u(t)=\mathrm{e}^{At}u(0)=S\mathrm{e}^{\Lambda t}S^{-1}u(0),\mathrm{e}^{At}=S\mathrm{e}^{\Lambda t}S^{-1}$。

$\mathrm{e}^{At}$称为矩阵指数，将指数展开成幂级数形式来定义
$$
\mathrm{e}^x=1+\frac{1}{2}x^2+\frac{1}{6}x^3+\cdots\\
\mathrm{e}^{At}=I+At+\frac{(At)^2}{2}+\frac{(At)^3}{6}+\cdots+\frac{(At)^n}{n!}+\cdots\\
\mathrm{e}^x=\sum_{0}^{\infty}{\frac{x^n}{n!}}\\
\frac{1}{1-x}=\sum_{0}^{\infty}{x^n}\\
(I-At)^{-1}=I+(At)+(At)^2+\cdots+(At)^n+\cdots\\
$$
由$\mathrm{e}^{At}$的定义，得
$$
\begin{aligned}
\mathrm{e}^{At}&=SS^{-1}+S\Lambda S^{-1}t+\frac{S\Lambda^2S^{-1}t^2}{2}+\cdots\\
&=S(I+\Lambda t+\frac{(\Lambda t)^2}{2}+\cdots)S^{-1}\\
&=S\mathrm{e}^{\Lambda t}S^{-1}\\
\end{aligned}\\
$$
$\mathrm{e}^{At}$的展开式恒成立，但是上述公式成立的前提是$A$可以用特征值和特征向量对角化，即$A$有$n$个线性无关的特征向量。
$$
\mathrm{e}^{\Lambda t}=\left[
\begin{array}{cccc}
\mathrm{e}^{\lambda_1t}&0&\cdots&0\\
0&\mathrm{e}^{\lambda_2t}&\cdots&0\\
\vdots&\vdots&\cdots&\vdots\\
0&0&\cdots&\mathrm{e}^{\lambda_nt}\\
\end{array}
\right]\\
$$
$S,S^{-1}$固定，因此当$t$不断增长，$\mathrm{e}^{At}$趋向于$0$的条件是所有特征值的实部为负数（$\mathrm{Re}(\lambda) < 0$）。

在复平面上，左半平面的特征值（$\mathrm{Re}(\lambda) ≤ 0$）使得微分方程存在稳定的解，即$t$趋近于无穷时函数的值不变。

在复平面上，以原点为圆心的单位圆内的特征值使得矩阵的幂收敛于$0(|\lambda|<1)$，即这是表达式包含矩阵的幂的函数的稳定区域（$t$趋近于无穷时函数的值不变）。
$$
\mathrm{e}^{i\theta}=\cos\theta+i\sin\theta\\
上述公式将三个函数的泰勒级数在0展开证得\\
(\mathrm{e}^{i\theta})^n=\mathrm{e}^{in\theta}=(\cos\theta+i\sin\theta)^n=\cos n\theta+i\sin n\theta\\
任意复数x=a+bi，令r=\sqrt{a^2+b^2}\\
x=r(\frac{a}{r}+\frac{b}{r}i)，令\alpha=\arccos{\frac{a}{r}}\\
x=r(\cos\alpha+i\sin\alpha)\\
x^n=r^n(\cos\alpha+i\sin\alpha)^n=r^n(\cos{n\alpha}+i\sin{n\alpha})\\
即在复平面上，复数由角度和模长表示\\
复数的n次幂由模长的n次幂和角度旋转n倍表示\\
$$

### 高阶微分方程的求解

对于三阶微分方程
$$
y'''+ay''+by'+cy=0\\
$$
令$u=\left[\begin{array}{c}y''\\y'\\y\end{array}\right]$，增加两个方程$\left\{\begin{aligned}y''&=y''\\y'&=y'\end{aligned}\right.$，将$u$当作未知量，原方程可以写成
$$
u'=
\left[\begin{array}{c}y'''\\y''\\y'\end{array}\right]
=\left[\begin{array}{ccc}
-a&-b&-c\\
1&0&0\\
0&1&0
\end{array}\right]
\left[
\begin{array}{c}y''\\y'\\y\end{array}
\right]
=\left[\begin{array}{ccc}
-a&-b&-c\\
1&0&0\\
0&1&0
\end{array}\right]u
\\
$$
一般来说，对于一个$n$阶微分方程，可以得到$n*n$的矩阵，原方程的系数出现在矩阵的第一行，其余$n-1$行各有一个$1$，表示其余方程($eg:y'=y'$)，这个矩阵使得原来的$n$阶方程转化为一阶向量方程，通过矩阵的特征值可以求得方程的解。

## Lecture 24

### 马尔科夫矩阵（Markov matrix）

马尔科夫矩阵是关于概率的矩阵

1. 所有元素大于等于$0$
2. 每一列的和为$1$

由定义得马尔科夫矩阵的幂还是马尔科夫矩阵。

马尔科夫矩阵一定有特征值为$1$，其他所有的特征值的绝对值（模长）小于等于$1$。
$$
A_{n*n}=\left[\begin{array}{cccc}a_1&a_2&\cdots&a_n\end{array}\right]是马尔科夫矩阵\\
对Ax=\lambda x,设x=\left[\begin{array}{c}x_1\\x_2\\\vdots\\x_n\end{array}\right]\\
Ax=x_1a_1+x_2a_2+x_na_n=u\\
此处定义|u|为u各元素的和\\
\begin{aligned}
|u|&=(x_1a_{11}+x_2a_{12}+\cdots+x_na_{1n})\\
&\ \ \ \ +(x_1a_{21}+x_2a_{22}+\cdots+x_na_{2n})\\
&\ \ \ \ +(x_1a_{n1}+x_2a_{n2}+\cdots+x_na_{nn})\\
&=x_1(a_{11}+a_{21}+\cdots+a_{n1})\\
&\ \ \ \ +x_2(a_{12}+a_{22}+\cdots+a_{n2})\\
&\ \ \ \ +\cdots\\
&\ \ \ \ +x_n(a_{1n}+a_{2n}+a_{nn})\\
&=x_1|a_1|+x_2|a_2|+\cdots+x_n|a_n|\\
&=x_1+x_2+\cdots+x_n\\
&=|x|\\
\end{aligned}\\
因此|\lambda x|=\lambda|x|=|x| \Rightarrow 若|x|≠0则\lambda=1，若\lambda≠1，则|x|=0\\
$$
定义2决定了一定有特征值$1$。$A-I$在$A$的每一列中减去$1$，每一列的和为$0$，将行向量全部相加得$0$因此行向量线性相关，因此$A-I$是奇异矩阵，$(A-I)x=0$有非零解，$1$是特征值，$x$为对应的特征向量。

$A$和$A^T$的特征值是一样的。考虑矩阵$A-\lambda I$和$(A-\lambda I)^T=A^T-\lambda I$，$|A-\lambda I|=0 \Rightarrow |(A-\lambda I)^T|=0 \Rightarrow |A^T-\lambda I|=0$，因此第一个方程的解就是第三个方程的解，$A$和$A^T$特征值相等。

考虑$A^T$，$A$的列和为$1$则$A^T$的行和为$1$。
$$
为简便将A^T记作A，此处A行和为1，每个元素都非负\\
对Ax=\lambda x,设x_k为x绝对值最大的元素\\
Ax的任意元素有\\
\begin{aligned}
x_i&=a_{i1}x_1+a_{i2}x_2+\cdots+a_{in}x_n\\
|x_i|&≤a_{i1}|x_1|+a_{i2}|x_2|+\cdots+a_{in}|x_n|\\
&≤a_{i1}|x_k|+a_{i2}|x_k|+\cdots+a_{in}|x_k|\\
&=|x_k|\\
\end{aligned}\\
所以Ax相比于x没有放大，|\lambda|只能小于等于1\\
否则A^{100}仍为马尔科夫矩阵，A^{100}x不会放大x,A^{100}x=\lambda^{100}x\\
\lambda^{100}x的某个元素将接近无穷大，矛盾\\
$$
哈佛课本证明：

> $$
> 为简便将A^T记作A，此处A行和为1，每个元素都非负\\
> A为实数矩阵，\lambda和x属于复数域，此处|x|表示模长\\
> 对Ax=\lambda x,取x的分量x_k使得任取j，|x_j|≤|x_k|\\
> Ax的第k个分量\\
> \begin{aligned}
> \sum_{i=1}^{n}{a_{ki}x_i}&=\lambda x_k\\
> |\sum_{i=1}^{n}{a_{ki}x_i}|&≤\sum_{i=1}^{n}{a_{ki}|x_i|}\\
> &≤\sum_{i=1}^{n}{a_{ki}|x_k|}\\
> &=|x_k|\\
> |\lambda x_k|=|\lambda||x_k|&≤|x_k|\\
> |\lambda|&≤1
> \end{aligned}\\
> $$
>
> 

对于$u_k=A^ku_0,\lambda_1=1,|\lambda_i|<1$，将$u_0$展开成$A$的特征向量的形式（需要$A$有n个线性无关的特征向量），可得$u_k$的稳态为$c_1x_1(+ c_ix_i\ if\ \lambda_i=1)$，$u_k,u_0$的元素之和相等。(如果某个$\lambda=-1$则$u_k$会振荡，不收敛，没有稳态。)

$x_1(\lambda_1=1)$的所有元素为非负值（证明似乎比较复杂）

### 傅里叶级数

标准正交矩阵$Q$的各列为$q_i$，它们构成$n$维空间的一组基，任意向量$v=\sum_{i=1}^{n}{x_iq_i}=Qx,x=Q^{-1}v=Q^Tv,x_i=q_i^Tv$

傅里叶级数：周期函数$f(x)=a_01+a_1\cos{x}+b_1\sin{x}+a_2\cos{2x}+b_2\sin{2x}+\cdots\\=\sum_{i=0}^{+\infty}{a_i\cos{ix}+b_i\sin{ix}}$

与向量内积$v^Tw=\sum{v_iw_i}$类似，定义函数内积，假设有两个连续函数$f(x),g(x)$，它们的内积$f^Tg=\int_{0}^{2\pi}{f(x)g(x)\mathrm{d}{x}}$（上述周期函数的周期为$2\pi$）

由定义，$\int_{0}^{2\pi}\cos{x}\sin{x}\mathrm{d}{x}=\frac{1}{2}\sin^2{x}|_{0}^{2\pi}=0$，$\cos{nx}$与$\sin{nx}$正交。

同样的方法可以证明傅里叶级数的不同项都是正交的，称为无穷正交基。

傅里叶级数系数公式：与向量类似，使用内积求系数。例如求$a_n,\int_{0}^{2\pi}f(x)\cos{nx}\mathrm{d}{x}=a_n\int_{0}^{2\pi}{\cos^2{nx}\mathrm{d}{x}}$

## Lecture 25

复习课

## Lecture 26

### 谱定理

典型的（一般的）实对称矩阵$A$（相对于特殊的单位阵$I$）的特征值也是实数，可以选出$n$个正交的特征向量（例如对$I$就需要选），组成的矩阵$S$可化为标准正交矩阵$Q$，$A=S\Lambda S^{-1}=Q\Lambda Q^T$

> **(Spectral Theorem)** Every **real symmetric** matrix has the factorization $S=Q\Lambda Q^T$ with **real** eigenvalues in $\Lambda$ and **orthonormal** eigenvectors in the columns of $Q$:
>
> **Symmetric diagonalization**  $S=Q\Lambda Q^{-1}=Q\Lambda Q^T \ with \ Q^{-1}=Q^T$

谱是矩阵特征值的集合

$Ax=\lambda x \Rightarrow \overline{A}\overline{x}=\overline{\lambda}\overline{x}$总是成立（分成实部虚部相乘再相加），对于实矩阵，$A\overline{x}=\overline{\lambda}\overline{x}$,说明如果$A$有特征值和特征向量$\lambda,x$就一定有$\overline{\lambda},\overline{x}$
$$
Ax=\lambda x\\
\Rightarrow\overline{x}^TAx=\overline{x}^T\lambda x\\
又有A\overline{x}=\overline{\lambda}\overline{x}\\
\Rightarrow \overline{x}^TA^T=\overline{x}^T\overline{\lambda}\\
\Rightarrow\overline{x}^TA^Tx=\overline{x}^T\overline{\lambda}x\\
由A=A^T得\\
\overline{x}^TAx=\overline{x}^T\lambda x=\overline{x}^TA^Tx=\overline{x}^T\overline{\lambda}x\\
\lambda\overline{x}^Tx=\overline{\lambda}\overline{x}^Tx\\
不考虑零向量，\lambda=\overline{\lambda}\\
$$
因此$\lambda$为实数，$x$由$(A-\lambda I)x=0$解得，也是实数。

**对称矩阵不同的特征值对应的特征向量一定是垂直的**
$$
设对称矩阵S有不同的特征值\lambda_1,\lambda_2且Sx=\lambda_1x,Sy=\lambda_2y\\
(\lambda_1x)^Ty=x^T\lambda_1y=x^TS^Ty=x^TSy=x^T\lambda_2y\\
\lambda_1x^Ty=\lambda_2x^Ty \Rightarrow x^Ty=0\\
$$
对称矩阵一定有$n$个线性无关的特征向量

> 课本的不严谨证明：
> $$
> S如果特征值都不相同那么一定有n个正交的特征向量\\
> 此时S可以分解成对角阵\\
> 如果S有相同的特征值，考虑矩阵C\\
> C的对角线为1c,2c,\cdots,nc,其余为0\\
> Sx=\lambda_1x,Cy=\lambda_2y\\
> S+C一定特征值都不相同，如果相同则再次进行上述过程\\
> 令c\rightarrow0,则S+C近似于对称矩阵S且特征向量相互正交\\
> 所以对称矩阵S一定有n个正交的特征向量(无语)\\
> $$
> 

$A=Q\Lambda Q^T,Q=\left[\begin{array}{cccc}q_1&q_2&\cdots&q_n\end{array}\right]$，将乘积展开成列乘行的形式，$A=\sum{\lambda_iq_iq^T_i}$，注意投影矩阵$P=\frac{q_iq^T_i}{q^T_iq_i}$,因为$q_i$为单位向量，所以$q_iq^T_i$为投影矩阵，因此对称矩阵可以分解成一些特征向量的投影矩阵的组合，每一个投影矩阵所投影到的特征向量是标准正交的。

对称矩阵的主元正负数的个数与特征值正负数的个数相同。(课本缺乏严谨证明)

### 正定矩阵

所有特征值都是正数的对称矩阵是正定矩阵$\Leftrightarrow$所有的主元为正数的对称矩阵是正定矩阵$\Leftrightarrow$所有的子行列式（从左上开始$1*1,2*2,\cdots$）都是正的对称矩阵是正定矩阵（所以正定矩阵也都是方阵）

## Lecture 27

### 复向量

向量$z$的元素为复数，$z \in C^n,|z|^2=\overline{z}^Tz=z^Hz,H(Hermit埃尔米特),z^H就表示\overline{z}^T,z^Hz$也是向量内积。

### 复矩阵

lecture 26的谱定理扩充到复数域，$A^T=A$的要求变为$\overline{A}^T=A$，满足这个要求的矩阵称为Hermit（埃尔米特）矩阵（对应实数域对称矩阵），特征值是实数，特征向量相互垂直（证明同实矩阵，$T$换$H$）。

酉矩阵unitary matrix（对应实数域标准正交矩阵）$Q$的各列$q_i$相互垂直，满足$q_i^Hq_j=0(i≠j),q_i^Hq_j=1(i=j),Q^HQ=I$

### 傅里叶矩阵

傅里叶矩阵是酉矩阵
$$
F_n=\frac{1}{\sqrt{n}}\left[\begin{array}{ccccc}
1&1&1&\cdots&1\\
1&w&w^2&\cdots&w^{n-1}\\
1&w^2&w^4&\cdots&w^{2(n-1)}\\
\vdots&\vdots&\vdots&\cdots&\vdots\\
1&w^{n-1}&w^{2(n-1)}&\cdots&w^{(n-1)(n-1)}\\
\end{array}\right]\\
F_{ij\\i,j=0,\cdots,n-1}=w^{ij}\\
w^n=1\\
w=\mathrm{e}^{\frac{i2\pi}{n}}在复平面的单位圆上，是1的n次方根，w称为原根\\
矩阵的行列从0开始编号\\
w^0,w^1,w^2,\cdots,w^{n-1}是x^n=1的n个解\\
令t=x^k(t \in N^+)\\
w^{0k},w^{1k},w^{2k},\cdots,w^{k(n-1)}是t^n=1的n个解t\\
t^n=1可以写成(t-t_0)(t-t_1)\cdots(t-t_{n-1})=0\\
由韦达定理或待定系数\\
t_0+t_1+\cdots+t_{n-1}=0\\
即w^{0k}+w^{1k}+w^{2k}+\cdots+w^{k(n-1)}=0\\
F_n任意两列q_m,q_l(l≠m)的内积\\
q_m^Hq_l=\frac{1}{n}\sum_{k=0}^{n-1}{\overline{w^{mk}}w^{lk}}
=\frac{1}{n}\sum_{k=0}^{n-1}{\mathrm{e}^{-\frac{i2\pi}{n}mk}\mathrm{e}^{\frac{i2\pi}{n}lk}}
=\frac{1}{n}\sum_{k=0}^{n-1}{\mathrm{e}^{\frac{i2\pi}{n}k(l-m)}}\\
不失一般性，设l>m\\
q_m^Hq_l=\frac{1}{n}(w^{0(l-m)}+w^{1(l-m)}+w^{2(l-m)}+\cdots+w^{(n-1)(l-m)})=0\\
显然l=m时上式为1\\
F_n^HF_n=I\\
F_n^{-1}=F_n^H\\
F_n^{-1}与F_n具有相同的性质\\
$$

### 快速傅里叶变换(FFT)

$$
F_{2n}=
\left[\begin{array}{cc}
I_n&D_n\\
I_n&-D_n\\
\end{array}\right]
\left[\begin{array}{cc}
F_n&\\
&F_n\\
\end{array}\right]
\left[\begin{array}{c}
even\mathrm{-}odd\\
permutation\\
\end{array}\right](\leftarrow P_{2n})\\
D_n=\left[\begin{array}{cccccc}
1& & & & &\\
& w & & & &\\
& & w^2 & & &\\
& & & . & &\\
& & & & . &\\
& & & & & w^{n-1}\\
\end{array}\right]\\
P_{2n}=\left[\begin{array}{ccccccc}
1 & & & & & & \cdots\\
& & 1 & & & & \cdots\\
& & & & 1 & & \cdots\\
\vdots & & \vdots & & \vdots & \\
& 1 &\\
& & & 1 \\
& & & & & 1 &\cdots\\
& \vdots & & \vdots & & \vdots\\
\end{array}\right]\\
P_{2n}的前n行奇数列有1，呈对角线排列；后n行偶数列有1，呈对角线排列\\
通过FFT，将计算F_n的n^2次乘法降低为\frac{1}{2}n\log{n}次\\
$$

## Lecture 28（线代的重点，将所有知识融合）

### 正定矩阵

以下针对实矩阵

$A$是对称矩阵，$A$满足以下条件之一则为正定矩阵，各个条件等价

1. 特征值都$>0$
2. 从左上开始的子行列式（主子行列式）都$>0$
3. 主元都$>0$
4. $x^TAx>0,x$为不等于零的任意向量

上述大于号取等的时候称为半正定矩阵。

可以通过主元来迅速判断是否正定

以二阶为例，$x^TAx=\left[\begin{array}{cc}x&y\end{array}\right]\left[\begin{array}{cc}a&b\\b&c\end{array}\right]\left[\begin{array}{c}x\\y\end{array}\right]=ax^2+2bxy+cy^2=f(x,y)$

$A$正定则$f(x,y)>0,f(x,y)=a(x+\frac{b}{a}y)^2+(c-\frac{b^2}{a})y^2$，注意$a,c-\frac{b^2}{a}$为$A$的主元，$\frac{b}{a}$为消元的系数，配方的过程其实就是高斯消元的过程，可以看出$f(x,y)$保证大于0就等价于主元大于0（同时由没有严谨证明的定理得到特征值大于0），同时1阶行列式对应主元$a$，二阶行列式对应主元$a(c-\frac{b^2}{a})$，即子行列式的值跟主元的乘积一一对应。

$f(x)$有极小值的条件为一阶导数等于0，二阶导数大于0。推广到多元的情况，以二阶为例，$f(x,y)$的二阶导数矩阵$\left[\begin{array}{cc}f_{xx}&f_{xy}\\f_{yx}=f_{xy}&f_{yy}\end{array}\right]$为正定矩阵，在$x和y$方向的二阶导数必须大于0，并且需要足够大来抵消混合导数的影响。

令$f(x,y)=1$则可以截出二维平面上的椭圆，椭圆的主轴的方向为特征向量的方向，长度为特征值的大小（未证明），称为主轴定理（力学）。

以上推广到$n$维都成立。

## Lecture 29

### 逆矩阵的特征值等于原矩阵特征值的倒数

$$
A可逆方阵\\
Ax=\lambda x\\
A^{-1}Ax=\lambda A^{-1}x\\
不考虑零向量,且A可逆\lambda≠0\\
\frac{1}{\lambda}x=A^{-1}x\\
根据定义，\frac{1}{\lambda}为A^{-1}特征值，x为对应特征向量\\
$$

### 正定矩阵

$A,B$为正定矩阵则$A+B$为正定矩阵（$x^TAx>0$证）

任意矩阵$A_{m*n},A^TA$一定是半正定矩阵。$x^TA^TAx=(Ax)^T(Ax)=|Ax|^2≥0$

我们希望上式仅当$x$为零向量时等号成立，即$A$的零空间内只有零向量，$A$没有自由列，$A$列满秩，$A$的各列线性无关。所以矩阵$A$各列线性无关时$A^TA$一定是正定矩阵。

### 相似矩阵

存在可逆矩阵$M_{n*n}$使得$A_{n*n},B_{n*n}$可以表示为$B=M^{-1}AM$，则称$A,B$为相似矩阵

相似矩阵具有同样的特征值
$$
Ax=\lambda x\\
AMM^{-1}x = \lambda x\\
M^{-1}AMM^{-1}x=\lambda M^{-1}x\\
B(M^{-1}x)=\lambda(M^{-1}x)\\
因此\lambda为B的特征值，对应的特征向量为M^{-1}x\\
除此之外，A和B线性无关的特征向量的数量也相等\\
否则举例设A有无关的x,y，B的M^{-1}x,M^{-1}y相关\\
M^{-1}x=kM^{-1}y\\
MM^{-1}x=kMM^{-1}y\\
x=ky\\
矛盾\\
$$
最好的相似矩阵是对角阵，特征值不变，特征向量可以选为标准正交向量组($eg:(1,0)(0,1)$)，因此对于有$n$个线性无关的特征向量的矩阵$A$来说，$\Lambda=S^{-1}AS$是最好的相似矩阵。

特征值互不相同的矩阵自然有$n$个线性无关的特征向量，下面考虑特征值相同的情况。

特征值相同分为两类，第一类矩阵为$nI$，它们仅和自己相似。对任意可逆矩阵$M,M^{-1}nIM=nI$，无论$M$是什么，都不会增加新的相似矩阵（比如$\left[\begin{array}{cc}4&\\&4\end{array}\right]$）。

另一类矩阵比如$\left[\begin{array}{cc}4&1\\&4\end{array}\right]$，它们无法对角化，否则就和第一类矩阵相似（如果可以对角化，可以将对角阵的数字变为第一类矩阵，系数提到$M$中，就和第一类相似），无法对角化意味着没有$n$个线性无关的特征向量。这类矩阵中最简洁的形式是$\left[\begin{array}{cc}4&1\\&4\end{array}\right]$这样的矩阵（上方只有一个1），它们最接近对角阵又无法对角化，称为Jordan标准型（Jordan form）。

注意有相同特征值以及相同数量线性无关的特征向量的矩阵不一定是相似矩阵
$$
eg:
\left[\begin{array}{cccc}
0&1&0&0\\
0&0&1&0\\
0&0&0&0\\
0&0&0&0\\
\end{array}\right],
\left[\begin{array}{cccc}
0&1&0&0\\
0&0&0&0\\
0&0&0&1\\
0&0&0&0\\
\end{array}\right]\\
$$
以上两个矩阵不相似，Jordan认为第一个矩阵分为一个3阶的块和一个1阶的块，第二个矩阵分为两个2阶的块。

$i$阶若尔当块(Jordan block)：
$$
J_i=\left[\begin{array}{ccccc}
\lambda_i&1\\
&\lambda_i&1\\
&&\cdots&\cdots\\
&&&\lambda_i&1\\
&&&&\lambda_i\\
\end{array}\right]\\
$$
每个Jordan block只有一个重复的特征值，对角线上全是$\lambda_i$，下面是$0$,对角线上一条斜线全是$1$，只有一个特征向量。

以上两个方阵有相同特征值以及相同数量线性无关的特征向量，但是分块大小不一样，所以并不相似。

若尔当(Jordan)定理：每一个方阵$A$都相似于一个若尔当阵(Jordan matrix)$J$。
$$
J=\left[\begin{array}{cccc}
J_1\\
&J2\\
&&\ddots\\
&&&J_d\\
\end{array}\right]\\
J_i为i阶Jordan块\\
块的数量和J特征向量的数量相等，因为每个块对应1个无关的特征向量(n维)\\
考察J_i时，将J_i的特征向量填入n个元素的对应位置，其余置0\\
这里不写特征值的数量是因为特征值可能相等\\
$$
在$n$维空间中，当$d=n$时，则$J=\Lambda$，每个分块只有1阶，也是现在好的（比较重要的）情况。

## Lecture 30

### AB的特征值与BA特征值相等

$$
A,B为n阶方阵\\
ABx=\lambda x\\
若\lambda≠0\\
BA(Bx)=\lambda(Bx)\\
不考虑x为零向量\\
Bx不为零向量否则ABx=0矛盾\\
所以\lambda为BA的特征值\\
若\lambda=0\\
则|AB|=|BA|=0，BA为奇异矩阵\\
存在非零向量x使得BAx=0=\lambda x\\
所以\lambda为BA的特征值\\
证毕\\
$$



### 奇异值分解（SVD）

~~教授表示这是矩阵最终和最好的分解~~

几何表示：行空间的一组标准正交基$v$，施加矩阵$A$变换，得到列空间的一组标准正交基$u$的倍数$\sigma>0,eg:Av_1=\sigma_1u_1$，写成矩阵形式$AV=U\Sigma$。特例是正定矩阵$AQ=Q\Lambda$。

$A=U\Sigma V^{-1},V$是标准正交方阵，$A=U\Sigma V^T$,消去$U$,求解$V$
$$
A^TA=V\Sigma^T\Sigma V^T=V\left[\begin{array}{ccc}\sigma_1^2&\\&\sigma_2^2\\&&\ddots\\\end{array}\right]V^T\\
A^TAV=V\left[\begin{array}{ccc}\sigma_1^2&\\&\sigma_2^2\\&&\ddots\\\end{array}\right]\\
$$
因此$\sigma_i^2$是$A^TA$的特征值，$V$是特征向量矩阵

同理$AA^T=U\Sigma\Sigma^TU^T$,特征值与$A^TA$相同($AB,BA$特征值相同)，特征向量矩阵为$U$

假设矩阵$A_{m*n}$的秩为$r,(v_1,\cdots,v_r)$是行空间的标准正交基，$(u_1,\cdots,u_r)$是列空间的标准正交基，用$(v_{r+1},\cdots,v_n)将v$补充完整，它们是$N(A)$的标准正交基，$(u_{r+1},\cdots,u_m)将u$补充完整，它们是$N(A^T)$的标准正交基，$V_{n*n},U_{m*m},\Sigma_{m*n},\Sigma$从左上开始的$r*r$个元素构成对角阵，其余全0，$r$个非零元素由计算$A^TA$后求特征值得到，一般可以从$A^TA$的秩（是否满秩）、行列式（特征值的积，不满秩则行列式0则必定有特征值0）、迹（特征值的和）快速求出。

#### 理解

实际上$U,V$并不能随便选择行空间和列空间的标准正交基，教授上课时没有说清楚，$U,V$需要满足特征向量的条件，仍然需要计算$A^TA,AA^T$得到

做SVD需要的$U,V$恰好可以作为矩阵的4个空间的基，所以SVD可以理解为从矩阵的4个空间寻找合适的基的过程。对于$Av_{1,\cdots,r}=\sigma_{1,\cdots,r}u_{1,\cdots,r}$,将行空间的基映射到列空间中，恰好映射成与列空间的基同一方向，$u,v$需要满足的条件就是它们是$AA^T,A^TA$的特征向量。$V$中还剩$n-r$个特征向量的方向，$N(A)$恰好是$n-r$维，因此刚好可以从零空间里找到满足条件的基补足$V$（$V$共$n$个方向，行空间和零空间加起来也是$n$个方向，所以一定可以一一对应），此时$Av=0$，等式右边仅需要满足新的向量与已经找好的$u$垂直且为$AA^T$的特征向量，特征向量的方向还剩$m-r$个，从左零空间中恰好可以找到这样的基，所以可以补全$U$，对应的$\sigma$填0，不失一般性设$n>m$，否则先求$A^T$即可，此时因为$Av_{>r}=0,\Sigma$剩下的部分全部填0即可。

## Lecture 31

### 线性变换

判断线性变换的两大条件：加法和数乘，两种常见的向量运算，线性变换应该保证这两种运算的不变性
$$
T(v+w) = T(v) + T(w)\\
T(cv)=cT(v)\\
$$
显然$T(0)=0$对线性变换一定成立

平面平移不是一个线性运算，因为$T(0)≠0$

每一个矩阵都表示一个线性变换，矩阵乘向量表示对向量进行变换，很容易验证矩阵乘向量满足两大条件

只要确定空间的基的变换就可以确定整个输入空间的变换

坐标源自基，每个向量都可以表示成基的线性组合，将基定作坐标轴，基的长度定义为单位1，组合的系数就是坐标值，表示向量由多少个基组成。

求线性变换对应的矩阵的基本方法：给定两组基$v,w$，将$T(v_i)$写成$w$的线性组合的形式，组合的系数就是矩阵的第$i$列。

## Lecture 32

### 基变换

对图像$P$,原来用标准基表示，在图像压缩中会用别的基表示，新的基为$W$（常见傅里叶基，小波基），基变换为$P=Wc,c=W^{-1}P$，$c$是新的基的系数。

因此对于$W$，要求计算快速，求逆快速，小波基由$1,-1,0$构成，并且标准正交，构成正交矩阵，因此$W^{-1}=W^T$

对基的第二个要求是少量的基向量就可以接近图像，以此略去系数小的基达到压缩（本课的录像用傅里叶基）

给定变换$T$和两组基$v,w$，$T$在$v$上对应的变换矩阵为$A$(以$v$同时作为输入和输出基，求输出相应系数得到的矩阵，变换后的矩阵$\left[\begin{array}{ccc}T(v_1)&\cdots&T(v_n)\end{array}\right]=输出基\left[\begin{array}{ccc}v_1&\cdots&v_n\end{array}\right]*变换矩阵A,$见 lecture31)，$T$在$w$上对应的变换矩阵为$B$，则$A,B$为相似矩阵，$B=M^{-1}AM,M$是基变换矩阵（教授没说，理解是$v_i$用$w$表示时各个$w_{j=1,\cdots,n}$对应的系数作为元素的向量$c_i$组成第$i$列的矩阵）。

## Lecture 33

### A具有正交的特征向量的条件

$AA^T=A^TA$

反对称矩阵$A^T=-A$的特征向量是正交的

典型的有对称阵、反对称阵、正交矩阵

### 正交矩阵的特征值绝对值为1

正交矩阵对向量的作用只有旋转，没有放缩
$$
Qx=\lambda x\\
(Qx)^TQx=x^TQ^TQx=x^Tx=\lambda x^T\lambda x=\lambda^2x^Tx\\
不考虑零向量，|\lambda|=1\\
$$

## Lecture 34

### $rank(A^TA)=rank(A)=rank(A^T)=rank(AA^T)$

行秩等于列秩，秩表示线性无关的行数和列数，因此$rank(A)=rank(A^T)$显然
$$
Ax=0\\
A^TAx=0\\
因此Ax=0的解也是A^TAx=0的解\\
A^TAx=0\\
x^TA^TAx=0\\
Ax=0\\
因此A^TAx=0的解也是Ax=0的解\\
因此Ax=0与A^TAx=0的解相同\\
意味着有相同数量的主列、自由列、秩\\
rank(A^TA)=rank(A)\\
证明rank(A)=rank(AA^T)，将上面的A换成A^T即可\\
$$

### 左逆、右逆、伪逆

$A$列满秩，$r=n<m$，则$A^TA$满秩可逆，$A$的左逆为$A_{left}^{-1}=(A^TA)^{-1}A^T$

$A$行满秩，$r=m<n$，则$AA^T$满秩可逆，$A$的右逆为$A_{right}^{-1}=A^T(AA^T)^{-1}$

$A$乘左逆得到投影矩阵，投影到列空间

右逆乘$A$得到行空间的投影矩阵

#### 行空间和列空间的向量一一对应

$$
x,y是A的行空间中不同向量则Ax≠Ay\\
若Ax=Ay\\
A(x-y)=0\\
则x-y在A的零空间中，而零空间是行空间的正交补，x-y属于行空间\\
因此x-y=0,x=y,矛盾\\
$$

如果只看行空间和列空间，则$A$是可逆的，从行空间到列空间的映射是$A$，从列空间到行空间的映射就是伪逆，记作$A^+,y=Ax,x=A^+y=A^+Ax$

大多数矩阵$r<m且r<n$，此时最小二乘法常用的$A^TA$不满秩不可逆，因此需要伪逆。多用在统计学中

#### 求伪逆

一种方法是从SVD开始
$$
A_{m*n}=U_{m*m}\Sigma_{m*n}V_{n*n}^T\\
\Sigma_{m*n}=\left[\begin{array}{cccccc}
\sigma_1\\
&\sigma_2\\
&&\ddots\\
&&&\sigma_r\\
&&&&0\\
&&&&&\ddots\\
\end{array}\right]\\
\Sigma_{n*m}^+=\left[\begin{array}{cccccc}
1/\sigma_1\\
&1/\sigma_2\\
&&\ddots\\
&&&1/\sigma_r\\
&&&&0\\
&&&&&\ddots\\
\end{array}\right]\\
A_{n*m}^+=V_{n*n}\Sigma_{n*m}^+U_{m*m}^T\\
\Sigma^+填0的地方可以填其他数，但是0是最简最小的矩阵\\
$$

## Lecture 35

总复习



