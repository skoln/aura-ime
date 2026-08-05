1. 项目定位与范围

· 产品形态：Android 平台输入法（IME），提供软键盘界面与候选词推荐功能。
· 核心能力：集成本地大语言模型推理（llama.cpp），实现离线候选词生成；同时支持云端候选作为补充。
· 工程边界：仅包含 Android 客户端应用，不涉及云端服务端实现。

---

2. 模块职责划分（强制三模块）

模块名 包路径 / 命名空间 职责范围
service xyz.skoln.auraime.service 继承 InputMethodService，管理输入法生命周期、系统回调（onCreate、onStartInput、onDestroy）、与系统 IME 框架的交互。
keyboard xyz.skoln.auraime.keyboard 负责键盘视图的绘制（基于传统 View 体系）、触摸事件处理、候选窗 UI 渲染及点击响应。
bridge xyz.skoln.auraime.bridge（Java） aura_ime::core（C++） 封装 JNI 桥接层，管理 llama.cpp 的模型加载、推理调用、资源释放；对外提供 Java 层可调用的同步/异步接口。

强制约束：

· 禁止跨模块直接依赖（如 keyboard 不得直接调用 bridge 的 JNI 方法，必须通过 service 中转或定义明确的接口类）。
· 新增子模块必须经过本说明书版本更新。

---

3. UI 渲染技术栈（强制锁定）

· 键盘视图：强制使用传统 View 体系（LinearLayout、ViewGroup 及自定义绘制），严禁引入 Jetpack Compose。
· 设置页面：可选用 Compose 或传统 View，但需在 AndroidManifest.xml 中明确 Activity 声明，且不影响键盘弹出性能。
· 图标资源：采用 Android 自适应图标规范（mipmap-anydpi-v26），背景必须指定纯色（禁止透明背景）。

---

4. 线程模型与性能基线

· UI 线程（主线程）：仅负责视图绘制、点击响应及系统回调（如 onStartInput）。禁止执行任何 I/O、数据库查询或推理操作。
· 推理线程：必须在 service 初始化时创建专用的 HandlerThread，并设置为 THREAD_PRIORITY_BACKGROUND。所有本地推理（getCandidates）及词库查询必须切至此线程执行。
· 超时处理机制：对于可能阻塞的本地操作（如推理），必须实现超时截断逻辑。超时后立即返回空结果，严禁抛出异常或继续等待。
· 云端请求：云端候选请求必须独立于推理线程，采用异步网络请求；回调结果不得覆盖本地首帧数据，仅允许作为列表后缀追加。

---

5. 数据存储与隔离规范

存储位置 用途 约束
Context.getFilesDir()（内部存储） 存放运行时词库、用户词典、本地模型文件（.bin） 强制唯一读写路径，严禁将核心数据存放于外部存储
外部公开目录（如 /storage/emulated/0/光环输入法） 用户手动导入/导出的备份文件 仅用于备份恢复操作，应用启动时不得自动扫描或加载此目录

· 权限要求：若需对外部存储进行写入，必须在 AndroidManifest.xml 中声明 READ_EXTERNAL_STORAGE 和 WRITE_EXTERNAL_STORAGE，并适配分区存储（requestLegacyExternalStorage 仅在 API 29 兼容模式下启用）。

---

6. Native / JNI 层基建约定

· 编译构建：使用 CMake（最低版本 3.22.1），启用 -O3 及 -flto（链接时优化）。
· 命名空间：C++ 代码强制使用 namespace aura_ime::core，禁止全局命名空间污染。
· JNI 函数命名：Java 侧 Native 方法所在类必须为 xyz.skoln.auraime.bridge.NativeBridge，JNI 函数名必须遵循 Java_xyz_skoln_auraime_bridge_NativeBridge_* 格式。
· 头文件约定：所有 Native 回调接口（如进度通知）必须使用 .h 头文件声明，并以 extern "C" 包裹，确保符号可被 dlopen 正确解析。

---

7. 输入法组件注册规范

· 服务声明：AndroidManifest.xml 中必须注册 InputMethodService，并强制包含：
  · android:permission="android.permission.BIND_INPUT_METHOD"
  · intent-filter 匹配 android.view.InputMethod
· 主 Activity（设置入口）：必须设置 launchMode="singleTask"，避免键盘弹出时干扰 Activity 栈顶状态。
· 元数据：必须提供 @xml/method 配置文件，声明输入法子类型（如有）。

---

8. 外围参照文件（互补事实源）

· variable_register.xlsx：记录所有全局常量、配置参数的具体值（类型、默认值、取值范围）。
· api_contract.xlsx：记录所有公开接口的方法签名、入参出参说明、异常定义。
· changelog.xlsx：按时间倒序记录每次变更的模块、类型、影响范围。

优先级仲裁：当本说明书（规范）与 Excel 表（具体值）发生冲突时，以本说明书的约束为准；Excel 表应在下一版本中修正对齐。

---

9. 变更维护流程

1. 任何涉及架构、模块职责、线程模型或存储策略的修改，必须在本说明书中更新对应章节。
2. 同步在 changelog.xlsx 中追加变更记录（日期、内容、更新人）。
3. 版本号按语义化版本规则递增（主版本号.次版本号.补丁号），冻结版本后不得更改。