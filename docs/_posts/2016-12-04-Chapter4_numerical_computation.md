---
title: 数值计算
layout: post
share: false
---

机器学习算法通常需要大量的数值计算。
这通常是指通过迭代地更新解来解决数学问题的算法，而不是解析地提供正确解的符号表达。
常见的操作包括优化（找到最小化或最大化函数值的参数）和线性方程组的求解。
对数字计算机来说实数无法在有限内存下精确表示，因此仅仅计算涉及实数的函数也是困难的。


# 上溢和下溢

在数字计算机上实现连续数学的根本困难是，我们需要通过有限数量的位模式来表示无限多的实数。
这意味着我们在计算机中表示实数时，几乎总会引入一些近似误差。
在许多情况下，这仅仅是舍入误差。
如果在理论上可行的算法没有被设计为最小化舍入误差的累积，可能就会在实践中失效，因此舍入误差会导致一些问题（特别是许多操作复合时）。

一种特别的毁灭性舍入误差是下溢。
当接近零的数被四舍五入为零时发生下溢。
许多函数在其参数为零而不是一个很小的正数时才会表现出质的不同。
例如，我们通常要避免被零除（一些软件环境将在这种情况下抛出异常，有些会返回一个非数字(not-a-number)的占位符）或避免取零的对数（这通常被视为$-\infty$，进一步的算术运算会使其变成非数字）。

<!-- % -- 77 -- -->

另一个极具破坏力的数值错误形式是上溢。
当大量级的数被近似为$\infty$或$-\infty$时发生上溢。
进一步的运算通常导致这些无限值变为非数字。

必须对上溢和下溢进行数值稳定的一个例子是softmax函数。
softmax函数经常用于预测与multinoulli分布相关联的概率，定义为
\begin{align}
 \text{softmax}(\Vx)_i = \frac{\exp(\Sx_i)}{\sum_{j=1}^n \exp(\Sx_j)} .
\end{align}
考虑一下当所有$\Sx_i$都等于某个常数$\Sc$时会发生什么。
从理论分析上说，我们可以发现所有的输出都应该为$\frac{1}{n}$。
从数值计算上说，当$\Sc$量级很大时，这可能不会发生。
如果$\Sc$是很小的负数，$\exp(c)$就会下溢。
这意味着softmax函数的分母会变成0，所以最后的结果是未定义的。
当$\Sc$是非常大的正数时，$\exp(c)$的上溢再次导致整个表达式未定义。
这两个困难能通过计算$\text{softmax}(\Vz)$同时解决，其中$\Vz = \Vx - \max_i \Sx_i$。
简单的代数计算表明，$\text{softmax}$解析上的函数值不会因为从输入向量减去或加上标量而改变。
减去$\max_i x_i$导致$\exp$的最大参数为$0$，这排除了上溢的可能性。
同样地，分母中至少有一个值为1的项，这就排除了因分母下溢而导致被零除的可能性。

还有一个小问题。
分子中的下溢仍可以导致整体表达式被计算为零。
这意味着，如果我们在计算$\log ~\text{softmax}(\Vx)$时先计算$\text{softmax}$再把结果传给$\log$函数，会错误地得到$-\infty$。
相反，我们必须实现一个单独的函数，并以数值稳定的方式计算$\log \text{softmax}$。
我们可以使用相同的技巧稳定$\log \text{softmax}$函数。

在大多数情况下，我们没有明确地对本书描述的各种算法所涉及的数值考虑进行详细说明。
底层库的开发者在实现深度学习算法时应该牢记数值问题。
本书的大多数读者可以简单地依赖保证数值稳定的底层库。
在某些情况下，有可能在实现一个新的算法时自动保持数值稳定。
Theano{cite?}就是这样软件包的一个例子，它能自动检测并稳定深度学习中许多常见的数值不稳定的表达式。

<!-- % -- 78 -- -->


# 病态条件数


条件数表明函数相对于输入的微小变化而变化的快慢程度。
输入被轻微扰动而迅速改变的函数对于科学计算来说是可能是有问题的，因为输入中的舍入误差可能导致输出的巨大变化。

考虑函数$f(\Vx) = \MA^{-1} \Vx$。
当$\MA \in \SetR^{n \times n}$ 具有特征值分解时，其条件数为
\begin{align}
 \underset{i,j}{\max}~ \Bigg| \frac{\lambda_i}{ \lambda_j} \Bigg|.
\end{align}
这是最大和最小特征值的模之比。
当该数很大时，矩阵求逆对输入的误差特别敏感。

这种敏感性是矩阵本身的固有特性，而不是矩阵求逆期间舍入误差的结果。
即使我们乘以完全正确的矩阵逆，病态条件数的矩阵也会放大预先存在的误差。
在实践中，该错误将与求逆过程本身的数值误差进一步复合。




