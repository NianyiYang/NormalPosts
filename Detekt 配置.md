# Detekt - kotlin 静态代码检测工具配置及使用

> 说明：网络上的资料要么过时，要么不全；因此自己通过实践，重新整理了一份使用攻略
>
> 作者：一哥的马甲
>
> 创建时间： 2020年09月06日
>
> 开发环境：Mac + Android Studio 

## 简介

Detekt 是 kotlin 的静态代码检测工具，有丰富的配置供开发者选择。本文只介绍基本配置和使用，剩下的内容请开发者自行探索

项目地址[点这里](https://github.com/detekt/detekt)

---

## 配置

打开项目的根配置`build.gradle`，配置插件

> 注意：**Gradle 版本要大于 5.4**

```groovy
buildscript {
    repositories {
			jcenter()
    }
}
// 1. 添加 detekt 插件
plugins {
    id("io.gitlab.arturbosch.detekt").version("1.13.1")
}

// 2. 添加 detekt 配置文件
detekt {
    toolVersion = "1.13.1"
    config = files("config/detekt/detekt.yml")
    // failFast = true // fail build on any finding
    buildUponDefaultConfig = true // preconfigure defaults

    reports {
        reports {
            xml {
                enabled = true
                destination = file("path/to/destination.xml")
            }
            html {
                enabled = true
                destination = file("path/to/destination.html")
            }
            txt {
                enabled = true
                destination = file("path/to/destination.txt")
            }
        }
    }
}
```

同步项目，运行 `./gradlew tasks`，可以看到`detekt`命令则表示配置成功

运行命令 `./gradlew detektGenerateConfig` 生成默认的检测配置文件，生成位置在项目根目录 `config` 文件下。可以手动修改生成的配置文件定制检测规则

在需要使用检测的 module 中的`build.gradle`配置中加上

```groovy
// 配置 detekt 插件
apply plugin: 'io.gitlab.arturbosch.detekt'
```

也可以在根目录的`build.gradle`中这样配置使检测全局生效

```groovy
allprojects {
    repositories {
        ... ...
    }
  	
  	// 这样配置即可全局生效
  	apply plugin: 'io.gitlab.arturbosch.detekt'
}
```

---

## 使用

直接在命令行运行 `./gradlew detekt` 即可，执行结束后会将检测结果显示出来；

![image-20200915152415601](/Users/nianyi.yang/Library/Application Support/typora-user-images/image-20200915152415601.png)

此时也会在项目的`/build/report`下生成可视化的文档（xml/html/txt）也可以点开查看

![image-20200915152656519](/Users/nianyi.yang/Library/Application Support/typora-user-images/image-20200915152656519.png)

---

## 添加 Git Hook

可以在每次提交之前通过 hook 自动运行检测脚本；在项目的`.git/hooks`文件夹下`touch pre-commit`，然后填入脚本

> 项目的`.git/hooks`文件夹下有一些 samples ，新建脚本必须与这些 samples 文件名一致（可以省略后缀）

```shell
echo "✅  开始执行 kotlin 代码检查"

./gradlew detekt

result=$?

if [ "$result" = 0 ] ; then    
   echo "✅  代码检查完毕，没有问题"     
   exit 0
else
   echo "❎  代码检查完毕，发现问题，请修复后重新提交"
   echo "❎  请到项目的 /build/report 目录下查看可视化文档"  
   exit 1
fi
```

保存，**并将文件设置为可执行文件** `chmod +x pre-commit`。这样每次提交之前都会运行检测脚本了

>  如果不想运行 hook 可以使用 `git commit --no-verify`

---

## 共享 Git Hook

使用 Git （version >= 2.9）映射功能

在项目根目录下新建文件夹`hooks`，然后将`pre-commit`文件放到文件夹中

执行命令 `git config core.hooksPath hooks` 即可

---

