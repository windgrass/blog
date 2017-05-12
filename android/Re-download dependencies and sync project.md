# 打开新项目，报错
错误如下：

> Error:Failed to open zip file.
Gradle's dependency cache may be corrupt (this sometimes occurs after a network connection timeout.)
Re-download dependencies and sync project (requires network)
Re-download dependencies and sync project (requires network)

I changed all the dependency versions to latest one bu still it is showing the same message.

It has two modules under the main project 1) BaseGameUtils & 2)ChaseWhisply. The above modules are not shown in bold letters (I think it should be shown in bold letters like module "app").

## 解决

down vote
accepted
While importing a libGDX project I got the same message from Android Studio 2.2.

Went to File->Project Structure then selected Project and set

Gradle version

and

Android Plugin Version

to values identical to another project I had recently created with Android Studio 2.2 and updated SDK. Could then sync gradle. Did Build->Build Clean with no issues. Then did Run->Clean and Rerun and project ran fine.

Hope this helps.