## 设计原则

### 最少知识原则

当设计一个类时，应该关注与它交互的类有哪些，它们之间是如何交互的；

#### 应用方法：

对于任何对象而言，在该对象的方法内，只应该调用属于以下范围的方法：

- 该对象本身；
- 被当做参数传递进来的对象；
- 此方法创建的或实例化的对象；
- 该对象的任何组件；

#### 举例：

```Go
type Temp struct {
    station TempStation
}

//不符合原则的实现
func (t *Temp) GetTemp()  float32 {
    thermometer := t.station.GetThermometer() //先从气象站获得温度计对象
    return thermometer.GetTemp() // 再从温度计获得温度
}

//符合原则的实现
func (t *Temp) GetTemp()  float32 {
    return t.station.GetTemp()
}

```