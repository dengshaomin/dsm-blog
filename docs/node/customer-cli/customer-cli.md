***
##### 环境
* mac
* node：v18.11.0
* agp：7.5

*** 
##### 目标
* 搭建一个创建Android项目的Cli
* 支持自定义projectName、packageName

*** 

##### 步骤
创建cli项目名
```
mkdir my-cli-test
```

创建入口文件
```
touch index.js
```

修改package.json指定入口文件,增加type：module
```
{
  "name": "dsm-vue-cli",
  "version": "0.0.1",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC",
  "type": "module",
  "bin": {
    "dsm-vue-cli": "./index.js"
  },
  "dependencies": {
    "@code_balance/my-test-cl": "^0.0.3",
    "chalk": "^5.2.0",
    "commander": "^10.0.1",
    "ejs": "^3.1.9",
    "inquirer": "^9.2.0",
    "ora": "^6.3.0"
  }
}

```

npm install 安装依赖包
```
chalk：终端打印输出各种样式的字符（颜色、背景色、下划线等）
commander：便捷的进行命令注册及解析，可直接通过命令行参数设置项目名、包名（暂未用到，使用inquirer交互式获取）
ejs：向文件注入变量
inquirer：命令行交互工具，用于用户输入项目名，包名
ora：命令行提示图标或小动画，用于下载等待

```
[更多依赖包使用](https://juejin.cn/post/7178666619135066170#heading-30)

新建android(empty activity)项目，将以下文件中的对应packageName和packageName改成模板类型以便后续ejs进行注入

<font color=red>my-cli-test/build.gradle</font>
```
buildscript {
    ext {
        applicationId = "<%= packageName %>"
        compileSdk = 33
        minSdk = 24
        targetSdk = 33
    }
}
```
<font color=red>my-cli-test/app/src/main/res/values/strings.xml</font>
```
<resources>
    <string name="app_name"><%= projectName %></string>
</resources>
```

<font color=red>my-cli-test/app/src/main/java/com/android/sample/MainActivity.kt</font>
```
package <%= packageName %>
import androidx.appcompat.app.AppCompatActivity
import android.os.Bundle
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
    }
}

```

<font color=red>my-cli-test/settings.gradle</font>
```
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
    }
}
rootProject.name = "<%= projectName %>"
include ':app'

```
将模板项目上传到github：[AndroidSampleCli](https://github.com/dengshaomin/AndroidSampleCLI)
***
my-test-cli/index.js修改，用于获取用户输入的projectName和packageName
```
#!/usr/bin/env node

import fetch from 'node-fetch';
import chalk from 'chalk';
import {program} from 'commander';
import inquire from 'inquirer'
import {handleDownload} from './download.js'

inquire.prompt([
    {
        type:'input',
        name:'projectName',
        message:'project name:',
        default:'dsm-vue-cli-sample'
    },
    {
        type:'input',
        name:'packageName',
        message:'package name:',
        default:'com.android.sample'
    }
]).then(answers=>{
    handleDownload(answers.projectName,answers.packageName);
})

```

编写download.js，提供handleDownload方法，用于从github拉取模板项目、删除.git文件夹、修改packageName、projectName

```
import download from 'download-git-repo';
import ora from 'ora';
import shell from 'child_process'
import fs from 'fs'
import ejs from 'ejs'
const repo = "https://github.com/dengshaomin/AndroidSampleCLI";
async function handleDownload(projectName,packageName){
    const packageNamePath = packageName.replace(/\./g, '/');
    const loading = ora('start download template').start();
    //adb拉取git项目
    shell.exec(`git clone ${repo}`,(error,stdout,stderr)=>{
        if(error){
            loading.fail('download template fail');
        }else{
            //重命名projectName
            fs.renameSync('AndroidSampleCLI',`${projectName}`)
            //重命名packageName
            fs.renameSync(`${projectName}/app/src/main/java/com/android/sample`,`${projectName}/app/src/main/java/${packageNamePath}`)
            //移除git隐藏文件
            fs.rm(`${projectName}/.git`, { recursive: true }, (err) => {
                if (err) {
                    loading.fail('download template fail');
                } else {
                    try{
                        const tmpData = {"packageName" : `${packageName}`,"projectName":`${projectName}`};
                        //修改模板文件的projectName和packageName
                        var filePaths = [`${projectName}/build.gradle`,`${projectName}/app/src/main/res/values/strings.xml`,`${projectName}/app/src/main/java/${packageNamePath}/MainActivity.kt`,`${projectName}/settings.gradle`];
                        filePaths.forEach((path,index)=>{
                            ejs.renderFile(path,tmpData).then(data =>{
                                fs.writeFileSync(path,data)
                            });
                        });
                        loading.succeed('download template success');
                    }catch(err){
                        loading.fail('download template fail');
                    }
                }
              });
            
        }
    });
    
};
export {handleDownload}
```
[my-test-cli](https://github.com/dengshaomin/my-test-cli)

***
npm link链接到全局，用于本地cli执行调试
```
npm link 

➜  my-test-cli git:(master) dsm-vue-cli
? project name: dsm-vue-cli-sample
? package name: com.android.sample
⠙ start download template
```
发布项目到npm仓库
```
npm publish

npm notice 
npm notice 📦  dsm-vue-cli@0.0.1
npm notice === Tarball Contents === 
npm notice 274B   .idea/modules.xml                                                          
npm notice 336B   .idea/my-test-cli.iml                                                      
npm notice 2.0kB  download.js                                                                
npm notice 169B   dsm-vue-cli-sample/.idea/compiler.xml                                      
npm notice 680B   dsm-vue-cli-sample/.idea/gradle.xml                                        
npm notice 468B   dsm-vue-cli-sample/.idea/misc.xml                                          
npm notice 180B   dsm-vue-cli-sample/.idea/vcs.xml                                           
npm notice 1.2kB  dsm-vue-cli-sample/app/build.gradle                                        
npm notice 750B   dsm-vue-cli-sample/app/proguard-rules.pro                                  
npm notice 873B   dsm-vue-cli-sample/app/src/main/AndroidManifest.xml                        
npm notice 299B   dsm-vue-cli-sample/app/src/main/java/com/android/sample/MainActivity.kt    
npm notice 1.7kB  dsm-vue-cli-sample/app/src/main/res/drawable-v24/ic_launcher_foreground.xml
npm notice 5.6kB  dsm-vue-cli-sample/app/src/main/res/drawable/ic_launcher_background.xml    
npm notice 425B   dsm-vue-cli-sample/app/src/main/res/layout/activity_main.xml               
npm notice 272B   dsm-vue-cli-sample/app/src/main/res/mipmap-anydpi-v26/ic_launcher_round.xml
npm notice 272B   dsm-vue-cli-sample/app/src/main/res/mipmap-anydpi-v26/ic_launcher.xml      
npm notice 343B   dsm-vue-cli-sample/app/src/main/res/mipmap-anydpi-v33/ic_launcher.xml      
npm notice 2.9kB  dsm-vue-cli-sample/app/src/main/res/mipmap-hdpi/ic_launcher_round.webp     
npm notice 1.4kB  dsm-vue-cli-sample/app/src/main/res/mipmap-hdpi/ic_launcher.webp           
npm notice 1.8kB  dsm-vue-cli-sample/app/src/main/res/mipmap-mdpi/ic_launcher_round.webp     
npm notice 982B   dsm-vue-cli-sample/app/src/main/res/mipmap-mdpi/ic_launcher.webp           
npm notice 3.9kB  dsm-vue-cli-sample/app/src/main/res/mipmap-xhdpi/ic_launcher_round.webp    
npm notice 1.9kB  dsm-vue-cli-sample/app/src/main/res/mipmap-xhdpi/ic_launcher.webp          
npm notice 5.9kB  dsm-vue-cli-sample/app/src/main/res/mipmap-xxhdpi/ic_launcher_round.webp   
npm notice 2.9kB  dsm-vue-cli-sample/app/src/main/res/mipmap-xxhdpi/ic_launcher.webp         
npm notice 7.8kB  dsm-vue-cli-sample/app/src/main/res/mipmap-xxxhdpi/ic_launcher_round.webp  
npm notice 3.8kB  dsm-vue-cli-sample/app/src/main/res/mipmap-xxxhdpi/ic_launcher.webp        
npm notice 805B   dsm-vue-cli-sample/app/src/main/res/values-night/themes.xml                
npm notice 378B   dsm-vue-cli-sample/app/src/main/res/values/colors.xml                      
npm notice 80B    dsm-vue-cli-sample/app/src/main/res/values/strings.xml                     
npm notice 805B   dsm-vue-cli-sample/app/src/main/res/values/themes.xml                      
npm notice 478B   dsm-vue-cli-sample/app/src/main/res/xml/backup_rules.xml                   
npm notice 551B   dsm-vue-cli-sample/app/src/main/res/xml/data_extraction_rules.xml          
npm notice 440B   dsm-vue-cli-sample/build.gradle                                            
npm notice 1.4kB  dsm-vue-cli-sample/gradle.properties                                       
npm notice 59.2kB dsm-vue-cli-sample/gradle/wrapper/gradle-wrapper.jar                       
npm notice 230B   dsm-vue-cli-sample/gradle/wrapper/gradle-wrapper.properties                
npm notice 5.8kB  dsm-vue-cli-sample/gradlew                                                 
npm notice 2.8kB  dsm-vue-cli-sample/gradlew.bat                                             
npm notice 335B   dsm-vue-cli-sample/settings.gradle                                         
npm notice 557B   index.js                                                                   
npm notice 524B   package.json                                                               
npm notice === Tarball Details === 
npm notice name:          dsm-vue-cli                             
npm notice version:       0.0.1                                   
npm notice filename:      dsm-vue-cli-0.0.1.tgz                   
npm notice package size:  98.0 kB                                 
npm notice unpacked size: 123.5 kB                                
npm notice shasum:        4806c053fd9e0e40c3fee36df6a8bce017b24096
npm notice integrity:     sha512-h/fsRLf+BXO97[...]eYNdxKI5Ja2oQ==
npm notice total files:   42                                      
npm notice 
npm notice Publishing to https://registry.npmjs.org/
+ dsm-vue-cli@0.0.1
```
没有npm账号需要到npm官网注册账号，第一次publish的时候需要输入账号和密码，根据console的提示操作即可;发布成功后即可和其他npm包一样通过npm install **进行安装和使用了；



