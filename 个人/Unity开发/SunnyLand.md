# 文档
## Manager of Mangers
需要采用 Manager of Manager
我这可能不是 Manager of Manager 框架

- 一个大单例 MainManager 将一系列小 Manager 合在一起，仅做了整合
- 每个 Manager 作为小单例
- 根本就不是框架，先写起来吧，一点点改进
### Managers
- LevelManager
  - 管理游戏场景的切换，关卡切换
- AudioManager
  - 管理游戏全程的音效
- GUIManager
  - 管理所有的UI
- ItemDataManager
  - 管理静态配置数据， 比如：
    - 怪物的配置属性
    - 道具的配置属性
    - 陷阱的配置属性
  - 这些都是静态的数据，也就是说用来初始化用的，这个Manager仅方便于数据的统一输入，后期肯定是要改成文件读取来配置数据的
- InputSystemManager
  - 只需要全局 new 一个controller就行，用于被各个脚本获取与控制


## 挂载用脚本
- Player
  - PlayerAction
    - 获取 InputSystem 控制角色移动
    - 控制受伤等行为
    - 主要聚焦于玩家的行为
  - PlayerManager （单例）
    - 配置玩家数据
    - 更新玩家数据
- Enemy
  - EnemyAction_Frogs
    - 移动，攻击判断等
  - EnemyManager
- Item
  - ItemAction_Cherry/Gem
    - 触发回血，计数，掉落等行为
- Clicker
  - 点击事件大集合

## Level Pack
- UI
  - 没问题，就那几个页面，UI啥的
  - 


## 烂尾，架构一坨，需要学学其他的



















# BUG
## 使用2DCharacterController遇到的问题
- 发现move函数正常触发，但无法正常移动
  - 允许control in air时可以正常移动，推断是由于地面检测坐标过高，导致一直无法接触到地面从而无法操控，且地面layer需要设置合理
  - 同理，发现头顶有天花板时触发了蹲下的减速，原因是天花板检测坐标过高，导致天花板检测不合理

## Animator创建预制件
- 想创建animation和controller预置件需要先创建个空的gameobject，把动画帧拖到gameobject上生成这俩东西，然后把gameobject删了就行
- 关于只有两帧的跳跃动画，或许我们想实现在跳跃上升时，使用动画帧1而在下降时使用另一个动画帧的动画逻辑，然而，如果使用检测是否跳跃至顶点并进行动画切换的逻辑非常复杂，Brackey使用的方法是将两个动画帧直接生成为一个动画，并将其sampler设置为2便可实现这个跳跃动画逻辑，具体原理不明，有待后续补充

## tilemap在移动时砖块之间产生蓝线的问题
- 一是主相机的反锯齿关掉
- 二是可以将砖块之间的距离设为一个小的负值，让砖块更紧密也行
- 三是有个使用shader的解决方法，太长不看

## 背景跟随镜头
- 直接将背景对象背景作为相机子对象

## Managers的简单单例
```c#
public static PlayerManager instance;

 private void Awake() {
    instance = this;
 }

 // 使用
 PlayerManager.instance.func();
```

## ***小狐狸教程，b站P47，对青蛙设置动画的问题
- 为什么需要对父对象添加animator，用来控制子类的spriteRender，而且为什么在做动画录制操作的时候拖入图片到时间轴上不行，但设置spriteRender的sprite就行
- 而且为什么这样子做可以让青蛙的上下移动动画不会覆盖代码的移动逻辑
- 只能说这种做法真能实现想要的效果，但就是不太能看懂，

## 使用transform生成特效时特效会随着死亡对象移动的bug
在 Unity 中，当你实例化一个 `GameObject` 并传递一个 `Transform` 作为第二个参数时，该对象会被设置为该 `Transform` 的子对象。因此，如果你实例化的动画对象 `anim.gameObject` 之后会随着 `transform` 一起移动，很可能是因为这个原因。

要解决这个问题，你可以避免在实例化时传递 `transform` 参数，或者确保提供的坐标和旋转是你想要的。例如：

```csharp
// Instantiate without setting it as a child
Instantiate(anim.gameObject, transform.position, transform.rotation);

// If you don't want it to move with the transform, avoid setting it as a child
GameObject newObject = Instantiate(anim.gameObject); // Instantiate with default position
newObject.transform.position = transform.position; // Set position manually if required
```

这样，实例化的对象将与 `transform` 的位置对齐，但不会成为其子对象，因此不会跟随它移动。根据你的需求选择合适的方法设置对象的位置和父子关系即可。

## 吃一个钻石来两个的问题
- 原因是短时间内连续触发了 player 身上的两个collider
- 一个简单的标志位即可解决

## 区分player踩踏攻击事件与敌人碰撞受伤事件
- 在玩家的攻击触发函数中给 playerManager 的攻击标志位赋值 true， 怪物的碰撞事件触发时检查该标志位，若为 true 则仅仅触发击退函数，否则触发伤害函数
- 总感觉是有点脱裤子放 p ，肯定有其他好方法

## 击退函数中作用力的问题
- 击退的效果会随着 player 的运动而改变，简单的说就是player的势能会抵消击退函数中force施加的力
- 比如速度跑起来受到击退时击退距离变近，从上面掉下来触发击退时，向上的力被重力压过去了，导致 player 做平移运动

解决：感觉说不定是 RigidBody.MovePosition 模拟一个固定的击退效果？这个暂且不改了

## 左右巡逻点没做好销毁的时机
- 没解决

## 每个脚本中的 inputsystem new出来的是独立的，注意奥

# 妈的，有的BUG得靠重启编辑器解决，焯
- 啧， 有的bug也不是这意思，总是没找到到底哪里出错了