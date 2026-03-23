# WeKit Xposed模块开发学习指南

> 📚 **完整的 WeKit 项目开发教程**  
> 🎯 从零开始掌握 Xposed模块开发  
> ⚡ 基于实际项目的实战教学

---

## 📋 目录

- [第一阶段：理解基础](#第一阶段理解基础)
- [第二阶段：核心概念](#第二阶段核心概念)
- [第三阶段：实战演练](#第三阶段实战演练)
- [第四阶段：调试与配置](#第四阶段调试与配置)
- [第五阶段：进阶练习](#第五阶段进阶练习)
- [常见问题 FAQ](#常见问题-faq)
- [参考资源](#参考资源)

---

## 第一阶段：理解基础

### 1.1 Xposed Framework 简介

**什么是 Xposed？**

Xposed 是一个运行在 Android 系统上的框架，允许开发者在不修改 APK 文件的情况下，动态修改应用程序的行为。

**核心原理：**
```
┌─────────────────┐
│   Xposed 框架   │
│  (系统级服务)   │
└────────┬────────┘
         │ Hook 机制
         ▼
┌─────────────────┐
│   WeKit 模块    │
│  (注入微信)     │
└────────┬────────┘
         │ 拦截方法调用
         ▼
┌─────────────────┐
│     微信应用    │
│  (com.tencent.mm)│
└─────────────────┘
```

**关键概念：**
- **Hook（钩子）**: 拦截方法调用的技术
- **XC_MethodHook**: Xposed 提供的 Hook 工具类
- **XposedHelpers**: 辅助类，简化反射操作
- **XposedBridge**: 桥接类，提供底层 Hook 功能

### 1.2 WeKit 项目概述

**项目信息：**
- **类型**: Android Xposed 微信增强模块
- **包名**: `moe.ouom.wekit`
- **宿主**: 微信 (com.tencent.mm >= 8.0.67)
- **系统要求**: Android >= 10.0
- **许可证**: GPL-3.0

**核心特性：**
1. ✅ **MemoryDexLoader** - 内存加载 Dex，安全性更高
2. ✅ **KSP 代码生成** - 自动注册 HookItem
3. ✅ **DSL 简化语法** - 优雅的 Hook 编写方式
4. ✅ **DexKit 集成** - 特征码查找，适配多版本
5. ✅ **缓存机制** - 避免重复扫描 Dex
6. ✅ **多进程支持** - 区分主进程、工具进程

**警告：**
> ⚠️ 本项目仅供学习研究使用  
> ⚠️ 存在账号被封禁风险，请勿用于主号  
> ⚠️ 严禁用于非法用途

### 1.3 开发环境搭建

**必需工具：**
```bash
# 1. Android Studio (推荐最新稳定版)
# 2. JDK 17+
# 3. NDK (用于编译 C++ 代码)
# 4. Git

# 克隆项目
git clone https://github.com/你的仓库/WeKit.git
cd WeKit

# 使用 Gradle Wrapper 构建
./gradlew assembleDebug
```

**项目同步后结构：**
```
WeKit/
├── app/                          # 主模块
│   ├── src/main/java/           # Java/Kotlin 源码
│   ├── src/main/cpp/            # C++ Native 代码
│   ├── src/main/res/            # 资源文件
│   └── AndroidManifest.xml      # 模块配置
├── libs/common/                 # 通用库
├── build-logic/                 # 构建逻辑
└── README.md                    # 项目说明
```

---

## 第二阶段：核心概念

### 2.1 HookItem - 功能单元

**什么是 HookItem？**

HookItem 是 WeKit 中的基本功能单元，每个独立的功能都是一个 HookItem 类。

**两种类型：**

| 类型 | 继承类 | 用途 | 示例 |
|------|--------|------|------|
| 开关型 | `BaseSwitchFunctionHookItem` | 可启用/禁用 | 消息防撤回 |
| 点击型 | `BaseClickableFunctionHookItem` | 带 UI 交互 | 清除缓存 |

**基本结构：**
```kotlin
@HookItem(path = "分类/功能名称", desc = "功能描述")
class MyHookItem : BaseSwitchFunctionHookItem() {
    
    override fun entry(classLoader: ClassLoader) {
        // Hook 逻辑入口
    }
}
```

**关键注解：**
- `@HookItem` - 标记此类为 HookItem，KSP 会自动生成注册代码
- `path` - 功能路径，用于配置保存和 UI 显示
- `desc` - 功能描述，显示在设置界面

### 2.2 IDexFind - Dex 查找接口

**为什么需要 IDexFind？**

微信经过混淆，类名和方法名都是 `a.b.c` 这种无意义的短名。IDexFind 通过特征码定位目标方法。

**实现方式：**
```kotlin
interface IDexFind {
    fun dexFind(dexKit: DexKitBridge): Map<String, String>
}
```

**返回值：**
- `Map<String, String>` - 键为属性名，值为方法/类的 Descriptor 字符串
- 系统会自动缓存这些 Descriptor
- 当微信版本更新导致方法变化时，会自动提示重新扫描

### 2.3 DSL 委托 - 简化声明

**传统方式 vs DSL 方式：**

❌ **传统方式（繁琐）：**
```kotlin
// 需要手动管理 Key 和反射
private var methodDescriptor: String? = null

fun loadDescriptor() {
    methodDescriptor = config.getString("key_xxx")
}

fun hook() {
    val method = DexKit.findMethod(methodDescriptor)
    // ...
}
```

✅ **DSL 方式（简洁）：**
```kotlin
// 一行搞定，自动生成 Key，自动反射
private val TargetMethod by dexMethod()

override fun entry(classLoader: ClassLoader) {
    TargetMethod.toDexMethod {
        hook { /* ... */ }
    }
}
```

**支持的委托类型：**
```kotlin
// 方法委托
private val MethodA by dexMethod()

// 类委托
private val ClassB by dexClass()

// 带自定义 Key（不推荐）
private val MethodC by dexMethod(key = "custom_key")
```

### 2.4 执行流程

**模块启动流程：**
```
1. Xposed 框架加载 WeKit
   ↓
2. ModuleAppImpl.onCreate()
   ↓
3. StartupInfo 初始化
   ↓
4. UnifiedEntryPoint.entry()
   ↓
5. HookItemLoader.loadHookItem()
   ├─ 从缓存加载 Descriptor
   ├─ 检查缓存有效性
   └─ 异步修复过期缓存
   ↓
6. HookItemFactory.getAllItemList()
   ↓
7. 遍历所有 HookItem
   ├─ 检查 isEnabled (用户配置)
   ├─ 检查 targetProcess (进程匹配)
   └─ 调用 entry()
   ↓
8. Hook 生效
```

**进程控制：**
```kotlin
// 指定目标进程
override val targetProcess: Int
    get() = SyncUtils.PROC_MAIN  // 仅在主进程运行

// 或始终运行
override val targetProcess: Int
    get() = SyncUtils.PROC_ALL   // 在所有进程运行
```

---

## 第三阶段：实战演练

### 示例 1：Hello World - 修改版本号显示

**目标：** 将微信版本号改为自定义文本

**文件位置：** `app/src/main/java/moe/ouom/wekit/hooks/item/example/VersionModifier.kt`

```kotlin
package moe.ouom.wekit.hooks.item.example

import de.robv.android.xposed.XposedHelpers
import moe.ouom.wekit.core.model.BaseSwitchFunctionHookItem
import moe.ouom.wekit.hooks.core.annotation.HookItem
import moe.ouom.wekit.util.log.WeLogger

/**
 * 示例 1：修改微信版本号显示
 * 
 * 知识点：
 * - 如何创建简单的 HookItem
 * - 使用 XposedHelpers 修改静态字段
 * - WeLogger 日志输出
 */
@HookItem(path = "示例功能/修改版本号", desc = "将微信版本号改为自定义文本")
class VersionModifier : BaseSwitchFunctionHookItem() {

    override fun entry(classLoader: ClassLoader) {
        super.entry(classLoader)
        
        WeLogger.i("VersionModifier 开始加载")
        
        try {
            // 步骤 1: 找到 BuildInfo 类
            val buildInfoClass = XposedHelpers.findClass(
                "com.tencent.mm.BuildInfo",  // 微信的 BuildInfo 类
                classLoader                   // 类加载器
            )
            
            // 步骤 2: 修改 VERSION 字段
            XposedHelpers.setStaticObjectField(
                buildInfoClass,
                "VERSION",           // 字段名
                "WeKit 定制版 v1.0"   // 新值
            )
            
            WeLogger.i("VersionModifier 修改成功 ✨")
            
        } catch (e: Throwable) {
            WeLogger.e("VersionModifier 执行失败 ❌", e)
        }
    }
    
    override fun unload(classLoader: ClassLoader) {
        WeLogger.i("VersionModifier 卸载")
        super.unload(classLoader)
    }
}
```

**关键点解析：**
1. `@HookItem(path = "...")` - 定义功能路径
2. `XposedHelpers.findClass()` - 查找类
3. `XposedHelpers.setStaticObjectField()` - 修改静态字段
4. `WeLogger.i/e()` - 输出日志
5. 异常处理 - 防止崩溃

**测试方法：**
1. 编译安装模块
2. 在 Xposed 中激活
3. 打开微信 → 我 → 设置 → 关于微信
4. 查看版本号是否改变

---

### 示例 2：消息拦截器

**目标：** 拦截发送的消息并添加前缀

```kotlin
package moe.ouom.wekit.hooks.item.example

import de.robv.android.xposed.XC_MethodHook
import de.robv.android.xposed.XposedBridge
import de.robv.android.xposed.XposedHelpers
import moe.ouom.wekit.core.model.BaseSwitchFunctionHookItem
import moe.ouom.wekit.hooks.core.annotation.HookItem
import moe.ouom.wekit.util.log.WeLogger

/**
 * 示例 2：消息拦截器
 * 
 * 知识点：
 * - Hook 方法的三种方式
 * - 修改方法参数
 * - beforeHookedMethod vs afterHookedMethod
 */
@HookItem(path = "示例功能/消息拦截", desc = "拦截发送的消息并添加前缀")
class MessageInterceptor : BaseSwitchFunctionHookItem() {

    override fun entry(classLoader: ClassLoader) {
        super.entry(classLoader)
        
        try {
            // 方式 1: 使用 hookBefore/hookAfter (父类提供)
            testHookWay1(classLoader)
            
            // 方式 2: 使用 XposedBridge.hookMethod (原生方式)
            testHookWay2(classLoader)
            
            // 方式 3: 使用 XposedHelpers.hookAllMethods (批量 Hook)
            testHookWay3(classLoader)
            
        } catch (e: Throwable) {
            WeLogger.e("MessageInterceptor 初始化失败", e)
        }
    }
    
    // ========== 方式 1: 使用父类提供的 hookBefore/hookAfter ==========
    private fun testHookWay1(classLoader: ClassLoader) {
        val messageSendClass = XposedHelpers.findClass(
            "com.tencent.mm.plugin.messenger.foundation.a.j",
            classLoader
        )
        
        val sendMethod = XposedHelpers.findMethodExactIfExists(
            messageSendClass,
            "b",  // 方法名（混淆后的短名）
            String::class.java,      // 参数 1: 聊天对象 ID
            String::class.java,      // 参数 2: 消息内容
            Int::class.javaPrimitiveType  // 参数 3: 消息类型
        ) ?: return
        
        // 使用父类的 hookBefore 方法
        hookBefore(sendMethod) { param ->
            val originalMessage = param.args[1] as String
            WeLogger.i("【方式 1】拦截消息：$originalMessage")
            
            // 修改消息内容
            param.args[1] = "[WeKit] $originalMessage"
            
            WeLogger.i("【方式 1】修改后：${param.args[1]}")
        }
    }
    
    // ========== 方式 2: 使用 XposedBridge.hookMethod (原生方式) ==========
    private fun testHookWay2(classLoader: ClassLoader) {
        val messageSendClass = XposedHelpers.findClass(
            "com.tencent.mm.sdk.platformtools.t",  // 假设的日志类
            classLoader
        )
        
        val infoMethod = XposedHelpers.findMethodExactIfExists(
            messageSendClass,
            "i",  // info 方法
            String::class.java,  // tag
            String::class.java,  // message
            VarargParameter<Any>()  // 可变参数
        ) ?: return
        
        XposedBridge.hookMethod(infoMethod, object : XC_MethodHook() {
            override fun beforeHookedMethod(param: MethodHookParam) {
                val tag = param.args[0] as String
                val message = param.args[1] as String
                
                WeLogger.i("【方式 2】日志拦截：[$tag] $message")
                
                // 可以修改参数
                param.args[1] = "【 intercepted 】$message"
            }
            
            override fun afterHookedMethod(param: MethodHookParam) {
                WeLogger.i("【方式 2】日志已输出")
            }
        })
    }
    
    // ========== 方式 3: 批量 Hook 所有同名方法 ==========
    private fun testHookWay3(classLoader: ClassLoader) {
        val targetClass = XposedHelpers.findClassIfExists(
            "com.tencent.mm.ui.base.n",  // Toast 相关类
            classLoader
        ) ?: return
        
        // Hook 所有名为 "show" 的方法
        hookAfter(targetClass, "show") { param ->
            WeLogger.i("【方式 3】Toast 显示：${param.method}")
        }
    }
}
```

**Hook 时机选择：**

| 时机 | 方法 | 用途 |
|------|------|------|
| Before | `beforeHookedMethod` | 修改参数、阻止执行 |
| After | `afterHookedMethod` | 修改返回值、清理资源 |

**修改参数：**
```kotlin
hookBefore(method) { param ->
    param.args[0] = "新值"  // 修改第 1 个参数
    param.args[1] = 123     // 修改第 2 个参数
}
```

**修改返回值：**
```kotlin
hookAfter(method) { param ->
    param.result = "新返回值"  // 修改返回值
}
```

**阻止方法执行：**
```kotlin
hookBefore(method) { param ->
    param.result = null  // 直接返回 null，原方法不会执行
    // 或者
    param.throwable = Exception("故意抛出异常")
}
```

---

### 示例 3：使用 DexKit 查找方法（高级）

**目标：** 不依赖硬编码的类名和方法名，使用特征码查找

```kotlin
package moe.ouom.wekit.hooks.item.example

import moe.ouom.wekit.core.dsl.dexMethod
import moe.ouom.wekit.core.dsl.dexClass
import moe.ouom.wekit.core.model.BaseSwitchFunctionHookItem
import moe.ouom.wekit.dexkit.intf.IDexFind
import moe.ouom.wekit.hooks.core.annotation.HookItem
import moe.ouom.wekit.util.log.WeLogger
import org.luckypray.dexkit.DexKitBridge

/**
 * 示例 3：使用 DexKit 查找方法
 * 
 * 知识点：
 * - DSL 委托声明
 * - dexFind() 方法实现
 * - 特征码匹配
 * - toDexMethod() 使用
 */
@HookItem(path = "示例功能/DexKit 示例", desc = "使用特征码查找方法")
class DexKitExample : BaseSwitchFunctionHookItem(), IDexFind {

    // ========== DSL 委托声明 ==========
    
    // 方法委托 - 自动管理 Key 和反射
    private val SendMessageMethod by dexMethod()
    
    // 类委托 - 查找目标类
    private val MessageClass by dexClass()
    
    // 另一个方法委托
    private val LogMethod by dexMethod()

    // ========== Dex 查找逻辑 ==========
    
    /**
     * Dex 查找入口
     * 
     * @param dexKit DexKit 桥接对象
     * @return Map<属性名，descriptor 字符串>
     * 
     * 系统会：
     * 1. 调用此方法获取所有方法和类的描述符
     * 2. 将结果缓存到配置文件
     * 3. 下次启动时直接从缓存加载
     * 4. 当查找逻辑改变时，自动提示重新扫描
     */
    override fun dexFind(dexKit: DexKitBridge): Map<String, String> {
        val descriptors = mutableMapOf<String, String>()

        // --- 查找 SendMessageMethod ---
        SendMessageMethod.find(dexKit, descriptors = descriptors) {
            matcher {
                // 条件 1: 方法名（如果知道）
                name = "sendMessage"
                
                // 条件 2: 参数数量
                paramCount = 2
                
                // 条件 3: 使用的字符串常量（强特征）
                usingStrings("发送消息", "sendMessage")
                
                // 条件 4: 返回类型
                returnType = "boolean"
                
                // 条件 5: 所在类的特征
                classMatcher {
                    usingStrings("Message", "Chat")
                }
            }
        }

        // --- 查找 MessageClass ---
        MessageClass.find(dexKit, descriptors = descriptors) {
            matcher {
                // 类名包含特定字符串
                className = "Message"
                
                // 实现的接口
                implements("android.os.Parcelable")
                
                // 包含的字段
                hasField("content")
                hasField("sender")
            }
        }

        // --- 查找 LogMethod ---
        LogMethod.find(dexKit, descriptors = descriptors) {
            matcher {
                // 更复杂的组合条件
                paramCount = 3
                paramTypes(0) = "java.lang.String"  // 第 1 个参数是 String
                paramTypes(1) = "java.lang.String"  // 第 2 个参数是 String
                paramTypes(2) = "java.lang.Object[]" // 第 3 个参数是 Object[]
                
                // 方法体中调用了 System.out.println
                invoke("java.io.PrintStream", "println", "(Ljava/lang/String;)V")
            }
        }

        return descriptors
    }

    // ========== Hook 执行逻辑 ==========
    
    override fun entry(classLoader: ClassLoader) {
        super.entry(classLoader)

        WeLogger.i("DexKitExample 开始加载")

        // 方式 1: 使用 DSL 的 toDexMethod（推荐）
        SendMessageMethod.toDexMethod {
            hook {
                // beforeIfEnabled - 方法执行前（如果功能已启用）
                beforeIfEnabled { param ->
                    WeLogger.i("SendMessageMethod 被调用")
                    WeLogger.i("参数 1: ${param.args[0]}")
                    WeLogger.i("参数 2: ${param.args[1]}")
                    
                    // 可以在这里修改参数
                    // param.args[0] = "modified"
                }
                
                // afterIfEnabled - 方法执行后（如果功能已启用）
                afterIfEnabled { param ->
                    WeLogger.i("SendMessageMethod 执行完成")
                    WeLogger.i("返回值：${param.result}")
                    
                    // 可以在这里修改返回值
                    // param.result = true
                }
            }
        }

        // 方式 2: 使用 dexClass 直接访问类
        MessageClass.toDexClass {
            // 直接使用 clazz 访问器
            val instance = java.lang.reflect.Constructor.newInstance(
                this.clazz,  // 获取 Class 对象
                "param1", "param2"
            )
            
            WeLogger.i("创建了实例：$instance")
        }

        // 方式 3: 结合传统 XposedHelpers
        LogMethod.toDexMethod {
            hook {
                beforeIfEnabled { param ->
                    // 禁止输出某些日志
                    if ((param.args[0] as String).contains("敏感词")) {
                        param.result = null  // 阻止输出
                        WeLogger.w("拦截敏感日志")
                    }
                }
            }
        }
    }
}
```

**特征码选择技巧：**

| 特征类型 | 稳定性 | 推荐度 | 示例 |
|---------|--------|--------|------|
| 字符串常量 | ⭐⭐⭐⭐⭐ | ✅ | `usingStrings("固定文本")` |
| 参数数量 | ⭐⭐⭐⭐ | ✅ | `paramCount = 2` |
| 方法调用 | ⭐⭐⭐⭐ | ✅ | `invoke("类名", "方法名", "签名")` |
| 返回类型 | ⭐⭐⭐ | ⚠️ | `returnType = "void"` |
| 方法名 | ⭐ | ❌ | `name = "a"` (易混淆) |

**最佳实践：**
```kotlin
// ✅ 推荐：组合多个稳定特征
matcher {
    paramCount = 3
    usingStrings("固定文案", "AnotherString")
    invoke("android.util.Log", "d", "(Ljava/lang/String;Ljava/lang/String;)I")
}

// ❌ 不推荐：只使用不稳定的特征
matcher {
    name = "a"  // 混淆后的短名，极易变化
    returnType = "int"
}
```

---

### 示例 4：点击型功能（带 UI 交互）

**目标：** 创建一个点击按钮弹出对话框的功能

```kotlin
package moe.ouom.wekit.hooks.item.example

import android.content.Context
import com.afollestad.materialdialogs.MaterialDialog
import moe.ouom.wekit.core.model.BaseClickableFunctionHookItem
import moe.ouom.wekit.hooks.core.annotation.HookItem
import moe.ouom.wekit.util.log.WeLogger

/**
 * 示例 4：点击型功能
 * 
 * 知识点：
 * - 继承 BaseClickableFunctionHookItem
 * - 实现 onClick() 方法
 * - 使用 MaterialDialog 创建对话框
 * - noSwitchWidget() 的使用
 */
@HookItem(path = "开发者选项/测试对话框", desc = "点击弹出测试对话框")
class TestDialog : BaseClickableFunctionHookItem() {

    /**
     * 点击事件回调
     * 
     * @param context 当前 Activity 或 Application 上下文
     */
    override fun onClick(context: Context?) {
        context?.let { ctx ->
            // 创建 MaterialDialog 对话框
            MaterialDialog(ctx)
                .title(text = "WeKit 测试")
                .message(text = "这是一个测试对话框\n\n功能已正常加载！")
                .positiveButton(text = "确定") { dialog ->
                    WeLogger.i("用户点击了确定按钮")
                    dialog.dismiss()
                }
                .negativeButton(text = "取消") { dialog ->
                    WeLogger.i("用户点击了取消按钮")
                    dialog.dismiss()
                }
                .neutralButton(text = "了解更多") { dialog ->
                    WeLogger.i("用户点击了了解更多")
                    dialog.dismiss()
                }
                .show()
        }
    }

    /**
     * 是否隐藏开关控件
     * 
     * 返回 true: 不显示开关，只显示按钮
     * 返回 false: 显示开关和按钮（默认）
     */
    override fun noSwitchWidget(): Boolean = true
}
```

**扩展：更复杂的对话框**

```kotlin
@HookItem(path = "开发者选项/高级设置", desc = "打开高级设置面板")
class AdvancedSettings : BaseClickableFunctionHookItem() {

    override fun onClick(context: Context?) {
        context?.let { ctx ->
            MaterialDialog(ctx)
                .title(text = "高级设置")
                
                // 添加单选列表
                .listItems(items = listOf("选项 1", "选项 2", "选项 3")) { _, index, _ ->
                    when (index) {
                        0 -> WeLogger.i("选择了选项 1")
                        1 -> WeLogger.i("选择了选项 2")
                        2 -> WeLogger.i("选择了选项 3")
                    }
                }
                
                // 添加输入框
                .input(hint = "请输入...", waitForPositiveButton = true) { _, text ->
                    WeLogger.i("用户输入：$text")
                }
                
                // 添加复选框
                .checkBoxPrompt(text = "不再提示", initialState = false) { checked ->
                    WeLogger.i("复选框状态：$checked")
                }
                
                .positiveButton(text = "保存")
                .negativeButton(text = "取消")
                .show()
        }
    }
}
```

---

### 示例 5：综合实战 - 朋友圈图片下载

**目标：** 实现朋友圈长按照片保存功能

```kotlin
package moe.ouom.wekit.hooks.item.chat

import android.graphics.Bitmap
import android.os.Environment
import de.robv.android.xposed.XC_MethodHook
import de.robv.android.xposed.XposedBridge
import de.robv.android.xposed.XposedHelpers
import moe.ouom.wekit.core.dsl.dexMethod
import moe.ouom.wekit.core.dsl.dexClass
import moe.ouom.wekit.core.model.BaseSwitchFunctionHookItem
import moe.ouom.wekit.dexkit.intf.IDexFind
import moe.ouom.wekit.hooks.core.annotation.HookItem
import moe.ouom.wekit.util.log.WeLogger
import org.luckypray.dexkit.DexKitBridge
import java.io.File
import java.io.FileOutputStream

/**
 * 综合实战：朋友圈图片下载
 * 
 * 功能：
 * - 长按朋友圈图片时自动保存到相册
 * - 使用 DexKit 查找图片加载方法
 * - 监听长按事件
 * - 保存图片到指定目录
 */
@HookItem(path = "朋友圈/图片自动保存", desc = "长按朋友圈图片自动保存到相册")
class MomentsImageSaver : BaseSwitchFunctionHookItem(), IDexFind {

    // DSL 委托
    private val LoadImageMethod by dexMethod()
    private val ImageLoaderClass by dexClass()
    private val OnLongClickListenerClass by dexClass()

    // 保存目录
    private val saveDir = File(
        Environment.getExternalStorageDirectory(),
        "WeKit/SavedImages"
    )

    override fun dexFind(dexKit: DexKitBridge): Map<String, String> {
        val descriptors = mutableMapOf<String, String>()

        // 查找图片加载方法
        LoadImageMethod.find(dexKit, descriptors = descriptors) {
            matcher {
                paramCount = 3
                paramTypes(0) = "java.lang.String"  // URL
                paramTypes(1) = "android.widget.ImageView"  // ImageView
                usingStrings("download", "image", "bitmap")
            }
        }

        // 查找图片加载器类
        ImageLoaderClass.find(dexKit, descriptors = descriptors) {
            matcher {
                className = "ImageLoader"
                hasField("cache")
                hasField("url")
            }
        }

        // 查找长按监听器
        OnLongClickListenerClass.find(dexKit, descriptors = descriptors) {
            matcher {
                implements("android.view.View\$OnLongClickListener")
            }
        }

        return descriptors
    }

    override fun entry(classLoader: ClassLoader) {
        super.entry(classLoader)

        WeLogger.i("MomentsImageSaver 开始加载")

        try {
            // 确保保存目录存在
            if (!saveDir.exists()) {
                saveDir.mkdirs()
                WeLogger.i("创建保存目录：${saveDir.absolutePath}")
            }

            // Hook 图片加载完成回调
            hookImageLoad(classLoader)
            
            // Hook 长按事件
            hookLongClick(classLoader)

        } catch (e: Throwable) {
            WeLogger.e("MomentsImageSaver 加载失败", e)
        }
    }

    /**
     * Hook 图片加载
     */
    private fun hookImageLoad(classLoader: ClassLoader) {
        LoadImageMethod.toDexMethod {
            hook {
                afterIfEnabled { param ->
                    // 方法执行后获取加载好的 Bitmap
                    val bitmap = param.result as? Bitmap ?: return@afterIfEnabled
                    
                    WeLogger.i("图片加载成功：${bitmap.width}x${bitmap.height}")
                    
                    // 保存到文件
                    saveBitmap(bitmap)
                }
            }
        }
    }

    /**
     * Hook 长按事件
     */
    private fun hookLongClick(classLoader: ClassLoader) {
        val imageViewClass = XposedHelpers.findClassIfExists(
            "android.widget.ImageView",
            classLoader
        ) ?: return

        // Hook 所有 ImageView 的 setOnLongClickListener
        XposedBridge.hookAllMethods(
            imageViewClass,
            "setOnLongClickListener",
            object : XC_MethodHook() {
                override fun beforeHookedMethod(param: MethodHookParam) {
                    val listener = param.args[0] as? View.OnLongClickListener
                    
                    // 包装原始监听器
                    val wrapper = View.OnLongClickListener { view ->
                        WeLogger.i("ImageView 长按事件")
                        
                        // 先执行原始逻辑
                        listener?.onLongClick(view)
                        
                        // 尝试保存图片
                        val drawable = (view as? ImageView)?.drawable
                        if (drawable is BitmapDrawable) {
                            saveBitmap(drawable.bitmap)
                        }
                        
                        true
                    }
                    
                    param.args[0] = wrapper
                }
            }
        )
    }

    /**
     * 保存 Bitmap 到文件
     */
    private fun saveBitmap(bitmap: Bitmap) {
        try {
            val fileName = "IMG_${System.currentTimeMillis()}.jpg"
            val file = File(saveDir, fileName)
            
            FileOutputStream(file).use { out ->
                bitmap.compress(Bitmap.CompressFormat.JPEG, 90, out)
            }
            
            WeLogger.i("图片已保存：${file.absolutePath}")
            
            // 通知媒体库刷新（可选）
            // MediaScannerConnection.scanFile(...)
            
        } catch (e: Throwable) {
            WeLogger.e("保存图片失败", e)
        }
    }

    override fun unload(classLoader: ClassLoader) {
        WeLogger.i("MomentsImageSaver 卸载")
        super.unload(classLoader)
    }
}
```

---

## 第四阶段：调试与配置

### 4.1 日志查看

**方法 1：ADB Logcat**
```bash
# 过滤 WeKit 日志
adb logcat | grep -E "WeKit|wekit"

# 只看错误日志
adb logcat | grep "WeLogger.E"

# 保存日志到文件
adb logcat > wekit_log.txt
```

**方法 2：Android Studio Logcat**
```
Filter: WeKit
Level: Debug
Package: com.tencent.mm
```

**日志级别：**
```kotlin
WeLogger.d("Debug 信息")    // 详细调试
WeLogger.i("Info 信息")     // 一般信息
WeLogger.w("Warning 信息")  // 警告
WeLogger.e("Error 信息", e) // 错误 + 异常堆栈
WeLogger.v("Verbose 信息")  // 最详细
```

### 4.2 配置管理

**读取配置：**
```kotlin
val config = WeConfig.getDefaultConfig()

// 读取布尔值
val enabled = config.getBooleanOrFalse("$PrekXXX$path/isEnabled")

// 读取字符串
val customText = config.getString("custom_key", "默认值")

// 读取整数
val maxCount = config.getInt("max_count", 10)
```

**写入配置：**
```kotlin
val editor = WeConfig.getDefaultConfig().edit()
editor.putBoolean("my_key", true)
editor.putInt("count", 100)
editor.apply()
```

### 4.3 调试技巧

**技巧 1：添加详细日志**
```kotlin
override fun entry(classLoader: ClassLoader) {
    WeLogger.i("===== ${this::class.java.simpleName} 开始加载 =====")
    WeLogger.i("ClassLoader: $classLoader")
    WeLogger.i("isEnabled: $isEnabled")
    
    // ... 业务逻辑
    
    WeLogger.i("===== ${this::class.java.simpleName} 加载完成 =====")
}
```

**技巧 2：捕获所有异常**
```kotlin
try {
    // 危险操作
} catch (e: Throwable) {  // 捕获所有异常和错误
    WeLogger.e("发生异常", e)
    WeLogger.printStackTrace()  // 打印完整堆栈
}
```

**技巧 3：检查类和方法是否存在**
```kotlin
// 安全查找，不存在返回 null
val clazz = XposedHelpers.findClassIfExists("com.xxx.ClassName", classLoader)
if (clazz == null) {
    WeLogger.w("类不存在：com.xxx.ClassName")
    return
}

// 安全查找方法
val method = XposedHelpers.findMethodExactIfExists(...)
```

### 4.4 常见问题 FAQ

#### Q1: Hook 不生效怎么办？

**排查步骤：**
1. ✅ 检查模块是否在 Xposed 中激活
2. ✅ 检查微信是否在作用域内
3. ✅ 重启微信（强制重启）
4. ✅ 查看日志确认 `entry()` 是否被调用
5. ✅ 检查类名和方法签名是否正确
6. ✅ 确认功能开关已启用

#### Q2: 找不到类或方法？

**解决方案：**
```kotlin
// 1. 使用 findClassIfExists 安全查找
val clazz = XposedHelpers.findClassIfExists("xxx", classLoader)

// 2. 使用 DexKit 特征码查找（推荐）
private val TargetMethod by dexMethod()

override fun dexFind(dexKit: DexKitBridge) {
    TargetMethod.find(dexKit) {
        matcher {
            usingStrings("稳定字符串")
            paramCount = 2
        }
    }
}
```

#### Q3: 如何处理多个微信版本？

**方案 1：使用 DexKit 缓存**
```kotlin
// 系统会自动为不同版本维护不同的缓存
// 当检测到版本变化时，会提示重新扫描
```

**方案 2：版本判断**
```kotlin
val versionCode = HostInfo.getVersionCode()

when (versionCode) {
    in 1000..1999 -> hookVersion1()
    in 2000..2999 -> hookVersion2()
    else -> WeLogger.w("未知版本：$versionCode")
}
```

#### Q4: 模块导致微信崩溃？

**解决方法：**
1. 查看 `/data/data/com.tencent.mm/files/tombstones/` 下的崩溃日志
2. 检查是否有未捕获的异常
3. 在 `try-catch` 中包裹所有代码
4. 使用 `WeLogger.printStackTrace()` 记录堆栈

#### Q5: 配置不保存？

**检查点：**
```kotlin
// 1. 确保使用了正确的 path
@HookItem(path = "分类/名称", desc = "...")

// 2. 读取配置时使用完整路径
val key = "$PrekXXX${path}/isEnabled"

// 3. 保存后调用 apply()
editor.apply()
```

---

## 第五阶段：进阶练习

### 练习任务 1：修改微信步数 ⭐⭐

**目标：** Hook 微信运动，修改步数显示

**提示：**
1. 查找包含 "step"、"walk"、"sport" 等关键词的类
2. 找到返回步数的方法（通常返回 int）
3. 修改返回值

**参考答案框架：**
```kotlin
@HookItem(path = "朋友圈/修改步数", desc = "自定义微信运动步数")
class StepModifier : BaseSwitchFunctionHookItem() {
    override fun entry(classLoader: ClassLoader) {
        // TODO: 实现你的逻辑
    }
}
```

### 练习任务 2：自动回复消息 ⭐⭐⭐

**目标：** 收到特定消息时自动回复

**要求：**
- 检测收到的消息内容
- 匹配关键词自动回复
- 可配置回复内容

**提示：**
1. 找到消息接收回调方法
2. 在 `afterHookedMethod` 中检测消息
3. 调用发送方法

### 练习任务 3：朋友圈小视频下载 ⭐⭐⭐⭐

**目标：** 一键下载朋友圈小视频

**要求：**
- 长按视频播放按钮
- 弹出下载菜单
- 保存到相册

**提示：**
1. 找到视频播放器类
2. Hook 播放完成回调
3. 获取视频文件路径
4. 复制到指定目录

### 练习任务 4：消息防撤回 ⭐⭐⭐⭐⭐

**目标：** 阻止别人撤回消息

**要求：**
- 拦截撤回请求
- 保留原始消息
- 提示"对方尝试撤回消息"

**挑战：**
- 需要精准定位撤回方法
- 处理各种消息类型（文字、图片、视频）
- 避免被检测

---

## 参考资源

### 官方文档
- [Xposed API 文档](https://api.xposed.info/)
- [DexKit 官方文档](https://dexkit.github.io/)
- [Material Dialogs 文档](https://afollestad.github.io/material-dialogs/)

### 参考项目
- [QAuxiliary](https://github.com/cinit/QAuxiliary) - QQ 辅助工具
- [WAuxiliary](https://github.com/HdShare/WAuxiliary_Public) - 微信辅助工具
- [TimTool](https://github.com/suzhelan/TimTool) - Tim功能性增强

### 工具推荐
- **JADX** - APK 反编译工具
- **Frida** - 动态插桩工具
- **MT 管理器** - 手机端文件管理
- **Xposed Installer** - Xposed 框架管理

### 学习路线
```
Week 1-2: Xposed 基础
  ├─ 了解 Xposed 原理
  ├─ 学习 XC_MethodHook 使用
  └─ 完成示例 1、2

Week 3-4: DexKit 入门
  ├─ 理解特征码查找
  ├─ 学习 DSL 语法
  └─ 完成示例 3

Week 5-6: 实战练习
  ├─ 实现简单功能
  ├─ 调试和优化
  └─ 完成练习任务 1、2

Week 7-8: 进阶提升
  ├─ 复杂功能实现
  ├─ 多版本适配
  └─ 完成练习任务 3、4
```

---

## 附录

### A. 常用代码片段

**快速 Hook 方法：**
```kotlin
hookBefore(targetClass, "methodName") { param ->
    WeLogger.i("方法被调用：${param.args.joinToString()}")
}
```

**查找类：**
```kotlin
val clazz = XposedHelpers.findClass("com.xxx.ClassName", classLoader)
```

**查找方法：**
```kotlin
val method = XposedHelpers.findMethodExact(
    clazz,
    "methodName",
    String::class.java,
    Int::class.javaPrimitiveType
)
```

**修改字段：**
```kotlin
XposedHelpers.setObjectField(instance, "fieldName", newValue)
XposedHelpers.setStaticObjectField(clazz, "STATIC_FIELD", newValue)
```

**调用方法：**
```kotlin
XposedHelpers.callMethod(instance, "methodName", arg1, arg2)
XposedHelpers.callStaticMethod(clazz, "methodName", arg1)
```

### B. 命名规范

**文件命名：**
- Kotlin 文件：`PascalCase.kt` (如 `MessageInterceptor.kt`)
- Java 文件：`PascalCase.java`

**变量命名：**
```kotlin
// 驼峰命名
val userName = "xxx"
val messageCount = 10

// 常量全大写
const val MAX_COUNT = 100
val PREK_XXX = "prek_xxx"
```

**类命名：**
```kotlin
// 功能描述性命名
class MessageInterceptor      // ✅ 好
class HookDemo1               // ❌ 不好

// 后缀规范
class XXXModifier             // 修改器
class XXXSaver                // 保存器
class XXXCleaner              // 清理器
```

### C. 提交清单

在提交代码前请检查：
- [ ] 代码已通过编译
- [ ] 添加了必要的日志输出
- [ ] 异常已妥善处理
- [ ] 遵循了代码规范
- [ ] 测试了基本功能
- [ ] 更新了相关文档

---

## 结语

恭喜你完成了 WeKit 开发入门教程！🎉

**记住：**
- 实践是最好的老师，多动手写代码
- 遇到问题先查日志，再查文档
- 保持好奇心，探索更多可能性
- 遵守开源协议，尊重他人劳动成果

**下一步：**
1.  fork 项目到本地
2.  尝试实现一个简单功能
3.  阅读优秀 PR 学习经验
4.  参与社区讨论

祝你学习愉快！🚀

---

*最后更新：2026-03-23*  
*维护者：WeKit Team*  
*许可证：GPL-3.0*
