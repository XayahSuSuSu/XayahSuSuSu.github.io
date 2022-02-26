---
title: 成为Wheel Maker：发布Android库到MavenCentral
date: 2022-02-25 23:45:40
tags: [ 'Android', 'MavenCentral' ]
---

# 一、前言
**Android发展**日新月异，由此诞生了许多强大的**第三方库**，例如**Glide**、**OkHttp**。我们当然也可以**发布自己的库**，当一回**Wheel Maker**！以前大家用**Jcenter**作为平台，而现在大家用**MavenCentral**。

# 二、步骤
## 1. 注册Sonatype账号
**MavenCentral**由**Sonatype**运营，因此我们先 **[注册](https://issues.sonatype.org)** 一个 **Sonatype账号**。

## 2. 创建问题
首先填写**项目**和**问题类型**。
- 项目：Community Support - Open Source Project Repository Hosting (OSSRH)
- 问题类型：New Project
{% asset_img 创建问题.png 创建问题 %}
然后根据具体情况填写**库信息**。
{% asset_img 创建问题2.png 创建问题2 %}
收到了来自**工作人员**的**回复**
{% asset_img 回复.png 回复 %}
回复中提到我们需要验证**GroupId**对应的**域名**，我这里为了方便就选择**第二种方式**，将**GroupId**改为**io.github.xayahsususu**。
回到**GitHub**，新建一个名为**OSSRH-78521**（根据回复中的仓库名）的**仓库**。
{% asset_img 新建仓库.png 新建仓库 %}
**编辑**这个**问题**，把**GroupId**改为**io.github.xayahsususu**并**更新**，然后将**状态**重新**更新为开放**，**等待回复**。
几分钟之后，正常情况下会受到完成的回复，注意回复中提到的`s01.oss.sonatype.org`，这是我们**管理上传库**的**地址**，可能每个**时期**得到的地址**不同**。
{% asset_img 完成回复.png 完成回复 %}
**问题状态**变为**已解决**
{% asset_img 完成状态.png 完成状态 %}

## 3. 申请GPG密钥
发布到**MavenCentral**的库需要**签名**，因此我们 **[下载](https://www.gpg4win.org/get-gpg4win.html)** 相应工具来**生成密钥**，这里以**Windows**为例
安装**Gpg4win**，运行**Kleopatra**。
{% asset_img Kleopatra.png Kleopatra %}
左上角**文件 - 新建密钥对 - 创建个人 OpenPGP 密钥对**
{% asset_img 个人密钥.png 个人密钥 %}
{% asset_img 个人密钥2.png 个人密钥2 %}
完成后在**主界面**打开生成的**密钥**，记住**指纹后8位**（后面要用到），然后**生成吊销证书**并**保存**
{% asset_img 个人密钥3.png 个人密钥3 %}
接下来回到**主界面**右键**密钥 - 在服务器上发布**，若网络没有问题即可**发布成功**。
{% asset_img 密钥发布成功.png 密钥发布成功 %}
**发布成功**后，再次右键**密钥 - 备份私钥**，将私钥导出**（将后缀改为.gpg）**。
接下来打开密钥，**修改密码**。
{% asset_img 修改密码.png 修改密码 %}

## 4. Android库发布脚本配置
在项目**根目录**下新建`publish-mavencentral.gradle`
输入以下代码（如果上文中回复得到的地址不为`s01.oss.sonatype.org`，则将其**改为得到的地址**）
```
apply plugin: 'maven-publish'
apply plugin: 'signing'

task androidSourcesJar(type: Jar) {
    classifier = 'sources'
    from android.sourceSets.main.java.source
}

ext["signing.keyId"] = ''
ext["signing.password"] = ''
ext["signing.secretKeyRingFile"] = ''
ext["ossrhUsername"] = ''
ext["ossrhPassword"] = ''

File secretPropsFile = project.rootProject.file('local.properties')
if (secretPropsFile.exists()) {
    println "Found secret props file, loading props"
    Properties p = new Properties()
    p.load(new FileInputStream(secretPropsFile))
    p.each { name, value ->
        ext[name] = value
    }
} else {
    println "No props file, loading env vars"
}
publishing {
    publications {
        release(MavenPublication) {
            // The coordinates of the library, being set from variables that
            // we'll set up in a moment
            groupId PUBLISH_GROUP_ID
            artifactId PUBLISH_ARTIFACT_ID
            version PUBLISH_VERSION

            // Two artifacts, the `aar` and the sources
            artifact("$buildDir/outputs/aar/${project.getName()}-release.aar")
            artifact androidSourcesJar

            // Self-explanatory metadata for the most part
            pom {
                name = PUBLISH_ARTIFACT_ID
                description = PUBLISH_DESCRIPTION
                // If your project has a dedicated site, use its URL here
                url = PUBLISH_GITHUB_URL
                licenses {
                    license {
                        name = PUBLISH_LICENSE_NAME
                        url = PUBLISH_LICENSE_URL
                    }
                }
                developers {
                    developer {
                        id = PUBLISH_DEVELOPER_ID
                        name = PUBLISH_DEVELOPER_NAME
                        email = PUBLISH_DEVELOPER_EMAIL
                    }
                }
                // Version control info, if you're using GitHub, follow the format as seen here
                scm {
                    connection = PUBLISH_CONNECTION
                    developerConnection = PUBLISH_CONNECTION_DEVELOPER
                    url = PUBLISH_CONNECTION_URL
                }
                // A slightly hacky fix so that your POM will include any transitive dependencies
                // that your library builds upon
                withXml {
                    def dependenciesNode = asNode().appendNode('dependencies')

                    project.configurations.implementation.allDependencies.each {
                        def dependencyNode = dependenciesNode.appendNode('dependency')
                        dependencyNode.appendNode('groupId', it.group)
                        dependencyNode.appendNode('artifactId', it.name)
                        dependencyNode.appendNode('version', it.version)
                    }
                }
            }
        }
    }
    repositories {
        // The repository to publish to, Sonatype/MavenCentral
        maven {
            // This is an arbitrary name, you may also use "mavencentral" or
            // any other name that's descriptive for you
            name = PUBLISH_MAVEN_NAME

            def releasesRepoUrl = "https://s01.oss.sonatype.org/service/local/staging/deploy/maven2/"
            def snapshotsRepoUrl = "https://s01.oss.sonatype.org/content/repositories/snapshots/"
            // You only need this if you want to publish snapshots, otherwise just set the URL
            // to the release repo directly
            url = version.endsWith('SNAPSHOT') ? snapshotsRepoUrl : releasesRepoUrl

            // The username and password we've fetched earlier
            credentials {
                username ossrhUsername
                password ossrhPassword
            }
        }
    }
}
signing {
    sign publishing.publications
}
```

打开项目**根目录**中的`local.properties`，添加以下代码

```
signing.keyId=$密钥指纹后八位（无空格）
signing.password=$密钥密码
signing.secretKeyRingFile=$导出密钥的文件路径，例如：E\:\\ProgramDesign\\AndroidProjects\\Signature\\Xayah_0xEB95FB34_SECRET.gpg
ossrhUsername=$Sonatype帐号
ossrhPassword=$Sonatype密码
```
打开**待发布库目录**下的`build.gradle`，行位添加以下代码（根据示例参考修改填写）
```
ext {
    PUBLISH_GROUP_ID = "io.github.xayahsususu"
    PUBLISH_ARTIFACT_ID = 'materialyoufileexplorer'
    PUBLISH_DESCRIPTION = 'A file explorer with the style of Material You.'
    PUBLISH_GITHUB_URL = 'https://github.com/XayahSuSuSu/Android-MaterialYouFileExplorer'
    PUBLISH_VERSION = '1.0.0'
    PUBLISH_LICENSE_NAME = 'GPL-3.0'
    PUBLISH_LICENSE_URL = 'https://choosealicense.com/licenses/gpl-3.0/'
    PUBLISH_DE_URL = 'https://choosealicense.com/licenses/gpl-3.0/'
    PUBLISH_DEVELOPER_ID = 'Xayah'
    PUBLISH_DEVELOPER_NAME = 'Xayah'
    PUBLISH_DEVELOPER_EMAIL = 'zds1249475336@gmail.com'
    PUBLISH_CONNECTION = 'scm:git:github.com/XayahSuSuSu/Android-MaterialYouFileExplorer.git'
    PUBLISH_CONNECTION_DEVELOPER = 'scm:git:ssh://github.com/XayahSuSuSu/Android-MaterialYouFileExplorer.git'
    PUBLISH_CONNECTION_URL = 'https://github.com/XayahSuSuSu/Android-MaterialYouFileExplorer/tree/main'
    PUBLISH_MAVEN_NAME = "MaterialYouFileExplorer"

}
apply from: "${rootProject.projectDir}/publish-mavencentral.gradle"
```

## 5. 发布Android库到MavenCentral
打开**Settings - Experimental - 取消勾选 Do not build Gradle task list during Gradle sync**
{% asset_img Experimental.png Experimental %}
点击左上角**File - Sync Project with Gradle Files**
{% asset_img Sync.png Sync %}
完成后打开**右侧Gradle窗口**，双击运行**build - assemble**
{% asset_img assemble.png assemble %}
**编译完成**后，双击运行**publishing - publishReleasePublicationTo${LibraryName}Repository** 发布到[https://s01.oss.sonatype.org](https://s01.oss.sonatype.org) （具体地址由上文回复中可得）
{% asset_img Publish.png Publish %}
打开**管理地址**后，进入**Staging Repositories**，即可看到我们上传的库**记录**。
{% asset_img NRM.png NRM %}
打开**库记录**，点击**Close**，**输入描述**后**Confirm**，稍等片刻即可刷新查看，若**Close**成功，则点击**Release**发布。**发布成功**后，之前的问题会收到**发布成功**的**回复**。
{% asset_img 发布成功回复.png 发布成功回复 %}
假以时日，我们即可在[https://repo1.maven.org/maven2/](https://repo1.maven.org/maven2/) 和 [https://search.maven.org/](https://search.maven.org/) 中查看到我们发布的**库信息**。
{% asset_img Maven.png Maven %}

## 6. 使用已发布的Android库
打开**需要引用库**的项目的**app**目录下的`build.gradle`，输入以下代码**引用**。
格式为：
```
implementation '$PUBLISH_GROUP_ID:$PUBLISH_ARTIFACT_ID:$PUBLISH_VERSION'
```
例如：
```
implementation 'io.github.xayahsususu:materialyoufileexplorer:1.0.0'
```