# 基于梯度的优化方法


大多数深度学习算法涉及某种形式的优化。
优化指的是改变$\Vx$以最小化或最大化某个函数$f(\Vx)$的任务。
我们通常以最小化$f(\Vx)$指代大多数最优化问题。
最大化可经由最小化算法最小化$-f(\Vx)$来实现。

我们把要最小化或最大化的函数称为目标函数或准则。
当我们对其进行最小化时，我们也把它称为代价函数、损失函数或误差函数。
虽然有些机器学习著作赋予这些名称特殊的意义，但在这本书中我们交替使用这些术语。

我们通常使用一个上标$*$表示最小化或最大化函数的$\Vx$值。
如我们记$\Vx^*=\argmin f(\Vx)$。

我们假设读者已经熟悉微积分，这里简要回顾微积分概念如何与优化联系。

<!-- % -- 79 -- -->

假设我们有一个函数$\Sy = f(\Sx)$， 其中$\Sx$和$\Sy$是实数。
这个函数的导数记为$f^\prime(x)$或$\frac{dy}{dx}$。
导数$f^\prime(\Sx)$代表$f(\Sx)$在点$x$处的斜率。
换句话说，它表明如何缩放输入的小变化才能在输出获得相应的变化：
$f(\Sx+\epsilon) \approx f(\Sx) + \epsilon f^\prime(\Sx) $。

因此导数对于最小化一个函数很有用，因为它告诉我们如何更改$x$来略微地改善$y$。
例如，我们知道对于足够小的$\epsilon$来说，$f(\Sx-\epsilon \text{sign}(f^\prime(\Sx)) )$是比$f(\Sx)$小的。
因此我们可以将$\Sx$往导数的反方向移动一小步来减小$f(\Sx)$。
这种技术被称为梯度下降{cite?}。
\fig?展示了一个例子。
\begin{figure}[!htb]
\ifOpenSource
\centerline{\includegraphics{figure.pdf}}
\else
\centerline{\includegraphics{Chapter4/figures/gradient_descent_color}}
\fi
\caption{梯度下降。 
梯度下降算法如何使用函数导数的示意图，即沿着函数的下坡方向（导数反方向）直到最小。}
\end{figure}

<!-- % -- 80 -- -->

当$f^\prime(\Sx)=0$，导数无法提供往哪个方向移动的信息。
$ f^\prime(\Sx)=0 $的点称为临界点或驻点。
一个局部极小点意味着这个点的$f(\Sx)$小于所有邻近点，因此不可能通过移动无穷小的步长来减小$f(\Sx)$。
一个局部极大点是$f(x)$意味着这个点的$f(\Sx)$大于所有邻近点，因此不可能通过移动无穷小的步长来增大$f(x)$。
有些临界点既不是最小点也不是最大点。这些点被称为鞍点。
见\fig?给出的各种临界点的例子。
\begin{figure}[!htb]
\ifOpenSource
\centerline{\includegraphics{figure.pdf}}
\else
\centerline{\includegraphics{Chapter4/figures/critical_color}}
\fi
\caption{临界点的类型。 
一维情况下，三种临界点的示例。
临界点是斜率为零的点。
这样的点可以是局部极小点，其值低于相邻点; 局部极大点，其值高于相邻点; 或鞍点，同时存在更高和更低的相邻点。
}
\end{figure}

使$f(x)$取得绝对的最小值（相对所有其他值）的点是全局最小点。
函数可能只有一个全局最小点或存在多个全局最小点，
还可能存在不是全局最优的局部极小点。
在深度学习的背景下，我们优化的函数可能含有许多不是最优的局部极小点，或许多被非常平坦的区域包围的鞍点。
尤其是当输入是多维的时候，所有这些都将使优化变得困难。
因此，我们通常寻找$f$非常小的值，但在任何形式意义下并不一定是最小。
见\fig?的例子。
\begin{figure}[!htb]
\ifOpenSource
\centerline{\includegraphics{figure.pdf}}
\else
\centerline{\includegraphics{Chapter4/figures/approx_opt_color}}
\fi
\caption{近似最小化。 
当存在多个局部极小点或平坦区域时，优化算法可能无法找到全局最小点。
在深度学习的背景下，即使找到的解不是真正最小的，但只要它们对应于代价函数的显著低的值，我们通常就能接受这样的解。
}
\end{figure}

我们经常最小化具有多维输入的函数：$f: \SetR^n \rightarrow \SetR $。 
为了使"最小化"的概念有意义，输出必须是一维的(标量)。

<!-- % -- 81 -- -->

