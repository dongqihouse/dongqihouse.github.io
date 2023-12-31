---
layout:     post
title:      "关于下载的一些记录"
subtitle:   " \" URLSession \""
date:       2018-10-7 16:00:00
author:     "QD"
header-img: "/img/post-bg-2015.jpg"
catalog: true
tags:
    - iOS
---

国庆这几天，同学问了一个问题。app 进入后台就处于挂起休眠状态，为什么各大视频app都在在后台缓存（锁屏状态下也可以缓存啊）。

因为自己没有怎么做过这方面的东西，瞬间有点懵逼。第一反应就是，iOS有一个短暂的在后台工作的代码
```swift
application.beginBackgroundTask(withName: "test") {

}

//do something
```

这个方法会让app有短暂几分钟的活跃状态。但是面对长时间而且情况复杂的下载任务，明显是不够的。

在翻了google和文档之后。确定以前几点
* 利用URLSession 去建立后台下载任务
* 后台下载任务在应用挂起的时候，下载任务实在系统进程中的
> If your app is in the background, the system may suspend your app while the download is performed in another process.
* 进入后台之后，再回到当前应用要利用后台的`identity`来恢复下载状态
* 当系统进程下载完成之后会回调appDelegate的回调方法之中返回一个完成的闭包， 这个闭包**必须**在session的complete的代理中调用
> You should immediately store this handler wherever it makes sense for your app, perhaps as a property of your app delegate, or of your class that implements URLSessionDownloadDelegate

这一点就爽歪歪了， 无论我应用是否挂起还是杀死。都不能阻止下载任务的进行。 以前流量不够的时候，误碰到下载按钮都是秒杀死app来防止自己倾家荡产。 现在看在自己还是太年轻。

代码差不多这个样子
```
func beginDownload() {
let url = "http://flv2.bn.netease.com/videolib3/1706/07/gDNOH8458/HD/gDNOH8458-mobile.mp4"

let configure = URLSessionConfiguration.background(withIdentifier: "com.aa.a..a...a.a.")
let session = URLSession(configuration: configure, delegate: self, delegateQueue: nil)

session.getAllTasks { (tasks) in
if tasks.count > 0 {
// 恢复下载
for task in tasks {
task.resume()
}
} else {
// 创建下载
let task =  session.downloadTask(with: URL.init(string: url)!)
task.resume()
}
}     
}


func urlSession(_ session: URLSession, downloadTask: URLSessionDownloadTask,
didFinishDownloadingTo location: URL) {
guard let httpResponse = downloadTask.response as? HTTPURLResponse,
(200...299).contains(httpResponse.statusCode) else {
print ("server error")
return
}
do {
let documentsURL = try
FileManager.default.url(for: .documentDirectory,
in: .userDomainMask,
appropriateFor: nil,
create: false)
let savedURL = documentsURL.appendingPathComponent(
location.lastPathComponent)
try FileManager.default.moveItem(at: location, to: savedURL)
} catch {
print ("file error: \(error)")
}
}



func urlSessionDidFinishEvents(forBackgroundURLSession session: URLSession) {
DispatchQueue.main.async {
let delegate = UIApplication.shared.delegate as? AppDelegate
delegate!.backgroundCompletionHandler()
}

}


func urlSession(_ session: URLSession, downloadTask: URLSessionDownloadTask, didWriteData bytesWritten: Int64, totalBytesWritten: Int64, totalBytesExpectedToWrite: Int64) {
let a = Double(totalBytesWritten) / Double(totalBytesExpectedToWrite)
print(a)
DispatchQueue.main.async {
self.progressView.progress = Float(a)
self.label.text = String(a)
}

//        let a = Double(totalBytesWritten) / Double(totalBytesExpectedToWrite)
//        print(a)
}

```

上面3个代理
* `didFinishDownloadingTo `下载完成之后告你 沙盒中下载内容的位置。
以前下载大文件都要边下载变存贮，现在都帮你搞好了。只管调用api就行。
* `urlSessionDidFinishEvents` 后台任务下载结束之后要处理appDelegate传过来的完成闭包。 随便苹果没告诉你为啥这样做，但是文档指明一定要调用
* `URLSessionDownloadTask ` 来提示下载进度的


其它的一点东西
顺便了看了断点下载。
我记得以前搞这个大文件下载和断点下载都挺麻烦的。但是更新URLSession之后都是很容易就搞定
断点下载
```
task.suspend()
task.resume()
```
都不用记录什么数据了
以后感觉马上就是UI工程师了


