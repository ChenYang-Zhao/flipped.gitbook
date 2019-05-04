## 命令模式

### 概念

#### 出发点

将动作的执行者从动作的请求者对象中解耦，命令模式的请求者只关心如何发起请求，不关心请求的具体实现，甚至不关心请求具体的执行者是谁；

#### 实现方法

将请求封装成对象，对象实现Command接口，对外提供execute方法，请求的发起者只需调用此方法即可完成请求；

#### 定义

命令模式将请求封装成对象，以便使用不同的请求、队列或者日志来参数化其它对象。命令模式也可以支持撤销操作；

### 命令类图及角色

![](../imgs/command-pattern-class-diagram.jpeg)

### 代码示例

以遥控器指挥电灯、空调等设备的开关为例

```Go
//Command接口，请求的执行者统一实现该接口
type Command interface {
    Execute() error
}

const SlotCount = 10

//遥控器类的实现
type RemoteControl struct {
    Commands [SlotCount]Command //该遥控器每个slot对应的Command对象
}

func NewRemoteControl() *RemoteControl {
    r := &RemoteControl{
        Commands: [SlotCount]Command{}
    } 

    for _, command := range r.Commands{
        command = NewNoCommand() //Command的默认实现，什么都不做
    }
}
//给遥控器的slot设置具体的命令对象
func (rc *RemoteControl) SetCommand(slot int, c Command) {
    rc.Commands[slot] = c
}
//按下遥控器的按钮触发此方法
func (rc *RemoteControl) PutButton(slot int) {
    rc.Commands[slot].Execute()
}

// 台灯命令的实现
type LightOnCommand struct {
    L *Light //台灯对象，有On/Off方法控制台灯的开关
}

func (loc *LightOnCommand) Execute() error {
    loc.L.On()
}

type LightOffCommand struct {
    L *Light
}

func (loc *LightOffCommand) Execute() error {
    loc.L.Off()
}

func main() {
    rc := NewRemoteControl()
    rc.SetCommand(1, &LightOnCommand{L:xx})
    rc.SetCommand(2, &LightOffCommand{L:xx})

    //test on and off
    rc.PutButton(1)
    rc.PutButton(2)
}

```

