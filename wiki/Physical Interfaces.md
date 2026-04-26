# Sable 物理接口与用法总览

> 本文按“**获取信息**”与“**施加影响**”两个方向整理 Sable 当前代码仓库中可直接使用的物理相关接口。示例均面向二次开发（Mod 集成 / 联动）。

---

## 0. 总体入口

### 0.1 获取子关卡容器（SubLevelContainer）
- 入口：`SubLevelContainer.getContainer(level)`
- 作用：拿到某个 `Level` 的子关卡容器，后续查询、遍历、分配子关卡都从它开始。

```java
// 功能：从当前维度获取 Sable 子关卡容器
SubLevelContainer container = SubLevelContainer.getContainer(level);
if (container == null) return;
```

### 0.2 获取刚体句柄（RigidBodyHandle）
- 入口：`RigidBodyHandle.of(ServerSubLevel subLevel)` / `RigidBodyHandle.of(ServerLevel, PhysicsPipelineBody)`
- 作用：对某个子关卡刚体做脉冲、力矩、速度、传送等操作。

```java
// 功能：为某个 ServerSubLevel 获取刚体操作句柄
RigidBodyHandle handle = RigidBodyHandle.of(serverSubLevel);
if (handle == null || !handle.isValid()) return;
```

---

## 1. 获取信息方向

> 目标示例：获取“结构里某个方块在世界中的坐标”、获取质心、速度、角速度、碰撞范围等。

### 1.1 坐标系转换与姿态（Pose）

`SubLevel.logicalPose()` 返回子关卡当前逻辑姿态，`SubLevel.lastPose()` 返回上一 tick 姿态。可用于**局部坐标 <-> 世界坐标**转换。

常见用法（源码中已有同类调用）：
- `logicalPose().transformPosition(local)`：局部点 -> 世界点
- `logicalPose().transformPositionInverse(world)`：世界点 -> 局部点
- `logicalPose().transformNormal(vec)`：局部方向 -> 世界方向
- `logicalPose().transformNormalInverse(vec)`：世界方向 -> 局部方向

```java
// 功能：把结构内方块中心(local)转成世界坐标(world)
Vector3d localBlockCenter = new Vector3d(x + 0.5, y + 0.5, z + 0.5);
Vector3d worldPos = serverSubLevel.logicalPose().transformPosition(localBlockCenter, new Vector3d());
```

### 1.2 直接读取子关卡状态

可从 `SubLevel` / `ServerSubLevel` 读取：
- `boundingBox()`：世界空间包围盒
- `getUniqueId()` / `getName()`：标识
- `getPlot()`：子关卡内容所在 plot
- `getMassTracker()`：总质量、质心、惯性张量

```java
// 功能：读取结构当前世界包围盒 + 质心（局部）
var worldAabb = serverSubLevel.boundingBox();
var mass = serverSubLevel.getMassTracker().getMass();
var comLocal = serverSubLevel.getMassTracker().getCenterOfMass();
```

### 1.3 查询“某位置属于哪个子关卡 / 影响了哪些子关卡”

通过 `SubLevelContainer`：
- `getAllSubLevels()`：遍历所有已加载子关卡
- `queryIntersecting(bounds)`：查询与某世界包围盒相交的子关卡
- `getSubLevel(UUID)`：按 UUID 取子关卡

如果要做“世界点 -> 所属子关卡”判定，常见做法是：
1) 用点构造一个极小包围盒；
2) 调 `queryIntersecting` 做快速候选过滤；
3) 再按你需要做精细判断。

```java
// 功能：查询与一个世界 AABB 相交的子关卡
for (SubLevel sub : container.queryIntersecting(bounds)) {
    // 在这里处理候选 sub-level
}
```


### 1.4 判断某个 BlockPos 是否在子维度内，以及属于哪一个子维度

最直接的 API 是 `Sable.HELPER.getContaining(level, pos)`（`pos` 可传 `BlockPos` / `Vec3i` / `Position` / `Vector3dc` 等）。

- 返回 `null`：该位置不在任何子维度 plot 内。
- 返回 `SubLevel`：该位置属于该子维度。

```java
// 功能：判断一个方块坐标是否在子维度内，并拿到所属子维度
BlockPos pos = new BlockPos(x, y, z);
SubLevel owner = Sable.HELPER.getContaining(level, pos);

if (owner == null) {
    // 功能：当前位置在世界主维度（不在任何子维度）
} else {
    // 功能：当前位置在 owner 这个子维度内，可继续读取 owner.getUniqueId()/getName()
}
```

> 补充：如果你需要“一点可能命中多个旋转后包围盒”的近似查询，先用 `getAllIntersecting(...)` 做候选，再按业务精筛。

### 1.5 读取速度信息

#### A) 刚体速度（推荐）
`RigidBodyHandle` 提供：
- `getLinearVelocity(dest)`：世界系线速度
- `getAngularVelocity(dest)`：世界系角速度

