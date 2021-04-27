# loam
## 1.loadm简介
### 1.1 loam组成
由四个部分组成。特征点提取、高频低精度odom(laserOdometry)、低频高精度odom(laserMapping)、双频odom融合功能。
### 1.2特征点提取
分为coner和surface点,通过曲率来区分。  
**曲率计算**:选取一个scan上的点（最前和最后五个除外)计算其与前后五个点的曲率大小进行判断（差值的平方和）  
### 1.3 laserMapping阶段
通过LiDAR在map坐标系中的位姿T,将LiDAR坐标系下的特征点转到map坐标系下.针对map坐标系下的每个特征点,寻找与之接近的线或者面(corner对应线,surface对应面),然后计算点与线/面的距离,这个距离就是该点对应的loss.然后loss对位姿T求雅可比,使用高斯牛顿的方法,迭代优化T减小loss直到收敛。  
Paper与代码的差别:  
paper中使用LM方法进行优化,而代码中使用的是高斯牛顿法则.  
paper中使用的是angle-axis进行优化,而代码中使用的是旋转矩阵直接对欧拉角求导,最终优化的是欧拉角.
