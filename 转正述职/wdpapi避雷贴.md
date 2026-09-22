WdpApi 踩雷记录
1. 网格（批量添加 range）与路径叠加显示规则
当网格与路径位置重合时，渲染顺序与配置会影响显示效果：
- coordZRef = "surface"，coordZOffset 按需配置
- 先添加网格，后添加路径：重合点位高度异常，路径会悬浮空中
- 先添加路径，后添加网格：显示正常
- coordZRef = "Altitude" 或 "ground"
需保证 coordZOffset ＞ 路径坐标数组中所有点的 z 值
- 若 coordZOffset 偏小：路径显示不全，部分被遮挡
- 若 coordZOffset 偏大：路径完整显示，支持先网格后路径

2. 实体跟随路径移动（对象存储与删除规则）
- 如需单独删除实体 / 移动对象且保留路径，三维引擎对象需单独存放至非响应式变量中（避免 Vue 代理与响应式开销）。
- 存储方式：
data() {
this.toolObjs = { ... };
return {};
}
- 删除跟随路径的对象时，必须取 movingObj（真实移动对象）。
- 若直接删除 animObjResp.result.object，会连带删除路径。
- 正确存储示例：
const animObjResp = await coveringMove(
poiObj.result.objects[0],
this.toolObjs.patrolPaths[i],
{ time: 60, bLoop: true, state: "play" }
);
if (animObjResp?.success) {
this.toolObjs.anim.push(animObjResp.result.object.movingObj);
}

3. 模型与路径绑定（朝向 / 旋转设置）
使用 App.Bound 方法绑定模型与路径时：
- 默认规则：模型X 轴自动朝向路径前进方向
- 若模型自身 X 轴并非正前方，会出现朝向异常，可通过 API 调整转角修正
- 注意：直接设置模型自身 rotator 不生效，必须在模型与路径绑定配置中设置 rotator
- 配置示例：
rotator: {
pitch: config.pitch || 0,
yaw: config.yaw || 0,
roll: config.roll || 0
}

4. 骨骼模型使用说明（云端 / 本地）
- 云端模型：直接通过 seedId 即可请求加载使用
- 本地模型：需先在项目案例 - 编辑案例中将模型拖入场景，发布给个人后下载到本地；未下载直接请求会导致请求超时

5. 镜头飞行抖动问题及解决方案
镜头飞行过程中出现抖动，优先使用 SetCameraPose 接口，避免使用聚焦到坐标点的 FlyTo API。
- 若必须使用 FlyTo，需将 distance 数值 +0.1 避免抖动
原因：当距离参数为 0 时，特定场景下极易触发镜头异常抖动，小幅偏移可规避该渲染问题。
- 推荐直接使用 SetCameraPose 实现镜头飞行
FlyTo 用法（不推荐，易抖动）
const jsondata = {
"targetPosition": [121.48533665,31.24164246,30],
"rotation": {
"pitch": -30, //俯仰角, 参考(-90~0)
"yaw": 0,     //偏航角, 参考(-180~180; 0:东; 90:南; -90:北)
},
"distance":0.1, //距离(单位:米)
"flyTime": 1     //过渡时长(单位:秒)
}
// 必加：distance + 0.1
推荐用法（无抖动）
const jsondata = {
"location": [121.48537621,31.23840069,900],
"rotation": {
"pitch": -35, //俯仰角, 参考(-90~0)
"yaw": 0      //偏航角, 参考(-180~180)
},
"flyTime": 1     //过渡时长(单位:秒)
}
const res = await App.CameraControl.SetCameraPose(jsondata);
console.log(res);

6. 通过ClearByEids关闭window弹窗请求超时问题
运行环境: cmp公网地址
操作及现象：点击window弹窗关闭按钮，通过OnWebJSEvent监听关闭事件，然后调用ClearByEids删除window实体，平台报错Request timeout，场景完全卡死
出现频率：偶发




[图片]
[图片]

解决方案：wdpapi v2.2.0的bug，需要将前端和场景包的wdpapi都升级到v2.3.0解决

