# 笔记 navmesh 

[TOC]

<br />

# 1 简介
RecastNavigation 是一个导航寻路工具集，Unity和UE4就是用的这个库，开发者叫 Mikko Mononen。  
它包括Recast和Detour两个子集。  

## 1.1 Recast
Recast 是一个网格导航工具集，一般是用在游戏里。  
优势：  

* 自动化的。 用的时候就是把任意层次的几何体扔进去处理，然后会给你一堆网格信息。  
* 处理比较快。  
* 开源的，可以自己改代码定制。  

处理过程大概分为三个步骤：  

* 体素化模型  
    * 用输入的三角形网格体构建出多层的高度场。  
        * 先体素化，用源几何体构造出实心的高度场，表示障碍空间，包括地面、墙体等；  
        * 然后将实心高度场的上表面中连续的区间合并为联通区域，构造出紧凑高度场，表示可行走的开放空间。  
    * 这个时候可以做一些简单的筛选，去除掉角色不可到达的位置，比如空气墙外面、空间太小不能容纳角色的地方、大石头里面。  
* 分割联通区域  
    把角色可行走的区域划分成简单的二维叠加区域，会得到一个不重叠的轮廓线，这大大简化了最后一步。  
* 将联通区域简化成多边形  
    通过先追踪区域轮廓，再对轮廓进行简化，使导航多边形从区域中剥离出来。最终得到的多边形被转化成了凸多边形，非常适合用于寻路和层次空间推理。

## 1.2 Detour
Detour 是一个寻路和空间推理工具包，recast必须要跟Detour结合起来用。 任何导航网格都可以和Detour结合用，但recast生成的是最切合的。  
Detour会提供一个适用于许多简单情况的简单静态导航网格，以及允许添加和删除片段的分块导航网格。  
分块的网格，可以随着玩家在关卡中前进，输入和输出新的导航数据，或者随着地图的变化重新生成tiles。  


<br />

# 2 Recast Demo
在源码的RecastDemo目录下，a kitchen sink demo，包含了这个库所有功能。 用来学习这个库的，而且可以编译出一个可执行文件，可以用来测试验证。  
官方建议，刚开始学的话，先去看**Sample_SoloMesh.cpp**这个文件，可以搞清楚编译导航网格的步骤；看**NavMeshTesterTool.cpp**这个文件，可以知道Detour是怎么寻路的。  

