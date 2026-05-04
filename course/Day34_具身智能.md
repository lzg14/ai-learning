# Day 34：具身智能

> 之前的 Agent 都活在数字世界里，具身智能让 Agent 有了"身体"，能在物理世界里移动、感知、操作、做事。

## Part 1：什么是具身智能？为什么这是 AI 的下一个大时代？

### 什么是具身智能？

**简单说：有身体的 AI，能在物理世界里真实做事的 AI。**

| AI 时代 | "活"在哪里 | 能做什么 |
| --- | --- | --- |
| 第一代 AI | 服务器里 | 下棋、推荐、分类 |
| 第二代 AI（2020-2025） | 数字世界里 | 聊天、写代码、画画 |
| **第三代 AI（2025+）** | **物理世界里** | **走路、拿东西、做饭、打扫、开车** |

> 💡 之前的 AI 都是"瘫子"：只有脑子，没有手，没有脚。具身智能给 AI 装上了手和脚。

---

### 为什么具身智能这么重要？

**整个人类文明，99% 的工作都是在物理世界做的：**
- 工厂里的工人
- 餐厅里的服务员
- 家里的保姆
- 医院里的护士
- 快递员、司机

**这些工作，之前的 AI 都做不了，因为它没有身体。
具身智能一出来，这些工作就都能被 AI 做了！**

---

### 为什么是现在？

**三个技术同时成熟了：**
1. ✅ **大模型成熟了：** AI 终于有"脑子"了，能理解世界
2. ✅ **视觉模型成熟了：** AI 终于能"看"了
3. ✅ **机器人硬件便宜了：** 成本降到了普通人能承受的范围

---

## Part 2：具身 Agent 的完整架构

### 具身 Agent 的三层架构

```plaintext
┌─────────────────────────────────────────────────────┐
│                  第三层：行动层                        │
│        机械臂、轮子、手、腿、各种执行器                   │
├─────────────────────────────────────────────────────┤
│                  第二层：决策层                        │
│        大模型 + 规划算法 + 强化学习                      │
├─────────────────────────────────────────────────────┤
│                  第一层：感知层                        │
│    摄像头、雷达、深度传感器、触觉传感器...               │
└─────────────────────────────────────────────────────┘
```

是不是和我们人的架构一模一样？

---

### 第一层：感知层

**核心传感器：**

| 传感器 | 作用 | 成本 |
| --- | --- | --- |
| RGB 摄像头 | 拍彩色照片 | 几十块 |
| 深度摄像头/激光雷达 | 知道距离 | 几百-几千块 |
| 触觉传感器 | 碰到东西能感觉到 | 几十-几百块 |
| IMU/陀螺仪 | 知道姿势 | 几十块 |

---

### 第二层：决策层

**核心组成：**

1. **世界模型 World Model**
   - 机器人脑子里有一个对世界的理解
   - 能在脑子里"预演"接下来会发生什么

2. **规划算法**
   - 我要去那边拿杯子，怎么走？
   - 路上有个椅子，怎么绕过去？

3. **大模型**
   - 理解自然语言指令
   - 分解复杂任务

---

### 第三层：行动层

**核心执行器：**

| 执行器 | 作用 |
| --- | --- |
| 轮子/腿 | 移动 |
| 机械臂/手 | 拿东西，放东西 |
| 各种工具头 | 吸尘器头、拖把、螺丝刀 |

---

## Part 3：主流的机器人平台和开发框架

### 消费级机器人平台

| 平台 | 价格 | 特点 | 适合做什么 |
| --- | --- | --- | --- |
| Unitree 宇树 Go2 | 1-2 万 | 四足机器狗，能跑能跳 | 巡检、科研 |
| 小米 CyberDog 2 | 1 万多 | 四足，小米生态 | 爱好、科研 |
| UBtech Walker S | 几十万 | 双足人形机器人 | 科研、商用 |

### 具身智能开发框架

| 框架 | 出品方 | 特点 |
| --- | --- | --- |
| ROS / ROS 2 | 开源社区 | 最成熟，生态最好 |
| NVIDIA Isaac Sim | NVIDIA | 模拟器，虚拟世界训练 |
| Unitree ROS SDK | 宇树 | 宇树机器人官方 SDK |
| OpenVLA | 开源 | 通用视觉-行动模型 |

---

## Part 4：实战——家居控制机器人

### 完整代码：具身家居 Agent

