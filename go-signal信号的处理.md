在生产环境下运行的系统要求优雅退出，即程序接收退出通知后，会有机会先执行一段清理代码，将收尾工作做完后再真正退出。我们采用系统Signal来 通知系统退出，即kill pragram-pid。我们在程序中针对一些系统信号设置了处理函数，当收到信号后，会执行相关清理程序或通知各个子进程做自清理。kill -9强制杀掉程序是不能被接受的，那样会导致某些处理过程被强制中断，留下无法恢复的现场，导致消息被破坏，影响下次系统启动运行。

golang 的系统信号处理主要涉及os包、os.signal包以及syscall包。其中最主要的函数是signal包中的Notify函数：
```
func Notify(c chan <-os.Signal, sig …os.Signal)
```

该函数会将进程收到的系统Signal转发给channel c。转发哪些信号由该函数的可变参数决定，如果你没有传入sig参数，那么Notify会将系统收到的所有信号转发给c。如果你像下面这样调用Notify：
```
signal.Notify(c, syscall.SIGINT, syscall.SIGUSR1, syscall.SIGUSR2)
```

则Go只会关注你传入的Signal类型，其他Signal将会按照默认方式处理，大多都是进程退出。因此你需要在Notify中传入你要关注和处理的Signal类型，也就是拦截它们，提供自定义处理函数来改变它们的行为。

下面是一个signal处理实例：

```go
package main
import "fmt"
import "time"
import "os"
import "os/signal"
import "syscall"
//信号处理句柄
type signalHandler func(s os.Signal, arg interface{})
//信号处理集
type signalSet struct {
   m map[os.Signal]signalHandler
}
//新建信号处理集
func signalSetNew()(*signalSet){
   ss := new(signalSet)  
   ss.m = make(map[os.Signal]signalHandler)
   return ss
}
//注册信号处理句柄
func (set *signalSet) register(s os.Signal, handler signalHandler) {
   if _, found := set.m[s]; !found { 
     set.m[s] =  handler   
   }
}
//信号处理集处理信号
func (set *signalSet) handle(sig os.Signal, arg interface{})(err error) {
   if _, found := set.m[sig]; found {
      set.m[sig](sig, arg) 
      return nil   
   } else {
      return fmt.Errorf("No handler available for signal %v", sig)   
   }   panic("won't reach here")
}

func main() {
   go sysSignalHandleDemo()
   time.Sleep(time.Hour) // make the main goroutine wait!
}

func sysSignalHandleDemo() {
   ss := signalSetNew()   
   handler := func(s os.Signal, arg interface{}) { 
     fmt.Printf("handle signal: %v\n", s)   
   }
   ss.register(syscall.SIGINT, handler)
   for {
      c := make(chan os.Signal)
      var sigs []os.Signal 
      for sig := range ss.m {
         sigs = append(sigs, sig)
      }
      signal.Notify(c)
      // Block until a signal is received. 
      sig := <-c
      //处理接收到的信号
      err := ss.handle(sig, nil)
      if err != nil {
         fmt.Printf("unknown signal received: %v\n", sig)
         os.Exit(1)
      }   
   }
}
```

上例中Notify函数只有一个参数，没有传入要关注的sig，因此程序会将收到的所有类型Signal都转发到channel c中。build该源文件并执行程序：

$&gt; go build signal.go

$&gt; signal

在另外一个窗口下执行如下命令：

$&gt; ps -ef\|grep signal

tonybai 25271 1087 0 16:27 pts\/1 00:00:00 signal

$&gt; kill 25271

我们在第一个窗口会看到如下输出：

$&gt; signal

handle signal: interrupt

