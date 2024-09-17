[toc]

# MIT18.065 数据分析、信号处理和机器学习中的矩阵方法

https://www.bilibili.com/video/BV1b4411j7V3?p=2

## Lecture 1 The Column Space of A Contains All Vectors Ax

### A=CR

$$
\left[\matrix{
2&1&3\\
3&1&4\\
5&7&12\\
}\right]
=\left[\matrix{
2&1\\
3&1\\
5&7\\
}\right]
\left[\matrix{
1&0&1\\
0&1&1\\
}\right]
$$

$A=CR$，$C$是从左到右取出$A$中线性无关的非零列，$R$的列是由$C$的各列组成$A$的各列的线性组合的系数构成。同时$C$是$A$列空间的基，$R$是行空间的基，也可以看作$C$中各行由$R$的各行组成$A$的各行的线性组合的系数构成。列空间和行空间的基数量相等，由此行秩=列秩。

$R$是$A$的$RREF(reduced\ row\ echelon\ form)$简化行阶梯形式（不包括全零行），即消元使得主元系数为$1$，且其他行该主元系数为$0$

### A=CMR

$C$同上节，$R$是从上到下直接从$A$取出的线性无关的非零行，那么$rank(A_{m*n})=r,C_{m*r},R_{r*n},M_{r*r}$

为了求$M$，注意$C^TC,RR^T$满秩可逆

$C^TAR^T=C^TCMRR^T,M=(C^TC)^{-1}C^TAR^T(RR^T)^{-1}$

这个分解的意义在于保存了$A$的属性和原数据，$QR$分解和$SVD$中这些属性丢失。例如$A$非负则$CR$非负，$A$稀疏则$CR$稀疏。

## Lecture 2 Multiplying and Factoring Matrices

### LU分解的解释

$\left[\matrix{2&3\\4&7}\right]$通过消元得到$\left[\matrix{2&3\\0&1}\right]$，第二行减去第一行的2倍
$$
\left[\matrix{2&3\\4&7}\right]=\left[\matrix{1&0\\2&1}\right]\left[\matrix{2&3\\0&1}\right]=\left[\matrix{1\\2}\right]\left[\matrix{2&3}\right]+\left[\matrix{0\\1}\right]\left[\matrix{0&1}\right]=\left[\matrix{2&3\\4&6}\right]+\left[\matrix{0&0\\0&1}\right]
$$

$$
A_1=LU=(col_1\ of\ L)(row_1\ of\ U)+\left[\matrix{0&O\\O&A_2}\right]\\
=(col_1\ of\ L)(row_1\ of\ U) + (col_2\ of\ L)(row_2\ of\ U) + \left[\matrix{0&0&O\\0&0&O\\O&O&A_3}\right]
$$

在消元过程中，第一行不需要操作，所以$L$的第一行为$1,0...$，第一列为为了消去第一列元素所乘的第一行系数，$U$的第一行为$A$的第一行，因此$(col_1\ of\ L)(row_1\ of\ U)$包含了第一行第一列的所有信息，剩下的$A_2$为原矩阵用第一行消元后剩余的信息。同理继续分解$A_2$，$L$为下三角阵，所以$(col_2\ of\ L)(row_2\ of\ U)$不会在$(col_1\ of\ L)(row_1\ of\ U)$的基础上再给第一行的结果增加信息，且$col_2\ of\ L$是在$A$完成第一列消元后的矩阵上继续消元得到（即在基础上为了消去第二列元素所乘的第二行系数），因为后续不会再对第二行操作，所以$U$的第二行即为此时消元第一列后的$A$的第二行，也是$A_2$第一行，所以$(col_1\ of\ L)(row_1\ of\ U)+(col_2\ of\ L)(row_2\ of\ U)$包含了原矩阵第一二行和第一二列所有信息，其中$(col_2\ of\ L)(row_2\ of\ U)$包含$A_2$第一行第一列信息，$A_3$包含剩余信息。

~~写了那么多其实这部分内容并不重要，在这门课不会再出现了~~

## Lecture 3 Orthonormal Columns in Q Give $Q^TQ=I$

当$Q$是方阵时，$QQ^T=I$也成立，因为对方阵来说，左逆等于右逆

$(Qx)^TQx=||Qx||^2=x^TQ^TQx=x^Tx=||x||^2,||Qx||=||x||$

$Q$不改变向量的长度

例如对二维平面有$Q=\left[\matrix{\cos{\theta}&-\sin{\theta}\\\sin{\theta}&\cos{\theta}}\right],Q\left[\matrix{1\\0}\right]=\left[\matrix{\cos{\theta}\\\sin{\theta}}\right],Q\left[\matrix{0\\1}\right]=\left[\matrix{-\sin{\theta}\\\cos{\theta}}\right]$

### HouseHolder reflection

$Q$(rotation matrix)将坐标轴旋转了$\theta$，等于将整个平面的向量旋转了$\theta$

$Q=\left[\matrix{\cos{\theta}&\sin{\theta}\\\sin{\theta}&-\cos{\theta}}\right],Q\left[\matrix{1\\0}\right]=\left[\matrix{\cos{\theta}\\\sin{\theta}}\right],Q\left[\matrix{0\\1}\right]=\left[\matrix{\sin{\theta}\\-\cos{\theta}}\right]$

$Q$(reflect matrix，反射矩阵)将向量关于直线$y=\cos{\frac{\theta}{2}}x$做对称

高阶的反射矩阵可以由HouseHolder(豪斯霍尔德) reflection算法获得，n阶向量$u^Tu=1,H=I-2uu^T$。$H$是对称正交矩阵

$H^TH=H^2=I-4uu^T+4uu^Tuu^T=I$

### Hadamard matrices

$$
H_2=\frac{1}{\sqrt{2}}\left[\matrix{1&1\\1&-1}\right],H_4=\frac{1}{\sqrt{4}}\left[\matrix{H_2&H_2\\H_2&-H_2}\right],H_{2^{n+1}}=\frac{1}{\sqrt{2^{n+1}}}\left[\matrix{H_{2^n}&H_{2^n}\\H_{2^n}&-H_{2^n}}\right]
$$

$H$是正交矩阵，由$1,-1$构成。对$H_n$，$n$是4的整数倍时仍为Hadamard matrix，即由$1,-1$构成的正交矩阵，为2的次幂时如上构造，否则有另外的构造方法。在傅里叶变换、编码、信号有重要作用。

### 正交矩阵的特征向量矩阵

置换矩阵（permutation matrix）是正交矩阵，且特征向量相互正交，n阶置换矩阵的特征向量矩阵是n阶傅里叶矩阵，例如

$Q_4=\left[\matrix{&1&&\\&&1&\\&&&1\\1&&&}\right]$的特征向量矩阵是4阶傅里叶矩阵$F_4=\left[\matrix{1&1&1&1\\1&i&i^2&i^3\\1&i^2&i^4&i^6\\1&i^3&i^6&i^9}\right]$

## Lecture 4 Eigenvalues and Eigenvectors

假设$A_{n*n}$有$n$个线性无关的特征向量$x$，任意向量$v$可以写成$v=\sum_{i=1}^{n}{c_ix_i},v_k=A^kv=\sum_{i=1}^{n}{c_i\lambda_i^kx_i}$

$\frac{\mathrm{d}{v}}{\mathrm{d}{t}}=Av$的解为$v=\sum_{i=1}^{n}{C_ic_i\mathrm{e}^{\lambda_it}x_i},C$为需要初始值确定的常数。

### 相似矩阵有相同特征值

$A_{n*n},B_{n*n}$均可逆，$AB$与$BA$有相同特征值

取$M=B,BA=M(AB)M^{-1}$

### 反对称矩阵

反对称矩阵没有实特征值
$$
A=-A^H\\
Ax=\lambda x\\
x^HAx=\lambda x^Hx\\
\overline{A}\overline{x}=\overline{\lambda}\overline{x}\\
x^HA^H=\overline{\lambda}x^H\\
-x^HA^Hx=-\overline{\lambda}x^Hx\\
又A=-A^H\\
\lambda x^Hx=-\overline{\lambda}x^Hx\\
不考虑零向量，\lambda=-\overline{\lambda}\\
令\lambda=a+bi\\
a+bi=-a+bi\\
a=0\\
所以\lambda必定为虚数\\
$$
例如$A=\left[\matrix{0&-1\\1&0}\right]$，作用在二维实向量上，$A\left[\matrix{1\\0}\right]=\left[\matrix{0\\1}\right],A\left[\matrix{0\\1}\right]=\left[\matrix{-1\\0}\right]$，等于将向量逆时针旋转90度，没有任何实$\lambda$能满足$Ax$与$\lambda x$同方向，因此特征值和特征向量都是复数。

## Lecture 5 Positive Definite and Semidefinite Matrices

## Lecture 6 Singular Value Decomposition(SVD)

### SVD

$A=U\Sigma V^T,AV=U\Sigma$

实际中为了避免特征向量符号不对（例如对$\left[\matrix{1&&\\&1&\\&&5}\right]$,特征值$1$的特征向量为$\left[\matrix{x\\y\\0}\right]$，需要选择），先做$A^TA$求出$V$和$\Sigma$，再通过$Av=\sigma u,u=\frac{Av}{\sigma}$求$U$

为了证明这样的$U$是正确的，需要证明$u$正交且为$AA^T$的特征向量

（实际上并不需要证明$u$正交，因为$v$是在这个前提下求出的，所以逆回来验算必定成立，**用$Av$求$u$是为了确定$u$的符号**，否则$-u$同样可以从$AA^T$求出但是不满足$Av=\sigma u$的条件，并且在假设了$A=U\Sigma V^T$这个条件后，做$AA^T,u$是特征向量显然成立，而$v$正是在这个条件下求出的，所以并不需要证明，教授此处为了说明确定$u$的符号循环论证了）

$u_1^Tu_2=(\frac{Av_1}{\sigma_1})^T\frac{Av_2}{\sigma_2}=\frac{v_1^TA^TAv_2}{\sigma_1\sigma_2}=\frac{\sigma_2^2v_1^Tv_2}{\sigma_1\sigma_2}=0$,也说明正交向量$v$可以从行空间选择，经过$A$变换后，得到列空间中的正交向量。

正交矩阵对向量变换不改变向量的模长，奇异值分解说明线性变换$A$，作用在向量$x$上，以二维为例，等于将$x$的两个分量$[0\ 1]^T,[1\ 0]^T$做旋转$V^T$，再用$\Sigma$拉长各个分量，再做一个通常来说不同的旋转$U$，即将单位圆拉伸成旋转的椭圆，且椭圆的长短轴就是$\sigma$，规定$\sigma_1≥\sigma_2≥\cdots≥\sigma_r>0$，因此$\sigma_1$作用在$x$的第一个分量上，为长轴。<a name="线性变换Ax的几何解释"></a>

假设$A$是方阵，奇异值之积也是$A$的行列式，$|A|=|U\Sigma V^T|=|U||\Sigma||V^T|=|\Sigma|=\Pi{\sigma}$，$A$如果满秩则$\Sigma$对角线上没有0，不满秩则奇异值填不满整条对角线，将零视为奇异值相乘，$|A|=0$，符合。

完整的SVD中$U_{m*m},V_{n*n}$，下标大于$r$的向量从$N(A^T),N(A)$取，但因为这部分在计算中会全部等于0，没有信息量，所以SVD的矩阵中可以只取下标小于等于$r$的部分。

### Polar Decomposition<a name="polar decomposition"></a>

任意矩阵$A=SQ,S$是对称矩阵，$Q$是各列正交的矩阵（不是方阵）

$A_{m*n}=U_{m*r}\Sigma_{r*r}V_{n*r}^T=U\Sigma U^TUV^T=(U\Sigma U^T)(UV^T)=SQ$

$(UV^T)^T(UV^T)=VU^TUV^T=I$所以$UV^T$是$m*n$正交阵。