```python
"""
具身家居控制 Agent
能看图片、理解 3D 空间、导航、控制智能设备
"""
import asyncio
import random
from typing import List, Dict, Any, Optional, Tuple
from dataclasses import dataclass
from enum import Enum

@dataclass
class Position3D:
    """3D 位置"""
    x: float
    y: float
    z: float

@dataclass
class ObjectInfo:
    """物体信息"""
    name: str
    position: Position3D
    category: str
    is_smart_device: bool = False
    device_type: str = None
    device_state: str = "off"

class PerceptionSystem:
    """感知系统"""

    def __init__(self):
        self.camera_enabled = True
        self.depth_sensor_enabled = True

    async def capture_scene(self) -> Dict:
        """捕获场景"""
        return {
            "rgb_image": "模拟 RGB 图像",
            "depth_map": "模拟深度图",
            "timestamp": "2026-05-02T00:00:00Z"
        }

    async def detect_objects(self, scene: Dict) -> List[ObjectInfo]:
        """检测物体"""
        # 模拟检测结果
        objects = [
            ObjectInfo("台灯", Position3D(0.5, 0.8, 1.2), "家具", True, "light", "off"),
            ObjectInfo("茶杯", Position3D(0.6, 0.5, 1.0), "日用品", False),
            ObjectInfo("沙发", Position3D(2.0, 1.5, 0.5), "家具", True, "sofa_light", "off"),
            ObjectInfo("电视", Position3D(3.0, 2.0, 0.3), "电器", True, "tv", "off"),
            ObjectInfo("空调", Position3D(1.0, 3.0, 2.5), "电器", True, "ac", "off"),
            ObjectInfo("窗帘", Position3D(0.0, 2.0, 1.5), "家具", True, "curtain", "closed"),
        ]
        return objects

    async def estimate_depth(self, point: Position3D) -> float:
        """估计深度"""
        return (point.x ** 2 + point.y ** 2 + point.z ** 2) ** 0.5

class WorldModel:
    """世界模型"""

    def __init__(self):
        self.objects: List[ObjectInfo] = []
        self.room_map: Dict = {}

    def update_world(self, objects: List[ObjectInfo]):
        """更新世界模型"""
        self.objects = objects
        print(f"🌍 世界模型更新：检测到 {len(objects)} 个物体")

    def find_object(self, name: str) -> Optional[ObjectInfo]:
        """查找物体"""
        for obj in self.objects:
            if name.lower() in obj.name.lower():
                return obj
        return None

    def calculate_path(self, from_pos: Position3D, to_pos: Position3D) -> List[Position3D]:
        """计算路径"""
        steps = 5
        return [
            Position3D(
                from_pos.x + (to_pos.x - from_pos.x) * i / steps,
                from_pos.y + (to_pos.y - from_pos.y) * i / steps,
                from_pos.z + (to_pos.z - from_pos.z) * i / steps
            )
            for i in range(steps + 1)
        ]

    def predict_action_result(self, device_type: str, action: str) -> str:
        """预测动作结果"""
        if device_type == "light":
            return "on" if action == "turn_on" else "off"
        elif device_type == "tv":
            return "on" if action == "turn_on" else "off"
        elif device_type == "curtain":
            return "open" if action == "open" else "closed"
        return "unknown"

class ActionExecutor:
    """动作执行器"""

    def __init__(self):
        self.current_position = Position3D(0, 0, 0)
        self.action_history: List[Dict] = []

    async def move_to(self, target: Position3D):
        """移动到目标位置"""
        distance = ((target.x - self.current_position.x) ** 2 +
                   (target.y - self.current_position.y) ** 2 +
                   (target.z - self.current_position.z) ** 2) ** 0.5

        print(f"🚶 移动中：距离 {distance:.2f} 米")

        # 模拟移动时间
        await asyncio.sleep(distance * 0.5)

        self.current_position = target
        self.action_history.append({
            "action": "move",
            "target": target,
            "distance": distance
        })

        print(f"✅ 到达：({target.x:.1f}, {target.y:.1f}, {target.z:.1f})")

    async def control_device(self, device: ObjectInfo, action: str):
        """控制设备"""
        print(f"🎛️ 控制设备：{device.name} - {action}")

        await asyncio.sleep(0.3)

        device.device_state = "on" if action == "turn_on" else "off"

        self.action_history.append({
            "action": action,
            "device": device.name,
            "result": "success"
        })

        print(f"✅ {device.name} 已{action}")

    async def grab_object(self, obj: ObjectInfo):
        """抓取物体"""
        print(f"🤲 抓取物体：{obj.name}")

        await asyncio.sleep(0.5)

        self.action_history.append({
            "action": "grab",
            "object": obj.name,
            "result": "success"
        })

        print(f"✅ 已抓取 {obj.name}")

class EmbodiedHomeAgent:
    """具身家居 Agent"""

    def __init__(self):
        self.perception = PerceptionSystem()
        self.world_model = WorldModel()
        self.action_executor = ActionExecutor()
        self.conversation_history: List[Dict] = []

    async def perceive(self):
        """感知环境"""
        print("\n👁️ 感知环境...")
        scene = await self.perception.capture_scene()
        objects = await self.perception.detect_objects(scene)
        self.world_model.update_world(objects)

    async def execute_command(self, command: str):
        """执行自然语言命令"""
        print(f"\n🎯 收到命令：{command}")
        self.conversation_history.append({"role": "user", "content": command})

        # 命令解析
        command_lower = command.lower()

        if "开灯" in command or "关灯" in command:
            await self._handle_light_command(command)
        elif "电视" in command:
            await self._handle_tv_command(command)
        elif "窗帘" in command:
            await self._handle_curtain_command(command)
        elif "拿" in command or "取" in command:
            await self._handle_grab_command(command)
        elif "走到" in command or "去" in command:
            await self._handle_navigation_command(command)
        else:
            print(f"🤔 抱歉，我还不理解这个命令")

    async def _handle_light_command(self, command: str):
        """处理灯光命令"""
        device = self.world_model.find_object("灯")
        if device:
            action = "turn_on" if "开" in command else "turn_off"
            await self.action_executor.control_device(device, action)

    async def _handle_tv_command(self, command: str):
        """处理电视命令"""
        device = self.world_model.find_object("电视")
        if device:
            action = "turn_on" if "开" in command else "turn_off"
            await self.action_executor.control_device(device, action)

    async def _handle_curtain_command(self, command: str):
        """处理窗帘命令"""
        device = self.world_model.find_object("窗帘")
        if device:
            action = "open" if "开" in command else "close"
            await self.action_executor.control_device(device, action)

    async def _handle_grab_command(self, command: str):
        """处理抓取命令"""
        for obj in self.world_model.objects:
            if obj.name in command:
                # 导航到物体位置
                await self.action_executor.move_to(obj.position)
                # 抓取
                await self.action_executor.grab_object(obj)
                break

    async def _handle_navigation_command(self, command: str):
        """处理导航命令"""
        for obj in self.world_model.objects:
            if obj.name in command:
                await self.action_executor.move_to(obj.position)
                break

    def get_status(self) -> Dict:
        """获取状态"""
        return {
            "position": self.action_executor.current_position,
            "objects_count": len(self.world_model.objects),
            "actions_count": len(self.action_executor.action_history),
            "devices_on": sum(1 for o in self.world_model.objects if o.is_smart_device and o.device_state == "on")
        }

async def main():
    print("=" * 70)
    print("🏠 具身家居 Agent 测试")
    print("=" * 70)

    agent = EmbodiedHomeAgent()

    # 1. 感知环境
    await agent.perceive()

    # 2. 执行命令
    commands = [
        "把客厅的灯打开",
        "把电视打开",
        "打开窗帘",
        "去拿茶杯",
        "走到沙发旁边",
    ]

    for cmd in commands:
        await agent.execute_command(cmd)
        await asyncio.sleep(0.5)

    # 3. 获取状态
    status = agent.get_status()
    print(f"\n📊 状态：")
    print(f"  当前位置：({status['position'].x:.1f}, {status['position'].y:.1f}, {status['position'].z:.1f})")
    print(f"  检测物体数：{status['objects_count']}")
    print(f"  执行动作数：{status['actions_count']}")
    print(f"  开启设备数：{status['devices_on']}")

if __name__ == "__main__":
    asyncio.run(main())
```

---

## 📌 今日要点

- ✅ 理解了什么是具身智能，为什么这是 AI 的下一个大时代
- ✅ 掌握了具身 Agent 的三层架构：感知层、决策层、行动层
- ✅ 知道了主流的机器人平台和开发框架
- ✅ 实现了一个完整的具身家居控制 Agent

---

## 自测题

1. 什么是具身智能？和传统 Agent 有什么区别？
2. 具身 Agent 的三层架构分别是什么？各有什么作用？
3. 世界模型的作用是什么？为什么需要它？
4. 三个技术同时成熟指的是什么？

## 动手作业

1. 把上面的具身家居 Agent 跑起来
2. 添加一个"温度调节"功能
3. 实现更复杂的路径规划算法
