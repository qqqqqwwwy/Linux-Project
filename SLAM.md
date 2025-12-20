## 1. slam构图，有障碍变化（家具移动），且会遇到动态障碍

**SLAM 只负责构建“长期静态地图”（墙、门、固定柜体）**  
 **导航时使用“静态地图 + 实时感知生成的动态代价地图”**  
 **椅子、桌子、人 → 不进 SLAM 地图，只进实时代价地图**

## 得到的地图应包含：

✅ 墙  
✅ 门洞  
✅ 固定柜体  
❌ 人  
❌ 椅子  
❌ 桌子

 **这是一张“长期地图”**

---

## 2. 导航阶段：动态世界由 Costmap 处理（关键）

### 2.1 ROS2 Nav2 的核心：**Costmap**

需要两个 costmap：

---

### 1.Global Costmap（全局）

- 来源：**SLAM 保存的静态地图**
- 分辨率：低一点（0.05–0.1 m）  
- 作用：
	 - 全局路径规划（A* / Dijkstra / Smac）

 **不频繁更新**

---

### 2.Local Costmap（局部，实时）
- 来源：
    - 激光雷达
    - 深度相机 

- 作用：
    - 实时避障
    - 处理椅子、人、移动桌子

 **10–20 Hz 实时刷新**
 
---
## 3. 椅子、桌子、人 “特殊处理”
### 语义分离
如果你以后要做**高级行为**，比如：
- “不要撞人，但可以贴着椅子走”
- “椅子可以推开，人不可以”

可以：
#### 视觉 + YOLO
- 人 → 标记为 **高代价 / 不可通行**
- 椅子 → 中代价
- 桌子 → 高代价

然后写入 **不同 costmap layer**

---
## 4. 首次SLAM建图 —— Frontier 探索

---
### 4.1 Frontier 的“严格定义”
在**栅格地图（Occupancy Grid）**中：
- 栅格状态通常是：
    - `FREE`（空）  
    - `OCCUPIED`（障碍）
    - `UNKNOWN`（未知）

### 4.2 一个格子是 Frontier，当且仅当：
1. 该格子是 **FREE**
2. 它的 **8 邻域中至少有一个是 UNKNOWN**

 这意味着：
- 机器人能站过去
- 站在那里就“能看到未知区域”

---

##  Frontier 探索的标准算法流程
### Step 1：从 SLAM 获取当前地图
- 订阅：
    - `/map`（nav_msgs/OccupancyGrid）
- 这是：
    - 实时更新的地图
    - 包含已知 + 未知

---

### Step 2：扫描地图，找 Frontier 点

---
### Step 3：Frontier 聚类（否则点太多）

不能对每一个格子都导航。
### 做法：
- 用 BFS / DFS
- 把相邻 frontier 格子聚成一个 **frontier cluster**

每个 cluster 表示：
> “一整片未知区域的入口”

---
### Step 4：为每个 Frontier Cluster 打分
#### 最基础评分函数（示例）
`score = α · frontier_size       - β · distance_to_robot`
- `frontier_size`：cluster 中格子数量（≈ 能看到多少未知）
- `distance`：当前机器人 → frontier 中心的路径长度

---

### Step 5：选分数最高的 Frontier
- 作为下一个探索目标
- 计算 cluster 的 **中心点 / 最近点**
- 转换成 `PoseStamped`
    

---
### Step 6：交给 Nav2 去“走过去”

- 通过 `NavigateToPose` action
- Nav2 负责：
    - 路径规划
    - 动态避障
    - 到达判定

---

### Step 7：判断是否“探索完成”
满足任一：
- 没有 frontier
- frontier 都不可达
- 未知区域 < 阈值
- 超时
