# PublishLibrary
对各种发布库的使用和总结

## 发布到Maven仓库
### 1.`maven-publish` 插件

 `maven-publish` 插件提供了发布Maven格式的功能。
 
 `publishing` 插件创建一个类型为`PublishingExtension`名称为`publishing`的参数，这个参数里面提供了`publications`容器和`repositories`容器。
 `maven-publish` 依赖的是`MavenPublication`和`MavenArtifactRepository`。(意思是”maven-publish”是在”publishing”上再一次的封装)
 
 应用”maven-publish”插件的例子。
 ```
 // build.gradle
apply plugin: 'maven-publish'
 ```
 
 应用”Maven Plugin”需要做如下事情：
 - 使用”publishing”插件 (就是需要apply plugin: 'maven-publish')
 - 创建一个规则自动为每一个MavenPublication创建GenerateMavenPom任务
 - 创建一个规则自动的为每一个MavenPublication和MavenArtifactRepository的组合创建一个PublishToMavenRepository任务
 - 创建一个规则自动的为每一个MavenPublication创建PublishToMavenLocal任务
 
 > MavenPublication相当于是需要发布的内容，
 而MavenArtifactRepository则相当于是发布的仓库。
 
 `maven-pluign`主要是做三件事：
- GenerateMavenPom可以用来生成pom文件；
- PublishToMavenRepository可以用来发布到指定仓库，本地某个路径或者远程的服务器；
- PublishToMavenLocal则是用来发布到本地的.m2仓库(例如我自己电脑的地址是/Users/用户名/.m2)
### 2.Publications
Publication对象描述的是一次发布的结构和配置。Publications通过任务发布到仓库，Publication对象的配置则精确地确定哪些内容需要发布。所有的发布配置信息定义在PublishingExtension.getPublications()容器中，在一个项目中每一个发布到都有一个唯一的名字。

对于”maven-publish”插件有哪些影响，需要创建一个MavenPublication，这个发布决定了哪些内容需要发布以及包含在POM文件中的详细信息。发布时可以通过增加模块、自定义构建的内容以及修改生成的POM文件来配置。

#### 1.发布一个软件模块

最简单的发布一个Gradle项目到Maven仓库的方法就是指定一个软件模版去发布。目前已经支持的模版有。

| Name	| Provided By |	Artifacts	| Dependencies |
| ------ | ---------- | ----------| ------------|
| java	| The Java Plugin	| Generated jar file	| Dependencies from ‘runtime’ configuration |
| web	| The War Plugin	| Generated war file	| No dependencies |

下面的例子，构建内容和运行的依赖都来自在Java插件中定义的java信息。
```gradle
// build.gradle
publishing {
    publications {
        mavenJava(MavenPublication) {
            from components.java
        }
    }
}
```
> 什么意思呢？上面一句from components.java代表了使用默认的配置，从上表可以看出会生成jar文件，同时需要运行时的一些依赖。

#### 2. 发布一个自定义构建内容

可以通过配置artifact明确的指定需要生成的内容。通常会提供原始的文件或者AbstractArchiveTask对应的实例(例如：Jar，Zip等等)。

对于每一个自定义的构建内容，可以指定扩展名和分类信息(classifier)。注意只有一个发布的构建内容可以拥有空的分类信息(classifier)，并且所有的构建信息必须拥有唯一的分类信息(classifier)和扩展名的组合。

如下配置自定义的构建内容：
```gradle
// build.gradle
task sourceJar(type: Jar) {
    from sourceSets.main.allJava
}

publishing {
    publications {
        mavenJava(MavenPublication) {
            from components.java

            artifact sourceJar {
                classifier "sources"
            }
        }
    }
}
```

> 什么意思呢？ 插件自身可以支持发布出去生成的文件是jar和war，但是这并不能满足所有的情况，所以可以自定义构建内容，比如：aar等等。怎么自定义呢？创建相应的任务以及在publications中使用artifact就可以了。

#### 3. 在生成的POM中区别值

生成的POM文件的值包含了如下的属性：

- groupId Project.getGroup()
- artifactId Project.getName()
- version Project.getVersion()

覆盖默认的标识属性非常的简单：只需要在MavenPublication配置时指定groupId，artifactId，version就行。
```gradle
// build.gradle
publishing {
    publications {
        maven(MavenPublication) {
            groupId 'org.gradle.sample'
            artifactId 'project1-sample'
            version '1.1'

            from components.java
        }
    }
}
```
Maven限制了groupId和artifactId在一个有限的字符集([A-Za-z0-9_\\-.]+)，并且Gradle也遵循这样的限制。对于version(以及构建内容中的extension和classifier)，只需要是有效的Unicode字符就行。唯一明确禁止的Unicode字符是”\”,”/”和ISO控制字符。会在publication之前对字符进行校验。

