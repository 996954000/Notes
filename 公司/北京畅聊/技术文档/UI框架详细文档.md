# Unity UI框架详细技术文档

## 概述

这是一个基于PureMVC架构的Unity UI框架，采用PMVC模式建立除ViewComponent之外的通知机制、数据更新机制等。框架的核心目标是让ViewComponent与Mediator关联起来，使Mediator有对应的操作对象。

## 核心架构组件

### 1. 数据结构层

#### 1.1 ContextViewVo（配置层）
```csharp
public class ContextViewVo
{
    public string ViewCmptClassName = "";        // ViewComponent类名
    public string MediatorClassName = "";        // Mediator类名
    public bool IsFullScreenView;                // 是否全屏视图
    public bool IsPopView;                       // 是否弹窗
    public ContextViewVo[] Children;             // 子视图配置
    public object Data;                          // 自定义数据
    public int ZIndex = 0;                       // 层级索引
    public int FunType = ContextFunType.Default; // 功能类型
    public string DefaultBgColor = null;         // 默认背景色
    public string CustomMediatorName = "";       // 自定义Mediator名称
    public BaseCmptAttribute CmptAttribute;      // 组件属性
}
```

**作用**：配置层数据结构，在config中定义UI的静态配置信息。

#### 1.2 Context（运行时上下文）
```csharp
public class Context
{
    public string MediatorClassStr;              // Mediator类名字符串
    public string ViewComponentClassStr;         // ViewComponent类名字符串
    public object Data;                          // 运行时数据
    public Context Parent;                       // 父Context
    public List<Context> Children;               // 子Context列表
    public ContextLoadStep Step;                 // 加载步骤状态
    public ContextType Type;                     // 上下文类型（CMPT/SCENE/WINDOW/NAVGATION）
    public int FunType;                          // 功能类型
    public string DefaultBgColor;                // 默认背景色
    public int ZIndex;                           // 层级索引
    public BaseCmptAttribute CmptAttribute;      // 组件属性
    public bool IsViewShow;                      // 视图是否显示
    public bool SendOpenNativeJumpMsg;           // 是否发送原生跳转消息
}
```

**作用**：运行时数据结构，从ContextViewVo实例化而来，管理UI节点的运行时状态和父子关系。

#### 1.3 LayerBodyVo（操作请求体）
```csharp
public class LayerBodyVo
{
    public enum TYPE
    {
        NONE,           // 无操作
        ADD,            // 添加
        WAITINGREMOVE,  // 等待移除
        REMOVE,         // 移除
        HIDE,           // 隐藏
        SHOW,           // 显示
    }
    
    public TYPE Type;                    // 操作类型
    public Context Context;              // 上下文
    public Action Callback;              // 回调函数
    public Context ParentContext;        // 父上下文
    public Context PrevContext;          // 前一个上下文
    public bool IsForceAddLayer;         // 是否强制添加
    public bool IsLoading;               // 是否正在加载
    public long createTime;              // 创建时间
}
```

**作用**：封装UI操作请求的完整信息，包含操作类型、上下文、回调等。

#### 1.4 ViewInfoClass（视图信息类）
```csharp
public class ViewInfoClass
{
    public object Data;                          // 数据
    public Dictionary<string, object> ChildData; // 子视图数据
    public Context Context;                      // 上下文
    public Context ParentContext;                // 父上下文
}
```

**作用**：包装视图创建所需的信息，包含数据、上下文等。

### 2. 核心类层次结构

#### 2.1 BaseMediator（中介者基类）
```csharp
public class BaseMediator : BaseNotifier, IMediator
{
    public BaseCmpt ViewComponent { get; set; }  // 关联的ViewComponent
    public Context Context { get; set; }         // 关联的Context
    public string MediatorName { get; set; }     // Mediator名称
    
    // 生命周期方法
    public virtual void ViewWillAppear()         // 视图即将出现
    public virtual void ViewDidAppear()          // 视图已经出现
    public virtual void ViewDidDisAppear()       // 视图已经消失
    public virtual void AllSubViewDidAppear()    // 所有子视图已出现
    
    // 子视图管理
    public virtual List<Context> OnAddSubView()  // 添加子视图
    public virtual void SubViewDidAppear(string mediatorName)  // 子视图已出现
    
    // 事件绑定
    public void Bind(string eventStr, ITEmitter.ActionType callback)
    
    // 视图操作
    public void Show(object viewData = null)     // 显示视图
    public void Hide()                           // 隐藏视图
    public void CloseLayer()                     // 关闭层
}
```