```java
// 功能：读取刚体线速度与角速度（世界参考系）
Vector3d v = handle.getLinearVelocity(new Vector3d());
Vector3d w = handle.getAngularVelocity(new Vector3d());
```

#### B) 点相对空气速度
`SubLevelHelper.getVelocityRelativeToAir(level, pos, dest)` 可得到某世界点相对空气速度（考虑风提供器）。

```java
// 功能：读取世界点相对空气速度，用于空气动力学计算
Vector3d vAir = SubLevelHelper.getVelocityRelativeToAir(level, worldPos, new Vector3d());
```

### 1.6 质量与惯性信息

`MassData` / `MassTracker` 核心可读项：
- `getMass()` / `getInverseMass()`
- `getCenterOfMass()`（局部坐标，可能为 null）
- `getInertiaTensor()` / `getInverseInertiaTensor()`
- `getInverseNormalMass(position, direction)`（指定点、指定方向的等效质量）

```java
// 功能：读取结构质量参数，供控制器或稳定器使用
MassData md = serverSubLevel.getMassTracker();
double mass = md.getMass();
Vector3dc com = md.getCenterOfMass();
```

---

## 2. 施加影响方向

> 目标示例：在结构质心施加“世界参考系向量”的特定大小力，或者在任意点施加冲量并产生转动。

### 2.1 直接对刚体施加冲量（RigidBodyHandle）

`RigidBodyHandle` 常用接口：
- `applyImpulseAtPoint(position, force)`：在某点施加冲量（会产生线/角效应）
- `applyLinearImpulse(impulse)`：纯线冲量（局部）
- `applyAngularImpulse(impulse)` / `applyTorqueImpulse(torque)`：纯角冲量（局部）
- `applyLinearAndAngularImpulse(impulse, torque, wakeUp)`：同时施加
- `addLinearAndAngularVelocity(v, w)`：直接改速度
- `teleport(position, orientation)`：传送刚体

```java
// 功能：在局部坐标某一点施加局部冲量
handle.applyImpulseAtPoint(
    new Vector3d(localX, localY, localZ),
    new Vector3d(localFx, localFy, localFz)
);
```

> 注意：当前接口注释把单位写作 N（牛顿），但方法命名和实现语义是 impulse（脉冲/动量增量）风格；工程上请统一你的时间步与单位约定。

### 2.2 用 ForceTotal 聚合“点力”并一次提交

`ForceTotal` 适合你先汇总多个推进器/气动面输出，再统一施加：
- `applyImpulseAtPoint(massData, position, force)`：按点力自动累积线力与力矩
- `applyLinearAndAngularImpulse(...)`
- `getLocalForce()` / `getLocalTorque()`
- `handle.applyForcesAndReset(forceTotal)`：通过 `RigidBodyHandle` 提交并清空

```java
// 功能：汇总多个作用点的力，并一次性提交给刚体
ForceTotal total = new ForceTotal();
for (Thruster t : thrusters) {
    total.applyImpulseAtPoint(
        serverSubLevel.getMassTracker(),
        t.localPoint(),
        t.localForce()
    );
}
handle.applyForcesAndReset(total);
```

### 2.3 使用 QueuedForceGroup 做分组记录（可观测/调试）

`ServerSubLevel.getOrCreateQueuedForceGroup(forceGroup)` 可按类别记录受力（如 LIFT / DRAG / PROPULSION）：
- `applyAndRecordPointForce(point, force)`：既累计到 `ForceTotal`，又记录点力（开启 tracking 时）
- `recordPointForce(point, force)`：仅记录

```java
// 功能：按“推进力”分组记录并施加
QueuedForceGroup propulsion = serverSubLevel.getOrCreateQueuedForceGroup(ForceGroups.PROPULSION.get());
propulsion.applyAndRecordPointForce(localPoint, localForce);
```

### 2.4 关键场景：在“质心”施加一个“世界参考系向量”力

你给出的需求可以直接落地为：
1) 取局部质心 `comLocal = massData.getCenterOfMass()`；
2) 将世界力向量转局部：`localForce = pose.transformNormalInverse(worldForce)`；
3) 在局部质心点施加：`handle.applyImpulseAtPoint(comLocal, localForce)`。

```java
// 功能：在结构质心施加世界系向量（先转局部再施加）
MassData md = serverSubLevel.getMassTracker();
Vector3dc comLocal = md.getCenterOfMass();
if (comLocal == null) return;

Vector3d worldForce = new Vector3d(wx, wy, wz);            // 世界参考系
Vector3d localForce = serverSubLevel.logicalPose()
    .transformNormalInverse(worldForce, new Vector3d());   // 转到局部

handle.applyImpulseAtPoint(comLocal, localForce);
```

### 2.5 若你在 BlockEntity 中实现持续受力

可实现 `BlockEntitySubLevelActor#sable$physicsTick(...)`，在每次 physics tick 中读取状态并施加作用：

