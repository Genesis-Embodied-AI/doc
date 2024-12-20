# 🦾 逆运动学与运动规划

在本教程中，我们将通过几个示例来说明如何在Genesis中使用逆运动学（IK）和运动规划，并执行一个简单的抓取任务。

首先，我们创建一个场景，加载你喜欢的机械臂和一个小立方体，构建场景，然后设置控制增益：

```python
import numpy as np
import genesis as gs

########################## 初始化 ##########################
gs.init(backend=gs.gpu)

########################## 创建场景 ##########################
scene = gs.Scene(
    viewer_options = gs.options.ViewerOptions(
        camera_pos    = (3, -1, 1.5),
        camera_lookat = (0.0, 0.0, 0.5),
        camera_fov    = 30,
        max_FPS       = 60,
    ),
    sim_options = gs.options.SimOptions(
        dt = 0.01,
        substeps = 4, # 为了更稳定的抓取接触
    ),
    show_viewer = True,
)

########################## 实体 ##########################
plane = scene.add_entity(
    gs.morphs.Plane(),
)
cube = scene.add_entity(
    gs.morphs.Box(
        size = (0.04, 0.04, 0.04),
        pos  = (0.65, 0.0, 0.02),
    )
)
franka = scene.add_entity(
    gs.morphs.MJCF(file='xml/franka_emika_panda/panda.xml'),
)
########################## 构建 ##########################
scene.build()

motors_dof = np.arange(7)
fingers_dof = np.arange(7, 9)

# 设置控制增益
# 注意：以下值是为实现Franka最佳行为而调整的
# 通常，每个新机器人都会有一组不同的参数。
# 有时高质量的URDF或XML文件也会提供这些参数，并会被解析。
franka.set_dofs_kp(
    np.array([4500, 4500, 3500, 3500, 2000, 2000, 2000, 100, 100]),
)
franka.set_dofs_kv(
    np.array([450, 450, 350, 350, 200, 200, 200, 10, 10]),
)
franka.set_dofs_force_range(
    np.array([-87, -87, -87, -87, -12, -12, -12, -100, -100]),
    np.array([ 87,  87,  87,  87,  12,  12,  12,  100,  100]),
)
```

```{figure} ../../_static/images/IK_mp_grasp.png
```

接下来，让我们将机器人的末端执行器移动到预抓取姿态。这分两步完成：

- 使用IK求解给定目标末端执行器姿态的关节位置
- 使用运动规划器到达目标位置

Genesis中的运动规划使用OMPL库。你可以按照[安装](../overview/installation.md)页面中的说明进行安装。

在Genesis中，IK和运动规划非常简单：每个都可以通过一个函数调用完成。

```python

# 获取末端执行器链接
end_effector = franka.get_link('hand')

# 移动到预抓取姿态
qpos = franka.inverse_kinematics(
    link = end_effector,
    pos  = np.array([0.65, 0.0, 0.25]),
    quat = np.array([0, 1, 0, 0]),
)
# 夹爪打开位置
qpos[-2:] = 0.04
path = franka.plan_path(
    qpos_goal     = qpos,
    num_waypoints = 200, # 2秒持续时间
)
# 执行规划路径
for waypoint in path:
    franka.control_dofs_position(waypoint)
    scene.step()

# 让机器人到达最后一个路径点
for i in range(100):
    scene.step()

```

如你所见，IK求解和运动规划都是机器人实体的两个集成方法。对于IK求解，你只需告诉机器人的IK求解器哪个链接是末端执行器，并指定目标姿态。然后，你告诉运动规划器目标关节位置（qpos），它会返回一个规划和平滑的路径点列表。注意，在我们执行路径后，我们让控制器再运行100步。这是因为我们使用的是PD控制器，目标位置和当前实际位置之间会有一个差距。因此，我们让控制器多运行一段时间，以便机器人能够到达规划轨迹的最后一个路径点。

接下来，我们将机器人夹爪向下移动，抓取立方体并将其抬起：

```python
# 到达
qpos = franka.inverse_kinematics(
    link = end_effector,
    pos  = np.array([0.65, 0.0, 0.135]),
    quat = np.array([0, 1, 0, 0]),
)
franka.control_dofs_position(qpos[:-2], motors_dof)
for i in range(100):
    scene.step()

# 抓取
franka.control_dofs_position(qpos[:-2], motors_dof)
franka.control_dofs_force(np.array([-0.5, -0.5]), fingers_dof)

for i in range(100):
    scene.step()

# 抬起
qpos = franka.inverse_kinematics(
    link=end_effector,
    pos=np.array([0.65, 0.0, 0.3]),
    quat=np.array([0, 1, 0, 0]),
)
franka.control_dofs_position(qpos[:-2], motors_dof)
for i in range(200):
    scene.step()
```

在抓取物体时，我们对2个夹爪自由度使用了力控制，并施加了0.5N的抓取力。如果一切顺利，你将看到物体被抓取并抬起。

