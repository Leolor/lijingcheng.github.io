---
layout: post
title: "用CocoaPods为iOS项目做依赖管理"
date: 2015-02-11 15:10:58 +0800
comments: true
categories: 
---


{% img left http://7x00ed.com1.z0.glb.clouddn.com/cocoapods.png 190 190 "" "" %}
>**[CocoaPods](http://www.cocoapods.org) - *Get on with building your app, not duplicating code.***   
可以为Cocoa和Swift项目使用的第三方类库和我们自己的私有类库提供依赖管理，就像Java领域的Maven。
我们只需要告诉它要使用的类库名称和版本，然后再执行一条命令，它就会自动将这个类库的源码从github上下载到本地，并且为工程设置好相应的系统依赖和编译选项。使用CocoaPods可以大大节省我们在设置和更新类库时所花的时间。

<!-- more -->
<br/>
#准备工作
由于CocoaPods是用ruby写的，并且主要用于管理github上的第三方类库，所以在安装前需要有Ruby环境和Git环境。Git环境可以通过下载Xcode中的Command Line Tools建立。Ruby环境可以使用Mac系统默认提供的版本，也可以通过以下命令将它更新至最新版。
```
sudo gem update –system
``` 
<br/>
为了提高下载速度，还需要将ruby的源更改为淘宝的  
```
gem sources -a http://ruby.taobao.org/  (添加)
gem sources -r http://rubygems.org/  (删除)
gem sources -l  (检查一下)
``` 

<br/>
#安装并使用CocoaPods
- 安装CocoaPods
```
sudo gem install cocoapods (更新命令也是这个)
```
- 下载CocoaPods维护的所有podspec文件到"~/.cocoapods/repos/master/Specs"
```
pod setup
```
>更新spec也用这个命令，podspec文件主要用来描述依赖库的名称、版本、作者、下载地址等信息。通过CocoaPods下载第三方类库，其实就是根据我们指定的类库名称找到相关的podspec，然后再根据podspec文件中指定的地址去下载。 
 
- 查看CocoaPods管理的依赖库信息 
```
pod search 依赖库的名字
```
- 新建Podfile文件，此文件用于配置项目所需要使用的依赖库
```
cd "项目根目录"
pod init
```
- 打开Podfile文件，按下面内容配置依赖关系，Podfile的更详细配置方法可参照[官方文档](http://guides.cocoapods.org/syntax/podfile.html)
```
source 'https://github.com/CocoaPods/Specs.git'

platform :ios, '7.0'
inhibit_all_warnings!

pod 'AFNetworking', '2.5.1' 
```
>上面内容的意思是，specs文件由CocoaPods提供，项目需要使用支持ios7.0及以上的依赖库，并且为主target配置了2.5.1版本的AFNetworking，并忽略依赖库中的所有警告。依赖库的版本号建议明确指定，这样可以避免更新依赖库后对现有工程造成影响。  

如果你的项目有多个target，那么需要按下面的方式配置pod，否则依赖关系只能作用于主target。  
```
#Tests target 不仅使用主target的依赖库还需要添加自己的
target 'Tests' do
  pod "Tests需要的依赖库"
end
或
#Tests target不需要主target的依赖库
target 'Tests', :exclusive => true do
  pod "Tests需要的依赖库"
end
```

- 根据Podfile下载安装依赖库
```
pod install (添加或删除依赖库后也是通过此命令更新)
```
> **Cocoapods会先更新相关podspec文件，然后自动为项目工程设置好相应的系统依赖和编译参数，当依赖库安装完成后，打开项目根目录，会发现多了以下文件及文件夹**  
> 
> 1. ".xcworkspace"，以后必须通过此workspace打开项目。  
> 2. "Pods"，CocoaPods将Profile中配置的所有依赖库都下载到这里，并且将所有依赖库打包成单独的静态库供主项目使用。
> 3. "Podfile.lock"，用于保存已经安装的依赖库版本信息。如果在配置依赖库时没有明确指定版本，那么必须将此文件加入到版本控制中，否则有可能造成团队开发中不同成员使用的依赖库版本不一致。

- 查看所下载的依赖库是否有新版本
```
pod outdated
```
- 下载新版本
```
pod update
```

<br/>
#让自己的开源项目支持Cocoapods
####通过Cocoapods创建项目会让整个事情变的简单一些
```
pod lib create FMDBHelper
```
> **创建完成后项目根目录会包含以下文件及文件夹**  
> 
> 1. ".travis.yml"，通过"travis-ci"做持续集成要用到的配置文件，一般情况下使用默认配置就可以，如果需要使用持续集成服务，还需要以github帐号登录[travis-ci](https://travis-ci.org)，并打开对应项目开关。
> 2. ".gitignore"，建议将Pods目录也加入到忽略范围  
> 3. "LICENSE"，默认为MIT 
> 4. "FMDBHelper.podspec"，通过Cocoapods下载项目时要用到的项目配置文件。  
> 5. "README.md"，通过markdown语法编写此文件，用于在github上显示项目介绍。  
> 6. "Pod"，将自己的开源代码和资源文件放到这里 
> 7. "Example"，demo工程，包含测试用的target。  

####在demo工程中开发并测试
- 将源代码和资源文件分别放到"Pod/Classes"和"Pod/Assets"目录下
- 用"pod install"命令为demo工程安装依赖库，以后只要新增依赖库的代码或资源文件都需要更新
- 开发测试完成后还需要修改podspec文件

####将podspec发布到CocoaPods的Git库中
- 向cocoapods注册你的信息，需要输入邮箱(与podspec中写的一致)和名字，稍后还需要验证email
```
pod trunk register bj_lijingcheng@163.com "lijingcheng"
```
- 登录github，release一版项目并打上标签，标签要与podspec中定义的一致
- 检查spec文件是否合格
```
pod spec lint FMDBHelper.podspec
```
- 发布spec文件，以后需要升级也是用这个命令
```
pod trunk push FMDBHelper.podspec
```
- 更新本地spec
```
pod setup (稍后便可以在"~/.cocoapods/repos/master/Specs"下看到你的开源项目了)
```

<br/>
#通过Cocoapods安装私有库
对于不能放在CocoaPods公共Git仓库中的类库，可选择以下方式设置  
```
pod 'FMDBHelper', :podspec => 'https://.../FMDBHelper.podspec'
pod 'FMDBHelper', :git => 'https://.../FMDBHelper.git'
pod 'FMDBHelper', :path => '~/documents/code/Pods/'
```