针对具有多维输入的函数，我们需要用到偏导数的概念。
偏导数$\frac{\partial}{\partial \Sx_i}f(\Vx)$衡量点$\Vx$处只有$x_i$增加时$f(\Vx)$如何变化。
梯度是相对一个向量求导的导数:$f$的导数是包含所有偏导数的向量，记为$\nabla_{\Vx} f(\Vx)$。
梯度的第$i$个元素是$f$关于$x_i$的偏导数。
在多维情况下，临界点是梯度中所有元素都为零的点。

在$\Vu$（单位向量）方向的方向导数是函数$f$在$\Vu$方向的斜率。
换句话说，方向导数是函数$f(\Vx + \alpha \Vu)$关于$\alpha$的导数（在$\alpha = 0$时取得）。
使用链式法则，我们可以看到当$\alpha=0$时，$\frac{\partial}{\partial \alpha} f(\Vx + \alpha \Vu) = \Vu^\Tsp \nabla_{\Vx} f(\Vx)$。

为了最小化$f$，我们希望找到使$f$下降得最快的方向。
计算方向导数：
\begin{align}
 \underset{\Vu, \Vu^\Tsp\Vu = 1}{\min} \Vu^\Tsp & \nabla_{\Vx} f(\Vx) \\
 = \underset{\Vu, \Vu^\Tsp\Vu = 1}{\min} \| \Vu \|_2 \| &\nabla_{\Vx}f(\Vx) \|_2 \cos \theta
\end{align}
其中$\theta$是$\Vu$与梯度的夹角。
将$ \| \Vu \|_2 = 1$代入，并忽略与$\Vu$无关的项，就能简化得到$ \underset{\Vu}{\min} \cos \theta $。 
这在$\Vu$与梯度方向相反时取得最小。
换句话说，梯度向量指向上坡，负梯度向量指向下坡。
我们在负梯度方向上移动可以减小$f$。
这被称为\textbf{最速下降法}(method of steepest descent)或梯度下降。

最速下降建议新的点为
\begin{align}
  \Vx' = \Vx - \epsilon \nabla_{\Vx} f(\Vx)
\end{align}
其中$\epsilon$为学习速率，是一个确定步长大小的正标量。
我们可以通过几种不同的方式选择$\epsilon$。
普遍的方式是选择一个小常数。
有时我们通过计算，选择使方向导数消失的步长。
还有一种方法是根据几个$\epsilon$计算$f(\Vx - \epsilon \nabla_{\Vx} f(\Vx))$， 并选择其中能产生最小目标函数值的$\epsilon$。
这种策略被称为线搜索。

最速下降在梯度的每一个元素为零时收敛（或在实践中，很接近零时）。
在某些情况下，我们也许能够避免运行该迭代算法，并通过解方程$\nabla_{\Vx} f(\Vx)= 0$直接跳到临界点。

<!-- % -- 82 -- -->

虽然梯度下降被限制在连续空间中的优化问题，但不断向更好的情况移动一小步（即近似最佳的小移动）的一般概念可以推广到离散空间。
递增带有离散参数的目标函数被称为爬山算法{cite?}。


## 梯度之上：Jacobian和Hessian矩阵

有时我们需要计算输入和输出都为向量的函数的所有偏导数。
包含所有这样的偏导数的矩阵被称为Jacobian矩阵。
具体来说，如果我们有一个函数：$\Vf: \SetR^m \rightarrow \SetR^n$，$\Vf$的Jacobian矩阵$\MJ \in \SetR^{n \times m}$定义为$J_{i,j} = \frac{\partial}{\partial \Sx_j} f(\Vx)_i$。

有时，我们也对导数的导数感兴趣，即二阶导数。
例如，有一个函数$f: \SetR^m \rightarrow \SetR$，$f$的一阶导数(关于$\Sx_j$)关于$x_i$的导数记为$\frac{\partial^2}{\partial \Sx_i \partial \Sx_j} f$。
在一维情况下，我们可以将$\frac{\partial^2}{\partial \Sx^2} f$为$f"(\Sx)$。
二阶导数告诉我们的一阶导数将如何随着输入的变化而改变。
它表示只基于梯度信息的梯度下降步骤是否会产生如我们预期的那样大的改善，因此是重要的。
我们可以认为，二阶导数是对曲率的衡量。
假设我们有一个二次函数（虽然很多实践中的函数都不是二次，但至少在局部可以很好地用二次近似）。
如果这样的函数具有零二阶导数，那就没有曲率。
也就是一条完全平坦的线，仅用梯度就可以预测它的值。
我们使用沿负梯度方向大小为$\epsilon$的下降步，当该梯度是$1$时，代价函数将下降$\epsilon$。
如果二阶导数是负的，函数曲线向下凹陷(向上凸出)，因此代价函数将下降的比$\epsilon$多。
如果二阶导数是正的，函数曲线是向上凹陷(向下凸出)，
因此代价函数将下降的比$\epsilon$少。
从\fig?可以看出不同形式的曲率如何影响基于梯度的预测值与真实的代价函数值的关系。
\begin{figure}[!htb]
\ifOpenSource
\centerline{\includegraphics{figure.pdf}}
\else
\centerline{\includegraphics{Chapter4/figures/curvature_color}}
\fi
\caption{二阶导数确定函数的曲率。
这里我们展示具有各种曲率的二次函数。
虚线表示我们仅根据梯度信息进行梯度下降后预期的代价函数值。
对于负曲率，代价函数实际上比梯度预测下降得更快。
没有曲率时，梯度正确预测下降值。
对于正曲率，函数比预期下降得更慢，并且最终会开始增加，因此太大的步骤实际上可能会无意地增加函数值。
}
\end{figure}

