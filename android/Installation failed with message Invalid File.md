# IDE更新后，报错
如下：
```
Installation failed with message Invalid File: K:\project\app\build\intermediates\split-apk\with_ImageProcessor\debug\slices\slice_0.apk. It is possible that this issue is resolved by uninstalling an existing version of the apk if it is present, and then re-installing.

WARNING: Uninstalling will remove the application data!

Do you want to uninstall the existing application?
```
解决方案：

```
Click Build tab ---> Clean Project

Click Build tab ---> Build APK

Run.
```
原文链接：
> http://stackoverflow.com/questions/42219784/installation-failed-with-message-invalid-file