```java
// 功能：在物理 tick 内持续施加控制力
@Override
public void sable$physicsTick(ServerSubLevel subLevel, RigidBodyHandle handle, double timeStep) {
    // 读取姿态、速度、质心
    // 计算控制输出
    // 调 handle.applyXXX(...)
}
```

---

## 3. 接口速查（按能力归类）

### 3.1 信息获取
- `SubLevelContainer.getContainer(level)`
- `SubLevelContainer.getAllSubLevels()` / `queryIntersecting(bounds)` / `getSubLevel(uuid)`
- `SubLevel.logicalPose()` / `lastPose()` / `boundingBox()`
- `ServerSubLevel.getMassTracker()`
- `MassData.getMass()` / `getCenterOfMass()` / `getInertiaTensor()`
- `RigidBodyHandle.getLinearVelocity(dest)` / `getAngularVelocity(dest)`
- `SubLevelHelper.getVelocityRelativeToAir(level, pos, dest)`

### 3.2 施加影响
- `RigidBodyHandle.applyImpulseAtPoint(...)`
- `RigidBodyHandle.applyLinearImpulse(...)`
- `RigidBodyHandle.applyAngularImpulse(...)`
- `RigidBodyHandle.applyLinearAndAngularImpulse(...)`
- `RigidBodyHandle.addLinearAndAngularVelocity(...)`
- `RigidBodyHandle.teleport(...)`
- `ForceTotal.applyImpulseAtPoint(...)`
- `ServerSubLevel.getOrCreateQueuedForceGroup(...)`
- `QueuedForceGroup.applyAndRecordPointForce(...)`

---

## 4. 实战建议（避免常见坑）

1. **先统一坐标系**：你的控制器建议内部统一使用“局部系”，只在输入/输出边界做世界<->局部转换。  
2. **检查质心 null**：极端情况下结构可能无有效质量，`getCenterOfMass()` 可能为空。  
3. **按 physics tick 施力**：持续推力不要放在普通 game tick，优先放 `sable$physicsTick`。  
4. **需要可视化就开分组记录**：`enableIndividualQueuedForcesTracking(true)` 后配合 `QueuedForceGroup` 便于调试。  
5. **单位保持一致**：当前接口命名与注释存在“force/impulse”混用，工程实现要自定并严格一致。


## 5. 进阶场景补充

### 5.1 让玩家坐在“与方块绑定的乘坐实体（如坐垫）”上时，随子维度正常移动/旋转

这个场景在 Sable 内部是按“**乘客跟随载具所在子维度**”处理的，关键点是：

1. **让坐垫实体留在子维度里，不要被踢出**（通常给实体类型打 `#sable:retain_in_sub_level` 标签）。
2. 玩家上座后，Sable 会把乘客的 tracking sub-level 继承为载具所在子维度，进而跟随子维度位姿变化。
3. 对需要“无抖动显示”的同步点，可配合 `EntitySubLevelUtil.setOldPosNoMovement(entity)` 在客户端/同步包处理后刷新旧位置。

```java
// 功能：示例——生成可乘坐实体后，确保其留在子维度并让玩家乘坐
Entity seat = ...; // 你的坐垫实体（建议在实体类型层面加入 retain_in_sub_level 标签）
SubLevel seatSubLevel = Sable.HELPER.getContaining(level, seat);

if (seatSubLevel != null) {
    // 功能：玩家开始乘坐后，将自动跟随 seat 所在子维度的平移/旋转
    player.startRiding(seat, true);

    // 功能：可选，消除一帧“看起来移动了”的旧坐标误差
    EntitySubLevelUtil.setOldPosNoMovement(player);
}
```

> 若你在“读取 NBT 后重建骑乘关系”这类流程里手动摆放乘客位置，可参考 `EntityRidingSubLevelVehicleHelper.kickRidingEntity(...)` 的做法：按眼睛锚点把乘客位置从子维度局部转换到世界系，避免高度错位。

### 5.2 方向转换：把子维度中方块朝向（东南西北上下）转成世界坐标系向量

核心做法：

1. 先把 `Direction` 变成局部单位向量（`direction.getNormal()`）；
2. 用 `subLevel.logicalPose().transformNormal(localDir)` 转到世界系。

```java
// 功能：把子维度中的方块朝向(Direction)转换成世界系方向向量
Direction localFacing = state.getValue(BlockStateProperties.FACING); // 例如 NORTH/UP 等
Vector3d localDir = JOMLConversion.atLowerCornerOf(localFacing.getNormal());

Vector3d worldDir = subLevel.logicalPose()
    .transformNormal(localDir, new Vector3d()) // 功能：局部方向 -> 世界方向
    .normalize();                               // 功能：归一化，便于作为射线/推力方向使用
```

如果你需要“世界向量 -> 子维度内朝向”，反向使用 `transformNormalInverse(...)`，再取 `Direction.getNearest(...)` 即可。