当我们的函数具有多维输入时，二阶导数也有很多。
可以这些导数合并成一个矩阵，称为Hessian矩阵。
Hessian矩阵$\MH(f)(\Vx)$定义为
\begin{align}
 \MH(f)(\Vx)_{i,j} = \frac{\partial^2}{\partial \Sx_i \partial  \Sx_j} f(\Vx).
\end{align}
Hessian等价于梯度的Jacobian矩阵。

<!-- % -- 83 -- -->

微分算子在任何二阶偏导连续的点处可交换，也就是它们的顺序可以互换：
\begin{align}
 \frac{\partial^2}{\partial \Sx_i \partial \Sx_j} f(\Vx) = \frac{\partial^2}{\partial \Sx_j\partial \Sx_i} f(\Vx) .
\end{align}
这意味着$H_{i,j} = H_{j,i}$， 因此Hessian矩阵在这些点上是对称的。
在深度学习背景下，我们遇到的大多数函数的Hessian几乎处处都是对称的。
因为Hessian矩阵是实对称的，我们可以将其分解成一组实特征值和特征向量的正交。
在特定方向$\Vd$上的二阶导数可以写成$\Vd^\Tsp \MH \Vd$。
当$\Vd$是$\MH$的一个特征向量时，这个方向的二阶导数就是对应的特征值。
对于其他的方向$\Vd$，方向二阶导数是所有特征值的加权平均，权重在0和1之间，且与$\Vd$夹角越小的特征向量有更大的权重。
最大特征值确定最大二阶导数，最小特征值确定最小二阶导数。

<!-- % -- 84 -- -->

我们可以通过（方向）二阶导数预期一个梯度下降步骤能表现得多好。
我们在当前点$\Vx^{(0)}$处作函数$f(\Vx)$的近似二阶泰勒级数：
\begin{align}
 f(\Vx) \approx f(\Vx^{(0)}) + (\Vx - \Vx^{(0)})^\Tsp \Vg + 
 \frac{1}{2}  (\Vx - \Vx^{(0)})^\Tsp \MH  (\Vx - \Vx^{(0)}),
\end{align}
其中$\Vg$是梯度，$\MH$是$ \Vx^{(0)}$点的Hessian。
如果我们使用学习速率$\epsilon$，那么新的点$\Vx$将会是$\Vx^{(0)}-\epsilon \Vg$。
代入上述的近似，可得
\begin{align}
 f(\Vx^{(0)} - \epsilon \Vg ) \approx f(\Vx^{(0)})  - \epsilon \Vg^\Tsp \Vg + \frac{1}{2} \epsilon^2 \Vg^\Tsp \MH  \Vg.
\end{align}
其中有3项：函数的原始值、函数斜率导致的预期改善、函数曲率导致的校正。
当这最后一项太大时，梯度下降实际上是可能向上移动的。
当$\Vg^\Tsp \MH  \Vg$为零或负时，近似的泰勒级数表明增加$\epsilon$将永远导致$f$的下降。
在实践中，泰勒级数不会在$\epsilon$大的时候也保持准确，因此在这种情况下我们必须采取更启发式的选择。
当$\Vg^\Tsp \MH  \Vg$为正时，通过计算可得，使近似泰勒级数下降最多的最优步长为
\begin{align}
 \epsilon^* = \frac{ \Vg^\Tsp \Vg}{ \Vg^\Tsp \MH  \Vg} .
\end{align}
最坏的情况下，$\Vg$与$\MH$最大特征值$\lambda_{\max}$对应的特征向量对齐，则最优步长是$\frac{1}{\lambda_{\max}}$。
我们要最小化的函数能用二次函数很好地近似的情况下，Hessian的特征值决定了学习速率的量级。

