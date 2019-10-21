---
layout:     post
title:      Golang执行CMD方法
date:       2019-10-21
author:     JC
header-img: img/golang.jpg
catalog: false
tags:
    - golang
---

1. Golang执行系统命令使用 os/exec Command方法：

		func Command(name string, arg ...string) *Cmd

第一个参数是命令名称，后面参数可以有多个命令参数。

```
cmd := exec.Command("ls", "-a")

if stdout, err := cmd.StdoutPipe(); err != nil {     //获取输出对象，可以从该对象中读取输出结果

    log.Fatal(err)

}

defer stdout.Close()   // 保证关闭输出流

 

if err := cmd.Start(); err != nil {   // 运行命令

    log.Fatal(err)

}

 

if opBytes, err := ioutil.ReadAll(stdout); err != nil {  // 读取输出结果    

    log.Fatal(err)

} else {

    log.Println(string(opBytes))

}
```
 

2.  将命令的输出结果重定向到文件中： 
```
    stdout, err := os.OpenFile("stdout.log", os.O_CREATE|os.O_WRONLY, 0600)   

    if err != nil {

        log.Fatalln(err)

    }

    defer stdout.Close()

    cmd.Stdout = stdout   // 重定向标准输出到文件

    // 执行命令

    if err := cmd.Start(); err != nil {

        log.Println(err)

    }
```
 

3. cmd的Start和Run方法的区别： 

Start执行不会等待命令完成，Run会阻塞等待命令完成。
```
cmd := exec.Command("sleep", "10")

err := cmd.Run()  //执行到此处时会阻塞等待10秒

err := cmd.Start()   //如果用start则直接向后运行

if err != nil {

    log.Fatal(err)

}

err = cmd.Wait()   //执行Start会在此处等待10秒
```
 

4. 如果命令名称和参数写成一个字符串传给Command方法，可能会执行失败报错：file does not exist，但此时如果按以下方式强行启动一个DOS窗口（windows平台）进行执行，也是成功的。

在Windows平台，强行弹出DOS窗口执行命令行： 
```
cmdLine := pscp -pw pwd local_filename user@host:/home/workspace   
cmd := exec.Command("cmd.exe", "/c", "start " + cmdLine)
err := cmd.Run()
fmt.Printf("%s, error:%v \n", cmdLine, err)
```

5. 运行时隐藏golang程序自己的cmd窗口：

		go build -ldflags -H=windowsgui 


6. Windows平台上，执行系统命令隐藏cmd窗口：
```
cmd := exec.Command("sth")
if runtime.GOOS == "windows" {
    cmd.SysProcAttr = &syscall.SysProcAttr{HideWindow: true}
}
err := cmd.Run()
```