#### 2.2 BaseCmpt（视图组件基类）
```csharp
public class BaseCmpt : MonoBehaviour
{
    public ITEmitter _Emitter;                   // 事件发射器
    public BaseCmptAttribute _attribute;         // 组件属性
    public string prefabPath;                    // 预制体路径
    public string fGuiPath;                      // FairyGUI路径
    public string DependFGuiPackageName;         // 依赖的FGUI包名
    public string ViewComponentClassName;        // 组件类名
    protected bool isLoaded;                     // 是否已加载
    protected bool isRemoveCmpt;                 // 是否移除组件
    
    // 静态加载方法
    public static void Load(Type T, Action<BaseCmpt> cb, BaseCmptAttribute attribute, string vcName = null)
    
    // 生命周期方法
    public virtual void OnUILoaded(GameObject node)  // UI加载完成
    public void Enter()                              // 进入
    public virtual void Exit()                       // 退出
    public virtual void Attach(BaseCmpt parent)      // 附加到父节点
    public virtual void Detach(BaseCmpt parent)      // 从父节点分离
    
    // 视图状态
    public virtual void ViewWillAppear()         // 视图即将出现
    public virtual void ViewDidAppear()          // 视图已经出现
    public virtual void OnShow()                 // 显示
    public virtual void OnHide()                 // 隐藏
}
```

#### 2.3 BaseWindow（窗口基类）
```csharp
public class BaseWindow : BaseCmpt
{
    public static void Load(Type T, Action<BaseWindow> cb)  // 静态加载方法
    public override void Attach(BaseCmpt parent)            // 重写附加方法（窗口不需要附加）
}
```

### 3. 核心管理器

#### 3.1 GameMediator（游戏中介者）
**职责**：
- 处理所有UI相关的通知
- 管理UI的添加、移除、显示、隐藏操作
- 处理前台/后台切换时的UI状态管理

**关键方法**：
```csharp
// 处理通知
public override void OnHandleNotification(in BaseNotificationBody note)

// 添加子层
private void AddSubLayer(ViewInfoClass notiData)

// 加载上下文
private LayerBodyVo LoadContext(ViewInfoClass viewInfo)

// 添加场景
private void AddScene(ViewInfoClass viewInfo, int type)

// 添加窗口
private void AddWindow(ViewInfoClass notiData, string windowName)
```

#### 3.2 UIManager（UI管理器）
**职责**：
- 管理UI加载队列
- 控制每帧加载的UI数量
- 处理UI加载的异步操作

**关键特性**：
```csharp
public static bool EnableMainThreadAddLayer = true;  // 是否在主线程添加层
private static int MaxLayerQueueLoadCount = 50;      // 每帧最大加载数量

// 队列管理
private static readonly ConcurrentQueue<UILayerData> AddLayerQueue;
private static readonly ConcurrentQueue<UILayerData> AddWindowLayerQueue;

// 添加层到队列
public static void AddLayerToQueue(LayerBodyVo context, LayerProxy proxy)
public static void AddWindowLayerToQueue(LayerBodyVo context, LayerProxy proxy)
```

#### 3.3 LayerProxy（层代理）
**职责**：
- 管理UI层的加载、移除、显示、隐藏
- 处理UI组件的实例化和绑定
- 管理UI的生命周期

**关键方法**：
```csharp
// 主线程添加层
public async UniTask MainAddLayerContext(LayerBodyVo noteBody)

// 主线程加载层
private async UniTask MainLoadLayerWithContext(Context childContext, List<Context> layerContextArr)

// 主线程覆盖层
private async UniTask MainOverlay(BaseCmpt parent, string childClsStr, Context childContext, string vcName, Action<BaseCmpt> onCmptFinish, Action<BaseCmpt> onFinish, Action<Exception> onFailed)
```

#### 3.4 ContextProxy（上下文代理）
**职责**：
- 管理Context的存储和检索
- 管理窗口和场景的上下文栈
- 处理弹窗和全屏视图的特殊逻辑

**关键方法**：
```csharp
// 上下文管理
public void PushContext(Context context, string windowName)
public Context PopContext(string windowName)
public Context GetCurrentContext(string windowName)

// 弹窗管理
public void AddPopMediatorName(string mediatorName)
public void RemovePopMediatorName(string mediatorName)

// 全屏视图管理
public void AddForceCocosMediatorName(string mediatorName, int type, string bgColor, bool SendJumpMsg)
```