二阶导数还可以被用于确定一个临界点是否是局部极大点、局部极小点或鞍点。
回想一下，在临界点处$f'(x) = 0$。
而$f"(x) > 0$意味着$f'(x)$会随着我们移向右边而增加，移向左边而减小，也就是 $f'(x - \epsilon) < 0$ 和 $f'(x+\epsilon)>0$对足够小的$\epsilon$成立。 换句话说，当我们移向右边，斜率开始指向右边的上坡，当我们移向左边，斜率开始指向左边的上坡。
因此我们得出结论，当$f'(x) = 0$且$f"(x) > 0$时，$\Vx$是一个局部极小点。
同样，当$f'(x) = 0$且$f"(x) < 0$时，$\Vx$是一个局部极大点。
这就是所谓的二阶导数测试。
不幸的是，当$f"(x) = 0$时测试是不确定的。
在这种情况下，$\Vx$可以是一个鞍点或平坦区域的一部分。

<!-- % -- 85 -- -->

在多维情况下，我们需要检测函数的所有二阶导数。
利用Hessian的特征值分解，我们可以将二阶导数测试扩展到多维情况。
在临界点处（$\nabla_{\Vx} f(\Vx) = 0$），我们通过检测Hessian的特征值来判断该临界点是一个局部极大点、局部极小点还是鞍点。
当Hessian是正定的（所有特征值都是正的），则该临界点是局部极小点。
因为方向二阶导数在任意方向都是正的，参考单变量的二阶导数测试就能得出此结论。
同样的，当Hessian是负定的（所有特征值都是负的），这个点就是局部极大点。
在多维情况下，实际上可以找到确定该点是否为鞍点的积极迹象（某些情况下）。
如果Hessian的特征值中至少一个是正的且至少一个是负的，那么$\Vx$是$f$某个横截面的局部极大点，却是另一个横截面的局部极小点。
见\fig?中的例子。
最后，多维二阶导数测试可能像单变量版本那样是不确定的。
当所有非零特征值是同号的且至少有一个特征值是$0$时，这个检测就是不确定的。
这是因为单变量的二阶导数测试在零特征值对应的横截面上是不确定的。
\begin{figure}[!htb]
\ifOpenSource
\centerline{\includegraphics{figure.pdf}}
\else
\centerline{\includegraphics{Chapter4/figures/saddle_3d_color}}
\fi
\caption{既有正曲率又有负曲率的鞍点。
示例中的函数是$f(\Vx) = x_1^2 - x_2^2$。
函数沿$x_1$轴向上弯曲。
$x_1$轴是Hessian的一个特征向量，并且具有正特征值。
函数沿$x_2$轴向下弯曲。
该方向对应于Hessian负特征值的特征向量。
名称"鞍点"源自该处函数的鞍状形状。
这是具有鞍点函数的典型示例。
维度多于一个时，鞍点不一定要具有0特征值：仅需要同时具有正特征值和负特征值。
我们可以想象这样一个鞍点（具有正负特征值）在一个横截面内是局部极大点，另一个横截面内是局部极小点。
}
\end{figure}

多维情况下，单个点处每个方向上的二阶导数是不同。
Hessian的条件数衡量这些二阶导数的变化范围。
当Hessian的条件数很差时，梯度下降法也会表现得很差。
这是因为一个方向上的导数增加得很快，而在另一个方向上增加得很慢。
梯度下降不知道导数的这种变化，所以它不知道应该优先探索导数长期为负的方向。
病态条件数也导致很难选择合适的步长。
步长必须足够小，以免冲过最小而向具有较强的正曲率方向上升。
这通常意味着步长太小，以致于在其它较小曲率的方向上进展不明显。
见\fig?的例子。
\begin{figure}[!htb]
\ifOpenSource
\centerline{\includegraphics{figure.pdf}}
\else
\centerline{\includegraphics{Chapter4/figures/poor_conditioning_color}}
\fi
\caption{梯度下降无法利用包含在Hessian矩阵中的曲率信息。
这里我们使用梯度下降来最小化Hessian矩阵条件数为5的二次函数$f(\Vx)$。
这意味着最大曲率方向具有比最小曲率方向多五倍的曲率。
在这种情况下，最大曲率在$[1,1]^\top$方向上，最小曲率在$[1,-1]^\top$方向上。
红线表示梯度下降的路径。
这个非常细长的二次函数类似一个长峡谷。
梯度下降把时间浪费于在峡谷壁反复下降，因为它们是最陡峭的特征。
由于步长有点大，有超过函数底部的趋势，因此需要在下一次迭代时在对面的峡谷壁下降。
与指向该方向的特征向量对应的Hessian的大的正特征值表示该方向上的导数快速增加，因此基于Hessian的优化算法可以预测，在此情况下最陡峭方向实际上不是有前途的搜索方向。
}
\end{figure}

<!-- % -- 86 -- -->