7. POI或Path坐标点位高度不起作用
- 控制变量删除部分参数，呈现如下
原始：
const res = await App.Scene.Add(poi, {
calculateCoordZ: {
coordZRef: "surface", //surface:表面;ground:地面;altitude:海拔
coordZOffset: 50 //高度(单位:米)；
}
});
删除后结构如下：
const res = await App.Scene.Add(poi);

const res = await App.Scene.Add(poi, {
calculateCoordZ: {
coordZRef: "surface", //surface:表面;ground:地面;altitude:海拔
//删除高度
}
});

8. WDP云渲染请求 Pending 导致 POI 与镜头动画卡顿问题
问题描述：在 Vue3 + Vite 项目中，执行 addPOI、flyCameraTo 等云渲染接口时，dtp-api.51aes.com 请求在 Network 面板会 Pending 约 1 秒后才真正发出，导致 POI 加载与镜头飞行动画明显卡顿。
排查过程：
- 逐个禁用 Chrome 扩展：浏览器扩展可能会在请求发送前执行拦截逻辑，多个扩展耗时叠加后导致请求 Pending
- 环境对比验证：生成独立 HTML 页面、Vue2 项目及 Chrome 无痕模式下进行对比
- 代码层排查：将并行加载改为串行 await，移除 customData 等非必要字段，或重构加载逻辑，确认是不是业务代码或数据量导致
- 网络排查：继续排查网络与连接问题，包括禁用 STOMP 与直播 WebSocket、页面卸载时调用 Renderer.Stop() 释放资源、清理 Chrome 连接池等，但问题依旧存在。

9. POI因镜头旋转/移动导致图标与文字框偏移问题
- 问题描述：场景镜头旋转或移动时，POI 图标位置固定，但文字框/背景会跟随镜头方向产生偏移，出现图标与文字框错位、漂移的问题。
- 原因：
WDP 的 markerNormalUrl 与 labelBgImageUrl 虽然都可以显示图片，但底层渲染方式不同：
- markerNormalUrl ：使用 Billboard 精灵渲染，始终朝向相机，并固定在世界坐标位置，不会因镜头旋转产生漂移。
- labelBgImageUrl：属于 Label 层背景，会参与屏幕空间的 Label 偏移与旋转计算，镜头变化时会跟随 Label 一起移动。当 POI 图标放在 labelBgImageUrl 上时，镜头旋转或移动过程中，图标会跟随 Label 层产生偏移，因此出现漂移错位问题。
- 解决方式：
- POI 图标使用 markerNormalUrl / markerActivateUrl 渲染
- labelContent 仅用于显示文字
- 通过 labelBgOffset 调整文字框与 marker 对齐位置
调整后，镜头移动与旋转过程中，POI 图标与文字框可保持稳定对齐。

10. WDP推流画面卡死在最后一帧问题
- 问题
WDP 云渲染场景中，执行 Scene.ClearByTypes、FocusToEntities、点击 POI 或实体等操作后，画面会冻结在最后一帧，但API接口返回均为 success: true，无明显报错。
- 典型高发场景：
- DevTools Device Mode 模拟 4K+ 分辨率
- 首次进入场景（无 HTTP 缓存、WebRTC ICE 协商慢）
- 大屏设备硬件解码不够强
- 验证方式
卡死后，通过 DevTools Console 控制台跑以下代码，直接读取 WDP 推流 video 状态：
(() => {
const v = document.getElementById("streamingVideo");
if (!v) return "no streamingVideo element";

return {
paused: v.paused,
currentTime: v.currentTime,
readyState: v.readyState,
networkState: v.networkState,
videoWidth: v.videoWidth,
videoHeight: v.videoHeight,
srcObject: !!v.srcObject,
};
})();
- 验证发现：
➢卡死时 paused: true
➢currentTime > 0，说明视频已经播放过
➢WebRTC 流未断开，video 只是被暂停未恢复由此确认问题不是接口失败，而是 WDP 内部对推流 video 调用了 .pause() 后未自动 .play()。
- 原因
WDP 在执行部分场景操作时，会主动对 #streamingVideo 调用 .pause() 做流重协商，但结束后不会自动恢复播放。
普通分辨率下恢复时间很短，肉眼不易察觉；4K、大屏或首次进入场景时，由于解码与 WebRTC 协商耗时增加，冻结现象会明显放大。
- 解决方式
在 WDP 初始化前，对 #streamingVideo 增加 pause 自动恢复逻辑：
const patchVideo = (video) => {
if (!video || video.__wdpPatched) return;
video.__wdpPatched = true;

video.playsInline = true;
video.setAttribute("playsinline", "");
video.setAttribute("autoplay", "");

const resume = () => {
const p = video.play?.();
p?.catch?.(() => {
video.muted = true;
video.setAttribute("muted", "");
video.play?.().catch(() => {});
});
};

// 被 pause 后立即恢复
video.addEventListener("pause", resume, true);

// 首次主动 play
video.play?.().catch?.(() => {});
};