## 完整UI创建流程

### 阶段1：配置定义
1. **在config中定义ContextViewVo**：
   ```csharp
   // 示例配置
   new ContextViewVo
   {
       ViewCmptClassName = "LoginViewCmpt",
       MediatorClassName = "LoginViewMediator",
       IsPopView = true,
       FunType = ContextFunType.PopView,
       ZIndex = 100
   }
   ```

### 阶段2：请求发起
2. **发送UI创建通知**：
   ```csharp
   // 通过通知系统发送创建请求
   SendNoti(new CORE_ADD_SUB_LAYER_NOTI
   {
       ContextStr = "LoginView",
       Data = loginData,
       ParentContext = parentContext
   });
   ```

### 阶段3：GameMediator处理
3. **GameMediator接收并处理通知**：
   ```csharp
   case CORE_ADD_SUB_LAYER_NOTI.NAME:
   {
       ViewInfoClass notiData = new ViewInfoClass(note.getBody<CORE_ADD_SUB_LAYER_NOTI>());
       AddSubLayer(notiData);
   }
   ```

4. **LoadContext方法包装数据**：
   ```csharp
   private LayerBodyVo LoadContext(ViewInfoClass viewInfo)
   {
       // 1. 从ContextViewVo创建Context对象
       Context mainContext = new Context(viewInfo.Context);
       
       // 2. 构建父子关系
       while (contextArr.Count != 0)
       {
           Context childContext = contextArr.Shift();
           if (childContext.Parent == null)
           {
               mainContext = childContext;
           }
           else
           {
               childContext.Parent.AddChild(childContext);
           }
       }
       
       // 3. 创建LayerBodyVo
       LayerBodyVo bodyVo = new LayerBodyVo();
       bodyVo.Context = mainContext;
       bodyVo.ParentContext = viewInfo.ParentContext;
       
       return bodyVo;
   }
   ```

### 阶段4：命令执行
5. **发送LoadLayersCommand**：
   ```csharp
   SendCmd(new LoadLayersCommand.Vo { vo = nData });
   ```

6. **LoadLayersCommand执行**：
   ```csharp
   public override void onExecute(in BaseNotificationBody note)
   {
       LayerBodyVo layerVo = note.getBody<Vo>().vo;
       layerVo.Type = LayerBodyVo.TYPE.ADD;
       LayerProxy layerProxy = GetProxy<LayerProxy>();
       layerProxy.AddedToTheQueue(layerVo);
   }
   ```

### 阶段5：队列管理
7. **LayerProxy添加到队列**：
   ```csharp
   public void AddedToTheQueue(LayerBodyVo layerBody)
   {
       if (UIManager.EnableMainThreadAddLayer)
       {
           UIManager.AddLayerToQueue(layerBody, this);
       }
       else
       {
           // 添加到线程队列
           lock (_cacheWaitObj)
           {
               _cacheWaitObj.Add(layerBody);
           }
       }
   }
   ```

8. **UIManager队列处理**：
   ```csharp
   private async UniTaskVoid LoadLayerTask()
   {
       while (true)
       {
           var count = 0;
           while (AddLayerQueue.Count > 0 && count < MaxLayerQueueLoadCount)
           {
               var ok = AddLayerQueue.TryDequeue(out UILayerData data);
               if (ok && data != null)
               {
                   switch (data?.layerBodyVo.Type)
                   {
                       case LayerBodyVo.TYPE.ADD:
                           await data.proxy.MainAddLayerContext(data?.layerBodyVo);
                           break;
                   }
               }
           }
           await UniTask.Yield();
       }
   }
   ```

### 阶段6：UI实例化
9. **MainAddLayerContext执行**：
   ```csharp
   public async UniTask MainAddLayerContext(LayerBodyVo noteBody)
   {
       Context context = noteBody.Context;
       IMediator contextMediator = Facade.RetrieveMediator(context.MediatorName);
       
       // 检查是否已存在
       if (contextMediator != null && !noteBody.IsForceAddLayer)
       {
           return;
       }
       
       OnAddLayerBegin(noteBody);
       await MainAddLayer(context);
       OnAddLayerOver(noteBody);
   }
   ```