使用Hessian矩阵的信息指导搜索可以解决这个问题。
其中最简单的方法是牛顿法。
牛顿法基于一个二阶泰勒展开来近似$\Vx^{(0)}$附近的$f(\Vx)$：
\begin{align}
 f(\Vx) \approx f(\Vx^{(0)}) + (\Vx - \Vx^{(0)})^\Tsp \nabla_{\Vx} f(\Vx^{(0)}) + 
 \frac{1}{2}  (\Vx - \Vx^{(0)})^\Tsp \MH(f)(\Vx^{(0)})  (\Vx - \Vx^{(0)}).
\end{align}
接着通过计算，我们可以得到这个函数的临界点：
 \Vx^* =  \Vx^{(0)} -  \MH(f)(\Vx^{(0)})^{-1}  \nabla_{\Vx} f(\Vx^{(0)}) .
\end{align}
当$f$是一个正定二次函数时，牛顿法只要应用一次\eqn?就能直接跳到函数的最小点。
如果$f$不是一个真正二次但能在局部近似为正定二次，牛顿法则需要多次迭代应用\eqn?。
迭代地更新近似函数和跳到近似函数的最小点可以比梯度下降更快地到达临界点。
这在接近局部极小点时是一个特别有用的性质，但是在鞍点附近是有害的。
如\eqn?所讨论的，当附近的临界点是最小点（Hessian的所有特征值都是正的）时牛顿法才适用，而梯度下降不会被吸引到鞍点(除非梯度指向鞍点)。

仅使用梯度信息的优化算法被称为\textbf{一阶优化算法}(first-order optimization algorithms)，如梯度下降。
使用Hessian矩阵的优化算法被称为\textbf{二阶最优化算法}(second-order optimization algorithms){cite?}，如牛顿法。

在本书大多数上下文中使用的优化算法适用于各种各样的函数，但几乎都没有保证。
因为在深度学习中使用的函数族是相当复杂的，所以深度学习算法往往缺乏保证。
在许多其他领域，优化的主要方法是为有限的函数族设计优化算法。

在深度学习的背景下，限制函数满足Lipschitz连续或其导数Lipschitz连续可以获得一些保证。
Lipschitz连续函数的变化速度以Lipschitz常数$\CalL$为界：
\begin{align}
 \forall \Vx,~\forall \Vy, ~| f(\Vx) - f(\Vy)|  \leq \CalL \| \Vx - \Vy \|_2 .
\end{align}
这个属性允许我们量化我们的假设——梯度下降等算法导致的输入的微小变化将使输出只产生微小变化，因此是很有用的。
Lipschitz连续性也是相当弱的约束，并且深度学习中很多优化问题经过相对较小的修改后就能变得Lipschitz连续。

<!-- % -- 88 -- -->

最成功的特定优化领域或许是凸优化。
凸优化通过更强的限制提供更多的保证。
凸优化算法只对凸函数适用——即Hessian处处半正定的函数。
因为这些函数没有鞍点而且其所有局部极小点必然是全局最小点，所以表现很好。
然而，深度学习中的大多数问题都难以表示成凸优化的形式。
凸优化仅用作的一些深度学习算法的子程序。
凸优化中的分析思路对证明深度学习算法的收敛性非常有用，然而一般来说，深度学习背景下的凸优化的重要性大大减少。 
有关凸优化的详细信息，见{Boyd04}或{rockafellar1997convex}。



# 约束优化

有时候，在$\Vx$的所有可能值下最大化或最小化一个函数$f(x)$不是我们所希望的。
相反，我们可能希望在$\Vx$的某些集合$\SetS$中找$f(\Vx)$的最大值或最小值。
这被称为约束优化。
在约束优化术语中，集合$\SetS$内的点$\Vx$被称为可行点。

我们常常希望找到在某种意义上小的解。
针对这种情况下的常见方法是强加一个范数约束，如$\| \Vx \| \leq 1$。

约束优化的一个简单方法是将约束考虑在内后简单地对梯度下降进行修改。
如果我们使用一个小的恒定步长$\epsilon$，我们可以先取梯度下降的单步结果，然后将结果投影回$\SetS$。
如果我们使用线搜索，我们只能在步长为$\epsilon$范围内搜索可行的新$\Vx$点，或者我们可以将线上的每个点投影到约束区域。
如果可能的话，在梯度下降或线搜索前将梯度投影到可行域的切空间会更高效{cite?}。

一个更复杂的方法是设计一个不同的、无约束的优化问题，其解可以转化成原始约束优化问题的解。
例如，我们要在$\Vx \in \SetR^2$中最小化$f(\Vx)$，其中$\Vx$约束为具有单位$L^2$范数。
我们可以关于$\theta$最小化$g(\theta) = f([\cos \theta, \sin \theta]^\Tsp)$，最后返回$[\cos \theta, \sin \theta]$作为原问题的解。
这种方法需要创造性；优化问题之间的转换必须专门根据我们遇到的每一种情况进行设计。

