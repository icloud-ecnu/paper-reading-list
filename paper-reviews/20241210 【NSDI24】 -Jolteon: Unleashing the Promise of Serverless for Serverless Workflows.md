by YikunGu

### Jolteon: Unleashing the Promise of Serverless for Serverless Workflows

![image-20241129132657601](https://raw.githubusercontent.com/wty403/pic/main/picture/image-20241129132657601.png)

![Jolteon](https://archives.bulbagarden.net/media/upload/thumb/e/e3/0135Jolteon.png/250px-0135Jolteon.png)

#### Introduction

开发者对serverless的要求：latency，cost。

增加配置的资源能减少latency同时增加cost，AWS Lambda提供Power Tuning分析特定函数的延迟成本曲线，并基于延迟成本曲线优化一个函数的配置；但Power Tuning不适用于serverless workflow。

Orion（OSDI'22）使用blackbox model来获得serverless workflow的近似延迟成本曲线，黑盒方法的性质导致模型不准确，启发式方法发现的资源配置是次优的。

Ditto（SIGCOMM'23）使开发人员能够优化工作流的延迟或成本。它只提供了两个极端（最小延迟或最小成本），不允许开发人员探索延迟和成本之间的其他权衡。延迟减少7%可能会使成本降低1.8倍，这可能比延迟最小的配置更可取。 

我们提出Jolteon，这是一个工作流编排器，它提供自动资源配置，以满足服务器无感知应用程序的应用程序级要求。开发人员只需要为工作流指定延迟或成本限制。Jolteon会自动配置资源，以最小化成本限制的执行延迟或最小化延迟限制的成本。通过这样做，Jolteon为工作流编排提供了服务器无感知体验，

##### Challenges

构建一个准确的性能模型，以捕捉资源配置和应用程序需求之间的复杂关系。
Blackbox模型捕捉到了服务器无感知计算的性能变化，但缺乏可解释性，忽略了服务器无感知工作流的执行特性。白盒模型忽略了服务器无感知计算的固有可变性，无法保证性能界限。
Jolteon建立了一个随机性能模型。该模型使用分析公式来捕捉工作流执行的执行特征，这使得模型更加准确和高效（即白盒建模的好处）。该模型使用随机函数来捕捉性能变化，可用于限制性能（即黑盒建模的好处）。这种方法利用了白盒和黑盒模型的优点，并避免了它们的缺点。

给定随机性能模型，下一个挑战是在延迟或成本限制下找到最佳资源配置。当我们使用随机变量来模拟执行可变性时，该问题在数学上被表述为概率约束优化问题，这是一种随机优化问题。为了解决这个问题，我们首先通过蒙特卡洛采样将其转换为确定性公式，并通过一个新的不等式来保证性能界限。在确定性公式下枚举所有可能的配置是不切实际的，因为搜索空间很大，约束条件也很多。我们证明了我们的公式是凸的。因此，我们利用凸性并使用高效的梯度下降算法来找到边界下的最优资源配置。

#### Motivation

whitebox：利用白盒方法通过分析方程确定地预测函数执行时间。然而，该模型忽略了服务器无感知计算中固有的性能可变性，因此无法保证性能界限。

blackbox：采用分布感知性能模型来捕捉这种性能变化。具体来说，他们首先收集函数执行轨迹作为采样数据，然后拟合一个黑盒模型来覆盖函数执行过程。尽管这些技术捕捉到了性能的可变性，但它们忽略了底层的服务器无感知计算特性，导致预测不准确和耗时的黑盒拟合。

服务器无感知计算的特点是指函数执行的逐步分解，即初始化、传输和计算。我们可以利用每个步骤的特点来构建更准确、更高效的性能模型，例如，传输步骤的时间受到可用网络带宽的影响。

然而，服务器无感知工服务器无感知据依赖关系的函数组成。这些工作不足以支持服务器无感知工作流的优化。

##### Challenges

我们的目标是提供自动资源配置以满足应用程序级要求。具体而言，系统应自动配置资源，以最小化用户指定的成本限制的执行延迟，或最小化用户指定延迟限制的成本。为了实现这一目标，必须应对两个主要挑战。

**第一个挑战**是构建一个性能模型，以准确捕捉资源配置和应用程序需求（例如延迟和成本）之间的复杂关系。

whitebox：白盒模型成功地捕获了服务器无感知计算的底层逻辑，因此可以快速准确地预测平均性能。然而，它们的确定性无法捕捉到性能的可变性。

blackbox：黑盒模型侧重于性能变化，但忽略了潜在的系统特征，导致预测不准确和耗时的黑盒拟合。

希望开发一种性能模型，利用白盒和黑盒的优点，同时减轻它们的缺点。

**第二个挑战**是在给定的性能范围内确定最佳资源配置。

由于性能模型包括分布因素，优化问题变得随机，无法直接求解。即使问题是确定性的，资源配置的巨大搜索空间和问题的复杂表述也很难找到最优解。简而言之，第二个挑战在于制定具有性能保证的随机优化问题，并有效地解决问题以找到最优结果。

#### Jolteon Overview

![image-20241129143230601](https://raw.githubusercontent.com/wty403/pic/main/picture/image-20241129143230601.png)

Jolteon是一个服务器无感知工作流编排器，它有助于自动配置资源以满足应用程序级要求（即延迟或成本）。它采用了一种新的随机性能模型来准确地分析服务器无感知计算中的资源-性能关系。它阐述了概率约束优化问题。然后，它将其转换为蒙特卡洛采样的确定性问题，并通过一个新的不等式来保证性能界限。它采用梯度下降算法，利用问题的凸性来实现最优结果。图3显示了概览。

##### User interface

用户将他们的服务器无感知工作流DAG和需求（延迟或成本界限）定义为Jolteon的输入。

##### Jolteon orchestrator

###### **Performance profiler**：

注册工作流后，Jolteon的性能分析器会定期轮询相应工作流的数据日志。它学习并更新由随机函数建模的每个工作流阶段的性能模型。随机函数能够利用白盒和黑盒模型的优点，同时减轻它们的缺点。具体而言，性能模型将分析公式中的每个参数视为随机变量，并将其拟合为分布。

###### **Bound guaranteed sampler**：

Jolteon基于工作流信息和用户指定的边界，提出了概率约束优化问题。在公式中，资源配置被视为自变量。由于概率约束优化不能直接求解，该模块通过蒙特卡洛采样将其转化为确定性问题。采样次数越多，置信度越高，但公式越复杂，求解时间越长。Jolteon提出了一种新的不等式来确定保证高置信度性能界限的最小样本量。

###### **Convex optimizer**：

采样后，概率约束优化问题转化为确定性优化问题。用现有算法解决这样的问题既费时又次优。Jolteon利用了一个关键的见解：问题是凸的。因此，Jolteon采用梯度下降算法来有效地找到最优资源配置。

##### Serverless runtime

服务器无感知运行时从编排器接收服务器无感知工作流和最佳资源配置。然后，它执行（即通过函数调用）工作流并记录执行日志。分析器使用日志来构建性能模型。执行结果将返回给用户。

#### Jolteon Design

![image-20241129151410055](https://raw.githubusercontent.com/wty403/pic/main/picture/image-20241129151410055.png)

##### Performance Profiler

使用一个随机性能模型。

该模型首先将每个工作流阶段（即DAG的每个节点）划分为函数和阶段。一个阶段由许多并行函数实例组成。阶段的延迟是所有函数实例的最大延迟。

每个函数实例分为两个阶段：初始化和执行。

###### Initialization phase

分两步，第一部从API gateway收到请求开始，受网络延迟、请求验证等因素有关，此过程的延迟是不可预测的，由整体系统负载决定。用随机变量$D$表示。

![image-20241129150617494](https://raw.githubusercontent.com/wty403/pic/main/picture/image-20241129150617494.png)

第二步是初始化环境，如果是冷启动，拉容器镜像、初始化运行时。热启动，延迟可忽略。

冷启动时间为$C$
$$
G=\left\{
\begin{aligned}
0,&Warm\ start\\
C,&Cold\ start.
\end{aligned}
\right.
$$

###### Execution phase

分为两步，数据传输和计算。

传输：数据大小$S$（随机变量），$d$个并行执行的函数，带宽$b$，vCPU为$v$。

传播时间为$\cfrac{S}{d\times b}$，但$b$受$v$影响，$b=\min(v\times W,B)$，其中$W$为每个vCPU的带宽，$B$为最大带宽。API调用的overhead为$O_T$。

则传输时间为
$$
T(d,v)=\cfrac{S}{d\times\min(v\times W,B)}+O_T
$$
计算：

计算时间随计算逻辑而变化，关注两类函数逻辑：多项式时间复杂度和对数时间复杂度。

计算时间为：
$$
C(d,v)=\sum\limits_{i=0}^lA_i\times(\cfrac S{dv})^i+\ln\cfrac S{dv}\times\sum\limits_{i=0}^m B_i\times(\cfrac S{dv})^i.
$$
由于$v$和$d$是离散且有限的，其他的复杂度也能被多项式复杂度逼近，即拉格朗日插值多项式。

###### Function model to stage model

对于整个stage，其延迟为
$$
LS(d,v)=\max\{D_j+G_j+T_j(d,v)+C_j(d,v),j=1,...,d\}
$$
**每个函数vCPU相同？**

对于cost，由于平台不对初始化时间收费，$\alpha$是每vCPU每秒费用，$\beta$为每个调用价格。
$$
CS(d,v)=\sum\limits_{j=1}^d((T_j(d,v)+C_j(d,v))\times v\times\alpha+\beta).
$$


###### Stochastic model

对于上面的建模，将一些参数$C,W,A_i$视为静态参数无法捕获性能的可变性，相反，我们将这些参数建模为随机变量，将确定性函数转换为随机函数。这些随机变量被拟合为分布，并标记为大写字母。

![image-20241129154333278](https://raw.githubusercontent.com/wty403/pic/main/picture/image-20241129154333278.png)

与黑盒模型相比，这种随机模型通过简单而有力的公式捕捉了潜在的系统特征，这允许更高的精度和更低的拟合开销。简而言之，这种随机模型不仅捕捉到了可变性，还捕捉到了潜在的系统特征，从而利用了白盒和黑盒模型的优势。

##### Bound Guaranteed Sampler

###### Formulation of chance constrained optimization

整个cost
$$
CW(\mathbf d,\mathbf v)=\sum\limits_{i=1}^p CS_i(d_i,v_i).
$$
概率约束优化问题：
$$
Minimize\ \ CW(\mathbf d, \mathbf v).\\
s.t.\ \ Confidence(LW(\mathbf d, \mathbf v)\leq\epsilon_l)\geq\delta.
$$
在这个公式中，某些参数是随机变量，约束要求以预定的置信水平$\delta$保证延迟界限$\epsilon_l$。在成本限制下最小化延迟的公式是相似的。

###### Bound guaranteed sampling

将这种方法应用于我们的具体问题会带来两个挑战：

第一个挑战是目标函数和约束函数包含随机变量。

Jolteon采用蒙特卡洛抽样将概率约束转化为一组确定性约束。
$$
G(\mathbf d,\mathbf v,\mathbf X)=LW(\mathbf d,\mathbf v)-\epsilon_l
$$
$\mathbf X$代表随机变量向量。取样本后，原始随即约束变为确定约束
$$
\{G(\mathbf d,\mathbf v,\mathbf x_i)\leq0,i=1,...n\}.
$$


第二个挑战是确定样本数量，以确保高置信度的性能界限。

![image-20241129160835071](https://raw.githubusercontent.com/wty403/pic/main/picture/image-20241129160835071.png)

样本越多，解的置信度就越高。但大样本量引入了高采样开销和复杂的问题公式，求解时间长。

Jolteon利用Hoeffding不等式和样本近似理论来找到样本大小$n$的下限。这个下限使Jolteon能够保证置信水平$\delta$的性能界限，同时最小化采样开销并简化问题公式。样本量的下限如下：
$$
\cfrac{1}{2\times(1-percentile)^2}\log(\cfrac{|D|}{1-\delta}).
$$
其中$D$为独立变量$\mathbf d$和$\mathbf v$的定义域，$\delta$为置信度，设为$99.9\%$。

$percentile$为目标界限百分比，如P95延迟界限。

##### Convex Optimizer

Jolteon利用了一个重要的见解：优化问题是凸的。

###### Optimization algorithm

![image-20241129164328536](https://raw.githubusercontent.com/wty403/pic/main/picture/image-20241129164328536.png)

第一个过程是梯度下降算法。

第二个过程是探测过程。由于现实世界中资源配置的离散性，连续局部极值可能不可行。为了解决这个问题，探测过程迭代地检查这个极值周围的可行点，以找到最终的配置。假设问题是凸的，局部极值也是全局最优的。因此，优化算法能够在给定的性能边界下识别最优解。

过多的样本约束使优化过程复杂化，并加剧了约束检查开销。为了减轻这种情况，我们采用支持约束技术来修剪冗余约束。

#### Evaluation

目标：

(i) overall performance against state-of-the-art solutions (§ 6.1); 

(ii) performance guarantees provided by Jolteon (§ 6.2); 

(iii) effectiveness of the performance model (§ 6.3); 

(iv) time analysis for running Jolteon (§ 6.4).

##### Setup

Jolteon编排器部署在一个c5.12xlarge EC2实例上，该实例具有48个vCPU和96 GB内存。Jolteon在AWS Lambda上执行服务器无感知工作流应用程序，并使用AWS S3作为外部存储。

##### Workload

- ML Pipeline：包含四部分，PCA、training，merging，testing，随机森林
- Video Analytics：包含四个stage，视频分割、帧提取、预处理和，帧分类，YOLO
- TPC-DS Query：八个stage

![image-20241202102331078](https://raw.githubusercontent.com/wty403/pic/main/picture/image-20241202102331078.png)

##### Baselines

- Caerus（NSDI'2021）：使用基于输入数据大小的启发式比例资源分配策略来优化工作流的延迟和成本。
- Ditto（SIGCOMM'2023）：利用白盒建模为工作流执行提供最小延迟或最小成本，我们分别称之为Ditto-L和Ditto-C。
- Orion（OSDI'2022）：对函数实例大小采用黑盒建模，并使用确定性数量的并行函数。然后，它使用具有黑盒模型的启发式搜索算法来最小化不同延迟要求下的成本。

##### Overall Performance

![image-20241202101829110](https://raw.githubusercontent.com/wty403/pic/main/picture/image-20241202101829110.png)

- Jolteon在延迟和成本方面都优于Caerus。这是因为Caerus的性能模型只考虑输入数据大小，而不考虑服务器无感知函数的固有逻辑。因此，它无法准确捕捉性能，因此资源配置不是最优的。
- Ditto-L和Ditto-C分别实现了最小的延迟和成本。Ditto不支持在帕累托最优延迟成本曲线上进行权衡。
- Orion近似于延迟成本曲线，能够满足不同的性能要求。但它使用启发式搜索，只要一个配置满足性能要求，它就会返回，并且只调整函数大小。

##### Performance Guarantees

![image-20241202102425171](https://raw.githubusercontent.com/wty403/pic/main/picture/image-20241202102425171.png)

###### Latency guarantee

为了评估Jolteon的延迟保证，我们设置了不同的延迟界限并测量了延迟。图10显示了三个工作流的实际95%延迟和理想延迟。测量的实际95%延迟始终小于并接近理想线，这证明了Jolteon提供延迟保证的能力。当我们增加延迟界限时，实际的95%延迟单调增加，表明Jolteon可以适应不同的延迟界限。当绑定松动时（例如，图10（b）中的视频分析>80秒），实际延迟不会增加，因为资源配置达到了允许执行的极限。



![image-20241202102449797](https://raw.githubusercontent.com/wty403/pic/main/picture/image-20241202102449797.png)

###### Cost guarantee

我们还通过设定不同的成本界限来评估Jolteon的成本保证。图11显示了三个工作流的实际和理想95%成本。与延迟保证类似，Jolteon能够为所有工作流提供有限的成本。当成本界限相对宽松时（例如，图11（b）中每次运行>0.02美元），实际成本几乎不变。这是因为优化目标是延迟。增加资源并不能进一步减少延迟。在这种情况下，Jolteon避免了不必要的资源分配。

##### Effectiveness of the Performance Model

![image-20241202102803588](https://raw.githubusercontent.com/wty403/pic/main/picture/image-20241202102803588.png)

###### Step-level effectiveness

关键执行步骤的性能模型。

这三种工作流程表现出不同的特点。TPCDS查询是一个I/O密集型工作流，其中关键步骤是从S3读取数据。由于S3为读取提供了稳定的带宽，在相同的vCPU分配下，步骤的执行时间在图12（a）中保持稳定，并且可以通过性能模式精确预测，误差小于1%。视频分析和ML管道是计算密集型的，其中关键步骤分别是视频分割和模型训练阶段的计算步骤。由于服务器无感知计算的性能可变性，这些步骤的执行时间变化更大，如图12（b）和图12（c）所示。随机性能模型预测了覆盖大部分实际执行时间的执行时间分布。

###### Stage-level effectiveness

![image-20241202103112025](https://raw.githubusercontent.com/wty403/pic/main/picture/image-20241202103112025.png)

然后，我们使用大量、中型和少量资源来检验阶段级绩效模型的有效性。表3列出了视频分析中视频分割阶段（即关键阶段）的预测误差。d表示函数实例的数量，v表示每个函数的vCPU数量。误差范围为[-6.07%，6.87%]。P50和P95时间预测的平均绝对百分比误差（MAPE）分别为6.25%和3.42%，表明Jolteon是准确的，能够捕捉到该阶段的性能变化。

###### Workflow-level effectiveness

![image-20241202103206866](https://raw.githubusercontent.com/wty403/pic/main/picture/image-20241202103206866.png)

![image-20241202103223507](https://raw.githubusercontent.com/wty403/pic/main/picture/image-20241202103223507.png)

最后，我们评估了Jolteon在预测整个工作流程的端到端延迟分布时的误差，并将其与Orion的黑盒和Ditto的白盒模型进行了比较。我们通过不同数量的函数实例（即$\mathbf d$）和每个函数不同数量的vCPU（即$\mathbf v$）来改变资源配置。vCPU的总数是向量$\mathbf d$和$\mathbf v$的点积。详细配置如表4所示。如视频分析表5所示，Jolteon的P50和P95延迟预测误差范围为[-7.59%，9.29%]，MAPE低于4%。这两条基线的表现要差得多。Orion的P50和P95延迟预测的MAPE都超过了30%，因为它的线性插值在大配置空间下是不准确的。Ditto的P50和P95延迟预测的MAPE分别为33.98%和28.00%，这是由于其分析模型无法适应性能变化。

##### Time analysis for Jolteon

###### Performance model training time

![image-20241202103418230](https://raw.githubusercontent.com/wty403/pic/main/picture/image-20241202103418230.png)

我们评估离线时间来训练性能模型。对于每个工作流，Jolteon收集了数十个具有不同资源配置的执行配置文件。然后，它使用非线性最小二乘法来拟合性能模型中随机变量（参数）的分布。表6报告了三个工作流程的训练时间。训练时间小于70毫秒，与数十秒的端到端延迟相比可以忽略不计。

###### Solving time

![image-20241202103431234](https://raw.githubusercontent.com/wty403/pic/main/picture/image-20241202103431234.png)

我们在凸优化器中测量了Jolteon梯度下降算法的时间。我们的样本量从10到10000不等。图13显示了结果。在延迟限制的情况下，当样本量小于5000时，算法在0.5秒内完成。当指定了成本界限时，算法将在0.05秒内完成。延迟限制下的较高解决时间是由于我们实现的具体情况。以SLSQP为算法，约束函数平滑且可微。延迟约束中存在最大运算符违反了此条件。为了解决这个问题，我们将整个工作流的单一延迟约束替换为每个DAG路径的单独延迟约束。它消除了最大运算符的使用，但增加了约束的数量。

更复杂的工作流程会导致更长的求解时间。在这三个工作流中，TPC-DS Query具有最多的阶段和最复杂的数据依赖关系（例如，八个阶段）。我们在TPC-DS查询上的实验涉及4048个采样约束。如图13（a）所示，Jolteon可以在0.5秒内解决问题。根据Azure Durable Functions的特性，95%的服务器无感知工作流的阶段少于8个，这表明Jolteon为大多数用例提供了亚秒级的解决时间。