// WDP 初始化前执行
const scan = () => {
const el = document.getElementById("streamingVideo");
if (el) patchVideo(el);
};

scan();

const observer = new MutationObserver(() => scan());
observer.observe(document.body, {
childList: true,
subtree: true,
});
处理后，场景聚焦、清空及实体点击过程中，推流画面可自动恢复播放，不再冻结在最后一帧。

11. 场景加载到一定值，无法加载
- 75%
- 虚拟显卡干扰：用户设备存在虚拟显卡（如向日葵虚拟显卡），渲染任务被错误分配，导致加载失败（已在 wdp5.9 修复）
第一步：打开“设备管理器” > 显示适配器，检查是否存在虚拟显卡，若有请禁用。
第二步：若禁用后仍无效，请联系打包人员核查配置。  
- 项目打包过程中配置不当也可能导致加载卡住，或者底板没有ready：需要找工程
- 90%
- 推流后端问题，需要找后端人员：张克慧
- 70%
- 这个问题可能是原来某个项目已经使用这个AppId了，已经存档了，新的项目又使用了同一个AppID，就会导致存档问题
方法一：新建一个AppId，再上传一次这个新的项目
方法二：后端研发人员把原来存档的东西删掉

12. 通过取点获取的点位生成的路径没有出现在路面的表面
聚焦到POI查看是否点位是打到了底版上，是否是地面没有添加碰撞（找场景解决）
[图片]
[图片]
[图片]
     海拔Altitude取点                             表面Surface取点                             地面Ground取点

13. 快速连续操作导致场景元素残留
● 问题描述：
在需要"先弹出/展示某元素，再进入下一步，最后关闭并恢复上一状态"的交互链路中，若用户点击过快，前一个操作的清理或恢复尚未完成，后一个操作就已开始执行，导致上一步的元素（如信息弹窗、标注、临时对象等）没有被正确清除而残留在场景中。
● 原因：
多个改动场景的异步流程共用同一套状态，快速触发时会相互交错。凡是"创建后立即登记、清理时按标识批量删除并配合类型兜底"的元素，通常都能删干净；而那些需要"在某步之后异步重建、且引用在异步返回后才赋值、又用单一变量保存"的元素，容易出现时序空档——新操作清理时读到的还是旧的空引用，什么都没删，等异步重建完成后又已无人负责清理，最终残留。根因是异步重建加引用迟到，与具体走什么事件通道无关。
● 解决方式：
➢ 引入全局串行队列，把所有"用户动作级"入口按点击先后顺序逐个执行，从根本上消除异步交错。为避免死锁，只在最外层入口串行，内部下层逻辑不重复串行。
➢ 增加"代龄/版本"守卫作为兜底：凡是会改变当前状态的操作都递增一个标记，异步重建在开始前后各比对一次，一旦发现已被更晚的操作抢先，就撤销本次重建并清理掉刚产生的孤儿元素。

