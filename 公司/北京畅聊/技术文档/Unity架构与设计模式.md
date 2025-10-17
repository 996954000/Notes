# Unity架构与设计模式知识总结

## 目录
- [FGUI UI加载机制](#fgui-ui加载机制)
  - [UIManager启动与预加载机制](#1-uimanager启动与预加载机制)
  - [FGUI资源预加载流程](#2-fgui资源预加载流程)
  - [编辑器模式与运行模式的资源加载](#3-编辑器模式与运行模式的资源加载)
  - [UIPackageDataCachePool缓存机制](#4-uipackagedatacachepool缓存机制)
  - [UI层加载任务](#5-ui层加载任务loadlayertask)
  - [预加载机制总结](#6-预加载机制总结)
  - [FairyGUI 资源加载机制详解](#7-fairygui-资源加载机制详解)
  - [UI窗口分类管理系统](#8-ui窗口分类管理系统)
- [Lib文件夹架构](#lib文件夹架构)
- [MonoSingleton设计模式](#monosingleton设计模式)
- [位掩码技术](#位掩码技术)
- [代理模式与生命周期管理](#代理模式与生命周期管理)

---

## FGUI UI加载机制

### 项目中的FGUI架构
这个Unity项目采用了**MVC架构模式**，与FGUI官方基础用法不同：

#### 核心组件
- **Mediator（中介者）**：负责UI逻辑控制
- **Component（组件）**：负责UI显示和交互
- **Proxy（代理）**：负责数据管理
- **UIManager**：负责UI生命周期管理

#### UI加载流程

##### 1. UIManager启动与预加载机制

**启动时序**：
```csharp
// UIManager在游戏开始时通过单例代理获取Instance
public class UIManager : MonoSingleton<UIManager>
{
    protected override void OnStart()
    {
        StartLoadLayerTask().Forget();  // 启动UI加载任务
    }
}
```

**预加载等待机制**：
```csharp
private async UniTaskVoid StartLoadLayerTask()
{
    // 等待FGUI包预加载完成，最多等待30秒
    var cts = new CancellationTokenSource();
    cts.CancelAfterSlim(TimeSpan.FromSeconds(30));
    
    try
    {
        // 等待preLoadAllPackage标志位变为true
        await UniTask.WaitUntil(() => { 
            return FGUIAssetManager.PreLoadAllPackageIsFinish() == true; 
        }, cancellationToken: cts.Token);
        
        LogAPI.TagD(TAG, "UI Package预加载完成");
    }
    catch (OperationCanceledException ex)
    {
        LogAPI.TagE(TAG, "UI Package预加载超时!");
    }
    
    LoadLayerTask().Forget();  // 开始UI层加载任务
}
```

##### 2. FGUI资源预加载流程

**FGUIAssetManager初始化**：
```csharp
public async UniTask init()
{
    // 注册资源管理事件
    BaseCmpt.OnEnterCmptLoadedAsset += OnAssetOwnerAdd;
    BaseCmpt.OnExitCmptRemoveAsset += OnAssetOwnerRemove;
    GComponent.OnGComponetDispose += OnAssetOwnerRemove;
    
    // 异步加载FGUI包（只加载bytes文件）
    await AssetManager.Instance.LoadFGUIPackages();
    
    // 同步加载FGUI Shader
    AssetManager.Instance.LoadFGUIShader();
    
    // 加载基础UI GameObject
    LoadBaseUIGameObject();
    
    // 启动UI包资源检查任务
    CheckUIPackageAsset().Forget();
}
```

**重要说明**：
- `LoadFGUIPackages`是**异步**的，会等待完成
- `LoadFGUIShader`是**同步**的，但内部调用`SetFGUIShaderData`是异步的
- 初始化顺序：先加载包，再加载shader，最后加载基础UI

**preLoadAllPackage标志位控制**：
```csharp
// 在SetFGUIShaderData方法中设置标志位
public static async UniTask SetFGUIShaderData(FGUIShadersScriptableObject data)
{
    if (!data)
    {
        LogAPI.TagE(Tag, "FGUIShadersScriptableObject 加载失败");
    }
    
    FGUIShaderData = data;
    FGUIShaders = FGUIShaderData.GetAllShader();
    ShaderConfig.Get = GetFGUIShader;
    
    // 注册字体
    await AssetManager.Instance.RegFont();
    
    // 关键：shader加载完成后立即设置预加载完成标志
    preLoadAllPackage = true;
}
```

**重要说明**：
- `preLoadAllPackage`标志位在**shader加载完成后**就设置为true
- 由于`LoadFGUIPackages`在`LoadFGUIShader`之前执行且是异步等待的，所以包加载实际上已经完成
- 这种设计确保UI系统在shader和包都就绪后才开始工作

##### 3. 编辑器模式与运行模式的资源加载

**编辑器模式（EditorAssetLoader）**：
```csharp
public void LoadAllFGUIPackages()
{
    // 通过AssetDatabase查找所有_fui.bytes文件
    string[] ids = AssetDatabase.FindAssets("_fui t:textAsset");
    var data = new StringDictionary<byte[]>(ids.Length);

    foreach (var id in ids)
    {
        string assetPath = AssetDatabase.GUIDToAssetPath(id);
        int pos = assetPath.LastIndexOf("_fui", StringComparison.Ordinal);
        if (pos == -1) continue;

        assetPath = assetPath.Substring(0, pos);
        TextAsset asset = AssetDatabase.LoadAssetAtPath<TextAsset>($"{assetPath}_fui.bytes");
        if (asset)
        {
            // 直接加载bytes数据到缓存池
            data[$"{asset.name}.bytes"] = asset.bytes;
            FGUIAssetManager.Instance.UIPackageDataCachePool?.Add($"{asset.name}.bytes", asset.bytes);
        }
    }
}
```

**运行模式（AddressableLoader）**：
```csharp
public async UniTask LoadAllFGUIPackages()
{
    // 通过Addressables加载FGUIPackagesScriptableObject
    AsyncOperationHandle<FGUIPackagesScriptableObject> bytesHandle = 
        Addressables.LoadAssetAsync<FGUIPackagesScriptableObject>("FairyGUIPackage/FGUIPackagesScriptableObjectAsset.asset");
    
    await bytesHandle;
    
    if (bytesHandle.Status == AsyncOperationStatus.Succeeded)
    {
        var obj = bytesHandle.Result;
        var data = obj.GetDatas();  // 获取所有包的bytes数据
        
        // 将所有bytes数据存入缓存池
        var packagepool = FGUIAssetManager.Instance.UIPackageDataCachePool;
        if (packagepool != null)
        {
            foreach (var item in data)
            {
                packagepool.Add(item.Key, item.Value);
            }
        }
    }
    
    bytesHandle.Release();
}
```

##### 4. UIPackageDataCachePool缓存机制

**缓存池设计**：
```csharp
public sealed class FGUIPackageDataCachePool
{
    private readonly StringDictionary<byte[]> _dictionary;
    
    public FGUIPackageDataCachePool(int capacity)
    {
        _dictionary = new StringDictionary<byte[]>(capacity);  // 默认容量1000
    }
    
    // 添加缓存：键格式为 "{packageName}.bytes"
    public void Add(string key, byte[] value)
    {
        _dictionary[key] = value;
    }
    
    // 获取缓存：直接返回bytes数据
    public byte[] Get(string key)
    {
        _dictionary.TryGetValue(key, out var result);
        return result;
    }
}
```

**缓存机制的优势**：
- **预加载优化**：提前加载所有UI包的结构信息（bytes文件）
- **运行时加速**：使用时直接从缓存获取bytes数据，避免重复加载
- **内存管理**：统一管理UI包数据，便于内存控制和释放
- **性能提升**：减少运行时的IO操作和解析开销

##### 5. UI层加载任务（LoadLayerTask）

**主循环处理**：
```csharp
private async UniTaskVoid LoadLayerTask()
{
    while (true)
    {
        // 等待场景上下文加载完成
        await WaitSceneContexLoadFinish();
        
        // 优先处理Window层队列
        while (AddWindowLayerQueue.Count > 0)
        {
            var ok = AddWindowLayerQueue.TryDequeue(out UILayerData data);
            if (ok && data != null && data.proxy != null && data.layerBodyVo != null)
            {
                OnAddLayerBengin(data);
                await data.proxy.MainAddLayerContext(data?.layerBodyVo);
                OnAddLayerOver(data);
            }
        }

        // 处理普通层队列（每帧最多处理MaxLayerQueueLoadCount个，默认50个）
        var count = 0;
        while (AddLayerQueue.Count > 0 && count < MaxLayerQueueLoadCount)
        {
            var ok = AddLayerQueue.TryDequeue(out UILayerData data);
            if (ok && data != null && data.proxy != null && data.layerBodyVo != null)
            {
                await WaitSceneContexLoadFinish();
                await WaitLayerRemove(data);  // 等待移除操作完成

                switch (data?.layerBodyVo.Type)
                {
                    case LayerBodyVo.TYPE.ADD:
                        OnAddLayerBengin(data);
                        await data.proxy.MainAddLayerContext(data?.layerBodyVo);
                        OnAddLayerOver(data);
                        break;
                    case LayerBodyVo.TYPE.REMOVE:
                        data.proxy.MainRemoveLayerContext(data?.layerBodyVo);
                        break;
                    case LayerBodyVo.TYPE.SHOW:
                        data.proxy.MainShowLayerContext(data?.layerBodyVo);
                        break;
                    case LayerBodyVo.TYPE.HIDE:
                        data.proxy.MainHideLayerContext(data?.layerBodyVo);
                        break;
                }
                count++;
            }
        }

        await UniTask.Yield();  // 每帧执行一次
    }
}
```

**队列管理机制**：
```csharp
// 添加UI层到队列
public static void AddLayerToQueue(LayerBodyVo context, LayerProxy proxy)
{
    LogAPI.TagM(TAG, $"UI事件进入主线程等待执行 LayerToQueue TYPE :{context?.Type} MediatorName: {context?.Context?.MediatorName}");
    
    // 检查是否有等待加载的PopView
    if (!ExitWaitLoadQueuePopView())
    {
        // 队列中第一个pop界面立即发送打开消息
        proxy.SendOpenNativeJump(context.Context);
    }
    
    var data = new UILayerData(context, proxy);
    AddLayerQueue.Enqueue(data);

    // 记录全屏视图的加载时间
    if (context is { Type: LayerBodyVo.TYPE.ADD, Context: not null } && context.Context.isFullScreenView())
    {
        var info = new LoadingDebugInfo();
        info.StartLonagingTime = TimeHelper.LocalMs();
        info.StartLonagingFrameCount = Time.frameCount;
        MediatorLoadingTime[context.Context.MediatorName] = info;
    }
}
```

##### 6. 预加载机制总结

**完整的预加载流程**：
1. **游戏启动** → UIManager.Instance获取 → OnStart()调用
2. **StartLoadLayerTask启动** → 等待preLoadAllPackage标志位
3. **FGUIAssetManager.init()** → 异步等待LoadFGUIPackages完成（Addressables的资源加载是异步的） + 同步调用LoadFGUIShader
4. **SetFGUIShaderData完成** → 设置preLoadAllPackage = true
5. **StartLoadLayerTask继续** → 启动LoadLayerTask主循环
6. **LoadLayerTask运行** → 处理UI层队列，创建和显示UI

**关键设计特点**：
- **顺序预加载**：LoadFGUIPackages先异步等待完成，再加载LoadFGUIShader
- **完整预加载**：preLoadAllPackage在shader和包都加载完成后才设置为true
- **缓存优化**：UIPackageDataCachePool预缓存所有UI包的bytes数据（容量1000）
- **队列管理**：通过队列机制确保UI操作在主线程顺序执行（每帧最多50个）
- **性能监控**：记录UI加载时间，便于性能优化

**预加载的数据内容**：
- **bytes文件**：包含UI界面的组件结构、布局信息等
- **Shader数据**：FGUI渲染所需的着色器资源
- **字体资源**：UI文本显示所需的字体文件
- **基础UI**：BaseUI.prefab等基础UI组件

**运行时使用**：
当需要显示UI时，系统会：
1. 从UIPackageDataCachePool获取预缓存的bytes数据
2. 通过FairyGUI解析bytes数据创建UI组件
3. 加载具体的纹理、音频等资源（按需加载）
4. 显示UI界面

这种设计大大减少了UI显示时的加载时间，提升了用户体验。

##### 7. FairyGUI 资源加载机制详解

**WaitFGUITextureLoad 函数的作用**：
`WaitFGUITextureLoad` 是 FairyGUI 纹理资源加载的回调函数，主要用于：

```csharp
async void WaitFGUITextureLoad(string name, string extension, Type type, PackageItem ite)
{
    if (type == typeof(Texture))
    {
#if USE_EDITOR_LOADER
        // 编辑器模式：使用 AssetDatabase 从本地路径加载
        string imgPath = $"Assets/Res/FairyGUI/{name}{extension}";
        Texture t = (Texture)AssetDatabase.LoadAssetAtPath(imgPath, typeof(Texture));
        if (!t)
        {
            ite.owner.SetItemAssetOnLoadFail(ite, t, DestroyMethod.Custom);
            return;
        }
        ite.owner.SetItemAssetOnLoadOver(ite, t, DestroyMethod.Custom);
#else
        // 运行模式：使用 Addressables 异步加载
        Texture t = await Addressables.LoadAssetAsync<Texture>(name + extension).Task;
        if (!t)
        {
            ite?.owner?.SetItemAssetOnLoadFail(ite, t, DestroyMethod.Custom);
            return;
        }
        ite?.owner?.SetItemAssetOnLoadOver(ite, t, DestroyMethod.Custom);
#endif
    }
}
```

**参数说明**：
- `name`: 纹理资源的名称
- `extension`: 文件扩展名
- `type`: 资源类型（只处理 Texture 类型）
- `ite`: FairyGUI 的 PackageItem 对象

**使用场景**：
该函数作为回调传递给 `UIPackage.AddPackage()` 方法：
```csharp
package = UIPackage.AddPackage(pkgAsset, packageName, WaitFGUITextureLoad);
```

**FairyGUI 包中其他资源的加载机制**：

**1. 纹理资源** - 需要自定义加载回调
- 通过 `WaitFGUITextureLoad` 回调处理
- 编辑器模式使用 `AssetDatabase.LoadAssetAtPath`
- 运行模式使用 `Addressables.LoadAssetAsync`

**2. 其他资源类型** - 由 FairyGUI 内部自动处理

**音频资源（AudioClip）**：
- FairyGUI 内部自动处理音频资源加载
- 当 UI 组件需要播放音效时自动从包中加载
- 无需开发者提供自定义加载回调

**字体资源（Font）**：
- 通过 `AssetManager.Instance.RegFont()` 预注册
- 在 `FGUIAssetManager.SetFGUIShaderData()` 中调用
- 字体资源在包加载时自动关联

**材质和着色器（Material/Shader）**：
- 通过 `FGUIAssetManager.LoadFGUIShader()` 预加载
- 存储在 `FGUIShaders` 字典中
- 通过 `ShaderConfig.Get` 提供访问接口

**完整的资源加载流程**：
```csharp
// 1. 加载包描述文件（.fui）
var date = await AsyncLoadUIPackage(address);

// 2. 创建包，只提供纹理加载回调
package = UIPackage.AddPackage(date, packageName, WaitFGUITextureLoad);

// 3. 其他资源由 FairyGUI 内部自动处理
// - 音频：自动从包中加载
// - 字体：通过预注册的字体系统
// - 材质：通过预加载的着色器系统
```

**为什么只看到纹理回调？**

**原因分析**：
1. **纹理是最主要的资源**：UI 界面主要由图片组成
2. **其他资源使用频率较低**：音频、字体等资源相对较少
3. **FairyGUI 内部优化**：其他资源类型由 FairyGUI 内部自动管理，性能更好
4. **项目设计策略**：采用预加载 + 自动管理的混合策略

**自定义 GLoader 扩展**：
项目还实现了 `FairyGUILoader` 类来扩展 GLoader 的功能：

```csharp
public class FairyGUILoader : GLoader, IFairyCache
{
    protected override void LoadExternal()
    {
        if (url.StartsWith("https://") || url.StartsWith("http://"))
        {
            // HTTP 资源加载
            ImageLoader.Loader(url, (sp, resUrl, errMsg, index) => {
                OnLoadCompleteByType(FairyGUILoadType.HTTP, sp);
            });
        }
        else if (url.StartsWith("Unity_Atlas://"))
        {
            // Unity 图集资源加载
            string newUrl = url.Replace("Unity_Atlas://", "");
            string[] paths = newUrl.Split('/');
            if (paths.Length > 1)
            {
                string atlasName = paths[0];
                string spritName = paths[1];
                AssetManager.Instance.LoadAtlasSprite(atlasName, spritName, sprite => {
                    OnLoadCompleteByType(FairyGUILoadType.ATLAS, sprite);
                });
            }
        }
        else
        {
            // 普通纹理资源加载
            AssetManager.Instance.LoadAssetAsync<Texture2D>(url, (resUrl, tex) => {
                OnLoadCompleteByType(FairyGUILoadType.NORMAL, tex, url);
            });
        }
    }
}
```

**资源加载的层次结构**：
- **纹理资源**：需要自定义加载回调（`WaitFGUITextureLoad`）
- **其他资源**：由 FairyGUI 内部自动处理，无需开发者干预

这种设计既保证了灵活性（可以自定义纹理加载），又简化了开发复杂度（其他资源自动管理）。

##### 8. UI窗口分类管理系统

**双队列设计**：
UIManager使用两个队列来管理不同类型的UI操作：

```csharp
// Window层队列 - 高优先级，处理系统级窗口
private static readonly ConcurrentQueue<UILayerData> AddWindowLayerQueue = new ConcurrentQueue<UILayerData>();

// 普通层队列 - 处理所有其他UI操作
private static readonly ConcurrentQueue<UILayerData> AddLayerQueue = new ConcurrentQueue<UILayerData>();
```

**5个核心窗口容器**：
游戏启动时会加载5个系统级窗口容器，用于分类管理不同类型的UI：

   ```csharp
// 在Entrance.cs中加载所有Window容器
foreach (string name in WINDOW.indexArr)  // ["KEY", "POPUP", "GLOBAL_ANIM", "ALERT", "TUTORIAL"]
{
    ContextViewVo config = WINDOW.Config[name];
    facade.SendNoti(new CORE_ADD_WINDOW_NOTI(config, name));
}
```

**窗口容器详细说明**：

1. **KEY窗口** - 主页面UI组件容器
   ```csharp
   { KEY, new ContextViewVo("ITClient.KeyWindow", "ITClient.KeyWindowMediator", ITBase.GuiZIndex.WindowKey) }
   ```
   - **用途**：管理主页面常驻UI组件
   - **包含内容**：上方资源显示bar、右边功能按钮、底部导航栏、侧边栏按钮等
   - **特点**：常驻显示，不会因打开其他界面而消失

2. **POPUP窗口** - 功能页面容器
   ```csharp
   { Popup, new ContextViewVo("ITClient.PopupWindow", "ITClient.PopupWindowMediator", ITBase.GuiZIndex.WindowPopup) }
   ```
   - **用途**：管理各种功能页面
   - **包含内容**：背包界面、商店界面、设置界面、好友列表、任务界面等
   - **特点**：完整的页面界面，需要用户主动打开和关闭

3. **ALERT窗口** - 提示弹窗容器
   ```csharp
   { ALERT, new ContextViewVo("ITClient.AlertWindow", "ITClient.AlertWindowMediator", ITBase.GuiZIndex.WindowAlert) }
   ```
   - **用途**：管理系统提示和确认弹窗
   - **包含内容**：人物武力变化提示、升级成功提示、错误信息、确认对话框等
   - **特点**：临时性提示信息，通常会自动消失或需要用户确认

4. **GLOBAL_ANIM窗口** - 全局动画容器
   ```csharp
   { GLOBAL_ANIM, new ContextViewVo("ITClient.GlobalAnimWindow", "ITClient.GlobalAnimWindowMediator", ITBase.GuiZIndex.GlobalAnim) }
   ```
   - **用途**：管理全局动画效果
   - **包含内容**：主页面转场动画、货物收取动画、资源fly动画、升级特效等
   - **特点**：纯动画效果，不包含交互逻辑，主要用于视觉反馈

5. **TUTORIAL窗口** - 教程引导容器
   ```csharp
   { TUTORIAL, new ContextViewVo("ITClient.TutorialWindow", "ITClient.TutorialWindowMediator", ITBase.GuiZIndex.WindowTutorial) }
   ```
   - **用途**：管理新手引导和教程
   - **包含内容**：高亮提示框、箭头指引、遮罩层、操作提示文字等
   - **特点**：教学引导UI，帮助新用户了解游戏操作

**层级关系**：
```
TUTORIAL (最顶层 - 教程引导)
    ↓
ALERT (警告提示层 - 系统消息)
    ↓  
POPUP (功能页面层 - 业务界面)
    ↓
KEY (主页面UI层 - 常驻组件)
    ↓
GLOBAL_ANIM (动画效果层 - 视觉特效)
    ↓
场景背景 (最底层)
```

**使用示例**：
```csharp
// 显示到不同的窗口容器
ShowPopupCmpt("BagView", data, null, AppFacade.POPUP_WINDOW);    // 显示到功能页面容器
ShowPopupCmpt("AlertView", data, null, AppFacade.ALERT_WINDOW);  // 显示到警告容器
ShowPopupCmpt("TutorialView", data, null, AppFacade.TUTORIAL);   // 显示到教程容器
```

**设计优势**：
- **层级清晰**：不同类型的UI有明确的层级关系
- **管理方便**：可以批量操作同一类型的UI（如关闭所有弹窗）
- **性能优化**：动画和UI分离，便于性能管理
- **用户体验**：确保重要提示不会被其他UI遮挡
- **开发维护**：代码结构清晰，便于团队协作

##### 9. 通过UIManager加载UI
```csharp
// UIManager负责管理UI的加载队列
public static void AddLayerToQueue(LayerBodyVo context, LayerProxy proxy)
{
    var data = new UILayerData(context, proxy);
    AddLayerQueue.Enqueue(data);
}
```

##### 10. Mediator创建和初始化
   ```csharp
   public class RoleEquipFiveElemRefreshMediator : BaseMediator
   {
       public override void OnDidRegister()
       {
           base.OnDidRegister();
           _cmpt = GetViewComponent<RoleEquipFiveElemRefreshCmpt>();
           _proxy = GetProxy<RoleRefreshProxy>();
           // 绑定事件和初始化逻辑
       }
   }
   ```

#### FGUI组件加载方式

**方式一：异步加载**
```csharp
// AssetManager.LoadFairyGuiGComponent
public void LoadFairyGuiGComponent(string packageName, string resName, 
    in ActionClosure onLoadComplate, bool awaitAsyncLoad = false)
{
    string address = $"{packageName}_fui.bytes";
    LoadFairyGuiGComponentTask(address, packageName, resName, onLoadComplate, awaitAsyncLoad).Forget();
}
```

**方式二：同步加载**
```csharp
// AssetManager.SyncLoadFairyGuiGComponent
public GComponent SyncLoadFairyGuiGComponent(string packageName, string resName)
{
    string address = $"{packageName}_fui.bytes";
    return SyncLoadFairyGuiGComponent(address, packageName, resName);
}
```

#### FGUI包加载机制
```csharp
// 同步加载FGUI包
public void SyncLoadFGUIPackage(string address, string packageName)
{
    if (UIPackage.CheckPackageCreated(packageName))
        return;
        
    var pkgAsset = SyncLoadUIPackage(address);
    if (pkgAsset == null)
        return;
        
    // 添加包到FGUI系统
    UIPackage package = UIPackage.AddPackage(pkgAsset, packageName, WaitFGUITextureLoad);
    
    // 递归加载依赖包
    if (package != null && package.dependencies != null && package.dependencies.Length > 0)
    {
        foreach (var kv in package.dependencies)
        {
            kv.TryGetValue("name", out string depPackageName);
            if (!string.IsNullOrEmpty(depPackageName))
            {
                SyncLoadFGUIPackage($"{depPackageName}_fui.bytes", depPackageName);
            }
        }
    }
}
```

#### 与FGUI官方用法的区别

**FGUI官方用法**：
```csharp
// 官方基础用法
UIPackage.AddPackage("包名");
GComponent view = UIPackage.CreateObject("包名", "组件名").asCom;
GRoot.inst.AddChild(view);
```

**项目中的用法**：
```csharp
// 项目中的用法
// 1. 通过AssetManager加载
AssetManager.Instance.LoadFairyGuiGComponent("包名", "组件名", callback);

// 2. 通过Mediator管理
public class SomeMediator : BaseMediator
{
    private SomeCmpt _cmpt;
    
    public override void OnDidRegister()
    {
        _cmpt = GetViewComponent<SomeCmpt>();
    }
}
```

---

## Lib文件夹架构

### Lib文件夹的作用
`Lib`文件夹是这个Unity项目的**核心基础库**，包含了项目的基础框架和通用功能模块。

#### 目录结构
```
Scripts/
├── Lib/           # 基础库层 (ITLib程序集)
├── Base/          # 基础数据层 (ITBase程序集)  
├── Client/        # 客户端业务层 (ITClient程序集)
├── Config/        # 配置层
└── SeaWarRes/     # 海战资源层
```

#### Assembly Definition (程序集定义)
最关键的是`ITLib.asmdef`文件，这定义了Lib文件夹作为一个**独立的程序集**：

```json
{
    "name": "ITLib",
    "rootNamespace": "ITLib",
    "references": [
        "Plugins",
        "Unity.Addressables",
        "Unity.Mathematics",
        // ... 其他依赖
    ],
    "allowUnsafeCode": true,
    "autoReferenced": true
}
```

#### 核心模块
- **AssetManager**: 资源管理系统
- **Context**: MVC框架核心（PureMVC）
- **Camera**: 相机管理系统
- **Http**: 网络请求模块
- **Util**: 工具类集合
- **ClassEx**: 扩展方法
- **Behavior3**: AI行为树系统
- **Spine**: 2D动画系统
- **Pool**: 对象池系统

#### Rider中的特殊标识
Rider会识别`.asmdef`文件，并在IDE中显示特殊标识：
- **程序集图标**: 显示这是一个独立的程序集
- **命名空间标识**: 显示`ITLib`命名空间
- **依赖关系**: 显示与其他程序集的引用关系
- **编译标识**: 显示编译状态和错误

---

## MonoSingleton设计模式

### 什么是MonoSingleton
`MonoSingleton<T>`是一个**泛型单例模式**的实现，专门为Unity设计的单例基类。它结合了：
- **单例模式**：确保整个应用程序中只有一个实例
- **MonoBehaviour特性**：可以访问Unity的生命周期方法
- **自动管理**：自动处理创建、初始化和销毁

### 核心实现原理
```csharp
public abstract class MonoSingleton<T> where T : MonoSingleton<T>, new()
{
    private static T m_Instance;
    
    public static T Instance
    {
        get
        {
            if (m_Instance != null)
                return m_Instance;
                
            // 懒加载创建实例
            m_Instance = new T();
            m_Instance.Init();
            m_Instance.OnStart();
            m_Instance.OnAwake();
            
            // 注册到Unity生命周期
            Type type = m_Instance.GetType();
            MonoSingletonCenter center = MonoSingletonCenter.Instance;
            center.RegUpdateCallBack(type, m_Instance.OnUpdate);
            center.RegLateUpdateCallBack(type, m_Instance.OnLateUpdate);
            
            return m_Instance;
        }
    }
}
```

### 泛型约束语法解析
```csharp
public abstract class MonoSingleton<T> where T : MonoSingleton<T>, new()
```

#### 约束详解
- `where T : MonoSingleton<T>` - T必须继承自MonoSingleton<T>
- `where T : new()` - T必须有无参构造函数
- 组合约束：T必须同时满足两个条件

#### 为什么这样设计？
1. **类型安全** - 确保类型关系正确
2. **构造函数要求** - 确保可以实例化
3. **自引用泛型** - 让每个具体的单例类都"知道"自己的类型

### 使用示例
```csharp
// 定义具体的单例类
public class AssetManager : MonoSingleton<AssetManager>
{
    // 编译器自动生成无参构造函数
    
    public override void Init()
    {
        // 初始化逻辑
    }
    
    protected override void OnStart()
    {
        // 启动逻辑
    }
}

// 使用时
AssetManager manager = AssetManager.Instance;  // T = AssetManager
```

### 项目中的MonoSingleton使用
```csharp
// 核心Manager类
public sealed partial class AssetManager : MonoSingleton<AssetManager>
public class UIManager : MonoSingleton<UIManager>
public class AudioManager : MonoSingleton<AudioManager>
public class PoolManager : MonoSingleton<PoolManager>
public class CoroutineManager : MonoSingleton<CoroutineManager>
```

---

## 位掩码技术

### 什么是位掩码？
位掩码是一种使用二进制位来表示多个布尔状态的技术。每个位代表一个状态，通过位运算来组合和检查状态。

### Unity HideFlags的位掩码设计
```csharp
public enum HideFlags
{
    None = 0,                    // 0000 0000 (二进制)
    HideInHierarchy = 1,         // 0000 0001 (二进制)
    HideInInspector = 2,         // 0000 0010 (二进制)
    DontSaveInEditor = 4,        // 0000 0100 (二进制)
    NotEditable = 8,             // 0000 1000 (二进制)
    DontSaveInBuild = 16,        // 0001 0000 (二进制)
    DontUnloadUnusedAsset = 32,  // 0010 0000 (二进制)
    DontSave = 52,               // 0011 0100 (二进制) = 4+16+32
    HideAndDontSave = 61         // 0011 1101 (二进制) = 1+4+16+32+8
}
```

### 位掩码的运算原理
#### 加法运算（组合多个标志）：
```csharp
// HideAndDontSave = 1+4+16+32+8 = 61
// 二进制表示：0011 1101

// 分解：
HideInHierarchy = 1      // 0000 0001
DontSaveInEditor = 4     // 0000 0100  
NotEditable = 8          // 0000 1000
DontSaveInBuild = 16     // 0001 0000
DontUnloadUnusedAsset = 32 // 0010 0000

// 相加（实际上是按位或运算）：
// 0000 0001 (1)
// 0000 0100 (4)
// 0000 1000 (8)
// 0001 0000 (16)
// 0010 0000 (32)
// ---------
// 0011 1101 (61)
```

### 位掩码的常用操作
#### 设置标志（按位或）：
```csharp
HideFlags flags = HideFlags.None;
flags |= HideFlags.HideInHierarchy;  // 设置隐藏标志
flags |= HideFlags.DontSave;         // 设置不保存标志
```

#### 检查标志（按位与）：
```csharp
HideFlags flags = HideFlags.HideAndDontSave;

// 检查是否隐藏
if ((flags & HideFlags.HideInHierarchy) != 0)
{
    Console.WriteLine("对象被隐藏");
}

// 或者使用HasFlag方法
if (flags.HasFlag(HideFlags.HideInHierarchy))
{
    Console.WriteLine("对象被隐藏");
}
```

#### 清除标志（按位与取反）：
```csharp
HideFlags flags = HideFlags.HideAndDontSave;

// 清除隐藏标志
flags &= ~HideFlags.HideInHierarchy;
```

### 为什么使用位掩码？
#### 优势：
1. **节省内存** - 一个整数可以表示32个布尔状态
2. **高效运算** - 位运算比多个布尔变量快
3. **灵活组合** - 可以任意组合多个状态
4. **类型安全** - 枚举提供类型检查

---

## 代理模式与生命周期管理

### 代理模式的核心思想
在Unity中，只有继承自`MonoBehaviour`的类才能自动获得Unity的生命周期方法（如`Update`、`LateUpdate`等）。但是`MonoSingleton<T>`是一个普通的C#类，不是`MonoBehaviour`，所以它无法直接获得这些生命周期方法。

### 解决方案：代理模式
项目通过`MonoSingletonCenter`这个`MonoBehaviour`来代理所有`MonoSingleton`的生命周期调用。

```csharp
// 在MonoSingleton的Instance属性中
Type type = m_Instance.GetType();                    // 获取具体类型，如AssetManager
MonoSingletonCenter center = MonoSingletonCenter.Instance;  // 获取生命周期管理中心
center.RegUpdateCallBack(type, m_Instance.OnUpdate);        // 注册Update回调
center.RegLateUpdateCallBack(type, m_Instance.OnLateUpdate); // 注册LateUpdate回调
```

### MonoSingletonCenter的工作原理
```csharp
internal sealed class MonoSingletonCenter : MonoBehaviour
{
    // 存储所有注册的Update回调
    private readonly TypeDictionary<Action> _updateList = new TypeDictionary<Action>(32);
    private readonly TypeDictionary<Action> _lateUpdateList = new TypeDictionary<Action>(32);
    
    // Unity的Update方法
    private void Update()
    {
        // 遍历所有注册的MonoSingleton，调用它们的OnUpdate方法
        foreach (KeyValuePair<Type, Action> action in _updateList)
        {
            action.Value.Invoke();  // 调用具体的OnUpdate方法
        }
    }
    
    // Unity的LateUpdate方法
    private void LateUpdate()
    {
        // 遍历所有注册的MonoSingleton，调用它们的OnLateUpdate方法
        foreach (KeyValuePair<Type, Action> action in _lateUpdateList)
        {
            action.Value.Invoke();  // 调用具体的OnLateUpdate方法
        }
    }
}
```

### 具体执行流程
1. **第一次访问AssetManager.Instance**
2. **在MonoSingleton.Instance的getter中执行注册**
3. **在MonoSingletonCenter.RegUpdateCallBack中添加到待注册列表**
4. **在MonoSingletonCenter.Update中调用所有注册的Update方法**

### MonoSingletonCenter的自动创建
`MonoSingletonCenter`确实需要挂载在GameObject上，但它是**自动创建**的：

```csharp
public static MonoSingletonCenter Instance
{
    get
    {
        if (m_Instance != null)
            return m_Instance;
            
        // 自动查找或创建GameObject
        GameObject obj = GameObject.Find(nameof(MonoSingletonCenter));
        if (!obj)
        {
            // 自动创建GameObject并挂载MonoSingletonCenter组件
            obj = new GameObject(nameof(MonoSingletonCenter));
            m_Instance = obj.AddComponent<MonoSingletonCenter>();
        }
        
        // 设置为不保存，跨场景不销毁
        obj.hideFlags = HideFlags.HideAndDontSave;
        DontDestroyOnLoad(obj);
        
        return m_Instance;
    }
}
```

### 为什么Scene中找不到MonoSingletonCenter？
你没在Scene中找到`MonoSingletonCenter`是正常的，原因：

1. **HideFlags.HideAndDontSave** - 隐藏且不保存到Scene
2. **运行时动态创建** - 只在代码需要时创建
3. **自动管理** - 完全由MonoSingleton系统管理
4. **设计如此** - 避免污染Scene文件

### 启动流程
#### 完整的启动时序
```
1. Unity启动 -> Entrance.Update() (第3帧)
2. PreStartUp() -> 基础MonoSingleton初始化
3. StartUp() -> 核心Manager初始化
4. InitFacade() -> MVC框架初始化
5. InitModule() -> 业务模块注册
6. StartScene() -> 场景加载

在整个过程中：
- 任何Manager第一次访问Instance时，会自动注册到MonoSingletonCenter
- MonoSingletonCenter自动创建GameObject并开始生命周期
- 所有Manager的Update方法通过MonoSingletonCenter统一调用
```

### 代理模式的优势
1. **统一管理** - 所有MonoSingleton的生命周期都由一个中心管理
2. **性能优化** - 避免每个MonoSingleton都创建自己的MonoBehaviour
3. **类型安全** - 通过Type作为key，确保回调的正确性
4. **易于扩展** - 可以轻松添加新的生命周期方法

---

## 总结

这个Unity项目采用了非常优雅的架构设计：

1. **FGUI UI系统** - 通过MVC架构和自定义加载机制，提供了比官方更强大的UI管理能力
2. **Lib基础库** - 通过程序集分离，实现了清晰的模块化架构
3. **MonoSingleton模式** - 结合了单例模式和Unity生命周期，提供了强大的Manager管理能力
4. **位掩码技术** - 高效地管理对象状态，节省内存和提升性能
5. **代理模式** - 巧妙地解决了普通类无法获得Unity生命周期的问题

这些设计模式和技术在实际项目中相互配合，形成了一个高效、可维护、可扩展的Unity项目架构。
