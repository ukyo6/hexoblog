title: AS3自定义lint
date: 2018-10-28 22:59:09
tags:
- Android
categories:
- Dev
---
  
<br>
## 概述
lint是代码风格和语法规则的检查工具,不限于android平台,其他例如jslint,eslint..
在最新的稳定版本中,官方提供了342个定义好的lint规则,基本满足开发中的需要,基本介绍和使用方法参照[官方教程](https://developer.android.google.cn/studio/write/lint) 
 
此外我们还需要某些特殊的场景下的检查,例如 序列化的内部类也需要序列化,如果没有序列化,我们要给出相应的提示
![image](http://upload-images.jianshu.io/upload_images/4960358-e025ec4d8e5259e1?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
网上的demo大部分是3.0之前基于lamlok,使用aar依赖的,已经不适用于AS3.0  
自己基于AS3.2, lint-checks版本26.2.0定义了一些lint, demo请参考https://github.com/ukyo6/cuslint

下面介绍自定义lint的基本配置
## Lint parser  
> 自从Lint从第一个版本就选择了lombok-ast作为自己的AST Parser，并且用了很久。但是Java语言本身在不断更新，Android也在不断迭代出新，lombok-ast慢慢跟不上发展，所以Lint在25.2.0版增加了IntelliJ的PSI（Program Structure Interface）作为新的AST Parser。但是PSI于IntelliJ、于Lint也只是个过渡性方案，事实上IntelliJ早已开始了新一代AST Parser，UAST（Unified AST）的开发，而Lint也将于即将发布的25.4.0版中将PSI更新为UAST。  

更多的介绍请参考这篇博客[LintParser具体介绍](https://blog.csdn.net/u012448058/article/details/68935865), 简单的说从lambok->PSI->UAST,官方一直在优化lint的执行效率和扩展性,本文使用的是Lint26.2.0版本


## 配置
### 1. 创建java-library
在项目中新建java library, 添加最新稳定版lint依赖26.2.0
```
dependencies {
    compileOnly 'com.android.tools.lint:lint-api:26.2.0'
    compileOnly 'com.android.tools.lint:lint-checks:26.2.0'
}
```
### 2. 实现自定义的detector
detector是lint的核心, 每个规则都先继承抽象类 Detector，然后实现Detector.Scanner接口,详细的步骤会在接下来说明   

### 3. 创建自定义的IssueRegistry, 在app中使用该Registry
**继承IssueRegistry,添加自定义的detector**

    ```
    public class IssuesRegister extends IssueRegistry {
        @NotNull
        @Override
        public List<Issue> getIssues() {
            return new ArrayList<Issue>() {{
                //添加自定义的Detector
                add(SelfLogDetector.ISSUE);  
                add(NewThreadDetector.ISSUE);
                add(MessageObtainDetector.ISSUE);
                add(ViewIdCorrectnessDetector.ISSUE);
                add(LayoutNameDetector.ACTIVITY_LAYOUT_NAME_ISSUE);
                add(LayoutNameDetector.FRAGMENT_LAYOUT_NAME_ISSUE);
            }};
        }
    }
    ```
**在java moudle的build.gradle中声明自定义的IssueRegistry**
 
    ```
    jar {
        manifest {
            attributes("Lint-Registry-v2": "com.lintrules.IssuesRegister")
        }
    }
    ```

**app中引入自定义的lint**

- Gradle Plugin3.0之前,用的是包装aar的方式引入自定义的lint, 参照 [linkedIn方案](https://engineering.linkedin.com/android/writing-custom-lint-checks-gradle)
- 3.0后增加 **lintChecks**, 无需再使用包装aar的方式
 
  >New lintChecks dependency configuration allows you to build a JAR that defines custom lint rules, and package it into your AAR and APK projects. Your custom lint rules must belong to a separate project that outputs a single JAR and includes only compileOnly dependencies. Other app and library modules can then depend on your lint project using the lintChecks configuration:

    我们可以直接在项目的build.gradle里添加
    ```
    dependencies {
        lintChecks project(':lintrules') //这里添加java library
    }
    ```
    
## 实现Detector
detector是lint的核心,主要分为下面几步
1. 每个规则都先继承抽象类Detector，然后实现Detector.Scanner接口
2. 定义ISSUE的内容,严重级别,提示信息,提示位置
3. 实现getApplicableXX和visitXX方法; 在getApplicableXX中定义检查的域,visitXX中定义检查的规则
4. 调用JavaContext.reportIssue()提示异常
 
### 1.实现Scanner接口
Scanner包括以下几种
- SourceCodeScanner 扫描 Java 或符合JVM规范的源码文件（如 kotlin）
- ClassScanner 扫描编译后的 class 文件
- BinaryResourceScanner 扫描二进制资源文件（如.png）
- ResourceFloderScanner 扫描资源目录
- XmlScanner 扫描 xml 文件
- GradleScanner 扫描 Gradle 文件
- OtherFileScanner 扫描其他文件  

每个Scanner都实现了很多getApplicableXX和visitXX方法,这些方法都是成对使用的
### 2.定义ISSUE
ISSUE在每个Detector中定义，lint检查到相关项将ISSUE报告出来，示例：
```
public static final Issue ISSUE = Issue.create(
    "InnerClassSerializable", //问题 Id
    "内部类需要实现Serializable接口", //问题的简单描述
    "内部类需要实现Serializable接口",//问题的详细描述
    Category.SECURITY, //问题类型  
    5, // 0-10严重级别
    Severity.ERROR, //问题严重程度
    new Implementation(SerializableDetector.class,
    Scope.JAVA_FILE_SCOPE); //Detector 和 Scope 的对应关系
```
### 3.实现getApplicableXX和visitXX方法方法
- 例如实现SourceCodeScanner接口,我们applicableSuperClasses()中指定需要检查的父类名列表,visitClass()当检测到你指定的父类名列表时,就会进入该方法, 根据参数JavaContext, Uclass可以很方便的获取Class的继承关系,名字等信息
```
    @Nullable
    @Override
    public List<String> applicableSuperClasses() {
        //指定检查"java.io.Serializable"
        return Collections.singletonList(CLASS_SERIALIZABLE);
    }

    /**
     * 扫描到applicableSuperClasses()指定的list时,回调该方法
     */
    @Override
    public void visitClass(JavaContext context, UClass declaration) {
        //只从最外部开始向内部类递归检查
        if (declaration instanceof UAnonymousClass) {
            return;
        }
        sortClass(context, declaration);
    }
```

- 再比如XmlScanner, 可以利用getApplicableElements()和visitElement()方法来进行xml中节点的扫描
```
@Override
    public Collection<String> getApplicableElements() {
        return Arrays.asList( //指定检查这几个控件的命名规范,可自行扩展
                TEXT_VIEW,
                IMAGE_VIEW,
                BUTTON,
                EDIT_TEXT,
                CHECK_BOX
        );
    }
    
    /**
     * 扫描到getApplicableElements()指定的xml节点时,回调该方法
     */
    @Override
    public void visitElement(XmlContext context, Element element) {
        //这个detector只扫描android:id属性,其他属性跳过
        if (!element.hasAttributeNS(ANDROID_URI, ATTR_ID)) return;

        Attr attr = element.getAttributeNodeNS(ANDROID_URI, ATTR_ID);
        String value = attr.getValue();
        if (value.startsWith(NEW_ID_PREFIX)) {
        ....
        }
    }
```
### 4.报告ISSUE

在需要报告ISSUE的地方调用context.report()
```
context.report(ISSUE, //定义好的ISSUE
            uClass.getNameIdentifier(), //ISSUE对应的AST节点
            context.getLocation(uClass.getNameIdentifier()), //ISSUE提示的位置
            String.format("内部类 `%1$s` 需要实现Serializable接口", uClass.getName())); //ISSUE的描述
```
## 使用
除了在代码中的静态waring,error提示,还可以输出报告来查看
输入 **gradlew 工程名:lintDebug** 可以在build/reports目录下查看lint报告,demo中检测到的问题

![image](http://upload-images.jianshu.io/upload_images/4960358-71fc216384f86a2b?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
也可以查看详细的信息

![image](http://upload-images.jianshu.io/upload_images/4960358-78453d627908fda5?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## lint调试
开发完自定义lint规则后,可能需要对代码进行验证,调试方式如下
- 在自定义的lint代码中打好断点
- 选择 Run -> Eidt Configurations, 在Android Studio的运行参数(Run Configurations)中添加一个Remote类型，都取默认值即可(端口号5005)
![image](http://upload-images.jianshu.io/upload_images/4960358-e282214648639075?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 在Teminal窗口中输入gradlew lintDebug -Dorg.gradle.daemon=false -Dorg.gradle.debug=true
![image](http://upload-images.jianshu.io/upload_images/4960358-7401fc32db31664d?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 选择刚才配置的remote参数,点击debug的console看到以下输出: Connected to the target VM, address: 'localhost:5005', transport: 'socket' 就可以进行调试了
### Lint构建优化
随着官方lint-checks的更新,detector的数量越来越多,构建一次linttask的速度肯定是很慢的(即使每个detector指定了scope,都需要扫描项目内所有该scope的文件...)
lint的开发者也在google论坛上给出了lint构建的建议 [Lint Performance Tips](https://groups.google.com/forum/#!topic/lint-dev/RGTvK_uHQGQ),大概是以下几点:  
1. 使用graldew lintDebug, gradlew lintRelease代替gradlew lint, 因为后者会执行lint lintDebug lintRelease,造成成倍的开销
2. 使用gradlew 模块名:lintDebug可以检查指定的模块
3. lintOptions配置checkTestSources和ignoreTestSources
4. 不要使用checkAllWarnings,因为有一些lint检查，特别是id为“WrongThreadInterprocedural”的,非常慢

### 参考资料
- [官方教程](https://developer.android.google.cn/studio/write/lint) 
- [从lombok到UAST – 浅谈Android Lint的AST Parser](https://blog.csdn.net/u012448058/article/details/68935865) 
- [Android ------ 美团的Lint代码检查实践](https://my.oschina.net/zhangqie/blog/1796238)
- [GoogleSample](https://github.com/googlesamples/android-custom-lint-rules)
- [Google论坛Lint-dev](https://groups.google.com/forum/#!forum/lint-dev)