10. **MainLoadLayerWithContext处理单个Context**：
    ```csharp
    private async UniTask MainLoadLayerWithContext(Context childContext, List<Context> layerContextArr)
    {
        // 1. 检查Mediator是否已存在
        IMediator layerContextMediator = Facade.RetrieveMediator(childContext.MediatorName);
        if (layerContextMediator != null)
        {
            return;
        }
        
        // 2. 找到父Mediator
        BaseMediator parentMediator = null;
        if (parentContext != null)
        {
            parentMediator = Facade.GetMediator<BaseMediator>(parentContext.MediatorName);
        }
        else
        {
            Context curSceneContext = contextProxy.GetCurSceneContext();
            parentMediator = Facade.GetMediator<BaseMediator>(curSceneContext.MediatorName);
        }
        
        // 3. 获取父ViewComponent
        BaseCmpt parentViewComponent = parentMediator.ViewComponent;
        
        // 4. 调用MainOverlay进行实际加载
        await MainOverlay(parentViewComponent, className, childContext, className, onCmptFinish, onFinish, onFailed);
    }
    ```

### 阶段7：组件加载
11. **MainOverlay执行组件加载**：
    ```csharp
    private async UniTask MainOverlay(BaseCmpt parent, string childClsStr, Context childContext, string vcName, Action<BaseCmpt> onCmptFinish, Action<BaseCmpt> onFinish, Action<Exception> onFailed)
    {
        BaseCmpt _BaseCmpt = null;
        
        // 定义回调
        ITEmitter.ActionType onLoaded = delegate
        {
            if (_BaseCmpt != null)
            {
                _BaseCmpt.Attach(parent);  // 附加到父节点
                if (!_BaseCmpt.GetIsRemoveCmpt())
                {
                    _BaseCmpt.Emitter.On(BaseCmpt.DID_ENTER, didEnter);
                    onCmptFinish(_BaseCmpt);  // 组件完成回调
                    _BaseCmpt.Enter();        // 进入
                    parent.OnAddLayer(_BaseCmpt);
                }
            }
            return null;
        };
        
        // 根据类型加载
        Type childType = ReflectionUtils.FindTypeOnAllAssembly(childClsStr);
        if (childType.BaseType == typeof(BaseWindow))
        {
            BaseWindow.Load(childType, basecmpt =>
            {
                _BaseCmpt = basecmpt;
                basecmpt.Emitter.On(BaseCmpt.LOADED, onLoaded);
                basecmpt.OnUILoaded(basecmpt.gameObject);
            });
        }
        else
        {
            BaseCmpt.Load(childType, basecmpt =>
            {
                _BaseCmpt = basecmpt;
                basecmpt.Emitter.On(BaseCmpt.LOADED, onLoaded);
                basecmpt.OnUILoaded(basecmpt.gameObject);
            }, cmptAttribute, vcName);
        }
    }
    ```

### 阶段8：Mediator绑定
12. **onCmptFinish回调中的Mediator绑定**：
    ```csharp
    delegate (BaseCmpt viewComponent)
    {
        // 1. 设置Context状态
        childContext.Step = ContextLoadStep.ViewWillAppear;
        viewComponent.ViewComponentClassName = className;
        
        // 2. 创建Mediator实例
        Type mediatorType = ReflectionUtils.FindTypeOnAllAssembly(childContext.MediatorClassStr);
        BaseMediator mediator = (BaseMediator)mediatorType.Assembly.CreateInstance(childContext.MediatorClassStr);
        
        // 3. 缓存ViewComponent
        ViewCache.AddPrefabUiNodeCache(className, viewComponent.gameObject, childContext.CmptAttribute.CacheGameObjSec);
        
        // 4. 绑定关系
        mediator.ViewComponent = viewComponent;
        mediator.Context = childContext;
        mediator.MediatorName = childContext.MediatorName;
        
        // 5. 注册Mediator
        Facade.RegisterMediator(mediator);
        
        // 6. 调用Register方法
        mediator.Register();
        
        // 7. 处理子视图
        var addSubView = mediator.OnAddSubView();
        if (addSubView != null)
        {
            layerContextArr.AddRange(addSubView);
        }
        
        // 8. 调用ViewWillAppear
        mediator.ViewWillAppear();
    }
    ```