<!-- % -- 89 --  -->

Karush–Kuhn–Tucker方法\footnote{KKT方法是\textbf{Lagrange乘子法}（只允许等式约束）的推广}是针对约束优化非常通用的解决方案。
为介绍KKT方法，我们引入一个称为广义Lagrangian或广义Lagrange函数的新函数。

为了定义Lagrangian，我们先要通过等式和不等式的形式描述$\SetS$。 
我们希望通过$m$个函数$g^{(i)}$和$n$个函数$h^{(j)}$描述$\SetS$，那么$\SetS$可以表示为$\SetS = \{ \Vx \mid \forall i, g^{(i)}(\Vx) = 0 ~\text{and}~ \forall j, h^{(j)}(\Vx) \leq 0  \}$。
其中涉及$g^{(i)}$的等式称为等式约束，涉及$h^{(j)}$的不等式称为不等式约束。

我们为每个约束引入新的变量$\lambda_i$和$\alpha_j$，这些新变量被称为KKT乘子。广义Lagrangian可以如下定义：
\begin{align}
 L(\Vx, \Vlambda, \Valpha) = f(\Vx) + \sum_i \lambda_i g^{(i)}(\Vx)  + \sum_j \alpha_j h^{(j)}(\Vx).
\end{align}

现在，我们可以通过优化无约束的广义Lagrangian解决约束最小化问题。
只要存在至少一个可行点且$f(\Vx)$不允许取$\infty$，那么
\begin{align}
 \underset{\Vx}{\min}~  \underset{\Vlambda}{\max}~
 \underset{\Valpha, \Valpha \geq 0}{\max}   L(\Vx, \Vlambda, \Valpha) 
\end{align}
与如下函数有相同的最优目标函数值和最优点集$\Vx$
\begin{align}
 \underset{\Vx \in \SetS}{\min}~ f(\Vx).
\end{align}
这是因为当约束满足时，
\begin{align}
  \underset{\Vlambda}{\max}~
 \underset{\Valpha, \Valpha \geq 0}{\max}   L(\Vx, \Vlambda, \Valpha)  = f(\Vx) ,
\end{align}
而违反任意约束时，
\begin{align}
  \underset{\Vlambda}{\max}  
 \underset{\Valpha, \Valpha \geq 0}{\max}   L(\Vx, \Vlambda, \Valpha)  = \infty .
\end{align}
这些性质保证不可行点不会是最佳的，并且可行点范围内的最优点不变。

<!-- % -- 90 -- -->

要解决约束最大化问题，我们可以构造$-f(\Vx)$的广义Lagrange函数，从而导致以下优化问题：
\begin{align}
 \underset{\Vx}{\min}~ \underset{\Vlambda}{\max}  ~
 \underset{\Valpha, \Valpha \geq 0}{\max} 
  -f(\Vx) + \sum_i \lambda_i g^{(i)}(\Vx)  + \sum_j \alpha_j h^{(j)}(\Vx).
\end{align}
我们也可将其转换为在外层最大化的一个问题：
\begin{align}
 \underset{\Vx}{\max}~ \underset{\Vlambda}{\min}~
 \underset{\Valpha, \Valpha \geq 0}{\min} 
  f(\Vx) + \sum_i \lambda_i g^{(i)}(\Vx) - \sum_j \alpha_j h^{(j)}(\Vx).
\end{align}
等式约束对应项的符号并不重要；因为优化可以自由选择每个$\lambda_i$的符号，我们可以随意将其定义为加法或减法。

不等式约束特别有趣。
如果$h^{(i)}(\Vx^*)= 0$，我们就说说这个约束$h^{(i)}(\Vx)$是\textbf{活跃}(active)的。
如果约束不是活跃的，则有该约束的问题的解与去掉该约束的问题的解至少存在一个相同的局部解。
一个不活跃约束有可能排除其他解。
例如，整个区域（代价相等的宽平区域）都是全局最优点的的凸问题可能因约束消去其中的某个子区域，或在非凸问题的情况下，收敛时不活跃的约束可能排除了较好的局部驻点。
然而，无论不活跃的约束是否被包括在内，收敛时找到的点仍然是一个驻点。
因为一个不活跃的约束$h^{(i)}$必有负值，那么$
 \underset{\Vx}{\min}~  \underset{\Vlambda}{\max}~
 \underset{\Valpha, \Valpha \geq 0}{\max}   L(\Vx, \Vlambda, \Valpha) 