## 2.1 windows下编译Recast Demo
* 下载[premake5](https://premake.github.io/)(需要用它生成vs的编译项目)，解压到RecastDemo目录下。  
* 下载[SDL2](https://www.libsdl.org/download-2.0.php)(recast的功能依赖SDL库)，解压到RecastDemo\Contrib目录下，改名成SDL。到RecastDemo\Contrib\SDL目录下，如果没有lib目录：  
    * 那就到VisualC目录下，vs打开SDL.sln，选择Win32、Debug，编译。  
    * 把编出来的RecastDemo\Contrib\SDL\VisualC\Win32\Debug 下的文件复制到RecastDemo\Contrib\SDL\lib\x86  
    * 总之RecastDemo\Contrib\SDL\lib\x86这个路径下要有库文件。  
* 在RecastDemo目录下，cmd打开命令行，输入 premake5 vs2019 (vs版本要对应自己已安装的版本)  
* vs打开生成的RecastDemo\Build\vs2019\recastnavigation.sln，编译。  
* 编译出来的RecastDemo\Bin\RecastDemo.exe 即是可执行文件。  

## 2.2 Recast Demo使用
### 2.2.1 基本使用
* 在RecastDemo\Bin\Meshes目录下有三个样例文件。  
* 在RecastDemo\Bin\TestCases目录下可以添加测试文本，编译出来的RecastDemo\Bin\Tests.exe，可以用来做单元测试。 命令行运行它会指示哪些是失败的，并统计成功的数量。  
* 运行RecastDemo.exe后，右侧有两个选项 :   
    * Sample 三种编译类型选择，对应源码里Sample的三个派生类  
        * Solo Mesh 表示纯粹的邻接凸多边形集合。 就是直接整个网格体按步骤搞。  
        * Tile Mesh 表示基于tile划分的N个邻接凸多边形集合。 就是先分块然后再按步骤搞，可以很方便地**动态改变地形**。  
        * Temp Obstacles 表示支持动态障碍物。 也是基于tile划分的N个邻接凸多边形集合，但它可以动态添加删除障碍物，游戏里应该都是用这种。  
    * Input Mesh 就是输入进去处理的原始网格体，可以选三个样例文件试试。  
* 刚开始用，选Solo Mesh和nav_test.obj，然后点Build。  
* WSADQE键移动视角，拖动右键旋转。 点右键选开始点，点左键选结束点，就可以看到路径了。  

### 2.2.2 Properties属性参数 
```
rasterization 体素化 :
    Cell Size : 体素在X-Z-plane上的大小。 决定了网格的精度。 越小越精细，但时间和内存消耗相应也越大。 
               核心设置，会影响到其他参数。  
    Cell Height : 体素在Y-plane上的大小。 值较小并不会对内存消耗产生显著影响；
                 较低的值虽然能够使网格贴合原始图形，但是如果是凹凸不平的地形，较低的值可能会造成邻接的网格之间产生断裂，
                 导致本来应该连在一起的网格造分离。
Agent 角色 : 
    Height : 角色高度，单位米
    Radius : 角色半径，单位米
    MaxClimb : 角色最大可跨越不同地形时的高度，比如上楼梯时，单位米
    MaxSlope : 角色最大可通过的斜坡的倾斜度，单位度
Region 区域 : 
    Min Region Size : 区域最小尺寸，单位体素
                      regionMinSize = sqrt(regionMinArea)
    Merged Region Size : 合并区域尺寸，当一个区域小于该尺寸时，如果可以，则会被合并进一些大的区域。单位体素
                         regionMergeSize = sqrt(regionMergeArea)
Partitioning 分割区域的算法 : 
    Watershed : 分水岭算法，默认是用这个
    Monotone : 单调
    Layers : 按层次
Filtering 筛选，即要剔除的选项 : 
    Low Hanging Obstacles : 低挂障碍物
    Ledge Spans : 壁架
    Walkable Low Height Spans : 高度较低的区间
Polygonization 多边形 : 
    Max Edge Length : 边最大长度，单位米
    Max Edge Error : 边最大误差，单位体素
    Verts Per Poly : 每个多边形的最大顶点数
DetailMesh 详细Mesh数据设置 : 
    Sample Distance : Detail sample 的长度，单位体素
    Max Sample Error : Detail sample 的最大误差，单位体素高度
Keep Itermediate Results : 是否保持中间结果
Build All Tiles : 编译所有的tiles
Tiling : 
    Tile Size : tile边长，单位体素
    Tile Cache : tile缓存
Draw : 显示方式，很有用。 从原始几何体到导航网格，可以看到各个阶段的结果。
```

[CritterAI 的一些参数](http://www.critterai.org/projects/nmgen_study/config.html)：  
参考[CritterAI插件CritterAI与Recast Navigation寻路](https://blog.csdn.net/qq_42672770/article/details/107185058)  
```
minTraversableHeight : 最低可通过高度，设定从底部边界到顶部边界之间的最低高度，该高度为模型可通过的高度. 
                       设定值的大小至少得是cellHeight的2倍
maxTraversableStep : 可跨越不同地形时的高度，设定像是从普通平面移动到楼梯这样的地形是否可通过的高度阀值. 
                     值设定必须大于cellHeight的2倍
maxTraversableSlope : 最大可通过的斜坡的倾斜度
clipLedges : 边缘突出部分是否可以行走
traversableAreaBorderSize : 可行走区域与阻挡物之前的距离大小. 值设定必须大于cellSize的2倍
smoothingThreshold : 当产生用于表示派生区域的距离场时，会被使用该值
useConservativeExpansion : 是否应用一些算法来避免产生残缺的区域
minUnconnectedRegionSize : 最小的无法被连接的区域（这里的区域指的是在网格生成之前，某些要与其他区域连接的区域）
maxEdgeLength : 指示网格边界的多边形最大的边
edgeMaxDeviation : 网格边界与原始集合图形的偏离量
maxVertsPerPoly : >= 3, 每个多边形的最大顶点数
contourSampleDistance : 设置采样距离，类似游戏中的凹凸贴图类似概念，用于NavMesh过程中的Generate Detailed Mesh,
                        匹配原始集合图形表面的网格（利用生成更加精细的三角形保证网格来贴合那些凹凸不平的地表）.
                        当值在0.9以下时会关闭这个特性.
contourMaxDeviation : 最大的采样偏移距离，最好和contourSampleDistance结合起来看，其效果的精确度受到contourSampleDistance的影响. 
                      为0时，这个特性选项无效,在实际使用时发现在这个值越接近0，其生成的网格就越偏离原始图形. 
```

### 2.2.3 Tools
```
Test Navmesh : 可以做一些测试，甚至可以自己改源码添加接口，用来测试一些功能
    Pathfind Follow : 查找从起始多边形到终点多边形的路径
    Pathfind Straight : 查找多边形道路内从起点到终点的直线平滑路径
        Vertices at crossings : 选项，显示路径上的点
            None 
            Area 
            All 
    Pathfind Sliced : 切片
    Distance to Wall : 显示起点到最近墙壁的距离
    Raycast : 起点往终点打一条射线，遇到障碍物停下
    Find Polys in Circle : 在以起点为中心，起点到终点为半径的圆内，找出所有多边形
    Find Polys in Shape : 在长方形内，找出所有多边形
    Find Local Neighbourhood : 找出起点周围的多边形
    Set Random Start : 随机设置起点
    Set Random End : 随机设置终点
    Make Random Points : 随机选一些点
    Make Random Points Around : 在以起点为圆心的一个圆内，按权重找一些多边形，然后在这些多边形上找一些点。
                                注，这些点可能不在圆内 (这个有点坑，游戏里大概率也会用到这个接口，
                                即想在圆内随机一个点，但有可能随机出的点不在圆内)
    Include Flags : 地形筛选
    Exclude Flags 
        Walk 
        Swim 
        Door 
        Jump 

Prune Navmesh 

Create Off-Mesh Connections 

Create Convex Volumes 

Create Crowds 
```


<br />

# 3 recast navigation 源码分析
参考[Steve Pratt的文档](http://www.stevefsp.org/projects/rcndoc/prod/modules.html)，建议自己去看看，文档详细全面。  
主要三个模块：

* Recast : 创建网格数据，Detour的前置工作
* Detour : 用Recast生成的数据，创建、操作和查询导航网格
* Crowd  : 实现了局部转向和动态避障功能, 提供群体寻路行为  

### 3.1 Recast
```
构建导航网格数据有很多种方法，其中一种比较简单的做法是：
    1. 预处理输入进去的三角形网格
    2. 构建一个实心高度场rcHeightfield
    3. 构建一个紧凑高度场rcCompactHeightfield
    4. 构建一个轮廓线集合rcContourSet
    5. 构建凸多边形rcPolyMesh
    6. 构建详细数据rcPolyMeshDetail
    7. 用rcPolyMesh 和 rcPolyMeshDetail 构建出适用于Detour的导航网格tile
主要类的生命周期处理：
    1. 用Recast allocator分配对象 (如 rcAllocHeightfield)
    2. 初始化或构建对象 (如 rcCreateHeightfield)
    3. 根据需求更新对象 (如 rcRasterizeTriangles)
    4. 使用该对象
    5. 用Recast allocator释放对象 (如 rcFreeHeightField)
```

### 3.2 Detour


### 3.3 Crowd


### 3.4 Recast Demo源码
```
主要逻辑在recastnavigation\RecastDemo\Source\Sample_TileMesh.cpp Sample_TileMesh::handleBuild()  
```


<br />

# 4 如何用到自己项目里
源码的DebugUtils, Detour, DetourCrowd, DetourTileCache, Recast文件夹加到项目里去，可以都打成库，然后自己封装一下外层接口。  
关卡构建数据要用到DebugUtils, Recast, Detour。 游戏运行时只要用到Detour。  


<br />

# 5 扩展
```
RecastNavigation的所有操作都是基于地表面的，对于空中对象的交互是无法完成的，这时可以结合其他引擎，如physx进行对象的空中交互。  

```



<br />

# 参考
[recastnavigations github源码](https://github.com/recastnavigation/recastnavigation)  
[Recast Navigation文档 by Steve Pratt](http://www.stevefsp.org/projects/rcndoc/prod/index.html)  
[CritterAI 官方文档](http://www.critterai.org/projects/cainav/doc/#) (基于RecastNavigation的一个寻路插件)  
[Suppor Navmesh Generation](https://github.com/t01y/WebGL_Learning/issues/10)  
[谷阿莫带你十分钟看完 NavMesh 生成算法](https://wo1fsea.github.io/2016/08/21/A_Quick_Introduction_to_NavMesh/)  
[关于NavMesh，我所知道的](https://wo1fsea.github.io/2016/08/20/About_NavMesh/)  
[从零开始学习导航网格#2 编译RecastNavigation](https://www.jianshu.com/p/41dd36b43afb)  
[[Unity3d手游开发笔记]初识RecastNavmesh](https://zhuanlan.zhihu.com/p/138286658)  
[navmesh 生成网格信息](https://blog.csdn.net/icebergliu1234/article/details/87902528)  
[服务器3D场景建模（九）：RecastNavigation之Detour数据结构](https://blog.csdn.net/u013272009/article/details/80281642)  
[RecastNavigation-理解高度场](https://blog.csdn.net/you_lan_hai/article/details/77428891)  
[NavMesh生成原理](https://zhuanlan.zhihu.com/p/40177186)  
[RecastNavigation（3D场景建模、网格导航）](https://www.cnblogs.com/damonxu/p/9858669.html)  
[游戏寻路之平滑路径—拉绳(漏斗)](https://blog.csdn.net/romantic_jie/article/details/114012756)  