### 阶段9：Attach操作
13. **BaseCmpt.Attach执行**：
    ```csharp
    public virtual void Attach(BaseCmpt parent)
    {
        if (!parent) return;
        
        GameObject parentGo = parent.gameObject;
        if (_attribute.CustomParentGameObject != null)
        {
            parentGo = (GameObject)_attribute.CustomParentGameObject;
        }
        
        // 根据父节点类型处理
        if (_attribute.ParentType == BASE_CMPT_PARENT_TYPE.FGUI && gameObject.GetComponent<UIPanel>() != null)
        {
            // FGUI Panel处理
            DisplayObject goDisplayObj = SafeGetFGuiPanelContainer(gameObject);
            GObject parentDisplayObj = SafeGetFGuiObjectInfo(parentGo);
            
            if (goDisplayObj != null && parentDisplayObj.displayObject is Container ap)
            {
                goDisplayObj.RemoveFromParent();
                ap.AddChild(goDisplayObj);
                NeedReCheckScale(forceResetScale);
            }
        }
        else if (_attribute.ParentType == BASE_CMPT_PARENT_TYPE.FGUI && prefabPath == null)
        {
            // FGUI组件处理
            GObject goDisplayObj = SafeGetFGuiObjectInfo(gameObject);
            GObject parentDisplayObj = SafeGetFGuiObjectInfo(parentGo);
            
            if (goDisplayObj != null && parentDisplayObj is GComponent p)
            {
                goDisplayObj.RemoveFromParent();
                p.AddChild(goDisplayObj);
                UpdateDynamicObjectRelation(goDisplayObj, goDisplayObj.parent);
            }
        }
        else
        {
            // 普通GameObject处理
            gameObject.transform.SetParent(parentGo.transform);
            NeedReCheckScale(forceResetScale);
        }
    }
    ```

### 阶段10：完成回调
14. **onFinish回调执行**：
    ```csharp
    delegate
    {
        BaseMediator mediator = GetMediatorWithName<BaseMediator>(childContext.MediatorName);
        
        // 1. 调用ViewDidAppear
        mediator.ViewDidAppear();
        childContext.Step = ContextLoadStep.ViewDidAppear;
        
        // 2. 处理全屏视图
        if (childContext != null && childContext.isFullScreenView())
        {
            contextProxy.AddForceCocosMediatorName(childContext.MediatorName, 2, childContext.DefaultBgColor, SendJumpMsg: false);
        }
        
        // 3. 设置Panel顺序
        if (childContext.CmptAttribute.ParentType != BASE_CMPT_PARENT_TYPE.FGUI && mediator.ViewComponent.gameObject.GetComponent<UIPanel>() != null)
        {
            mediator.ViewComponent.SetPanelOrder(childContext.ZIndex);
        }
        
        // 4. 检查子Context是否全部完成
        CheckChildContextAllFinish(childContext);
    }
    ```

## 生命周期管理

### Mediator生命周期
1. **创建阶段**：MainAddLayerContext
   - 通过反射创建Mediator实例
   - 设置ViewComponent、Context、MediatorName

2. **注册阶段**：
   - `Facade.RegisterMediator(mediator)`
   - `mediator.Register()`
   - 绑定事件监听器
   - 注册通知监听

3. **出现阶段**：
   - `mediator.ViewWillAppear()`
   - `mediator.ViewDidAppear()`

4. **运行阶段**：
   - 处理用户交互
   - 响应通知
   - 管理子视图

5. **移除阶段**：
   - `mediator.OnRemove()`
   - 清理事件监听
   - 移除通知监听
   - `mediator.ViewDidDisAppear()`

### ViewComponent生命周期
1. **加载阶段**：
   - `BaseCmpt.Load()` 或 `BaseWindow.Load()`
   - 加载预制体或FGUI组件
   - `OnUILoaded()`

2. **初始化阶段**：
   - `Init()`
   - 设置初始状态

3. **附加阶段**：
   - `Attach(parent)`
   - 建立父子关系

4. **进入阶段**：
   - `Enter()`
   - `OnEnter()`
   - `OnActive()`

5. **显示阶段**：
   - `ViewWillAppear()`
   - `ViewDidAppear()`
   - `OnShow()`

6. **隐藏阶段**：
   - `OnHide()`
   - `ViewDidDisAppear()`

7. **退出阶段**：
   - `Exit()`
   - `OnUnActive()`
   - `OnExit()`

8. **分离阶段**：
   - `Detach()`
   - 清理资源

## 缓存机制

### ViewCache（视图缓存）
```csharp
// 缓存管理
ViewCache.AddPrefabUiNodeCache(className, viewComponent.gameObject, cacheTime);
GameObject cacheNode = ViewCache.GetPrefabIdleUiNode(vcName);
ViewCache.SetPrefabUiNodeClose(ViewComponentClassName, gameObject);
```