$中的$\alpha_i = 0$。
因此，我们可以观察到在该解中$\Valpha \odot \Vh(\Vx) = 0$。
换句话说，对于所有的$i$， $\alpha_i \geq 0$或$ h^{(j)}(\Vx) \leq 0$在收敛时必有一个是活跃的。
为了获得关于这个想法的一些直观解释，我们可以说这个解是由不等式强加的边界，我们必须通过对应的KKT乘子影响$\Vx$的解，或者不等式对解没有影响，我们则归零KKT乘子。

可以使用一组简单性质描述约束优化问题的最优点。
这些性质称为Karush–Kuhn–Tucker条件{cite?}。
这些是确定一个点是最优点的必要条件，但不一定是充分条件。
这些条件是：

+ 广义Lagrangian的梯度为零。
+ 所有关于$\Vx$和KKT乘子的约束都满足。
+ 不等式约束显示的"互补松弛性"：$\Valpha \odot \Vh(\Vx) = 0$。

有关KKT方法的详细信息，请参阅{NumOptBook}。

<!-- % -- 91 -- -->


# 实例：线性最小二乘

假设我们希望找到最小化下式的$\Vx$值
\begin{align}
 f(\Vx) = \frac{1}{2}\| \MA \Vx - \Vb \|_2^2 .
\end{align}
专门线性代数算法能够高效地解决这个问题；但是，我们也可以探索如何使用基于梯度的优化来解决这个问题，这可以作为这些技术是如何工作的一个简单例子。

首先，我们计算梯度：
\begin{align}
 \nabla_{\Vx} f(\Vx) = \MA^\Tsp (\MA \Vx - \Vb) = \MA^\Tsp \MA \Vx - \MA^\Tsp \Vb .
\end{align}

然后，我们可以采用小的步长，按照这个梯度下降。见\alg?中的详细信息。

\begin{algorithm}[ht]
\caption{从任意点$\Vx$开始，使用梯度下降关于$\Vx$最小化
$ f(\Vx) = \frac{1}{2} || \MA \Vx - \Vb ||_2^2$的算法。
}
\begin{algorithmic}
\STATE 将步长 ($\epsilon$) 和容差 ($\delta$)设为小的正数。
\WHILE{ $ || \MA^\top \MA \Vx - \MA^\top \Vb ||_2 > \delta $}
\STATE $\Vx \leftarrow \Vx - \epsilon \left( \MA^\top \MA \Vx - \MA^\top \Vb \right)$
\ENDWHILE
\end{algorithmic}
\end{algorithm}


我们也可以使用牛顿法解决这个问题。
因为在这个情况下真正的函数是二次的，牛顿法所用的二次近似是精确的，该算法会在一步后收敛到全局最小点。

现在假设我们希望最小化同样的函数，但受 $\Vx^\Tsp \Vx \leq 1$ 的约束。 
要做到这一点，我们引入Lagrangian
\begin{align}
 L(\Vx, \lambda) = f(\Vx) + \lambda (\Vx^\Tsp \Vx - 1).
\end{align}
现在，我们解决以下问题
\begin{align}
  \underset{\Vx}{\min}~
 \underset{\lambda, \lambda \geq 0}{\max}~ L(\Vx, \lambda) .
\end{align}

<!-- % -- 92 -- -->

可以用Moore-Penrose伪逆：$\Vx = \MA^+ \Vb$找到无约束最小二乘问题的最小范数解。
如果这一点是可行，那么这也是约束问题的解。
否则，我们必须找到约束是活跃的解。
关于$\Vx$对Lagrangian微分，我们得到方程
\begin{align}
 \MA^\Tsp \MA \Vx - \MA^\Tsp \Vb + 2 \lambda \Vx = 0.
\end{align}
这就告诉我们，该解的形式将会是
\begin{align}
\Vx =  (\MA^\Tsp \MA + 2 \lambda \MI )^{-1} \MA^\Tsp \Vb.
\end{align}
$\lambda$的选择必须使结果服从约束。
我们可以关于$\lambda$进行梯度上升找到这个值。
为了做到这一点，观察
\begin{align}
 \frac{\partial}{\partial \lambda} L(\Vx, \lambda)  = \Vx^\Tsp \Vx - 1.
\end{align}
当$\Vx$的范数超过1时，该导数是正的，所以为了跟随导数上坡并相对$\lambda$增加Lagrangian，我们需要增加$\lambda$。
因为$\Vx^\Tsp \Vx$的惩罚系数增加了，求解关于$\Vx$的线性方程现在将得到具有较小范数的解。
求解线性方程和调整$\lambda$的过程一直持续到$\Vx$具有正确的范数并且关于$\lambda$的导数是$0$。

本章总结了开发机器学习算法所需的数学基础。
现在，我们已经准备好建立和分析一些成熟学习系统。

<!-- % -- 93 -- -->

