## 1. 原理概述
Riru是一个通过Magisk模块实现的Android系统级Hook框架，其核心功能是注入到Zygote进程中，从而允许模块在所有应用进程和系统服务进程中运行代码。其主要工作原理包括：

1. 注入机制 ：通过native bridge机制（ ro.dalvik.vm.native.bridge ）注入到Zygote进程
2. Hook方式 ：Hook JNI注册函数 jniRegisterNativeMethods 来替换Zygote的关键函数
3. 进程控制 ：替换Zygote的fork相关函数，在应用进程和系统服务进程创建时执行自定义代码
4. 隐藏机制 ：将自身和模块的内存标记为匿名内存，防止被检测工具发现
## 2. 详细实现分析
### 2.1 注入Zygote进程
Riru通过修改 ro.dalvik.vm.native.bridge 属性，使其指向自己的native bridge库。当Zygote启动时，系统会自动加载并执行这个native bridge。

关键代码 ： `loader.cpp`

```
__used __attribute__((constructor)) void constructor() {
    if (getuid() != 0) return;
    
    char cmdline[ARG_MAX + 1];
    get_self_cmdline(cmdline, 0);
    
    // 检查是否在zygote进程中
    if (strcmp(cmdline, "zygote") != 0 && strcmp(cmdline, 
    "zygote32") != 0 && 
        strcmp(cmdline, "zygote64") != 0 && strcmp(cmdline, 
        "usap32") != 0 && 
        strcmp(cmdline, "usap64") != 0) {
        LOGW("not zygote (cmdline=%s)", cmdline);
        return;
    }
    
    // 读取Magisk tmpfs路径
    char magisk_path[PATH_MAX]{0};
    // ... 读取逻辑 ...
    
    // 加载Riru核心库
    char riru_path[PATH_MAX];
    strcpy(riru_path, magisk_path);
    strcat(riru_path, "/.magisk/modules/riru-core/lib");
#ifdef __LP64__
    strcat(riru_path, "64");
#endif
    strcat(riru_path, "/libriru.so");
    
    auto handle = dlopen_ext(riru_path, 0);
    if (handle) {
        auto init = (void(*)(void *, const char*)) dlsym(handle, 
        "init");
        if (init) {
            init(handle, magisk_path);
        }
    }
    
    // 处理原始native bridge
    // ...
}
```
### 2.2 Hook JNI注册函数
Riru使用xhook库hook了 jniRegisterNativeMethods 函数。这个函数负责将Java类的native方法与C/C++实现关联起来。

关键代码 ： `jni_hooks.cpp`

```
NEW_FUNC_DEF(int, jniRegisterNativeMethods, JNIEnv *env, const 
char *className,
             const JNINativeMethod *methods, int numMethods) {
    LOGD("jniRegisterNativeMethods %s", className);
    
    auto newMethods = handleRegisterNative(className, methods, 
    numMethods);
    int res = old_jniRegisterNativeMethods(env, className, 
    newMethods ? newMethods.get() : methods,
                                           numMethods);
    return res;
}

void JNI::InstallHooks() {
    XHOOK_REGISTER(".*\\libandroid_runtime.so$", 
    jniRegisterNativeMethods);
    
    if (xhook_refresh(0) == 0) {
        xhook_clear();
        LOGI("hook installed");
    } else {
        LOGE("failed to refresh hook");
    }
    
    // 处理没有jniRegisterNativeMethods的情况
    // ...
}
```
### 2.3 替换Zygote关键函数
当系统注册Zygote类的native方法时，Riru会拦截并替换三个关键函数：

- nativeForkAndSpecialize ：fork应用进程
- nativeSpecializeAppProcess ：特殊化应用进程
- nativeForkSystemServer ：fork系统服务进程
关键代码 ： `jni_hooks.cpp`

```
static std::unique_ptr<JNINativeMethod[]> onRegisterZygote(const 
char *className, const JNINativeMethod *methods, int numMethods) {
    
    auto newMethods = std::make_unique<JNINativeMethod[]>
    (numMethods);
    memcpy(newMethods.get(), methods, sizeof(JNINativeMethod) * 
    numMethods);
    
    JNINativeMethod method;
    for (int i = 0; i < numMethods; ++i) {
        method = methods[i];
        
        if (strcmp(method.name, "nativeForkAndSpecialize") == 0) {
            // 保存原始方法
            JNI::Zygote::nativeForkAndSpecialize = new 
            JNINativeMethod{method.name, method.signature, method.
            fnPtr};
            
            // 根据不同Android版本选择合适的替换函数
            if (strcmp(nativeForkAndSpecialize_r_sig, method.
            signature) == 0)
                newMethods[i].fnPtr = (void *) 
                nativeForkAndSpecialize_r;
            else if (strcmp(nativeForkAndSpecialize_p_sig, method.
            signature) == 0)
                newMethods[i].fnPtr = (void *) 
                nativeForkAndSpecialize_p;
            // ... 其他版本处理 ...
            
        } else if (strcmp(method.name, 
        "nativeSpecializeAppProcess") == 0) {
            // 类似处理nativeSpecializeAppProcess
            // ...
        } else if (strcmp(method.name, "nativeForkSystemServer") 
        == 0) {
            // 类似处理nativeForkSystemServer
            // ...
        }
    }
    
    return newMethods;
}
```
### 2.4 执行钩子函数
替换后的函数会在调用原始函数前后执行预定义的钩子函数。这些钩子函数由Riru核心和各个模块注册。