**缓存策略**：
- 根据`CacheGameObjSec`设置缓存时间
- 支持节点复用，避免重复创建
- 自动管理缓存节点的生命周期

### 资源缓存
- FGUI包缓存
- 预制体缓存
- 图集缓存

## 事件系统

### ITEmitter（事件发射器）
```csharp
public class ITEmitter
{
    public void On(string eventStr, ActionType callback)      // 监听事件
    public void Emit(string eventStr, params object[] pars)  // 发射事件
    public void RemoveEventListener(string eventStr, ActionType callback)  // 移除监听
    public void RemoveAllListeners()                         // 移除所有监听
}
```

### 标准事件常量
```csharp
public const string LOADED = "BaseViewComponent:LOADED";
public const string DID_ENTER = "BaseViewComponent:DID_ENTER";
public const string DID_EXIT = "BaseViewComponent:DID_EXIT";
public const string ON_BACK = "BaseViewComponent:ON_BACK";
public const string ON_CLOSE = "BaseViewComponent:ON_CLOSE";
public const string ON_SHOW = "BaseViewComponent:ON_SHOW";
public const string ON_HIDE = "BaseViewComponent:ON_HIDE";
```

## 通知系统

### PureMVC通知机制
```csharp
// 发送通知
SendNoti(new CORE_ADD_SUB_LAYER_NOTI { ... });

// 监听通知
public override string[] ListNotificationInterests()
{
    return new[] { CORE_ADD_SUB_LAYER_NOTI.NAME };
}

public override void OnHandleNotification(in BaseNotificationBody note)
{
    switch (note.Name)
    {
        case CORE_ADD_SUB_LAYER_NOTI.NAME:
            // 处理通知
            break;
    }
}
```

## 多线程处理

### 主线程UI操作
- `UIManager.EnableMainThreadAddLayer = true` 时，所有UI操作在主线程执行
- 通过`UniTask`实现异步操作
- 队列机制控制每帧处理的UI数量

### 后台通知处理
```csharp
public override string[] ListBackgroundNotificationInterests()
{
    return new[] { CORE_BACKGROUND_ADD_SUB_LAYER_NOTI.NAME };
}

public override void OnHandleBackGroundNotification(in BaseNotificationBody note)
{
    // 处理后台通知
}
```

## 特殊视图处理

### 弹窗视图
- 通过`ContextFunType.PopView`标识
- 特殊的层级管理
- 支持弹窗栈管理

### 全屏视图
- 通过`ContextFunType.FullScreen`标识
- 与原生端交互
- 特殊的背景处理

### 窗口管理
- 支持多窗口系统
- 窗口上下文栈管理
- 窗口层级控制

## 性能优化

### 异步加载
- 使用`UniTask`实现异步操作
- 避免主线程阻塞
- 支持超时控制

### 对象池
- ViewComponent缓存复用
- 减少GC压力
- 提高创建效率

### 资源管理
- 自动资源引用计数
- 智能资源释放
- 内存泄漏防护

## 错误处理

### 异常捕获
```csharp
private void CatchErrorProcess(Exception e)
{
    // 记录错误信息
    // 清理相关资源
    // 恢复系统状态
}
```

### 超时处理
```csharp
private TimeSpan LoadOverlayTimeoutTimeSpan = TimeSpan.FromSeconds(10);
await UniTask.WaitUntil(CheckCurrentLayerLoadComplete, PlayerLoopTiming.EarlyUpdate, LoadOverlayTimeoutController.Timeout(LoadOverlayTimeoutTimeSpan));
```

## 调试支持

### 日志系统
- 详细的加载日志
- 性能统计
- 错误追踪

### 性能监控
```csharp
public Action<string, string, long, int> PopViewSlowCb;  // 弹窗性能回调
```

## 总结

这个UI框架通过以下关键特性实现了ViewComponent与Mediator的完美关联：

1. **配置驱动**：通过ContextViewVo配置UI结构
2. **异步加载**：支持异步UI创建和加载
3. **生命周期管理**：完整的UI生命周期控制
4. **事件系统**：强大的事件通信机制
5. **缓存优化**：智能的缓存和复用机制
6. **多线程支持**：主线程UI操作，后台通知处理
7. **错误处理**：完善的异常处理和恢复机制

框架的核心价值在于将复杂的UI管理抽象为简单的配置和操作，同时保证了高性能和可维护性。