14. 多操作串行后镜头抢夺抖动问题
● 问题描述：快速连点多个面板/POI 时，镜头会连续被不同目标"抢夺"，出现来回跳动、抖动。
● 原因：
WDP 镜头接口（flyCameraTo / CameraControl.Focus / FlyTo / Around）虽然带 flyTime 飞行动画时长，但 await 是"命令下发即返回、不等飞行动画结束"。串行队列只保证业务逻辑不交错，无法阻止连续多条镜头命令连发——后一条会打断前一条还在进行的飞行动画，于是镜头从半路突然转向新目标，表现为抢夺抖动。
● 解决方式：
采用"最新者优先"策略。让串行队列记录尚未处理完的任务数量，对外提供"当前是否已是最后一个待处理操作"的判断；在所有会抢占该类资源的动作执行前先做判断，只让队列中最后一个操作真正生效，中间被快速跳过的操作只处理必要的数据与状态、不触发该类命令。单次操作时判断恒为放行，不影响正常表现，仅在检测到有更晚操作排队时才跳过，从而消除抖动。

15. 骨骼模型"前进一段又弹回一下"
- 问题描述：模型沿路径播放步行动画时，出现周期性"前进一段又弹回一下"的抖动。
- 原因：
  - Bound 路径绑定负责让模型沿轨迹匀速前进——这是模型真正的移动来源，位置由引擎按时间插值算出。
  - 步行动画本身也带位移（这类叫 root motion / 位移型 locomotion 动画）。这个模型的步行动画不是"原地踏步"，而是动画内部让根骨骼往前挪一段，播到循环末尾再"啪"地把根骨骼弹回起点，如此往复。
  - 两者叠加在同一个模型上：
4. 动画播放中 → 模型 = Bound 前进量 + 动画前进量（往前冲）
5. 动画循环重置那一刻 → 动画前进量瞬间归零（往后弹）
叠加起来就是你看到的"走一节 → 卡一下往后退"。
  - 为什么和速度无关：弹回是由动画循环周期触发的（每个走路循环结束弹一次），不是由移动速度触发的。所以把 speed 调快调慢，弹回照样规律出现
  - 为什么 bPause: true 就好了：暂停动画后，第 2 套位移（动画自带位移）被彻底关掉，只剩 Bound 单独驱动，没有冲突，自然平顺——代价是脚不迈步、看起来滑步。
- 解决办法：换一个"原地不带位移"的走路动画序号animSequenceIndex

16. WDP实现模型绑定路径拖拽进度条，更新模型位置后继续移动
- 方案一：按时间计算位置
  - 轨迹点带真实定位时间，前端按时间建立播放轴。播放过程中，根据当前播放时间计算模型对应的轨迹位置，通过 SetLocation([lng, lat, alt]) 持续更新模型位置。
  - SetLocation 属于 ObjectController 的位置设置 API，调用后直接更新实体坐标，本身不负责路径插值和地形高度计算，因此连续移动需要前端自行计算每一帧的位置和高度。
  - 拖拽进度条时，仅更新进度和时间；松手后，根据目标进度换算对应时间，找到对应轨迹位置，通过 SetLocation 将模型定位到目标位置，再继续播放。
- 方案二：引擎 Bound 路径播放
  - 根据轨迹距离和配置速度生成播放时间，将轨迹交给 Bound 沿路径播放，由引擎负责模型的路径运动及高度处理。
  - 拖拽进度条时，仅更新进度和时间；松手后，根据目标进度确定轨迹位置，截取目标位置后的轨迹，重新创建路径并绑定 Bound，从目标位置继续播放。
  注意事项
    - 轨迹数据：按真实时间计算位置的方案依赖轨迹点中的定位时间；无真实定位时间时，需要根据轨迹距离和配置速度生成模拟播放时间。
    - 模型高度：使用 SetLocation 逐帧更新模型位置时，不经过路径创建时的地形高度处理，需自行处理模型高度。 
    - 路径效果：使用 SetLocation 手动更新位置会绕过引擎的路径运动处理，可能影响模型的高度及路径运动效果。