## Lecture 7 Eckart-Young:The Closest Rank k Matrix to A

### Principal Component Analysis(主成分分析 PCA)

由SVD，$A=U\Sigma V^T=\sum_{i=1}^{r}\sigma_iu_iv_i^T$

$A$可以分解为$r$个秩1矩阵的和，且$\sigma$递减，因此构成$A$最主要的部分为$\sigma_1u_1v_1^T,\sigma_2u_2v_2^T,\cdots$

最近似于$A$的秩$k$矩阵为$A_k=\sum_{i=1}^{k}\sigma_iu_uv_i^T$

#### 范数(norm)

向量$v$的L2范数$l^2=||v||_2=\sqrt{\sum_{i=1}^{n}{v_i^2}}$,L1范数$l^1=||v||_1=\sum_{i=1}^{n}{|v_i|}$,L无限范数(infinity norm)$l^{\infty}=||v||_{\infty}=\max_{i=1,\cdots,n}{|v_i|}$

最小化L1范数时优秀的向量的稀疏向量

常数$c,||cv||=|c|||v||$

$||v+w||≤||v||+||w||$

$||A||$称为$A$的范数(the norm of $A$)，是矩阵尺度（大小）的一种测量

$||A||_2=\sigma_1$

Frobenius范数$||A||_F=\sqrt{\sum_{i=1,\cdots,m\\j=1,\cdots,n}{a_{ij}^2}}$

Nuclear范数$||A||_{Nuclear}=\sum_{i=1}^{r}{\sigma_i}$

正交矩阵$Q,||QA||=||A||,||Qv||=||v||$（用L2范数看，就是对$v$旋转）

$A$左乘或右乘正交阵不改变范数，原因是所有范数都与奇异值有关，左乘或右乘正交阵之后，仍然满足SVD分解的形式，可以看作另一个矩阵的SVD，因此奇异值不变，范数不变。

$QA=(QU)\Sigma V^T,QU$仍然是正交矩阵，正交矩阵的积是正交矩阵，$(QU)^TQU=U^TQ^TQU=I$

#### Eckart-Young Theorem

如果$B$是秩$k$矩阵，那么$A,B$的距离$||A-B||≥||A-A_k||$

对3种范数，定理都成立

#### PCA

对一组数据点构成的矩阵，先对每一项数据均值化（例如身高加和为0），再求协方差矩阵$\frac{AA^T}{N-1}$，求出近似直线，直线的斜率就是$\sigma_1$（？教授没说清楚）

跟最小二乘不同，这里误差是点到直线的垂直距离，最小二乘是竖直距离

（教授这部分说得比较模糊，似乎留到练习课了）

## Lecture 8 Norms of Vectors and Matrices

### 向量的范数

Lp范数$l^p=||v||_p=(\sum_{i=1}^{n}{|v_i|^p})^{1/p}$

L0范数$l^0=||v||_0=非零成分的个数,||cv||_0=||v||_0$

S范数$||v||_S=\sqrt{v^TSv}$，S表示正定矩阵。在2维平面上，S范数≤1的图像是椭圆，L1范数是等长的菱形，L2范数是圆，L无限范数是正方形

对于最优化问题，在2D平面上$\min{||x||_1}$或$\min{||x||_2}$使得$c_1x_1+c_2x_2=b$，将$x$视为自变量和因变量，画出直线的图像，交轴于$(a,0),(0,b)$

几何角度的解为从原点开始扩张L1范数和L2范数的图像，表示当范数固定时，满足范数为当前值的点。当菱形\圆慢慢扩张，第一次接触直线时的点，就是符合条件的解$x$。因此L2范数的解为从原点到直线的垂线交点（圆的切线），L1范数的解视直线斜率不同可能为$(a,0)$或$(0,b)$或菱形的某一边。~~扩张到高维也适用可是并不能画出高维图像呢~~

### 矩阵的范数

#### Spetral Norm（谱范数，L2范数）

$||A||_2=\sigma_1$

矩阵范数由向量范数得出，$||A||_2=\max_{for\ all\ x}{\frac{||Ax||_2}{||x||_2}}$，可以理解为将$||x||_2$放大的一个系数