关键代码 ： `jni_hooks.cpp`

```
void nativeSpecializeAppProcess_r(
        JNIEnv *env, jclass clazz, jint uid, jint gid, jintArray 
        gids, jint runtimeFlags,
        jobjectArray rlimits, jint mountExternal, jstring seInfo, 
        jstring niceName,
        jboolean startChildZygote, jstring instructionSet, jstring 
        appDataDir,
        jboolean isTopApp, jobjectArray pkgDataInfoList, 
        jobjectArray whitelistedDataInfoList,
        jboolean bindMountAppDataDirs, jboolean 
        bindMountAppStorageDirs) {
    
    // 执行所有模块的pre钩子
    nativeSpecializeAppProcess_pre(
            env, clazz, uid, gid, gids, runtimeFlags, rlimits, 
            mountExternal, seInfo, niceName,
            startChildZygote, instructionSet, appDataDir, 
            isTopApp, pkgDataInfoList,
            whitelistedDataInfoList, bindMountAppDataDirs, 
            bindMountAppStorageDirs);
    
    // 调用原始函数
    ((nativeSpecializeAppProcess_r_t *) 
    JNI::Zygote::nativeSpecializeAppProcess->fnPtr)(
            env, clazz, uid, gid, gids, runtimeFlags, rlimits, 
            mountExternal, seInfo, niceName,
            startChildZygote, instructionSet, appDataDir, 
            isTopApp, pkgDataInfoList,
            whitelistedDataInfoList, bindMountAppDataDirs, 
            bindMountAppStorageDirs);
    
    // 执行所有模块的post钩子
    nativeSpecializeAppProcess_post(env, clazz, uid, 
    startChildZygote);
}
```
### 2.5 加载和管理模块
Riru会加载所有已安装的模块，并管理它们的生命周期和钩子函数。

关键代码 ： `module.cpp`

```
// 模块加载逻辑
// ...

// 执行所有模块的onModuleLoaded钩子
for (auto module : modules) {
    if (module->apiVersion >= 25) {
        module->onModuleLoaded();
    }
}
```
### 2.6 隐藏机制
Riru实现了隐藏机制，将自身和模块的内存标记为匿名内存，防止被检测工具发现。

关键代码 ： `hide_utils.cpp`

```
void Hide::HideFromMaps() {
    auto self_path = Magisk::GetPathForSelfLib("libriru.so");
    auto modules = Modules::Get();
    std::set<std::string_view> names{};
    for (auto module : Modules::Get()) {
        if (strcmp(module->id, MODULE_NAME_CORE) == 0) {
            names.emplace(self_path);
        } else if (module->supportHide) {
            if (!module->isLoaded()) {
                LOGD("%s is unloaded", module->id);
            } else {
                names.emplace(module->path);
            }
        } else {
            LOGD("module %s does not support hide", module->id);
        }
    }
    if (!names.empty()) Hide::HidePathsFromMaps(names);
}

void Hide::HideFromSoList() {
    // 从solist中移除相关库的条目
    // ...
}
```
## 3. 工作流程总结
1. 注入阶段 ：
   
   - Zygote启动时加载native bridge
   - native bridge检查当前进程是否为Zygote
   - 如果是，则加载Riru核心库
2. 初始化阶段 ：
   
   - 初始化Magisk路径和隐藏机制
   - 安装JNI钩子
   - 加载所有模块
   - 调用模块的onModuleLoaded钩子
3. 运行阶段 ：
   
   - 当Zygote准备fork新进程时，触发相应的钩子函数
   - 模块可以在pre钩子中修改参数或执行准备工作
   - 调用原始函数完成进程创建
   - 模块可以在post钩子中执行后续操作
4. 隐藏阶段 ：
   
   - 将自身和模块的内存标记为匿名内存
   - 从solist中移除相关库的条目
   - 确保在/proc/maps中不可见
## 4. 技术亮点
1. 巧妙的注入方式 ：利用系统原生的native bridge机制，无需复杂的进程注入代码
2. 灵活的钩子系统 ：支持多个模块在不同阶段执行代码，便于扩展功能
3. 跨版本兼容性 ：支持多种Android版本的不同函数签名
4. 隐藏机制 ：实现了内存隐藏，提高了反检测能力
5. 模块化设计 ：核心功能与模块分离，便于维护和扩展
Riru通过这些技术实现了强大的系统级Hook能力，为Android开发和定制提供了灵活的扩展机制。