17. 场景初始化时poi点位过多，鼠标拖动底图会点选到POI点上
- 个人理解：本质是poi占据整个界面操作空间太多了，场景拖拽空间很有限，整个体验有问题，能想到的就是poi的click和press事件要分开
  玉和田兰州的解决办法：找WDP加了一个在POI上面拖动鼠标镜头无法移动,单独出了一个版本
  在找WDP同事的时候需要提供：POI种类、API版本号、提供UE LOG
[图片]
该问题在wdpapi 2.5.0 版本进行了修复
[图片]

18. 点聚合POI和普通POI创建共同存在时，点击点聚合POI正常，普通POI点击不了
- 问题描述：用点聚合方式创建的POI点点击正常，另外再单独不是以聚合方式创建了POI，点击后，前面聚合方式创建的POI就全部点击不了。事件全被单独创建的POI点的拦截了
- 原因：
  创建 customPOI 之后再创建其他的2D覆盖物 ，鼠标操作事件有冲突，后续需要进一步修复customPOI 。（api2.3.0版本）
  目前的方案是：在 insertJpegPoi 函数里面 创建用普通的POI创建（new app.Poi） 替换掉原来创建customPOI (new app.CustomPoi)。
创建poi的办法
async function insertJpegPoi(media, coord) {
  const app = window.__WDP_APP__;
  if (!app || !app.Scene || typeof app.CustomPoi !== "function" || !media || !media.originalUrl || !coord) {
    console.warn("[droneTrack] insertJpegPoi 参数无效或 WDP App/CustomPoi 不可用");
    return null;
  }

  // 已插入过则跳过
  if (droneJpegPoiObjs.some(o => o && o._jpegUuid === media.uuid)) {
    return null;
  }

  // 规范化 url：originalUrl 可能以 // 开头（协议相对地址），补上 https:
  let imgUrl = media.originalUrl;
  if (imgUrl.startsWith('//')) imgUrl = 'https:' + imgUrl;

  const labelSize = JPEG_POI_PHOTO_SIZE + JPEG_POI_BORDER * 2;

  try {
    const poi = new app.CustomPoi({
      location: [coord[0], coord[1], coord[2] || 0],
      poiStyle: {
        markerNormalUrl: imgUrl,
        markerActivateUrl: imgUrl,
        markerSize: [JPEG_POI_PHOTO_SIZE, JPEG_POI_PHOTO_SIZE],
        markerOffset: [0, 0],
        // label 白色背景作为照片四周 20px 白边；zIndex 0 = marker 在上、label 在下
        labelContent: [''],
        labelStyle: {
          width: labelSize,
          height: labelSize,
          visible: true,
          offset: [-55, 105],
          hideDistance: 2000,
          zIndex: 0,
          background: ['ffffffff'],
        },
        generalLabelStyle: {
          width: labelSize,
          height: labelSize,
          fontSize: 1,
          padding: '0 0 0 0',
          color: 'ffffffff',
          textAlign: 'center',
          autoWrap: false,
        },
      },
      visible2D: {
        camera: {
          hideDistance: 2000,
          hideType: "default",
          scaleMode: "2D",
        },
        interaction: {
          clickTop: true,
          hoverTop: true,
        },
        entity: {
          overlapOrder: 1,
        },
      },
      entityName: `droneJpegPoi_${media.uuid}`,
      customId: `drone_jpeg_poi_${media.uuid}`,
    });

    const res = await app.Scene.Add(poi, {
      calculateCoordZ: {
        coordZRef: "surface",
        coordZOffset: 0,
      },
    });

    const poiObj = res?.result?.object || res?.result?.objects?.[0] || null;

    if (poiObj) {
      poiObj._jpegUuid = media.uuid;
      droneJpegPoiObjs.push(poiObj);
      // 点击照片 POI → 弹出大图（PhotoPlayPopup.show 接收媒体数组，内部过滤 jpeg）
      if (typeof poiObj.onClick === "function") {
        poiObj.onClick(() => {
          console.log("[droneTrack] 点击 jpeg 照片 POI, url:", imgUrl);
          if (typeof window.showDronePhotoPopup === "function") {
            window.showDronePhotoPopup([media]);
          }
        });
      }
      console.log("[droneTrack] jpeg 照片 POI 创建成功, eid:", poiObj.eid, "markerSize:", JPEG_POI_PHOTO_SIZE, "labelSize:", labelSize, "coord:", coord);
    } else {
      console.warn("[droneTrack] jpeg 照片 POI 创建失败", res);
    }
    return poiObj;
  } catch (e) {
    console.error("[droneTrack] jpeg 照片 POI 创建异常:", e);
    return null;
  }
}
该问题在wdpapi 2.5.0 版本进行了修复
[图片]