#### 4. 修改生成的POM文件

生成的POM文件可能在发布之前需要修改。”maven-publish”插件提供了一个hook来允许这样的修改。
```gradle
// build.gradle
publications {
    mavenCustom(MavenPublication) {
        pom.withXml {
            asNode().appendNode('description',
                                'A demonstration of maven POM customization')
        }
    }
}
```
上面例子在生成的POM文件中增加了一个description元素。使用hook可以修改POM中任意的元素。例如：你可以将依赖关系的版本范围替换成生成构建的实际版本。

可以修改POM中的任意元素，这也意味着可以将POM修改成非法的POM，所以需要小心的使用这个功能。

发布模块的唯一标识属性(groupId,artifactId,version)是一个例外，这个元素不能通过withXMLhook来修改。

#### 5. 发布多个模块

有时候使用Gradle创建多个子项目发布多个模块是非常有用的。

下面的例子是在一个项目中使用Gradle发布多个模块。
```gradle
task apiJar(type: Jar) {
    baseName "publishing-api"
    from sourceSets.main.output
    exclude '**/impl/**'
}

publishing {
    publications {
        impl(MavenPublication) {
            groupId 'org.gradle.sample.impl'
            artifactId 'project2-impl'
            version '2.3'

            from components.java
        }
        api(MavenPublication) {
            groupId 'org.gradle.sample'
            artifactId 'project2-api'
            version '2'

            artifact apiJar
        }
    }
}
```

如果一个工程定义了多个发布信息，Gradle会将每一个发布信息发布到指定的仓库。每一个发布需要给定一个唯一的标示。

### 3. Repositories

发布信息(Publications)最终会发布到仓库，发布的仓库信息定义在PublishingExtension.getRepositories()中。

下面是定义一个发布的仓库
```gradle
// build.gradle
publishing {
    repositories {
        maven {
            // change to point to your repo, e.g. http://my.org/repo
            url "$buildDir/repo"
        }
    }
}
```
> The DSL used to declare repositories for publication is the same DSL that is used to declare repositories to consume dependencies from, RepositoryHandler. However, in the context of Maven publication only MavenArtifactRepository repositories can be used for publication.

上面一句是对于Repositories的描述，自己的理解是使用插件的仓库必须和发布的仓库一直 

### 4. 发布到maven仓库的一个例子

“maven-publish”插件会自动的为每一个各自声明在publishing.publications的MavenPublication和声明在publishing.repositories的MavenArtifactRepository的组合创建一个PublishToMavenRepository任务。

创建的任务的名字格式是publish«PUBNAME»PublicationTo«REPONAME»Repository。
```gradle
// build.gradle
apply plugin: 'java'
apply plugin: 'maven-publish'

group = 'org.gradle.sample'
version = '1.0'

publishing {
    publications {
        mavenJava(MavenPublication) {
            from components.java
        }
    }
}
publishing {
    repositories {
        maven {
            // change to point to your repo, e.g. http://my.org/repo
            url "$buildDir/repo"
        }
    }
}
```

```gradle
// Output of gradle publish
> gradle publish
:generatePomFileForMavenJavaPublication
:compileJava
:processResources NO-SOURCE
:classes
:jar
:publishMavenJavaPublicationToMavenRepository
:publish

BUILD SUCCESSFUL

Total time: 1 secs
```
在这个例子中，一个类型是PublishToMavenRepository，名称是publishMavenJavaPublicationToMavenRepository的任务会被创建，这个任务会被连接到发布任务的生命周期中。执行gradle publish会生成POM文件以及生成要发布的所有的构建内容，并将生成的构建内容传输到相应的仓库。

### 5. 发布到本地maven仓库

为了于本地maven集成，有时候将模块发布在本地的.m2仓库是非常有用的。Maven称之为“安装”模块。”maven-publish”插件会自动的为每一个声明在publishing.publications的内容创建一个PublishToMavenLocal任务。每一个这样的任务都会被连接到publishToMavenLocal的生命周期中。不需要在publishing.repositories中写mavenLocal。

创建的任务的名字格式是publish«PUBNAME»PublicationToMavenLocal。

> 网上很多的开源插件有一些没有发布到本地的方法，对于一些需要修改代码进行调试等，发布到本地还是非常方便的。

### 6. 不发布的情况下生成pom文件

