Sample

Bro offer a simple project named `sample` to show how to use each function of the Bro.For better presentation,sample is originazed by single project and multi Module (sample also can be splited to multi Repo),as whitch,app is the major project,others are individual service modules.

```
// package the sample project
./gradlew assembleDebug

// debug bro-complier , bro-gradle-plugin
./gradlew assembleDebug -Dorg.gradle.debug=true --no-daemon

```

# Hot Deployment

Local Hot Deployment is similar to a plug-in deployment.The modules are individually packaged to apk and loaded dynamically. In order to modularity more thorough,low cost of debugging,even can imcrease the packing rate by more than ten times.

Different from the common plug-in deployment is that Bro is conditioned on stabilization and compatibility.The package would be resource redundant,and 

## Usage

### Configure

1.open the switch of Bro Hot Deployment in the host project;

```
bro {
    isPlugadgetEnabled true
}
```

2.declare the package name of module in the modules which needs hot deployment(This is because when module as the library it needn't applicationId,but needs when packing as the application);

```
bro{
    packageName 'com.example.app.home'
}
```

### Package
1.First,packaging the host project and installing in the phone.There's no difference from your normal packaging.At this point,the package is containing all modules:
```
./gradlew clean installDebug
```

2.Separately packaging the module when changing a module which need to debug:
```
./gradlew clean assembleDebug -PbroPlugadget=app.settings -PbroAppOutputs=/path/to/your/app/build/bro/debug_bro_app_outputs.zip
```
Among these:
- `broPlugadget`,necessary  parameter,to definitely appoint this module as Plugadget mode，open the plug-in package mode of the module;
- `broAppoutputs`,necessary parameter,to appoint the package of the host project.Different variant can have different.zip files depending on the host when packaged
- After packaging,the result located on `module/build/bro/com.package.name.plugadget`,and it will automatically transmit module result to the phone by adb,shall be in effect after the app restart.


### Character
support
- any one java code change in the module 
- any resources change besides Activity animation in the module

not support
- [would not support in the future]any changes to manifest,including adding Activity and so on.
- [would not support in the future]Native Lib update with module dependencies (.so);
- [would not support in the future]Java Lib and its resource update with module dependencies.(.jar .aar);
- [would not support in the future]By getApplicationContext().getResources() obtain plugin add or modify resources,but you can using getResources() provided by BroModule to support this scenario.
- [maybe support in the future]The apk packaged by module has already exist multi-dex,which is 99.99% caused by no reasonable partition,so it just don't support at the moment.


### Attention
- Local hot deployment is not recommended for online use now because of the redundancy of resources in the plug-in package(Sure it is work towards local hot deployment + efficient stability,so the project needs a trade-off,swap space for complexity) 
- Owing to reading plug-ins in out-position storage requires "storage privileges",in Android 6.0 and later,the first installed App needs to obtain privileges and force stop completely(to prevent ClassLoader cache),after that,the local hot deployment will take effect.Pay attention to that many systems will not kill the progress by simply swiping your App in the task manager;
- For better isolation and improving plug-in packaging rate, currently the package name configured by the module is strongly verified to be consistent with the module code package name.Only classes prefixed by the package name will be packaged into the plug-in module.

- If the main project is repackaged, all plug-ins must be replayed and removed from the existing plug-ins in the SD card(Locate on /sdcard/bro/${com.app.package.name}/) Considering offer a task to delete plugin later.
- It's not recommended to debug more than one plugin at the same time .Although it's supported in theory,the time spent on packaging multiple plug-in is not much different form the time spent on packaging the whole package.Bro was not do too much test and support in this part.


## Best Practices

### Avoid SNAPSHOT trap:
Generally,in multi module project,before the new release,host always relies on individual business modules in the form of SNAPSHOT.At this time,once At this time,once someone accidentally publishes a problematic SNAPSHOT, it can often result in people being unable to package. Local hot deployment can help to avoid this problem to some extent, as long as you packa a baseline package every morning, and then only pack and debug your module plug-ins for the rest of the day (provided, of course, that you don't do business with new code from other modules today).
### Constraint and Freedom

At present, open the local hot-deployed module, and the code package name will be strongly verified (the resource prefix will be considered in the future). When packaging the module plug-in, only the code under the package name will be put into the Plugadget package.Though,to a certain extent,that reduces the degree of freedom of the module(the root of the module package name should be consistent),creates many benefits.

- Separate code namespace,is the base of separation measures in the future;
- It is beneficial to check the  online problems. You can find the corresponding module and person in charge by looking at the package name.
- Reduce class duplication by accident -- in modular architectures, each module is individually packaged and integrated (in the form of aar), and isolated packaging environments are more likely to have packaging errors of the same class if they are prefixed with duplicate package names.

The constraint is meant to provide more freedom and stability. If possible, make the package name and resource prefix independent from the moment the module is built.Regardless of whether the local hot deployment is turned on or not.

### Automatically Opening Local Hot Deployment

There are two ways to control whether to open hot deployment.The first is configure the build.gradle in host project:

```
bro {
    isPlugadgetEnabled true
}
```
The second is switched the plug-in,including `-PBroPlugadget`

About package the plugs,it is automated,the different environments require different command.But for host project,it needs a dynamic configuration to control whether to open.For example,open in Debug package,close in Release package:

```
def isDebug() {
    if(gradle.startParameter.getTaskNames().size() == 0) { // for clean etc..
        return true
    }

    List<String> tasks = gradle.startParameter.getTaskNames();
    for(String s : tasks) {
        if(s.contains("Debug")) {
            return true;
        }
    }

    return false;
}

bro {
    isPlugadgetEnabled isDebug() 
}
```
As above,you can custom a function judging package Task Name to deal with these problems.