19. 使用CMP无法用客户服务器的渲染资源
- 问题描述：访问 https://51cmp.51aes.com:779/...（HTTPS 页面）时，页面内部需要请求 WDP 场景服务地址 http://202.104.131.38:30080/service（HTTP，非加密，外网可以打开http://202.104.131.38:30080/oauth/login）
[图片]
[图片]
- 问题定位：浏览器控制台报错：https://51cmp.51aes.com:779/6d64d869890bd43287be0dfa390be432/#/?sceneUrl=http%3A%2F%2F202.104.131.38%3A30080%2Fservice&sceneOrder=33d688797a07885dba44a0f62e320067&_t=1788411731963' was loaded over HTTPS, but requested an insecure XMLHttpRequest endpoint 'http://202.104.131.38:30080/service/Renderers/Any/order'. This request has been blocked; the content must be served over HTTPS.
- 结论：这是浏览器强制的安全策略拦截，不是应用程序的 bug，前端代码无法绕过。
  只要满足"外层页面是 HTTPS + 内部请求目标是 HTTP"这个组合，Chrome/Edge/Firefox 等所有现代浏览器都会无条件拦截，跟我们的代码逻辑、跟场景服务器本身是否正常运行都没有关系）。
- 疑问回答2：为什么能进http://202.104.131.38:30080/oauth/login
  因为这是直接在地址栏整页跳转打开，不是"在一个已经打开的 HTTPS 页面内部发起的子请求"——这是两码事，Mixed Content 规则只管后者，不管前者。
场景
浏览器怎么判定
是否拦截
在地址栏直接输入 http://202.104.131.38:30080/oauth/login 打开
这是整个页面的导航跳转（top-level navigation），跳转后这个 tab 本身就变成了一个 HTTP 页面
✅ 完全不受限制，可以自由跳转到任何协议的地址
已经在 https://51cmp.51aes.com 页面里，页面内部 JS 用 XHR/fetch 去请求 http://202.104.131.38:30080/service/...
这是当前 HTTPS 页面内部发起的子请求（subresource request），页面本身没变，还是原来那个 HTTPS 文档

❌ 被 Mixed Content 拦截

Mixed Content 规范约束的对象是"一个已加载页面内部再去拉取的资源"（XHR、fetch、<script>、<img>、iframe 等），目的是防止一个本来加密安全的页面被内部混入的明文内容污染/劫持。但浏览器允许你随时把整个标签页导航到任意协议的地址——这跟点了个 http:// 链接、或者手动输网址是完全合法、无限制的操作，不属于"混合内容"的范畴，哪怕你是从一个 HTTPS 页面点击链接跳转过去的也一样能打开。

20. 使用wdpapi加载shp数据很慢
- 问题描述：使用wdpapi加载shp数据很慢
- 解决办法：使用gisapi即可https://wdpapidoc.51aes.com/apifunc/gisapi
shp数据加载： 
api
简述
优点
缺点
wdpapi
可以加载poi、range、path
（wdpapi文档只写了range，其余的在云端联调中查找）
可以自定义样式，基础poi\range\path等具有的样式都可以使用

加载慢
gisapi
可以加载shp数据为点、线、面
加载快
样式单一，只能修改颜色，无法动效