上述最优化问题中，获取最优解时的$x$是$A$的右奇异向量$v_1$（课上没有证明），从几何角度看线性变换（见[线性变换Ax的几何解释](#线性变换Ax的几何解释)），秩$r$的$A$作用于向量$x$，仅在$\Sigma$矩阵对向量做拉伸，而$\Sigma$中拉伸最大的方向为$\sigma_1$对应的右奇异向量对应的方向，因此最优解时的$x$为$v_1$，最优解为$\sigma_1$（或者$\frac{||Av_1||_2}{||v_1||_2}=||Av_1||_2=||\sigma_1 u_1||_2=|\sigma_1|||u_1||_2=\sigma_1$）

#### Frobenius Norm

$||A||_F=\sqrt{\sum_{i=1,\cdots,m\\j=1,\cdots,n}{a_{ij}^2}}=\sqrt{\sum_{i=1}^{r}{\sigma_i^2}}$

##### 矩阵的迹满足交换律

$$
tr(A_{m*n}B_{n*m})=\sum(AB)_{ii}=\sum_{i=1}^{m}{(\sum_{j=1}^{n}{a_{ij}b_{ji}})}=\sum_{j=1}^{n}{(\sum_{i=1}^{m}{b_{ji}a_{ij}})}=\sum{(BA)_{jj}}=tr(BA)
$$

$||A||_F=\sqrt{\sum_{i=1,\cdots,m\\j=1,\cdots,n}{a_{ij}^2}}=\sqrt{tr(A^TA)}=\sqrt{tr(V\Sigma^2V^T)}$

注意迹为特征值之和，上式为$A^TA$的特征值分解，$\Sigma^2$即特征值矩阵

$||A||_F=\sqrt{\sum_{i=1}^{r}\sigma_i^2}$

#### Nuclear Norm(trace norm)

$||A||_N=\sum_{i=1}^{r}{\sigma_i}$

## Lecture 9 Four Ways to Solve Least Squares Problems

### 伪逆(pseudo inverse)

$A_{m*n}$将$C(A^T)$的$x$映射到$C(A)$的$Ax$，伪逆将其逆映射回来，$A^+Ax=x$

$A$将零空间映射到零点，$A^+$将左零空间映射到零点，满秩矩阵$A^+=A^{-1}$，画4空间图很好理解，非满秩矩阵将零点扩张成空间即可。

$A=U\Sigma V^T$,如果$A$可逆，$A^{-1}=V\Sigma^{-1}U^T$,$A$不可逆，$A^+=V\Sigma^+U^T$（定义并不要求列满秩）

伪逆是使得$AA^+,A^+A$最接近$I$的矩阵，注意左右伪逆不相等，但公式都可以由SVD类推求得

$\Sigma^+$是$\Sigma$非零项取倒数，其他全0

### 最小二乘法

对一组数据点，不在同一条直线上，即$Ax=b$无解，拟合一条最优的直线，使得误差最小（线性回归），误差定义为$||Ax-b||_2^2$

损失函数(loss function)$||Ax-b||_2^2=(Ax-b)^T(Ax-b)=x^TA^TAx-2b^TAx+b^Tb$

令上式求导为0，得$A^TAx=A^Tb$，也就是正规方程（~~可是教授你没教过矩阵求导啊~~）

从几何角度，就是求构成列空间中最接近$b$的向量的系数$x$，这样使得误差最小，正规方程也在求投影中出现过，因此求导的结果就是求投影的系数，为表示是近似解而不是原方程的解，解写做$\hat{x}$

$A$如果列满秩，则$\hat{x}=(A^TA)^{-1}A^Tb$，同时左伪逆$A_{left}^+=(A^TA)^{-1}A^T,\hat{x}=A^+b$

$A_{left}^+A=V_{n*n}\Sigma_{n*m}^+U_{m*m}^TU\Sigma_{m*n}V^T=VI_{n*n}V^T=I$

$(A^TA)^{-1}A^TA=I$，因此得到$A_{left}^+=(A^TA)^{-1}A^T,\hat{x}=A^+b$

同样当A列满秩时，做Gram-Schmidt正交化可以得到QR分解，$R_{n*n}$满秩（否则A是R的行的线性组合，不能秩n）

$A=QR$

$\hat{x}=(A^TA)^{-1}A^Tb=(R^TR)^{-1}R^TQ^Tb=R^{-1}(R^T)^{-1}R^TQ^Tb=R^{-1}Q^Tb$

当A列不满秩时，$A^TA$不可逆，可以将正规方程修正（加入惩罚项）为$(A^TA+\delta^2I)\hat{x}=A^Tb$，误差为$||Ax-b||_2^2+\delta^2||x||_2^2$（线性回归变为嵴回归ridge regression），当$\delta\rightarrow0$时$\hat{x}$就是原正规方程的解

对任意$v≠0,v^T(A^TA+\delta^2I)v>0$，因此为正定矩阵,$\hat{x}=(A^TA+\delta^2I)^{-1}A^Tb$

代入SVD
$$
\begin{align}
A^TA+\delta^2I&=V(\Sigma^T\Sigma+\delta^2I)V^T\\
(A^TA+\delta^2I)^{-1}A^T&=V[(\Sigma^T\Sigma+\delta^2I)^{-1}\Sigma^T]U^T\\
(\Sigma^T\Sigma_{m*n}+\delta^2I)^{-1}\Sigma^T
&=\left[\matrix{
\sigma_1^2+\delta^2\\
&\sigma_2^2+\delta^2\\
&&\ddots\\
&&&\sigma_r^2+\delta^2\\
&&&&\delta^2\\
&&&&&\ddots\\
&&&&&&\delta^2\\
}\right]^{-1}\Sigma^T\\
&=\left[\matrix{
\frac{1}{\sigma_1^2+\delta^2}\\
&\frac{1}{\sigma_2^2+\delta^2}\\
&&\ddots\\
&&&\frac{1}{\sigma_r^2+\delta^2}\\
&&&&\frac{1}{\delta^2}\\
&&&&&\ddots\\
&&&&&&\frac{1}{\delta^2}\\
}\right]
\left[\matrix{
\sigma_1\\
&\sigma_2\\
&&\ddots\\
&&&\sigma_r\\
&&&&0\\
&&&&&\ddots\\
}\right]\\
&=\left[\matrix{
\frac{\sigma_1}{\sigma_1^2+\delta^2}\\
&\ddots\\
&&\frac{\sigma_r}{\sigma_r^2+\delta^2}\\
&&&0\\
&&&&\ddots\\
}\right]_{n*m}\\
\lim_{\delta\rightarrow0}{(\Sigma^T\Sigma_{m*n}+\delta^2I)^{-1}\Sigma^T}
&=
\left[\matrix{
\frac{1}{\sigma_1}\\
&\ddots\\
&&\frac{1}{\sigma_r}\\
&&&0\\
&&&&\ddots\\
}\right]\\
&=\Sigma_{n*m}^+\\
\lim_{\delta\rightarrow0}{(A^TA+\delta^2I)^{-1}A^T}&=V\Sigma^+U^T=A^+\\
\end{align}\\
$$
因此$\hat{x}=A^+b$总是成立的

## Lecture 10 Survey of Difficulties with Ax=b

## Lecture 11 Minimizing x subject to Ax=b

这一节是为了解决大尺寸的矩阵，或近似奇异（列非常接近相关，或者说奇异值非常接近0）矩阵的Ax=b问题，减小电脑的计算误差

### better Gram-Schmidt(with column pivoting)

$A=[\matrix{a_1&a_2&\cdots&a_n}]$，挑选L2范数最大的$a_i$作为$A_1$并标准化为$q_1$，然后其余列减去在$q_1$上的投影，L2范数最大的作为$A_2$，依此类推。不会增加计算量，因为原方法也要每个列向量都减去$q_1$投影一次，这里只是提前计算并比较。

否则当$||A_i||_2$太小时，标准化时其作为分母，计算机会引入非常大的误差。

### Krylov 空间 , Arnoldi 过程

用于解决很大的稀疏矩阵Ax=b的问题

$A_{n*n},b_n$，那么$b,Ab,A(Ab),\cdots,A^{j-1}b$构成Krylov子空间$K_j$，每一次不做矩阵乘法，只做矩阵乘向量，子空间维数不超过$j$

直接求$A$的逆和伪逆会非常困难，所以求近似解

令$x_j$为$K_j$里最近似的解，从$j$个向量中用Gram-Schmidt方法（这里没看出Arnoldi过程的区别）得到一组正交基$v$，$x_j$就是$x$在$K_j$里的投影，可以用$v$表示出来。（然而怎么求系数根本没提，这节课上了个寂寞，教授你在干什么）

## Lecture 12 Computing Eigenvalues and Singular Values

### QR method

满秩矩阵$A_{n*n}=A_0=Q_{n*n}R_{n*n}=Q_0R_0,A_1=R_0Q_0=Q_1R_1\cdots$

$A_1$和$A_0$相似，因此特征值相同。$A_1=R_0Q_0=R_0A_0R_0^{-1}=Q_0^{-1}A_0Q_0=Q_0^TA_0Q^0$

重复这个过程，对角线下的元素会越来越小，最后对角线会非常接近特征值，$\lambda_n$首先在对角线最后准确地出现。

#### QR method with shift

加入偏移量会加速上述过程，更快得到特征值

$A_0-sI=Q_0R_0,A_1=R_0Q_0+sI$

偏移量一个好的选择是$\lambda_n$，0会首先在对角线最后准确出现

$A_1=R_0Q_0+sI=Q_0^{-1}(A_0-sI)Q_0+sI=Q_0^{-1}A_0Q_0-sI+sI$

因此仍然相似。

程序（matlab）计算特征值的方式，先将矩阵消元（程序似乎用相似变换做）化为Hessenberg矩阵（上三角+下面多一行对角线），再进行有偏移的QR过程

如果A对称，则后续得到的$A_i$对称，Hessenberg矩阵是一个三对角线矩阵，只有中央三条对角线，同样是对称矩阵

算法分为两步，第一步得到很多0，且这些0的位置在第二步过程中仍然为0，第二步进行QR过程。相似变换不改变特征值，$B=MAM^{-1}$,M任意选取可逆矩阵。

对于奇异值来说，要保持奇异值不变，两边相乘的换为正交阵即可，结果视为另一个SVD分解。$B=Q_1AQ_2^T=(Q_1U)\Sigma(V^TQ_2^T)$

A先进行上述变换得到双对角线矩阵（主对角线和它上方的对角线）记作$A_1$，再进行$A_1^TA_1$，得到对称三对角线矩阵，再进行QR过程，得到特征值，再开方得到奇异值。

QR过程得到的是近似值，复杂度$O(N^3)$。对于很大的矩阵，考虑使用Krylov子空间做近似，表示出特征值、奇异值等矩阵的特征。（一百万规模的矩阵用一百维子空间就可以精确地近似，比如特征向量可以用一百维近似表示出来）。~~都是教授说的，没有证明，没有详述。~~

## Lecture 13 Randomized Matrix Multiplication

均值、期望$mean=E=p*对应样本x$

方差$\sigma^2=E[(x-mean)^2]=E[x^2]-(mean)^2$

从大矩阵A采样，范数大的列可能包含更多信息，因此各列取样的概率不应相等，例如各列的概率可以用norm squared决定（范数平方概率），$A=[\matrix{a_1&a_2}],||a_2||=2||a_1||$，取样概率$p_{a_2}=4p_{a_1}$

大矩阵乘法$A_{m*n}B_{n*p}=\sum(col\ of\ A)(row\ of\ B)$，分解成n个秩1矩阵相加

n个秩1矩阵计算量太大，从中选出1个矩阵，用来近似表示AB，一共取s次，取它们的平均值。假设每个秩1矩阵被选中的概率相等，因为原本AB有n个矩阵表示，所以秩1矩阵要放大n倍。

$AB≈\frac{\sum_{j=1}^{s}{a_{ij}b_{ij}^T*n}}{s}=\frac{\sum_{j=1}^{s}{\frac{a_{ij}b_{ij}^T}{\frac{1}{n}}}}{s},a_{i1},a_{i2},\cdots,a_{is}$表示对A每次随机选出的s个列

将平均值分摊到每一次，视为每一次为最后的期望贡献了多少。每次取矩阵平均值的数学期望$E=\sum{\frac{1}{n}(a_jb_j^T*n)}/s=\sum{a_jb_j^T}/s=AB/s$，总平均值的数学期望$E=\frac{AB}{s}*s=AB=\sum{\frac{1}{n}(\frac{a_jb_j^T}{\frac{1}{n}}/s)}*s=\sum{p_j(\frac{a_jb_j^T}{sp_j})}*s,\sum{p_j\frac{a_jb_j^T}{sp_j}}$即为每一次的数学期望对最后的期望做出多少贡献。

实际选中各个秩1矩阵的概率不应相等，令$p_j$为选中$a_jb_j^T$的概率，$b_j^T$表示B的第$j$行，$p_j=\frac{||a_j||||b_j^T||}{C},C=\sum{||a_j||||b_j^T||}$，使得概率和为1（后续会证明这种概率是使得方差最小的最优概率）

对照上面的数学期望，每一次取样的数学期望$E=\sum{p_j(\frac{a_jb_j^T}{sp_j})}=\frac{AB}{s},s$为样本数，总的数学期望仍为AB，说明这种取样方法是正确的。

~~到这里已经跟原视频说的不一样了，原视频说的非常不好理解，这是自己的理解，老教授你还行不行了……我甚至不确定这是正确的，感觉自己在强行解释~~

从形式上看，$\frac{a_jb_j^T}{sp_j}$即为概率$p_j$对应的样本值。

利用方差公式$\sigma^2=E[(x^2)]-(mean)^2=E[(x^2)]-[E[(x)]]^2$

一次抽样的方差$\sigma_1^2=\sum{p_j\frac{||a_j||^2||b_j^T||^2}{s^2p_j^2}}-\frac{||AB||_F^2}{s^2}$

总方差$\sigma^2=s\sigma_1^2=\frac{1}{s}(\sum\frac{||a_j||^2||b_j^T||^2}{p_j}-||AB||_F^2)$，利用$p_j=\frac{||a_j||||b_j^T||}{C}$，$\sigma^2=\frac{1}{s}(\sum{\frac{||a_j||||b_j^T||}{\frac{1}{C}}}-||AB||_F^2)=\frac{1}{s}(C^2-||AB||_F^2)$

~~也没解释为什么求方差的时候矩阵的平方这样处理，这节课真的看了个寂寞~~

现在证明上式是能得到的最小方差，选择的概率是最优概率

最小化方差即最小化$\sum{\frac{||a_j||^2||b_j^T||^2}{sp_j}}$，利用拉格朗日乘子，加入约束项
$$
\begin{align}
\min_{\sum{p_j=1}}{L}&=\min_{\sum{p_j=1}}{\frac{||a_j||^2||b_j^T||^2}{sp_j}}\\
&=\min_{\sum{p_j=1}}{\frac{||a_j||^2||b_j^T||^2}{sp_j}+\lambda(\sum{p_j}-1)}\\
\frac{\partial{L}}{\partial{p_j}}&=-\frac{||a_j||^2||b_j^T||^2}{sp_j^2}+\lambda=0\\
\frac{\partial{L}}{\partial{\lambda}}&=\sum{p_j}-1=0\\
\Rightarrow p_j&=\frac{||a_j||||b_j^T||}{\sqrt{s\lambda}},\sum{p_j}=1\\
\Rightarrow \sqrt{s\lambda}&=\sum{||a_j||||b_j^T||}=C,p_j=\frac{||a_j||||b_j^T||}{C}\\
\end{align}\\
$$

## Lecture 14 Low Rank Changes in A and Its Inverse

### Matrix inversation formula in signal process

还有很多其他名字，Sherman-Morrison-Woodbury(and so on) formula

当$u,v$为列向量时
$$
\begin{align}
(I-uv^T)^{-1}&=I+\frac{uv^T}{1-v^Tu}\\
(I-uv^T)(I+\frac{uv^T}{1-v^Tu})&=I-uv^T+\frac{uv^T-uv^Tuv^T}{1-v^Tu}\\
&=I-uv^T+\frac{uv^T-u(v^Tu)v^T}{1-v^Tu}\\
&=I\\
\end{align}
$$
当$u,v$为$n*k,k<n$的矩阵时（上式为$k=1$的特例）
$$
\begin{align}
(I_n-uv^T)^{-1}&=I_n+u(I_k-v^Tu)^{-1}v^T\\
(I_n-uv^T)(I_n+u(I_k-v^Tu)^{-1}v^T)&=I_n-uv^T+(I_n-uv^T)u(I_k-v^Tu)^{-1}v^T\\
&=I_n-uv^T+(u-uv^Tu)(I_k-v^Tu)^{-1}v^T\\
&=I_n-uv^T+u(I_k-v^Tu)(I_k-v^Tu)^{-1}v^T\\
&=I_n\\
\end{align}
$$
一般情况
$$
(A_{n*n}-U_{n*k}V_{k*n}^T)^{-1}=A^{-1}+A^{-1}U(I_k-V^TA^{-1}U)^{-1}V^TA^{-1}\\
$$


### 公式应用

一是解方程$(A-uv^T)x=b$

二是解决最小二乘（正规方程）中新数据点到来的问题
$$
A^TA\hat{x}=A^Tb\\
一个新数据点到来，增加新的一行，正规方程变为\\
\left[\matrix{A^T&v}\right]\left[\matrix{A\\v^T}\right]\hat{x}_{new}=\left[\matrix{A^T&v}\right]\left[\matrix{b\\b_{new}}\right]\\
(A^TA+vv^T)\hat{x}_{new}=A^Tb+vb_{new}\\
$$
上式$v_{n*1}$，新的正规方程可视为对$A^TA$增加了一个秩1的扰动

因为新的方程利用到原先的$A^TA$，只需要计算一个新的秩1矩阵，所以这种最小二乘也称为递归最小二乘（recursive least squared）

当增加$k$个数据点，即$v_{n*k}$时，视为对$A^TA$增加一个秩k的扰动，使用对应公式即可。

三、

假设$Aw=b$已解得

求解$(A-uv^T)x=b$，此处$u,v,w$列向量

变为求解$Az=u$，$z,w$可同时求解，$A[w\ z]=[b\ u]$

解$x=w+\frac{zv^Tw}{1-v^Tz}$
$$
\begin{align}
x&=(A-uv^T)^{-1}b\\
&=(A^{-1}+A^{-1}u\frac{1}{1-v^Tz}v^TA^{-1})b\\
&=w+z\frac{1}{1-v^Tz}v^Tw\\
&=w+\frac{zv^Tw}{1-v^Tz}\\
\end{align}
$$
这里结果跟教授不一样，感觉教授算错了。

## Lecture 15 Matrices A(t) Depending on t,Derivative=dA/dt

### A发生微小变化时，逆如何变化  $\frac{\mathrm{d}{A^{-1}}}{\mathrm{d}{t}}$

假设$A$是随时间变化的矩阵$A(t)$
$$
B^{-1}-A^{-1}=B^{-1}(A-B)A^{-1}
$$
令$B=A+\Delta{A},\Delta$表示小量变化，上式变为
$$
\Delta{A^{-1}}=(A+\Delta{A})^{-1}(-\Delta{A})A^{-1}
$$
$\Delta{(A^{-1})}$表示$A^{-1}$出现小量变化。两边同时除以$\Delta{t}$
$$
\frac{\Delta{A^{-1}}}{\Delta{t}}=（A+\Delta{A}）^{-1}\frac{-\Delta{A}}{\Delta{t}}A^{-1}
$$
令$\Delta{t}\rightarrow0$，则$A+\Delta{A}$中可忽略$\Delta{A}$
$$
\frac{\mathrm{d}{(A^{-1})}}{\mathrm{d}{t}}=-A^{-1}\frac{\mathrm{d}{A}}{\mathrm{d}{t}}A^{-1}
$$
假设$A$是一个变量$t$，就是普通的微积分
$$
\frac{\mathrm{d}{\frac{1}{t}}}{\mathrm{d}{t}}=-\frac{1}{t}\frac{\mathrm{d}{t}}{\mathrm{d}{t}}\frac{1}{t}=-\frac{1}{t^2}
$$

### A发生微小变化时，特征值如何变化  $\frac{\mathrm{d}{\lambda}}{\mathrm{d}{t}}$

矩阵以及特征值、特征向量、转置的特征向量随时间$t$变化
$$
A(t)x(t)=\lambda(t)x(t)\\
y^T(t)A(t)=\lambda(t)y^T(t)\\
增加约束y^Tx=1\\
当A是对称矩阵时，y=x=q为单位向量\\
不同特征值对应的A的右特征向量和左特征向量正交\\
Ax=\lambda_1x\\
y^TA=\lambda_2y^T\\
y^TAx=\lambda_2y^Tx=y^T\lambda_1x\\
(\lambda_2-\lambda_1)y^Tx=0\\
y^Tx=0
$$
矩阵形式
$$
AX=X\Lambda\\
Y^TA=\Lambda Y^T\\
Y^TX=I
$$
在$y^Tx=1$的约束条件下
$$
y^T(t)A(t)x(t)=\lambda(t)y^T(t)x(t)=\lambda(t)\\
\frac{\mathrm{d}{\lambda}}{\mathrm{d}{t}}=\frac{\mathrm{d}{y^T}}{\mathrm{d}{t}}A(t)x(t)+y^T(t)\frac{\mathrm{d}{A}}{\mathrm{d}{t}}x(t)+y^T(t)A(t)\frac{\mathrm{d}{x}}{\mathrm{d}{t}}\\
y^Tx=1\\
\frac{\mathrm{d}{y^Tx}}{\mathrm{d}{t}}=0\\
\frac{\mathrm{d}{y^T}}{\mathrm{d}{t}}x(t)+y^T(t)\frac{\mathrm{d}{x}}{\mathrm{d}{t}}=0\\
\begin{align}
\frac{\mathrm{d}{\lambda}}{\mathrm{d}{t}}&=\frac{\mathrm{d}{y^T}}{\mathrm{d}{t}}A(t)x(t)+y^T(t)\frac{\mathrm{d}{A}}{\mathrm{d}{t}}x(t)+y^T(t)A(t)\frac{\mathrm{d}{x}}{\mathrm{d}{t}}\\
&=\frac{\mathrm{d}{y^T}}{\mathrm{d}{t}}\lambda(t)x(t)+\lambda(t)y^T(t)\frac{\mathrm{d}{x}}{\mathrm{d}{t}}+y^T(t)\frac{\mathrm{d}{A}}{\mathrm{d}{t}}x(t)\\
&=\lambda(t)(\frac{\mathrm{d}{y^T}}{\mathrm{d}{t}}x(t)+y^T(t)\frac{\mathrm{d}{x}}{\mathrm{d}{t}})+y^T(t)\frac{\mathrm{d}{A}}{\mathrm{d}{t}}x(t)\\
&=y^T(t)\frac{\mathrm{d}{A}}{\mathrm{d}{t}}x(t)\\
\end{align}\\
\frac{\mathrm{d}{\lambda}}{\mathrm{d}{t}}=y^T(t)\frac{\mathrm{d}{A}}{\mathrm{d}{t}}x(t)
$$

### 秩发生变化时，特征值变化

$S$对称矩阵，特征值$\gamma$，令$\gamma_1≥\gamma_2≥\cdots$

$uu^T$为秩1半正定矩阵，有1个非零特征值其余全0，$uu^Tu=(u^Tu)u$，非零特征值为$u^Tu=||u||_2^2$，特征向量为$u$

$(S+uu^T)$特征值为$\lambda$，令$\lambda_1≥\lambda_2≥\cdots$

则有$\lambda_1≥\gamma_1≥\lambda_2≥\gamma_2\cdots$

假设$Su_i=\gamma_iu_i,u_i^Tu_i=1$，则$(S+Cu_iu^T_i)u_i=(\gamma_i+C)u_i,C$为常数

$S_n$为n阶对称阵特征值$\lambda$，$S_{n-1}$为$S_n$删掉最后一行和一列，特征值$\mu$，$\lambda_1≥\mu_1≥\lambda_2≥\mu_2\cdots$

## Lecture 16 Derivatives of Inverse and Singular Values

### A发生微小变化时，$A^2$如何变化   $\frac{\mathrm{d}{(A^2)}}{\mathrm{d}{t}}$

$$
\begin{align}
\frac{(A+\Delta{A})^2-A^2}{\Delta{t}}&=\frac{A(\Delta{A})+(\Delta{A})A+(\Delta{A})^2}{\Delta{t}}\\
\lim_{t\rightarrow0}\frac{(A+\Delta{A})^2-A^2}{\Delta{t}}&=A\frac{\mathrm{d}{A}}{\mathrm{d}{t}}+\frac{\mathrm{d}{A}}{\mathrm{d}{t}}A\\
\frac{\mathrm{d}{(A^2)}}{\mathrm{d}{t}}&=A\frac{\mathrm{d}{A}}{\mathrm{d}{t}}+\frac{\mathrm{d}{A}}{\mathrm{d}{t}}A\\
\end{align}\\
$$

### A发生微小变化时，奇异值如何变化    $\frac{\mathrm{d}{\sigma}}{\mathrm{d}{t}}$

将$u,v$均选作单位向量。假设$u^T(t)=(a_1(t),a_2(t)),\frac{\mathrm{d}{(u^Tu)}}{\mathrm{d}{t}}=\frac{\mathrm{d}{u^T}}{\mathrm{d}{t}}u+u^T\frac{\mathrm{d}{u}}{\mathrm{d}{t}}=\frac{\mathrm{d}{a_1}}{\mathrm{d}{t}}a_1+\frac{\mathrm{d}{a_2}}{\mathrm{d}{t}}a_2+a_1\frac{\mathrm{d}{a_1}}{\mathrm{d}{t}}+a_2\frac{\mathrm{d}{a_2}}{\mathrm{d}{t}}=2(\frac{\mathrm{d}{a_1}}{\mathrm{d}{t}}a_1+\frac{\mathrm{d}{a_2}}{\mathrm{d}{t}}a_2)=2\frac{\mathrm{d}{u^T}}{\mathrm{d}{t}}u=0$
$$
\begin{align}
A&=U\Sigma V^T\\
A^T&=V\Sigma U^T\\
A^TU&=V\Sigma\\
A^T(t)u(t)&=\sigma(t)v(t)\\
u^T(t)A(t)&=\sigma(t)v^T(t)\\
A(t)v(t)&=\sigma(t)u(t)\\
u^T(t)A(t)v(t)&=\sigma(t)u^T(t)u(t)\\
\sigma(t)&=u^T(t)A(t)v(t)\\
\frac{\mathrm{d}{\sigma}}{\mathrm{d}{t}}&=\frac{\mathrm{d}{u^T}}{\mathrm{d}{t}}Av+u^T\frac{\mathrm{d}{A}}{\mathrm{d}{t}}v+u^TA\frac{\mathrm{d}{v}}{\mathrm{d}{t}}\\
&=\sigma \frac{\mathrm{d}{u^T}}{\mathrm{d}{t}}u+\sigma v^T\frac{\mathrm{d}{v}}{\mathrm{d}{t}}+u^T\frac{\mathrm{d}{A}}{\mathrm{d}{t}}v\\
&=0+0+u^T\frac{\mathrm{d}{A}}{\mathrm{d}{t}}v\\
\frac{\mathrm{d}{\sigma}}{\mathrm{d}{t}}&=u^T\frac{\mathrm{d}{A}}{\mathrm{d}{t}}v\\
\end{align}\\
$$
$u$称为左奇异向量，$v$右奇异向量
$$
y^Tx=x^Ty\\
(\frac{\mathrm{d}{u}}{\mathrm{d}{t}})^T=\frac{\mathrm{d}{(u^T)}}{\mathrm{d}{t}}\\
所以\frac{\mathrm{d}{u^T}}{\mathrm{d}{t}}u=u^T\frac{\mathrm{d}{u}}{\mathrm{d}{t}}=0\\
$$

### 特征值的交错

对称矩阵$S$有特征值$\lambda_1≥\lambda_2≥\cdots,u$列向量

$S+\theta uu^T,\theta>0$有特征值$\mu_1≥\mu_2≥\cdots$，则$\mu_1≥\lambda_1≥\mu_2≥\lambda_2≥\cdots$

$\lambda_2$增大后不一定是$\mu_2$，例如$Su=\lambda_2u,(S+\theta uu^T)u=(\lambda_2+\theta)u$，当$\theta$足够大，$\lambda_2+\theta≥\lambda_1$时，它应该是$\mu_1$而不是$\mu_2$

#### Weyl's inequality

$S,T$对称阵，特征值从大到小排列，则
$$
\lambda_{i+j-1}(S+T)≤\lambda_i(S)+\lambda_j(T)
$$
取$j=1,\lambda_i(S+T)≤\lambda_i(S)+\lambda_{max}(T)$

令$T=\theta uu^T$，得到$\lambda_1(T)=\theta$，其余特征值为$0$，令$j=1$，得到$\lambda_i(S+T)≤\lambda_i(S)+\theta,\mu_1≤\lambda_1+\theta$

对奇异值也适用
$$
\sigma_{i+j-1}(A+B)≤\sigma_i(A)+\sigma_j(B)
$$

### 一些关于压缩感知(compressed sensing)

矩阵的nuclear norm

$||A||_N=\sum{\sigma_i}$

类似的，向量的L1范数$||v||_1=\sum{|v_i|}$

在最小化问题中，加上L1范数最小化的约束，可以得到稀疏解，即每个元素的绝对值都尽量接近0

一个不完整的矩阵，将其补完，一个想法是增加$||A||_N$最小化的约束

L1范数由L0范数得来（实际上不是范数，因为不满足$||cv||_0=|c|||v||_0$），L0范数表述向量非零元素的个数，L1范数是最接近L0的范数

推至矩阵，跟$||A||_N$最接近的是A的秩，即非零的奇异值的个数。

在深度学习中，猜测加上$||A||_N$最小化的约束，得到的梯度下降的答案是最优的，但是没有被证明。

## Lecture 17 Rapidly Decreasing Singular Values

### 为什么世界上有那么多低秩矩阵

#### 定义

一张$n*n$的图$X$，如果逐个元素发送，需要发送$n^2$次，将其SVD分解成$k$个秩1矩阵的和，每次发送构成秩1矩阵的一列一行共$2n$个元素，发送$k$次，则一共需要发送$2kn$次

当$2kn<n^2\Rightarrow k<n/2$时称矩阵$X$为低秩矩阵，这是严格定义。通常讨论的低秩矩阵认为$k\ll n/2$

#### Numerical rank

对$0<\epsilon<1$，矩阵的numerical rank $rank_\epsilon(X)=k$，使得$\sigma_{k+1}(X)≤\epsilon\sigma_1(X),\sigma_k>\epsilon\sigma_1(X)$

显然，$rank_0(X)=rank(X)$

Eckart-Young:$X=U\Sigma V^T=\sum{\sigma_iu_iv_i^T}$，最接近$X$的秩k矩阵$X_k$为前k项的和，则他们的差异可以用奇异值近似，$\sigma_{k+1}(X)=||X-X_k||_2$

Eckart-Young的表达式要结合上文看。给定$\epsilon$，则决定numerical rank，$rank_\epsilon(X)=k$，则最接近$X$的秩k矩阵$X_k=\sum_{i=1}^{k}{\sigma_iu_iv_i^T}$，差异用$\sigma_{k+1}$近似（其实就是在$\epsilon$满足numerical rank的不等式的情况下忽略$\sigma_{k+2}$和后续项），计算机将认为$X$和$X_k$是一样的。

有很多满秩矩阵，奇异值下降非常快，在numerical rank的条件下它们都是低秩的，比如Hilbert Matrix，$H_{jk}=\frac{1}{j+k-1}$

上述技术应用于压缩，用合适的$\epsilon$和$k$可以控制误差范围。

Vandermonde矩阵$V=\left[\matrix{1&x_1&\cdots&x_1^{n-1}\\\vdots&\vdots&&\vdots\\1&x_n&&x_n^{n-1}}\right],x_i$为实数，通常出现在多项式插值中，它是满秩但是numerical rank低秩矩阵，这时候低秩是不好的，求逆非常困难。

#### 世界是平滑（smooth）的

Reade认为世界上有那么多低秩矩阵的原因是世界是平滑的（意思应该是很多矩阵是从平滑函数采样而来）

例如多项式$P(j,k)=1+j+jk,X_{jk}=P(j,k)$，那么$X$在$\epsilon=0$的时候已经是低秩矩阵，$rank(X_{n*n})≤3$
$$
X=\left[\matrix{1&1&\cdots&1\\1&1&\cdots&1\\\vdots&\vdots&&\vdots\\1&1&\cdots&1}\right](全1矩阵，对应多项式的1)
+\left[\matrix{1&1&\cdots&1\\2&2&\cdots&2\\\vdots&\vdots&&\vdots\\n&n&\cdots&n}\right](对应多项式的j)
+\left[\matrix{1&2&\cdots&n\\2&4&\cdots&2n\\\vdots&\vdots&&\vdots\\n&2n&\cdots&n^2}\right](对应多项式的jk)
$$
三个矩阵秩全为1，因此$rank(X)\le3$

通常
$$
P(x,y)=\sum_{s=0}^{m-1}\sum_{t=0}^{m-1}a_{st}x^sy^t,X_{jk}=P(j,k),rank(X)\le m^2
$$
对于一般从平滑函数而不是多项式采样的矩阵，例如Hilbert Matrix $H_{jk}=\frac{1}{j+k-1}$，对应的函数$f(x,y)=\frac{1}{x+y-1}$

寻找$P$，近似于$f$，使得$|f(x,y)-P(x,y)|\le \frac{\epsilon}{n}||X||_2$（这里$n$应该是$X$为$n*n$矩阵）

从$P$采样，得到矩阵$Y$，$Y_{jk}=P(j,k)$，则$Y$近似于$X,||X-Y||_2\le\epsilon||X||_2$

（记了个寂寞）然而对于Hilbert矩阵效果不是很好，$rank(H_{1000}=1000)$，对于$\epsilon=10^{-15},rank_\epsilon(H)=28$，但是用Reade的方法得到近似的矩阵秩≤719

#### 世界是Sylvester的

意思是大多数矩阵满足Sylvester方程，即$AX-XB=C,X$是考察的矩阵。对于一个$X$，想要证明它是numerical low rank的，$A,B,C$通常比较难找。

对于Hilbert Matrix
$$
\left[\matrix{
1/2\\
&3/2\\
&&\ddots\\
&&&n-1/2
}\right]H-H
\left[\matrix{
-1/2\\
&-3/2\\
&&\ddots\\
&&&-n+1/2
}\right]=
\left[\matrix{
1&\cdots&1\\
\vdots&\ddots&\vdots\\
1&\cdots&1
}\right]
$$
定理：如果$X$满足$AX-XB=C$且$A,B$是正规矩阵（normal matrix），那么$\sigma_{rk+1}(X)\le Z_k(E,F)\sigma_1(X),r=rank(C)$

正规矩阵指方阵$AA^H=A^HA$，对于实矩阵，$AA^T=A^TA$

$Z$是Zolotarev数，$E$是包含$A$的特征值的集合，$F$是包含$B$的特征值的集合

关键点在于$E,F$不相交，当它们不相交时，$Z_k$随着$k$增加减小得非常快。对于Hilbert Matrix，正是这一点使得它是numerical rank 低秩，此时$r=1$，在$k$非常小时，$Z$就已经非常小，因此大多数$\sigma$都可以忽略。

用这种方法，Hilbert矩阵得到的$k=34$，即nume rank是34，而不是28，但也比719要更接近28得多。

## Lecture 18 Counting Parameters in SVD,LU,QR,Saddle Points

### 各个分解的自由参数

对于满秩的方阵，有$n*n$个元素，即$n^2$个自由参数

#### LU

L是下三角阵，对角元为1，因此有$\frac{1}{2}(1+n)n-n$个自由参数，U是上三角阵，没有限制，有$\frac{1}{2}(1+n)n$个自由参数，加起来为$n^2$

#### QR

Q的第一列有$n$个元素，但因为它是单位向量，所以自由参数有$n-1$个（当前$n-1$个元素决定，最后一个元素也决定了）。第二列在$n-1$的基础上，还要求与第一列内积为0，因此少一个自由参数(最后一项内积被其他项决定)，为$n-2$。依此类推，Q有$\frac{1}{2}(n-1)n$个自由参数

R是上三角阵，没有特殊限制，有$\frac{1}{2}(n+1)n$个自由参数。与Q相加为$n^2$

#### $X\Lambda X^{-1}$

$\Lambda$显然有$n$个自由参数

$X$可令每一列为单位向量，或手动限制每一列第一个元素为1，有$(n-1)n$个。$X^{-1}$由$X$决定，0个。加起来$n^2$

#### $Q\Lambda Q^T$

如果是对称矩阵，则不是$n^2$

$Q$有$\frac{1}{2}(n-1)n$个，$\Lambda$有$n$个，加起来$\frac{1}{2}(n+1)n$

#### QS

[polar decomposition](#polar decomposition)，$Q$有$\frac{1}{2}(n-1)n$个，$S$有$\frac{1}{2}(n+1)n$个，加起来$n^2$

#### SVD

假设$A_{m*n},m<n,r=m$，则$A=U_{m*m}\Sigma_{m*n}V_{n*n}^T$

$U$为正交阵，有$\frac{1}{2}(m-1)m$个，$\Sigma$有$m$个，$V$中有效的列仅仅有$m$个，其他实际上可没有限制，所以有$(n-1)+(n-2)+\cdots+(n-m)=mn-\frac{1}{2}(m+1)m$个，加起来为$mn$

假设$rank(A)=r$，则SVD可以写成$A=U_{m*r}\Sigma_{r*r}V_{r*n}^T$

$U$的限制是各列为单位正交向量，有$(m-1)+(m-2)+\cdots+(m-r)=mr-\frac{1}{2}(r+1)r$个，$\Sigma$有$r$个，$V$有$(n-1)+(n-2)+\cdots+(n-r)=nr-\frac{1}{2}(r+1)r$个，加起来为$mr+nr-r^2$个

### 向量和矩阵的求导

可参考https://www.bilibili.com/video/BV1xk4y1B7RQ/?p=4

工具书The Matrix Cookbook

$y=f(x)$，不管$y,x$是标量还是向量，本质是$y$的逐项对$x$的逐项进行求导

以下采用分母布局，即求导结果写成$x$纵向展开，$y$横向展开的形式，即$\left[\matrix{\part{f_1(x)}/\part{x_1}&\part{f_2(x)}/\part{x_1}&\cdots&\part{f_n(x)}/\part{x_1}\\\part{f_1(x)}/\part{x_2}\\\vdots\\\part{f_1(x)}/\part{x_n}}\right]$

$y=a^Tx=x^Ta,a^T=[a_1\ \cdots\ a_n]$
$$
\mathrm{d}y/\mathrm{d}x=
\left[\matrix{\part{y}/\part{x_1}\\\part{y}/\part{x_2}\\\vdots\\\part{y}/\part{x_n}}\right]
=\left[\matrix{a_1\\a_2\\\vdots\\a_n}\right]
=a
$$
$y_{n*1}=A_{n*n}x_{n*1},A=\left[\matrix{a_{11}&\cdots&a_{1n}\\\vdots\\a_{n1}&\cdots&a_{nn}}\right]$
$$
y=\left[\matrix{
f_1(x)=\sum_{j=1}^{n}{a_{1j}x_j}\\
f_2(x)=\sum_{j=1}^{n}{a_{2j}x_j}\\
\vdots\\
f_n(x)=\sum_{j=1}^{n}{a_{nj}x_j}
}\right]\\
\frac{\mathrm{d}y}{\mathrm{d}x}=
\left[\matrix{
a_{11}&a_{21}&\cdots&a_{n1}\\
a_{12}&a_{22}&\cdots&a_{n2}\\
\vdots\\
a_{1n}&a_{2n}&\cdots&a_{nn}
}\right]=A^T
$$
$y=x^TAx$
$$
y_{1*1}=x^T_{1*n}A_{n*n}x_{n*1}=
\left[\matrix{x_1&x_2&\cdots&x_n}\right]
[x_1\left[\matrix{a_{11}\\a_{21}\\\vdots\\a_{n1}}\right]+x_2\left[\matrix{a_{12}\\a_{22}\\\vdots\\a_{n2}}\right]+\cdots+x_n\left[\matrix{a_{1n}\\a_{2n}\\\vdots\\a_{nn}}\right]]\\
y=\sum_{i=1}^{n}{\sum_{j=1}^{n}{a_{ij}x_ix_j}}\\
\frac{\mathrm{d}y}{\mathrm{d}x}=\left[\matrix{
\frac{\part{\sum_{i=1}^{n}{\sum_{j=1}^{n}{a_{ij}x_ix_j}}}}{\part{x_1}}\\
\frac{\part{\sum_{i=1}^{n}{\sum_{j=1}^{n}{a_{ij}x_ix_j}}}}{\part{x_2}}\\
\vdots\\
\frac{\part{\sum_{i=1}^{n}{\sum_{j=1}^{n}{a_{ij}x_ix_j}}}}{\part{x_n}}
}\right]\\
=\left[\matrix{
\sum_{j=1}^{n}{a_{1j}x_j}+\sum_{i=1}^{n}{a_{i1}x_i}\\
\vdots\\
\sum_{j=1}^{n}{a_{nj}x_j}+\sum_{i=1}^{n}{a_{in}x_i}
}\right]\\
=\left[\matrix{
a_{11}&a_{12}&\cdots&a_{1n}\\
\vdots\\
a_{n1}&a_{n2}&\cdots&a_{nn}
}\right]
\left[\matrix{x_1\\\vdots\\x_n}\right]+
\left[\matrix{
a_{11}&a_{21}&\cdots&a_{n1}\\
\vdots\\
a_{1n}&a_{2n}&\cdots&a_{nn}
}\right]
\left[\matrix{x_1\\\vdots\\x_n}\right]\\
=Ax+A^Tx\\
=(A+A^T)x
$$
矩阵$A_{m*n}$对矩阵$B_{p*l}$求导，有$m*n*p*l$项元素，有4维，写不下，但是本质是一样的

矩阵求导的乘法公式
$$
\frac{\mathrm{d}{U^TV}}{\mathrm{d}{x}}=\frac{\mathrm{d}{V^TU}}{\mathrm{d}{x}}=\frac{\part{U}}{\part{x}}V+\frac{\part{V}}{\part{x}}U\\
U_{n*1}=\left[\matrix{u_1(x)\\u_2(x)\\\vdots\\u_n(x)}\right],
V_{n*1}=\left[\matrix{v_1(x)\\v_2(x)\\\vdots\\v_n(x)}\right],
x=\left[\matrix{x_1\\x_2\\\vdots\\x_n}\right]\\
U^TV=\sum_{i=1}^{n}{u_i(x)v_i(x)}\\
\frac{\mathrm{d}{U^TV}}{\mathrm{d}{x}}=\frac{\mathrm{d}{V^TU}}{\mathrm{d}{x}}=
\left[\matrix{
\frac{\part{\sum_{i=1}^{n}{u_i(x)v_i(x)}}}{\part{x_1}}\\
\vdots\\
\frac{\part{\sum_{i=1}^{n}{u_i(x)v_i(x)}}}{\part{x_n}}
}\right]\\
考察其中一项\\
\frac{\part{u_i(x)v_i(x)}}{\part{x_j}}=\frac{\part{u_i(x)}}{\part{x_j}}v_i(x)+\frac{\part{v_i(x)}}{\part{x_j}}u_i(x)（函数的求导法则）\\
\frac{\part{\sum_{i=1}^{n}{u_i(x)v_i(x)}}}{\part{x_j}}=\sum_{i=1}^{n}{\frac{\part{u_i(x)}}{\part{x_j}}v_i(x)}+\sum_{i=1}^{n}{\frac{\part{v_i(x)}}{\part{x_j}}u_i(x)}\\
=\frac{\part{U}}{\part{x_j}}V+\frac{\part{V}}{\part{x_j}}U（U视为y，横向展开，因此偏导得到行向量，结果仍然是一个数）\\
\frac{\mathrm{d}{U^TV}}{\mathrm{d}{x}}=\left[\matrix{
\frac{\part{U}}{\part{x_1}}V\\
\frac{\part{U}}{\part{x_2}}V\\
\vdots\\
\frac{\part{U}}{\part{x_n}}V
}\right]+
\left[\matrix{
\frac{\part{V}}{\part{x_1}}U\\
\frac{\part{V}}{\part{x_2}}U\\
\vdots\\
\frac{\part{V}}{\part{x_n}}U
}\right]\\
=\frac{\part{U}}{\part{x}}V+\frac{\part{V}}{\part{x}}U
$$

### Saddle points from constraints/from $\frac{x^TSx}{x^Tx}$

$$
解最优化问题\\
\min_{Ax=b}{\frac{1}{2}x^TSx}\\
A_{m*n},x_{n*1},b_{m*1},S对称正定\\
引入拉格朗日算子\\
L(x,\lambda_{m*1})=\frac{1}{2}x^TSx\pm(正负无所谓，这里取正)\lambda^T(Ax-b)\\
即L(x,\lambda)=\frac{1}{2}x^TSx+\lambda^T(Ax-b)\\
求导得到鞍点\\
\frac{\part{L}}{\part{x}}=Sx+A^T\lambda=0\\
\frac{\part{L}}{\part{\lambda}}=Ax-b=0\\
即求解\left[\matrix{S&A^T\\A&0}\right]\left[\matrix{x\\\lambda}\right]=\left[\matrix{0\\b}\right]\\
左边矩阵称为KKT矩阵\\
做块消元\\
\left[\matrix{S&A^T\\A&0}\right]\rightarrow\left[\matrix{S&A^T\\0&-AS^{-1}A^T}\right]\\
左上为正定矩阵，右下为负定矩阵，前n个主元为正，后n个主元为负，一半特征值为正，另一半负
$$

#### Rayleigh quotient（瑞利商）

$R(x)=\frac{x^TSx}{x^Tx}$，S为对称矩阵

$\min{R(x)}=\lambda_n,x=q_n$，上式最小值为$S$最小的特征值，$x$取对应特征向量

$\max{R(x)}=\lambda_1,x=q_1$，同样。

## Lecture 19 Saddle Points Continued, Maxmin Principle

## Lecture 20 Definitions and Inequalities

$$
mean=E[x]=\sum_{i=1}^{n}{p_ix_i}=m\\
variance=E[(x-m)^2]=E[x^2]-m^2=\sigma^2\\
$$

### 马尔可夫不等式（Markov Inequality）

样本$X(x_1,x_2,\cdots,x_n)\ge0$，概率$p_1,\cdots,p_n$，对于$a>0,P(X\ge a)=\sum{p_i}(备注：x_i\ge a)\le \frac{E[X]}{a}=\frac{mean(X)}{a}=\frac{\bar{X}}{a}$
$$
E[X]=\sum_{all\ x}{p_ix_i}\ge \sum_{x_i\ge a}{p_ix_i}\ge \sum_{x_i \ge a}{ap_i}\\
\sum_{x_i \ge a}{p_i}\le \frac{E[x]}{a}\\
$$

### 切比雪夫不等式（Chebyshev Inequality）

样本$X(x_1,x_2,\cdots,x_n)\ge0$，概率$p_1,\cdots,p_n$，对于$a>0,Prob(|x-m|\ge a)=\sum_{|x-m|\ge a}{p_i}\le \frac{\sigma^2}{a^2}$
$$
令Y=(X-m)^2\\
\bar{Y}=\sum{p_iy_i}=\sum{pi(x_i-m)^2}=\sigma^2\\
运用Markov不等式\\
\sum_{y_i\ge a^2}{p_i}=\sum_{(x_i-m)^2\ge a^2}{p_i}\le\frac{\bar{Y}}{a^2}=\frac{\sigma^2}{a^2}\\
即\sum_{|x-m|\ge a}{p_i}\le\frac{\sigma^2}{a^2}\\
$$

### 协方差矩阵（Covariance Matrix）

$V=V^T=\sum_{all\ x_i,y_i}{p_{ij}\left[\matrix{x_i-m_x\\y_j-m_y}\right][x_i-m_x\ y_j-m_y]}=\left[\matrix{\sigma_x^2&\sigma_{xy}\\\sigma_{yx}=\sigma_{xy}&\sigma_y^2}\right]$

$V$是对称矩阵、半正定或正定矩阵（用$x^TVx$证，$\sum$的每一项都是（半）正定的，概率是≥0的）

$p_{ij}$是$X=x_i,Y=y_j$时的联合概率（Joint Probability），$m$是均值（期望）。$X,Y$相关性越大，$\sigma_{xy}$越大（可以为负，表示负相关即$X,Y$结果相反）；两者完全独立时$\sigma_{xy}=0$，$V$为对角阵。

## Lecture 21 Minimizing a Function Step by Step

### 泰勒公式（Taylor Series）

$$
x标量\\
F(x+\Delta{x})≈F(x)+\Delta{x}\frac{\mathrm{d}{F}}{\mathrm{d}{x}}+\frac{1}{2}(\Delta{x})^2\frac{\mathrm{d}^2F}{\mathrm{d}{x}^2}\\
x=[x_1\ x_2\ \cdots\ x_n]^T\\
F(x+\Delta{x})≈F(x)+(\Delta{x})^T\nabla{F(x)}+\frac{1}{2}(\Delta{x})^TH(\Delta{x})\\
H(Hessian\ matrix)对称,H_{jk}=\frac{\part^2F}{\part{x_j}\part{x_k}}=H_{kj}\\
以上F是标量函数\\
f=[f_1(x)\ f_2(x)\ \cdots\ f_n(x)]^T\\
f(x+\Delta{x})≈f(x)+J\Delta{x}\\
J(Jacobian\ matrix),J_{jk}=\frac{\part{f_j}}{\part{x_k}}\\
$$

### 牛顿方法（Newton's Method）

用于获得方程的近似解，例如逼近$f=0$的解，$n$个方程和未知数。

假设$x^*$为$f(x)=0$的解（$f,x$都是向量），在$x^*$的邻域内选初始点$x_0$做泰勒展开取线性项，得到$f(x)≈f(x_0)+J(x_0)(x-x_0)$，令右边为0，得到近似解$x_1$。令$x_{k+1}=x_k+\Delta{x},f(x_k)+J(x_k)(x_{k+1}-x_k)=0,x_{k+1}=x_k-J^{-1}f(x_k)$

例如求$f(x)=x^2-9=0$
$$
x_{k+1}=x_k-\frac{1}{2x_k}(x_k^2-9)=\frac{1}{2}x_k+\frac{9}{2}\frac{1}{x_k}\\
x_{k+1}-3=\frac{1}{2x_k}(x_k-3)^2\\
$$
每一次迭代，$x_k$以平方的速度收敛，更接近真实解（因为是在真实解的邻域取初始值，平方项小于1，会越来越小。平方速度收敛的性质推广开来也适用）

### Minimizing F(x)

第一种方法，沿着图像最陡的方向下降（Steepest Descent）

$x_{k+1}=x_k-s\nabla{F(x_k)}$，$s$为学习率，防止步长过大图像由下降再转为上升，深度学习中精确计算学习率复杂，所以要人工设定调整。

第二种方法，问题等价于求导数为0，即$\nabla{F}=0$，运用牛顿方法

此处$f$即为$\nabla{F}$，因此$x_{k+1}=x_k-H^{-1}(x_k)\nabla{F}(x_k)$，$H$为Hessian矩阵。

第一种方法是粗略地直接选择梯度方向下降，牛顿方法区别在于下降的方向是精确的方向。

Steepest Descent收敛的速度是线性（$H$换成简单的一个数$s$，所以显然不能期待像牛顿方法一样以平方速度收敛）

考虑计算开销，实际中使用第一种方法。

### 凸性（Convexity）

以上最小化方法适用的范围是，所要最小化的函数是凸函数（现实中很多达不到这个要求，所以在图像局部表现为凸函数的时候，可以求得局部最小值？）

最典型，也最简单的问题是
$$
\min_{x\in K}{F(x)}\\
$$
$F$是一个凸函数，$K$是一个凸集，问题没有鞍点和局部最优解，只有全局最优解，一旦找到最小值就是全局最小值。常见的比如
$$
\min_{Ax=b}{F(x)}\\
$$
$Ax=b$解得的$x$构成一个仿射（affine）（其实就是从某个标准的向量空间做线性变换得到它，加法数乘平移），不能说子空间，因为解不一定包含零向量（仅当b=0包含）

### Convex Set K

凸集$K$的性质是从集合内任选两点做线段，线段上的点都在$K$内

以三角形内的点为例

凸集的并集一般不是交集

凸集的交集一定是交集

### Convex Function

在函数图像上以及函数图像上方的点构成凸集，则该函数为凸函数。

更常用的验证方法为，$f$输出为标量

当$x$为标量时，$\frac{\mathrm{d}^2{f}}{\mathrm{d}{x}^2}\ge0$恒成立，则$f$为凸函数

当$x$为向量时，Hessian矩阵$H$为正定或半正定矩阵恒成立，则为凸函数

多个凸函数$f_1,f_2,\cdots,f_n,\min\{f_1,f_2,\cdots,f_n\}$通常不是凸函数（类似并集），$\max\{f_1,f_2,\cdots,f_n\}$一定是凸函数（其图像本就在所有凸函数上方，构成凸集。类似交集）。

二次型函数$f(x),(eg:f(x)=x_1^2+x_2^2)$通常（也可能一定，这里没细说）可以写成$f(x)=\frac{1}{2}x^TSx$，这里的$S$就是Hessian矩阵$H$。

凸函数没有局部最小值，没有鞍点，一旦找到最小值，就是全局最小值。

## Lecture 22 Gradient Descent：Downhill to a Minimum

纯二次型函数（没有交叉相乘项），例如$f(x)=\frac{1}{2}(x^2+by^2)=\frac{1}{2}x^TSx,S=\left[\matrix{1\\&b}\right]$

$S$的条件数（Condition number）决定了函数收敛到最优解的速度（这里没说是什么方法，应该是steepest descent）。假设$b<1$，条件数就是$1/b$，即$S$对称时，是$\lambda_{max}/\lambda_{min}$，当条件数太大时，会遇到问题（步长太大？）。$S$不对称，则用$\sigma_{max}/\sigma_{min}$

最优化问题中$F$取得最优解的点$x^*$称作$\arg\min(F)$

凸函数$f(X_{n*n})=-\log(\det(X)),\nabla{f}=-(X^{-1})^T$

$X^{-1}_{ij}=\frac{C_{ji}}{\det(X)}$，梯度用链式求导法则以及将行列式按行展开得到

### 梯度下降

$f(x+\Delta{x})≈f(x)+(\Delta{x})^T\nabla{f},f(x+\Delta{x})-f(x)≈(\Delta{x})^T\nabla{f}$

右边是内积的形式，因此当$\Delta{x}$与梯度同向时值最大，逆着梯度方向就减小最多。

求$F(x)$最小值，迭代$x_{k+1}=x_k-s_k\nabla{F(x_k)},s_k$为第$k$次迭代的步长即学习率

最优的学习率可以通过线搜索获得(exact line search，思想大概是定一个区间，然后按比例$a$缩短左端或者右端，每次检查两端$F(x+\Delta{x})-F(x)$的值，保留最小者，类似二分查找)，得到最优的$F(x_{k+1})$

~~应该是因为计算开销太大所以现实没人用吧~~

梯度方向即为法向，垂直于切线、切面。

## Lecture 23 Accelerating Gradient Descent（Use Momentum）

### Momentum

$$
x_{k+1}=x_k-sz_k\\
z_k=\nabla{f(x_k)}+\beta z_{k-1}\\
$$

s为学习率，$\beta$也是人调的超参数

当$f=\frac{1}{2}x^TSx$时，$\nabla{f}=Sx$

公式变为
$$
x_{k+1}=x_k-sz_k\\
z_{k+1}-Sx_{k+1}=\beta z_k\\
\left[\matrix{
1&0\\
-S&1
}\right]
\left[\matrix{x\\z}\right]_{k+1}
=\left[\matrix{
1&-s\\
0&\beta
}\right]
\left[\matrix{x\\z}\right]_k\\
$$
这里假设$S$是对称正定矩阵，$x_k=c_kq,z_k=d_kq,Sx_k=c_k\lambda q$
$$
\left[\matrix{
1&0\\
-\lambda&1
}\right]
\left[\matrix{c\\d}\right]_{k+1}
=\left[\matrix{
1&-s\\0&\beta
}\right]
\left[\matrix{c\\d}\right]_k\\
\left[\matrix{c\\d}\right]_{k+1}=
\left[\matrix{
1&-s\\\lambda&\beta-\lambda s
}\right]
\left[\matrix{c\\d}\right]_{k}=R\left[\matrix{c\\d}\right]_{k}\\
$$
$R$的特征值由$s,\beta$决定，函数收敛的速度由$R$的条件数$\kappa=\frac{M}{m}$决定，$M$是$R$最大的特征值，$m$是最小特征值，条件数越小收敛越快

最优的选择为
$$
s=(\frac{2}{\sqrt{M}+\sqrt{m}})^2\\
\beta=(\frac{\sqrt{M}-\sqrt{m}}{\sqrt{M}+\sqrt{m}})^2\\
|eigenvalues\ of\ R|<(\frac{\sqrt{M}-\sqrt{m}}{\sqrt{M}+\sqrt{m}})^2\\
$$
~~不是，写了半天就讲了个特例？实际中好像也不这么用啊？~~

### Nesterov

$$
x_{k+1}=x_k+\beta(x_k-x_{k-1})-s\nabla{f(x_k+\gamma(x_k-x_{k-1}))}\\
$$

## Lecture 24 Linear Programming and Two-Person Games

### linear programming(线性规划)

$$
\min_{Ax=b\\x\ge0}{c^Tx}\\
$$

$c$是费用向量(cost vector)，因为$c^Tx$是一次方表达式，所以叫线性规划

一般$A_{m*n},m<n$

$x$的取值范围称为feasible set，记作$K$

因为费用是线性的，最小值会在边界点取到。一种计算最小值方法叫simplex method，从一个边界点出发，计算最陡的边（类似梯度下降），沿着边到达新的边界点，重复。实际消耗步骤是多项式级，但是理论上是指数级。另一种方法是interior point methods（没有详细讲，大致是不止遍历各个边界点，解决过程中$x$可以取$K$内部的点，可以使用微积分和梯度下降和牛顿方法。两种方法哪一种更快看情况。）

再另一种方法是解决它的对偶问题。

#### dual problem

$$
\max_{A^Ty\le c}{b^Ty}\\
$$

（对于$A_{n*m}^T,m<n$，方程数比未知数多，所以比较好解？）

##### 弱对偶性(weak duality)

$$
b^Ty=x^TA^Ty\le x^Tc=c^Tx\\
$$

$x\ge0$的作用体现在不等号那里。

##### （强）对偶性(duality)

the minimax theorem：线性规划里$b^Ty$的最大值和$c^Tx$的最小值相等，即最优解相等，$b^Ty^*=c^Tx^*$

simplex method可以同时解决两个对偶问题的最优解。（~~那不是废话吗都遍历了？~~）

在非线性规划里，以上最大值和最小值可能不相等，有差值（gap）。

##### max flow min cut

简单点了一下，没太看懂

##### Two-person Game

零和游戏，x付出，y收取，x需要付出最小，y需要收取最大

算代价矩阵的期望

理解的重点是让最坏的情况代价尽量小（收获尽量大），因此令每一行（列）期望相等，求得概率。举例说明，如果其中一行期望偏大，会被对方发觉，然后被利用让对方总是选更小的那个。因此期望相等是最优的情况。

## Lecture 25 Stochastic Gradient Descent

### SGD

梯度下降(batch gradient descent)
$$
a_{k+1}=a_k-s\frac{1}{n}\sum_{x=x_1}^{x_n}{\nabla f_{x}(a_k)}\\
a为要学习的参数，x为样本\\
$$
SGD
$$
a_{k+1}=a_k-s\nabla{f_{x_i}(a_k)}\\
x_i为从x_1,\cdots,x_n中随机选择的样本\\
$$
每一次迭代，不计算所有样本的梯度取平均，而是随机选一个样本计算梯度做迭代

当特征数量和样本数量非常大时，计算所有样本的梯度非常消耗时间

初期可以快速地减小误差（$\min{f(x)},f(x)$的值下降非常快），越接近最优解，步长越大波动越大，图像上看在最优解附近越分散

因为深度学习更看重在未见过的数据上的性能，所以在已见过的数据上达到最优也不一定在未见过的数据上最优，因此快速迭代达到最优解附近，在最优解附近波动是可以接受的，最终在新数据上不会差很多。

以一维为例
$$
\min{f(x)=\frac{1}{2}\sum_{i=1}^{n}{(a_ix-b_i)^2}}\\
$$
所有元素皆为标量

图像上，每一个组成都是一个向上的二次函数，一个组成$f_i(x)=\frac{1}{2}(a_ix-b_i)^2$

$f(x)$的最优解
$$
\nabla{f(x)}=\sum_{i=1}^{n}{(a_ix-b_i)a_i}=0\\
x^*=\frac{\sum{a_ib_i}}{\sum{a_i^2}}\\
$$
$f_i(x)$的最优解
$$
\nabla{f_i(x)}=(a_ix-b_i)a_i=0\\
x=\frac{b_i}{a_i}\\
$$
假设$\frac{b_p}{a_p}=\min{\frac{b_i}{a_i}},\frac{b_q}{a_q}=\max{\frac{b_i}{a_i}}$
$$
\frac{b_p}{a_p}=\frac{\sum{a_i^2\frac{b_p}{a_p}}}{\sum{a_i^2}}\le x^*=\frac{\sum{a_i^2\frac{b_i}{a_i}}}{\sum{a_i^2}}\le\frac{\sum{a_i^2\frac{b_q}{a_q}}}{\sum{a_i^2}}=\frac{b_q}{a_q}\\
$$
因此
$$
x^*\in[\frac{b_p}{a_p},\frac{b_q}{a_q}]\\
$$
在这个区间外，$\nabla{f(x)}$和$\nabla{f_i(x)}$符号相同，后者是前者的一个分量（一个组成方向），因此，在梯度下降的一开始，随机初始化的点落在区间外，梯度下降和SGD都可以达到回归区间（接近最优解）的效果，SGD计算开销小，回归非常快。

在区间内，符号不再一直相同，所以SGD有时会离最优解更远，有时更近，图像上就体现出波动和分散。

高维情况类似，低维为高维的思考提供直觉。

在实践中，按经验看如果SGD某一轮迭代得到的函数值已经落到最优附近（$\min{f(x)}$小于某值），则可以提前终止，否则一直波动浪费时间。当然也可以继续迭代让模型更普适化更稳定更鲁棒（健壮），一切看你的心情和想法（~~玄学开始了~~）

SGD有两种方法，一是每次从所有样本中随机取一种，可能重复取（replacement）；二是每取走一个样本就将它从可取样本中去除，不会重复取，直到所有样本都被取一遍（without replacement）。哪一个更好没有定论，实际中都使用第二种，实现为将数据集打乱然后遍历，因为第一种每次都从很大量的样本中随机选一个开销太大。然而实际上有数学理论分析作为基础的是第一种，而不是第二种。对mini-batch也一样。

取单个的样本可能有很大的方差，例如真实值是0，正无穷和负无穷的平均值是0，但是方差极大。为了减小方差实际中更多用mini-batch，一组batch可能10,100,1000个样本。
$$
a_{k+1}=a_k-s\frac{1}{|I_k|}\sum_{j\in I_k}{\nabla{f_{x_j}(a_k)}}\\
$$
按理说mini-batch越大越接近full gradient descent，理论上效果更好，但实际中更容易过拟合。加入一些噪声更能增加模型的普适性鲁棒性。

为什么SGD这种丑陋的优化方法比很多优雅的方法在深度学习中效果要好，还没有答案（课程是18年，还在找普适化的理论）；为什么SGD在深度学习中有如此好的效果也没有严格的理论支撑，更多的是上述的直觉。

## Lecture 26 Structure of Neural Nets for Deep Learning

不懂为什么讲了一节课平面将高维空间分成多少份的问题

$N$个平面两两相交将$m$维空间分成

$r(N,m)=C_0^N+C_1^N+\cdots+C_m^N$份

$r(N,m)=r(N-1,m)+r(N-1,m-1)$

## Lecture 27 Backpropagation：Find Partial Derivatives

讲了一些计算图，讲得并不好，说得不清楚。

用矩阵乘法举例说明计算顺序的重要性，后向传播是计算偏微分的正确顺序。

$A_{m*n},B_{n*p},C_{p*q},(AB)C$的计算开销是$mnp+mpq,A(BC)$的计算开销是$npq+mnq$，假设$q=1,m,n,p$很大，则$(AB)C$的开销为$mp(n+1)$，$A(BC)$的开销为$n(m+p)$，后者比前者小得多。

## Lecture 28 Completing a Rank-One Matrix，Circulants！

### Rank-One Matrix

一个秩1矩阵$uv^T,u_{m*1},v_{n*1}$，则它有$m+n-1$个自由度，因为可以将$u$第一个元素化为1，按倍数调整其他元素，结果不变。

任意给定$m+n-1$个矩阵元素，能将这个$m*n$矩阵填满为秩1矩阵当且仅当以行号为一部分节点，以列号为一部分节点，作二分图，以给定元素的位置为边连接节点（例如$a_{11}$位置给定了元素则连接行1和列1节点），图中无回路。

### Circulants

Circulants（回路、循环）矩阵如下

一个4x4的矩阵只有4个不同的元素，每个对角线有4个元素且都相同
$$
C=\left[\matrix{
2&5&1&0\\
0&2&5&1\\
1&0&2&5\\
5&1&0&2\\
}\right]\\
$$
置换矩阵$P=\left[\matrix{&1\\&&1\\&&&1\\1}\right]$是Circulants矩阵

两个Circulants矩阵的乘积仍是Circulants矩阵

以4阶为例，假设Circulants矩阵$C$的4个不同元素是$c_0,c_1,c_2,c_3$，则任意$C=c_0I+c_1P+c_2P^2+c_3P^3$，即任意$C,D$可以表达为$P$的多项式，$CD$仍是$P$的多项式，且$P^4=I$，因此最终表达式仍是4阶Circulants矩阵。

$C,P$特征向量相同

## Lecture 31 Eigenvectors of Circulant Matrices：Fourier Matrix

### cyclic convolution

循环卷积？cyclic convolution matrix（circulants matrix）$C$由其一行或者一列$c$决定，$Cv$相当于$c$和$v$做循环卷积$c\otimes v$，计算方式为将$c$和$v$做竖式乘法，举例
$$
c^T=[1,2,3]\\
v^T=[4,5,6]\\
c\otimes v=\\
\frac{\matrix{&&&1&2&3\\&&&4&5&6}}{\frac{\matrix{&&6&12&18\\&5&10&15\\4&8&12}}{\matrix{4&13&28&27&18}}}\\
$$
如果是非循环卷积$c*v$，则结果就是$[4,13,28,27,18]^T,1+2+3=6,4+5+6=15,6*15=90,4+13+28+27+18=90$，即两个向量的元素和的成绩等于卷积后向量的元素和，可以用这种方法检验卷积的正确性。

循环卷积$c\otimes v$，实际上可以对应circulants matrix，以及多项式乘法。例如$c$可以对应$1+2x+3x^2$，$v$对应$4+5x+6x^2$，且$x^0=x^3=1$，则$c\otimes v$对应$(1+2x+3x^2)(4+5x+6x^2)=4+13x+28x^2+27x^3+18x^4$，系数对应卷积结果，将$x^3,x^4$化为$1,x$，得到$31+31x+28x^2$，$c\otimes v=[31,31,28]^T,31+31+28=90=6*15$。实际上就是多一个将周期项相加的步骤。

还说了一些卷积矩阵（Convolution Matrix\Toeplitz Matrix）的内容，其乘以向量即为对向量做非循环卷积，感觉跟认知的卷积相差很多

Toeplitz矩阵
$$
T=\left[\matrix{
t_0&t_1&t_2&t_3&\cdots&t_n\\
t_{-1}&t_0&t_1&t_2&\cdots&t_{n-1}\\
t_{-2}&t_{-1}&t_0&t_1&t_2&\cdots\\
t_{-3}&t_{-2}&t_{-1}&t_0&t_1&\cdots\\
\vdots&\cdots\\
}\right]\\
$$
即每一条对角线的元素都相等

教授举例的卷积矩阵还要求关于主对角线对称，即$t_1=t_{-1},t_2=t_{-2},\cdots$

因为大量元素重复，所以可以节约很多存储空间，不需要那么多权重，以及计算开销，（第一行就决定了整个卷积矩阵）因此对一张图做卷积以及卷积神经网络特别重要。

### 置换矩阵P的特征值和特征向量

$P=\left[\matrix{&1\\&&1\\&&&1\\1}\right]$

$PP^T=I,P$为正交矩阵，奇异值为1。

特征值由解行列式求得，$n$阶$P$的特征值为$\lambda^n=1$

正交矩阵的特征向量也是正交的。

特征向量正交的矩阵有对称矩阵（对称矩阵特征值是实数），正交矩阵（特征值模长为1，因为正交矩阵不改变向量长度，$||Qx||=||x||$），反对称矩阵（特征值为虚数，对向量做旋转）。以上统称为正规矩阵（Normal Matrix），性质和检验方法是$A^HA=AA^H$

任意Circulants矩阵是正规矩阵，任意两个Circulants矩阵乘法满足交换律，任意Circulants矩阵的特征向量相同，都和P的特征向量一样（以上都说同阶，用P的多项式表达，一目了然）

$8$阶$P$的特征值$1,w=\mathrm{e}^{\frac{2\pi i}{8}},w^2,\cdots,w^7$，对应的特征向量构成8阶傅里叶矩阵$F_8$

$F_8$各列正交可以由$P$是正规矩阵得到，也可以做点乘（18.06笔记，暴力计算）。

简便一点的计算可以尝试如下方法，举例：
$$
e_1=\left[\matrix{1\\1\\1\\1\\1\\1\\1\\1}\right],
e_2=\left[\matrix{1\\w\\w^2\\w^3\\w^4\\w^5\\w^6\\w^7}\right]\\
e_1^Te_2=1+w+w^2+w^3+w^4+w^5+w^6+w^7=t\\
w^8=w^0=1\\
wt=t\\
t=0\\
$$
即求内积之后整个式子乘以$t$的第一项，推出$t=0$（没有对任意两列的情况做验证，应该可以）

## Lecture 32 ImageNet is a Convolutional Neural Network（CNN），The Convolution Rule

$$
c_{n*1},d_{n*1}\\
(c*d)_{k}=\sum_{i}{c_id_{k-i}}\\
用多项式展开看待。类似的，有\\
f(x),g(x)\\
f*g(x)=\int_{-\infin}^{\infin}f(t)g(x-t)\mathrm{d}{t}\\
$$

Circulant矩阵的特征值是$Fc,F$是特征向量构成的傅里叶矩阵，$c$是Circulant矩阵的第一行，也是用$P$表示的多项式展开的系数。

n阶Circulant矩阵$C,D$，$CD$由$c\otimes d$决定，分别是各自的第一行，结果是乘积的第一行。$F_n(c\otimes d)$是$CD$的特征值

$C,D$特征向量一样，做特征值分解，再相乘，得到$C=X\Lambda_CX^{-1},D=X\Lambda_DX^{-1},CD=X\Lambda_C\Lambda_DX^{-1}$，即$CD$特征值是$C,D$对应相同特征向量的特征值的乘积。

Convolution Rule：

$F_n(c\otimes d)=(F_nc)\cdot*(F_nd),\cdot*$表示两个向量对应位置元素相乘。

傅里叶矩阵乘以向量，由于FFT的存在，只需要$n\log(n)$的时间，因此计算$Fc$非常快。

Convolution Rule重要在于，$c\otimes d$复杂度是$n^2$，左侧复杂度为$n^2+n\log(n)$，右侧复杂度为$2n\log(n)+n$

2维函数卷积$f*g(x,y)=\int_{-\infin}^{\infin}{\int_{-\infin}^{\infin}{f(u,t)g(x-u,y-t)\mathrm{d}{u}\mathrm{d}{t}}}$

## Lecture 33 Neural Nets and the Learning Function

介绍了一些浅显基础的神经网络，跳过。

### Distance Matrix

给定距离矩阵$D$，表示$R^d$中任意两点$x_i,x_j$的距离，求各点的位置矩阵$X$

$d_{ij}=||x_i-x_j||^2=x_i^Tx_i-x_i^Tx_j-x_j^Tx_i+x_j^Tx_j$

定义$d_i=d_{ii}$上式写成矩阵形式

$D=\left[\matrix{1\\1\\\vdots\\1}\right]\left[\matrix{d_1&d_2&\cdots&d_n}\right]+\left[\matrix{d_1\\d_2\\\vdots\\d_n}\right]\left[\matrix{1&1&\cdots&1}\right]-2X^TX$

得到在$i$行$j$列有$d_i+d_j-2x_i^Tx_j=x_i^Tx_i+x_j^Tx_j-2x_i^Tx_j=d_{ij}$

$X^TX=G=-\frac{1}{2}D+\frac{1}{2}\left[\matrix{1\\1\\\vdots\\1}\right]\left[\matrix{d_1&d_2&\cdots&d_n}\right]+\frac{1}{2}\left[\matrix{d_1\\d_2\\\vdots\\d_n}\right]\left[\matrix{1&1&\cdots&1}\right]$

问题转化为给定$X^TX$，求$X$

$X$必定有多个解，如果求得一个解$X$，那么$QX$相当于对$X$即各个位置的点做旋转，$X^TX$不变，点之间的距离不变。

一种解法是求$X^TX$的特征值和特征向量，另一种是对它做消元

$X^TX$是对称（半）正定矩阵，分解为$X^TX=Q\Lambda Q^T$，求得其中一个解为$X=Q \Lambda^{1/2}Q^T$，同样为对称（半）正定矩阵。

消元（LU分解），得到$X^TX=LDU$，其中$L,U$对角线为$1,D$是对角阵。注意$X^TX$对称，所以结果必定为$X^TX=LDL^T$，得到其中一个解为$X=D^{1/2}L^T$。求得$LDL^T$的这个过程叫Cholesky Factorization。此时$X$是一个上三角矩阵。这种方法比特征分解快得多。

## Lecture 34 Distance Matrices，Procrustes Problem

当且仅当距离矩阵$D$中的数字满足三角不等式时，$X$位置矩阵才存在，$G$才是（半）正定矩阵。

### procrustes problem（普氏问题）

两组不同的向量$x_i$和$y_i$，寻找最合适的正交矩阵$Q$使得$YQ≈X$

如果$X,Y$都是各列正交的，则可以求得一个准确的$Q$

问题的表示为
$$
\min_{Q^TQ=I}{||YQ-X||_F^2}=tr((YQ-X)^T(YQ-X))=\sum{\sigma_{YQ-X}^2}\\
$$
注意$||QA||_F^2=||A||_F^2$，因为$QA=[Qa_1\ \cdots\ Qa_n]$，而$Q$乘以列向量相当于旋转，不改变列向量长度，$(Qx)^TQx=x^Tx$，因此每一列的长度不变，矩阵的F范数不变

$QA$或$AQ$不改变奇异值

$tr(A^TB)=tr(B^TA)(tr(A)=tr(A^T))$

$tr(A^TB)=tr(BA^T)(tr(AB)=tr(BA)),\sum$交换求和顺序证。教授用$AB,BA$特征值相等因此它们的和即迹相等证，但是没有详细说明，如果AB非方阵似乎有点麻烦。

上述最优化问题的解为（书本P258）
$$
Y^TX=U\Sigma V^T\\
Q_{best}=UV^T\\
$$

## Lecture 35 Finding Clusters in Graphs

上节课遗留了一个为什么神经网络有效的问题，没有视频。

一张图，将节点分成规模大致相等的k（这里k=2）类，并找到这k类的中心。

用数学语言描述，则将节点分为$a,b$两组，中心分别为$x,y$。寻找$x,y$使得
$$
\min_{a\cup b=all\ nodes\\a\cap b=\varnothing}{\sum{||a_i-x||^2}+\sum{||b_i-y||^2}}\\
$$

### k-means

一种方法是k-means算法。首先随机选取k个点，尽量分散，然后遍历各个点，将其并入最接近的那一类，重新计算类中心。重复此过程。

### spectral clustering

图的矩阵

incidence matrix，边为行号，节点为列号，边的起点写-1，终点写1，这里表示为A

degree matrix，对角矩阵，表示每个节点相连的边数，这里表示为D

adjacency matrix，行列都是节点，填1表示两个节点相连，对称矩阵，这里表示为B

拉普拉斯矩阵（Laplacian matrix）$L=A^TA=D-B$，对称半正定矩阵

$Lx=0$的解为全1，$\dim{N(L)}=1$，L有特征值0，对应向量全1。

Fiedler vector，L大于0的最小特征值对应的特征向量。

将图分为2部分的情况，Fiedler vecotr将有2个组成，一部分正，一部分负，对应将图的节点分为2堆。如果分为3堆，则需要求下一个特征值和特征向量（即一共3个）。（具体是不是下一个最小的特征值没说，也没说怎么分）

## Lecture 36 Alan Edelman and Julia Language

介绍Julia语言，反正我又不用。