有时候在不发布的情况下修改POM文件是非常有用的，因为POM文件的生成是一个单独的任务，所以非常容易实现。

生成POM文件的任务类型是GenerateMavenPom，名称是基于发布任务generatePomFileFor«PUBNAME»Publication。因此在下面的例子中，发布的名称是mavenCustom，所以生成POM文件的名称是generatePomFileForMavenCustomPublication。
```gradle
// 不发布的情况下生成POM文件
model {
    tasks.generatePomFileForMavenCustomPublication {
        destination = file("$buildDir/generated-pom.xml")
    }
}
```

```gradle
> gradle generatePomFileForMavenCustomPublication
:generatePomFileForMavenCustomPublication

BUILD SUCCESSFUL

Total time: 1 secs

## 打包android library到本地的例子
```gradle
// android-artifacts.gradle
apply plugin: "maven-publish"

task androidJavadocs(type: Javadoc) {
    source = android.sourceSets.main.java.srcDirs
    ext.androidJar = "${android.sdkDirectory}/platforms/${android.compileSdkVersion}/android.jar"
    classpath += files(ext.androidJar)
}

task androidJavadocsJar(type: Jar, dependsOn: androidJavadocs) {
    classifier = 'javadoc'
    from androidJavadocs.destinationDir
}

afterEvaluate { project ->
    tasks.all { Task task ->
        if (task.name.equalsIgnoreCase('publishPatchLibPublicationToMavenLocal')) {
            task.dependsOn tasks.getByName('assemble')
        }
    }
}

publishing {
    publications {
        PatchLib(MavenPublication) {

            artifactId project.getName()
            groupId group
            version version

            // artifact 打成aar
            artifact("$buildDir/outputs/aar/${project.getName()}-release.aar")
            artifact androidJavadocsJar
        }
    }
}
// publish«PUBNAME»PublicationToMavenLocal
task pushlishLibToLocal(dependsOn: ['build', 'publish PatchLibPublicationToMavenLocal']) {
    group = 'patch'
}
```
使用的时候只需要在需要打包到本地的lib工程的build.gradle目录中。
```gradle
apply plugin: 'com.android.library'

...
version "1.0.0" // 需要一个version传到android-artifacts.gradle，每一个需要打包的可能不一样
group APP_PACKAGE_NAME  // 同上

apply from: file('../gradle/android-artifacts.gradle')
```
## 打包Java library到本地的例子

Java library 就会更简单一点了，因为有一个模版，上面文章已经有介绍了。
```gradle
// java-artifacts.gradle

apply plugin: 'maven-publish'

sourceCompatibility = JavaVersion.VERSION_1_7
targetCompatibility = JavaVersion.VERSION_1_7

afterEvaluate { project ->
    tasks.all { Task task ->
        if (task.name.equalsIgnoreCase('publishPatchLibPublicationToMavenLocal')) {
            task.dependsOn tasks.getByName('assemble')
        }
    }
}

publishing {
    publications {
        PatchLib(MavenPublication) {
            from components.java
            groupId = group
            artifactId = project.getName()
            version = VERSION_NAME_DEV
        }
    }
}

// 发布在本地仓库
// 会自动创建'publishXXXXXPublicationToMavenLocal'  'XXXXX' 是 publishing->publications-> PatchLib
task pushlishLibToLocal(dependsOn: ['build', 'publishPatchLibPublicationToMavenLocal']) {
    group = 'patch'
}
```

## 总结

将项目模块(Android Library、Java Library以及Groovy Plugins等等)构建、打包发到Maven仓库(包括本地的Maven仓库)是一件很有意义的事情。对于公司来说，方便多个小组之间的调用、以及维护管理，同时也使代码看起来不会显的过于臃肿，也避免了一些网络不好的情况(公司内网还是相对快一点)。这篇文件介绍的’maven-publish’插件是一个已经可以直接使用的插件，个人觉得目前这个插件的主要作用是将构建内容发布在本地(.m2目录)。这样对于修改一些开源的框架还是蛮有作用的，以及将项目中的Library模块先发布在本地然后再依赖调用，也可以减少打包的时间。

对于需要打包发布在远程仓库的，最好还是使用[“maven”](https://docs.gradle.org/current/userguide/maven_plugin.html)插件和[“signing”](https://docs.gradle.org/current/userguide/signing_plugin.html)插件，当然还有可能发布在ivy仓库等，需要根据实际的需求来选择。所有的发布都可以在官方文档中找到。想要系统的学习Gradle插件还是得花点时间看看[userguide](https://docs.gradle.org/current/userguide/userguide.html)。
