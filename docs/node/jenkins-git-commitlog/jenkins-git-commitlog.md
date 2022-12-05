搭建环境：Mac Pro
[参考](https://blog.csdn.net/u011418943/article/details/108131146)

##### 完成以下任务
* jenkins搭建android ci
* shell的常用命令
* 获取当前编译版本所有commit log
* 将commit log文件写入assets下

##### 使用场景
项目中遇到测试同学不清楚当前的测试包包含哪些功能以及修复了哪些BUG

##### 安装jenkins
[jekins官网下载](https://www.jenkins.io/download/lts/macos/)最新版本，本文基于最新的jenkins
```
brew install jenkins-lts
```
或者指定版本
```
brew install jenkins-lts@YOUR_VERSION
```
启动服务
```
brew services start jenkins-lts
```
常用命令
```
Install the latest LTS version: brew install jenkins-lts
Install a specific LTS version: brew install jenkins-lts@YOUR_VERSION
Start the Jenkins service: brew services start jenkins-lts
Restart the Jenkins service: brew services restart jenkins-lts
Update the Jenkins version: brew upgrade jenkins-lts
```
Dashboard->系统管理->Configure System


![logo](0.png ':size=WIDTHxHEIGHT')

Dashboard->系统管理->全局工具配置


![logo](1.png ':size=WIDTHxHEIGHT')
![logo](2.png ':size=WIDTHxHEIGHT')
![logo](3.png ':size=WIDTHxHEIGHT')

Dashboard新建一个任务用(SampleAndroid)来编译git上的Android项目


![logo](4.png ':size=WIDTHxHEIGHT')

Dashboard->SampleAndroid对项目进行配置

jdk配置，可以选择安装的多个版本，应为当前是在本机，所以使用了系统默认的


![logo](5.png ':size=WIDTHxHEIGHT')

配置git仓库地址，`Repositories` 对应github项目ssh地址，`Credentials`默认是空的需要新建一个，账号密码对应github的账号密码

![logo](6.png ':size=WIDTHxHEIGHT')

要获取git的提交记录依赖一个插件`changelog-environment.hpi`，这个插件并不在jenkins的插件库中（[下载](https://github.com/KrisMarko/kr-changelog)）,下载后将`changelog-environment.hpi`拷贝到jenkins插件目录：
```
/Users/shaomin.deng/.jenkins/plugins
```
Dashboard->SampleAndroid->构建环境做下图配置
```
Entry Format：%3$s(at %4$s via %1$s)
Date Format：yyyy-MM-dd HH:mm:ss
```

![logo](7.png ':size=WIDTHxHEIGHT')

增加构建步骤：`执行shell` 获取当前版本的所有commit记录，由于`changelog-environment.hpi`只能获取到当前一次的提交信息，如果连续打了2个包，将会出现第一个包有commit信息，第二个包没有任何信息；因为变更已经记录到了第一个包中。 所以需要通过shell记录上次的版本号以及commit信息；用当次的版本号和上次的对比决定是覆盖还是插入


![logo](8.png ':size=WIDTHxHEIGHT')

```
#!/bin/bash
file="${WORKSPACE}/app/build.gradle"
line=$(sed -n '/versionName/=' $file) #获取要修改的字符串所在行,并将它保存到变量line
v=$(sed -n ${line}p $file)
v1=${v#*\"}
version=${v1%*\"}
logpath="${WORKSPACE}/app/src/main/assets/commitlog.log"
#rm $logpath
if  [ -f $logpath ];then
  commitlog=$(cat $logpath)
    if [ -z $commitlog ];then  #空文件
    commitlog="$version\n$SCM_CHANGELOG"
        echo $commitlog >> $logpath
    else 
      lineversion=$(sed -n 1p $logpath)
      if [ "$version" == "$lineversion" ];then
        if [ -n "$SCM_CHANGELOG" ]; then #提交log不为null
          #sed -i "2i ${content}" $logpath #linux插入内容到第2行,mac和linux下命令不同
sed -i "" "1a\ 
$SCM_CHANGELOG
" $logpath
    fi
      else
        cat /dev/null > $logpath #清空文件
        echo -e "$version\n$SCM_CHANGELOG" >> $logpath
      fi
  fi
else
  touch $logpath
  commitlog="$version\n${SCM_CHANGELOG}\nend"
  #iconv -f UTF-8 -t UTF-8 $logpath #转utf8
  echo -e $commitlog >> $logpath
fi
#commitlog=${cat $logpath} #此行会执行错误，用于阻断任务；调试使用
```

增加构建步骤：`gradle script` 用于启动android的编译
```
clean
assembleRelease
```


![logo](9.png ':size=WIDTHxHEIGHT')


启动项目构建，`shell`脚本已经将commitlog.log文件写入到了app/src/main/assets中，app启动后可从assets中读取文件内容；
```
1.0
test commit 3(at 2022-12-06 00:37:27 via shaomin.deng)
test commit 2(at 2022-12-06 00:36:17 via shaomin.deng)
test commit 1(at 2022-12-06 00:34:03 via shaomin.deng)
end
```
由于mac系统是原生于bsd系，sed命令和gnu不同，如果想用sed实现在某一行插入一行文本,须通过换行实现:
```
sed -i "" "1a\ 
$SCM_CHANGELOG
" $logpath
```
可以安装`gun-sed`使用更简单的shell执行插入命令

#### shell


[Linux 命令大全](https://www.runoob.com/linux/linux-command-manual.html)shell在线工具
```
https://www.runoob.com/try/runcode.php?filename=helloworld&type=bash
https://www.onlinegdb.com/online_bash_shell
```
