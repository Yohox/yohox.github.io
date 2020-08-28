---
title: 基于IBeacon+惯性实现精确实时导航小程序
date: 2020-08-27 22:26:36
tags: IBeacon, 室内定位, 小程序, 惯性导航, 组合导航, 室内导航
mathjax: true
---
作者: 李英濠(Yohox)  转载请注明出处。



代码我传到了GitHub 希望大家多给点Star 谢谢~

https://github.com/Yohox/IBeaconLocationMiniprogram

我的博客: http://yoho.ac.cn

# 背景

因为所在的公司举行极客大赛(就是30小时内写一个程序然后出来演示大众评委打分)，小伙伴提议说做室内导航，当初没多想以为挺简单，就接下来了。

初步调研方案我们采用了IBeacon作为我们的定位信号源设备，基本原理就是手机通过小程序捕获IBeacon的信号根据信号衰减程度推算出距离，因为IBeacon的X,Y坐标我们事先已经记录下来了，那么我们就能过三个两点距离公式推算出用户手机的X,Y坐标。

事后发现其中的技术难点远远没有想象中的那么简单，都是泪啊~，虽然只拿了铜奖，但其实我们得票数是全场第三，但是此次评分规则不是按总票数来算，是按各个维度来算最后只拿了铜奖可惜了。

# 原理

### 测距原理

我们首先需要获得手机与IBeacon之间的距离，IBeacon是一个按一定周期向外发射信号的一个蓝牙元件，手机可以通过蓝牙接收器捕获IBeacon发出的蓝牙信号，手机距离发射端越远信号则越差，也就是说距离与信号的衰减呈现某种关系，然后大佬们就总结出了以下公式：
$$
D=10^{(A-tx)/10n}
$$
D: 是终端(手机)与信号源(IBeacon)的距离

A: 是接受到的蓝牙信号强度(RSSI)

tx: 终端与信号源相隔1米时的信号强度(RSSI) 一般为-59 根据实际情况可做调整

n: 环境衰减因子 一般为2 根据实际情况调整

但是IBeacon发射的信号往往不是稳定的，所以我们通过不稳定的RSSI推算出的距离也只是一个估算值（后文会讲怎么处理）



### 定位原理

我们上学都学过两点之间的距离可以根据公式得到：
$$
|AB|=\sqrt{(X_{1}-X_{2})^2+(Y_{1}-Y_{2})^2}
$$
根据上面的测距原理，我们可以得知终端与信号源之间的距离，也就是AB，并且IBeacon是我们自己部署的，所以X,Y坐标我们也是提前知道的(也就是X2, Y2)是知道的，那么我们需要求的X1,Y1显然在这个式子里无法求解出来，但是如果我们知道了三个IBeacon终端与手机之间的距离呢？那么我们就可以得到三个两点之间的距离公式也就是：
$$
\begin{split}
|AB1|=\sqrt{(X_{1}-X_{2})^2+(Y_{1}-Y_{2})^2} \\
|AB2|=\sqrt{(X_{1}-X_{3})^2+(Y_{1}-Y_{3})^2} \\
|AB3|=\sqrt{(X_{1}-X_{4})^2+(Y_{1}-Y_{4})^2} \\
\end{split}
$$
已知AB1, AB2, AB3, X2, X3, X4, Y2, Y3, Y4 我们就可以解这个方程得出X1, Y1的值了(三元二次方程)

<b>注意: 因为我们的距离是估算值，所以我们得到的坐标也是估算值。</b>





### 惯性补偿

因为IBeacon 发射周期最低是100ms (iOS默认也是100ms)， 并且因为IBeacon的信号值并不是完全稳定的，所以我们需要预留几次IBeacon的RSSI值，这就会导致定位延迟，那么大家可以想想为什么我们的导航在经过隧道等没有GPS的地方仍然可以继续为我们提供导航服务呢？

答案就是通过 **惯性补偿机制**，通过手机的加速度传感器测得一段时间内的加速度值，对加速度进行二次积分则可以得出这段时间位移的距离的估算值，方向我们可以通过读取手机的电子落盘得到这段时间位移的方向，则我们可以推算出这段时间我们的位置距离GPS丢失信号开始的那个点的变化。

但是这里有个致命的点就是加速度值也不完全是精确的，经过二次积分之后误差会更大，并且经过时间的积累误差会越来越大。所以在室内导航里我用了另一种思路来推算用户位移的距离就是通过步数来决定。

如果你边走路边打开手机的加速度计并且做记录会发现我们在走路时加速度值会呈现波浪形波动，一般来说一个波动对应一步，我们只需要检测波动的出现的次数即可推算出所走的步数，微信步数，iOS步数等都是基于此类原理而来的。

![img](https://yoho-1302523027.cos.ap-nanjing.myqcloud.com/img/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20200825225754.jpg)



# 实现

### RSSI值获取

```javascript
wx.startBeaconDiscovery({
      uuids: ["fda50693-a4e2-4fb1-afcf-c6eb07647821"],
      success: () => {
        console.log("startBeaconDiscovery success")
        wx.onBeaconUpdate(() => {
          wx.getBeacons({
            success: (res) => {
              if (res.beacons.length == 0) {
                return
              }
              for (var beacon of res.beacons) {
    
                if (beacon.rssi == 0) {
                  continue
                }
                this.pushRssi(beacon.minor, beacon.rssi)
              }
            },
          })
        })
      },
      fail: (res) => {
        console.log("startBeaconDiscovery err: " + res.errMsg)
      }
    })
  },
```

我们通过小程序的startBeaconDiscovery, 开启IBeacon监听,onBeaconUpdate 注册IBeacon状态变更回调，每一次变更我们都用getBeacons获取所有Beacon的信息，这里的RSSI值我们不能直接使用，因为我们之前讲到了RSSI值它不稳定，所以我们需要一个数组来把一定周期内的RSSI值存起来并作出处理输出一个比较可靠的值。





### RSSI值与距离的转换

```javascript
export function getAccuracy(rssi, rrssi, n){
  let iRssi = Math.abs(rssi);
  let power = (iRssi + rrssi) / (10 * n);
  return Math.pow(10, power) * 100;
}
```

就是上述公式的实现 rrssi 是上文的txPower, n为环境衰减因子。





### 高斯模糊

我们不直接对RSSI值做处理，而是对RSSI值转换而来的距离做处理。

经过上面的操作我们可以把对应IBeacon的RSSI数组转换成了距离数组。

第一部分我们采用高斯模糊，来使数据做平滑化。

高斯模糊原理可看： http://www.ruanyifeng.com/blog/2012/11/gaussian_blur.html



高斯模糊的算法步骤：

假设我们求a下标的高斯模糊值，则我们求出这个数组的平均值x， 因为a下标的平均差为: x-a

高斯模糊根据a小标的平均差为数组各个下标都给予了不同的权重(q1, q2, q3……………)

我们要做的是让数组每个下标对应的值乘去他的权重并且求和则得出了a下标的对应高斯模糊值

``` javascript
guassion(accuracys) {
    let accuracysAvg = accuracys.reduce((acc, val) => acc + val, 0) / accuracys.length;
    let res = []
    for (var i in accuracys) {
        // 如果此值就是平均数 则不作高斯模糊
      if (accuracys[i] === accuracysAvg) {
        res.push(accuracys[i])
        continue
      }

      let guassions = getOneGuassionArray(accuracys.length, i, Math.abs(accuracys[i] - accuracysAvg))
      let sum = 0.0
      for (var j in guassions) {
        sum += accuracys[j] * guassions[j]
      }
      res.push(sum)
}
    
export function getOneGuassionArray (size, kerR, sigma) {
  if (size % 2 > 0) {
    size -= 1
  }
  if (!size) {
    return []
  }
  if (kerR > size-1){
    return []
  }
  let sum = 0;
  let arr = new Array(size);

  for (let i = 0; i < size; i++) {
    arr[i] = Math.exp(-((i - kerR) * (i - kerR)) / (2 * sigma * sigma));
    sum += arr[i];
  }

  return arr.map(e => e / sum);
}
```





### 卡尔曼滤波器

经过高斯模糊后，数据已经很平整了，但是还是会有一些毛刺，我们再经过卡尔曼滤波器进行二次处理

对于卡尔曼滤波器的概念可参考知乎用户: @[你忙吧我吃柠檬](https://www.zhihu.com/people/zhu-yitu-18) (不懂也没关系其实我也不懂)

> 通俗理解卡尔曼滤波
> 假设你有两个传感器，测的是同一个信号。可是它们每次的读数都不太一样，怎么办？
>
> 取平均。
>
> 再假设你知道其中贵的那个传感器应该准一些，便宜的那个应该差一些。那有比取平均更好的办法吗？
>
> 加权平均。
>
> 怎么加权？假设两个传感器的误差都符合正态分布，假设你知道这两个正态分布的方差，用这两个方差值，（此处省略若干数学公式），你可以得到一个“最优”的权重。
>
> 接下来，重点来了：假设你只有一个传感器，但是你还有一个数学模型。模型可以帮你算出一个值，但也不是那么准。怎么办？
>
> 把模型算出来的值，和传感器测出的值，（就像两个传感器那样），取加权平均。
>
> OK，最后一点说明：你的模型其实只是一个步长的，也就是说，知道x(k)，我可以求x(k+1)。问题是x(k)是多少呢？答案：x(k)就是你上一步卡尔曼滤波得到的、所谓加权平均之后的那个、对x在k时刻的最佳估计值。
>
> 于是迭代也有了。
>
> 这就是卡尔曼滤波。

对应的代码实现:

```javascript
export function kalmanSmoothing (data) {
  var prevData=0, p=10, q=0.0001, r=0.005, kGain=0;
  return data.map((inData) => {
    p = p+q; 
    kGain = p/(p+r);

    inData = prevData+(kGain*(inData-prevData)); 
    p = (1-kGain)*p;

    prevData = inData;

    return inData; 
  })
}
```





### 时间加权求平均

因为我们这里定的是收集近十次的RSSI值，那么就意味着定位会存在延时， 我们希望越近的RSSI值的出来的距离权重更高，而越久之前得到的值权重更低，这个也很简单我们只需要把数组的前一半再COPY一份 重组成一个新的数组即可。

```javascript
let accuracysAvg = accuracys.slice(0, parseInt(accuracys.length / 2)).concat(accuracys).reduce((acc, val) => acc + val, 0) / (accuracys.length * 1.5)
```

自此我们就得出来了一个相对稳定的IBeacon距离，对于每个IBeacon都要执行以上步骤。





### 最小二乘法实现定位方程求解

原理部分我们讲解了定位的核心原理，就是求解三元二次方程。

$$
\begin{split}
|AB1|=\sqrt{(X_{1}-X_{2})^2+(Y_{1}-Y_{2})^2} \\
|AB2|=\sqrt{(X_{1}-X_{3})^2+(Y_{1}-Y_{3})^2} \\
|AB3|=\sqrt{(X_{1}-X_{4})^2+(Y_{1}-Y_{4})^2} \\
\end{split}
$$


我们把这三个公式平方 然后第一个与第二个相减，第二个与第三个相减 得到：
$$
\begin{split}
AB1^2 - AB2^2 = X_1[2(X_{3}-X_{2})]+Y_1[2(Y_{3}-Y_{2})]+X_{2}^2-X_{3}^2+Y_{2}^2-Y_{3}^2 \\
AB2^2 - AB3^2 = X_1[2(X_{4}-X_{3})]+Y_1[2(Y_{4}-Y_{3})]+X_{3}^2-X_{4}^2+Y_{3}^2-Y_{4}^2
\end{split}
$$
然后我们就可以把他转成矩阵：
$$
\begin{bmatrix} 2(X_3-X_2) & 2(Y_3-Y_2) \\ 2(X_4-X_3) & 2(Y_4-Y_3) \end{bmatrix}
*
\begin{bmatrix} X_1 & Y_1 \end{bmatrix}
=
\begin{bmatrix} AB_1^2-AB_2^2-X_2^2+X_3^2-Y_2^2+Y_3^2 \\ AB_2^2-AB_3^2-X_3^2+X_4^2-Y_3^2+Y_4^2 \end{bmatrix}
$$
我们可以把第一个矩阵设为A 第二个矩阵设为x 第三个矩阵设为B 也就是:
$$
A
*
x
=
B
$$
因为我们得到的距离(AB1 AB2 AB3) 都不是准确值显然我们不可能得出准确的x值，但是我们可以利用最小二乘法求出最优的x矩阵即x 和 y值。

即最终转换为：
$$
x = (A^T * A)^{-1} * A^T * B
$$
参考:

https://blog.csdn.net/weixin_41987016/article/details/91869143

```javascript
let bases = combines[i].map(j => {
    let find = accuracys[j]
    let beacon = this.data.baeconMap[find.id]
    return {
        x: beacon.x,
        y: beacon.y,
        r: find.accuracy
    }
})
try {
    let nBases = trilateral(bases)
    // 最终再对的出来的 sumX 和 sumY 取平均。
    sumX += nBases.x;
    sumY += nBases.y;
    N ++
} catch {
}


export function trilateral(rounds){
  let a = [];
  let b = [];
  /*数组a初始化*/
  for(let i = 0; i < 2; i ++ ) {
      a.push([
          2*(rounds[i].x-rounds[rounds.length-1].x),
          2*(rounds[i].y-rounds[rounds.length-1].y)
      ])
  }

  /*数组b初始化*/
  for(let i = 0; i < 2; i ++ ) {
      b.push([
          Math.pow(rounds[i].x, 2)
              - Math.pow(rounds[rounds.length-1].x, 2)
              + Math.pow(rounds[i].y, 2)
              - Math.pow(rounds[rounds.length-1].y, 2)
              + Math.pow(rounds[rounds.length-1].r, 2)
              - Math.pow(rounds[i].r,2)
      ])
  }

  /*将数组封装成矩阵*/
  let b1 = new Matrix(b);
  let a1 = new Matrix(a);

  /*求矩阵的转置*/
  let a2  = a1.transpose();
  /*求矩阵a1与矩阵a1转置矩阵a2的乘积*/
  let tmpMatrix1 = a2.mmul(a1);
  let reTmpMatrix1 = inverse(tmpMatrix1);
  let tmpMatrix2 = reTmpMatrix1.mmul(a2);
  /*中间结果乘以最后的b1矩阵*/
  let resultMatrix = tmpMatrix2.mmul(b1);
  return {
    x: resultMatrix.get(0, 0),
    y: resultMatrix.get(1, 0)
  }
}
```





### 多点排列求定位

通过上述实现我们就可以让三个IBeacon和距离就可以实现定位了，但是如果我们有五个点如何最大化利用这些点求出一个相对稳定的X,Y值？其实也很简单我们只需要对其进行排列组合 每个组合都进行计算把X,Y结果累加起来求和即可。

代码：

```javascript
let combines = genCombines(accuracys.length, 3)
    let sumX = 0
    let sumY = 0
    let N = 0
    for (var i = 0; i < combines.length; i++) {
      let bases = combines[i].map(j => {
        let find = accuracys[j]
        let beacon = this.data.baeconMap[find.id]
        return {
          x: beacon.x,
          y: beacon.y,
          r: find.accuracy
        }
      })
      try {
        let nBases = trilateral(bases)
        sumX += nBases.x;
        sumY += nBases.y;
        N ++
      } catch {
      }
    }
    return {
      x: sumX / N,
      y: sumY / N,
    }
```





### 去除同行或同列三个IBeacon或以上参与运算的情况

因为IBeacon的有效距离大概是在1-5米左右 超过5米以外的信号衰减厉害，得到的值基本失去参考作用。所以我这里采取的是只让距离5米以内的IBeacon按距离排序截取前4个参与运算，为什么是四个而不是全部呢（这跟我们的排布关系有关，后面会讲）

所以我们期望得到计算的IBeacon应该是这样的：

![image-20200826215434085](https://yohox.oss-cn-shenzhen.aliyuncs.com/img/image-20200826215434085.png)

但是实际上我们可能获取的IBeacon 有可能是三个连在一起的。

![image-20200826215004877](https://yohox.oss-cn-shenzhen.aliyuncs.com/img/image-20200826215004877.png)

所以我们需要去除这类情况，我是准备了一个结果数组，然后遍历已经算出来的IBeacon节点，判定当前IBeacon节点是否在结果数组里存在相同的X或者Y(实际我们设定一个误差比如两个IBeacon节点相差不超过x米则认为是同行或者同列)，并且数量不超过两个的话则加入结果数组否则则判定结果数组里的距离是否跟当前IBeacon节点比把距离较远的替换掉。

代码如下：

```javascript
solveSameLine(accuracys){
   let res = []
   for(var i = 0; i < accuracys.length; i++){
     var a = this.data.baeconMap[accuracys[i].id]
     let xCnt = 0; let yCnt = 0;
      for(var j = 0; j < res.length; j++){
        var b = this.data.baeconMap[res[j].id]
        // x坐标 小于两米 则认为同行
        if(Math.abs(a.x - b.x) <= 280) {
          xCnt ++
        }
        // y坐标 小于三米 则认为同列
        if(Math.abs(a.y - b.y) <= 300){
          yCnt ++
        }
        if(xCnt == 2 || yCnt == 2){
          if(accuracys[i].accuracy < res[j].accuracy){
            res[j] = accuracys[i]
          }
          break
        }
      }
      if(xCnt == 2 || yCnt == 2){
        continue
      }

      res.push(accuracys[i])
   }
   return res
  },
```





### 根据加速度波动实现步数计算

在原理部分，我们已经解释了步数计算的原理，就是根据加速度的波动频率推算出所走的步数，则我们可以对加速度计定时的进行判定，我们对上一次加速度计的值进行缓存，把当前的加速度计的值与上一次加速度的计的值进行判别，如果是升高了则说明在上升阶段，降低则是下降阶段 如果上一次的加速度值比当前的大 且上一次处于上升阶段则 判定为波峰，我们则步数+1。

代码: 

```javascript
initAccelerometer(){
  wx.startAccelerometer({
    interval: 'game',
    success: () => {
      wx.onAccelerometerChange((res) => {
        this.data.step = stepManager.calcStep([res.x, res.y, res.z])
      })
    },
    fail: function(err) {
      console.log("startAccelerometer err:" + err)
    }
  })
},
    
//存放三轴数据
let valueNum = 5;
//用于存放计算阈值的波峰波谷差值
let tempValue = new Array(valueNum);
let tempCount = 0;
//是否上升的标志位
let isDirectionUp = false;
//持续上升次数
let continueUpCount = 0;
//上一点的持续上升的次数，为了记录波峰的上升次数
let continueUpFormerCount = 0;
//上一点的状态，上升还是下降
let lastStatus = false;
//波峰值
let peakOfWave = 0.0;
//波谷值
let valleyOfWave = 0.0;
//此次波峰的时间
let timeOfThisPeak = 0;
//上次波峰的时间
let timeOfLastPeak = 0;
//当前的时间
let timeOfNow = 0;
//当前传感器的值
let gravityNew = 0.0;
//上次传感器的值
let gravityOld = 0.0;
//动态阈值需要动态的数据，这个值用于这些动态数据的阈值
let initialValue = 0.8;
//初始阈值
let ThreadValue = 0.7;

//初始范围
let minValue = 9.0;
let maxValue = 18.6;

let currentStep = 0

export default class StepManager{
  calcStep(accelerometers){
    accelerometers = accelerometers.map(accelerometer => accelerometer * 9.81)
    let average =  Math.sqrt(Math.pow(accelerometers[0], 2)
                + Math.pow(accelerometers[1], 2) + Math.pow(accelerometers[2], 2));
    this.detectorNewStep(average);
    return currentStep
  }

  clearStep(){
    currentStep = 0
  }

  detectorNewStep(values) {
    if (gravityOld === undefined) {
        gravityOld = values;
    } else {
      // 判断是不是过了一个顶峰
      if (this.detectorPeak(values, gravityOld)) {
        timeOfLastPeak = timeOfThisPeak;
        let timeOfNow = Date.parse(new Date());
        if (timeOfNow - timeOfLastPeak >= 50
                && (peakOfWave  - valleyOfWave >= ThreadValue) && (timeOfNow - timeOfLastPeak) <= 2000) {
            timeOfThisPeak = timeOfNow;
            this.preStep();
        }
        if (timeOfNow - timeOfLastPeak >= 50
                && (peakOfWave - valleyOfWave >= initialValue)) {
            timeOfThisPeak = timeOfNow;
            // ThreadValue = this.peakValleyThread(peakOfWave - valleyOfWave);
        }
      }
    }
    gravityOld = values;
  }
  detectorPeak(newValue, oldValue) {
      lastStatus = isDirectionUp;
      if (newValue >= oldValue) {
          isDirectionUp = true;
          continueUpCount++;
      } else {
          continueUpFormerCount = continueUpCount;
          continueUpCount = 0;
          isDirectionUp = false;
      }
      if (!isDirectionUp && lastStatus
              && (continueUpFormerCount >= 2 && (oldValue >= minValue && oldValue < maxValue))) {
          peakOfWave = oldValue;
          return true;
      } else if (!lastStatus && isDirectionUp) {
          valleyOfWave = oldValue;
          return false;
      } else {
          return false;
      }
  }
  peakValleyThread(value) {
    let tempThread = ThreadValue;
    if (tempCount < valueNum) {
        tempValue[tempCount] = value;
        tempCount++;
    } else {
        tempThread = this.averageValue(tempValue, valueNum);
        for (let i = 1; i < valueNum; i++) {
            tempValue[i - 1] = tempValue[i];
        }
        tempValue[valueNum - 1] = value;
    }
    return tempThread;
  }
  averageValue(value, n) {
    let ave = 0;
    for (let i = 0; i < n; i++) {
        ave += value[i];
    }
    ave = ave / valueNum;
    // if (ave >= 8) {
    //     ave = 4.3;
    // } else if (ave >= 7 && ave < 8) {
    //     ave = 3.3;
    // } else if (ave >= 4 && ave < 7) {
    //     ave = 2.3;
    // } else if (ave >= 3 && ave < 4) {
    //     ave = 2.0;
    // } else {
    //     ave = 1.7;
    // }
    return ave;
  }
  preStep() {
    currentStep++;
  }
  toDegrees(angrad) {
    return angrad * 180.0 / Math.PI;
  }
  getRotationMatrix(R, I, gravity, geomagnetic) {
        // TODO: move this to native code for efficiency
        let Ax = gravity[0];
        let Ay = gravity[1];
        let Az = gravity[2];
        let Ex = geomagnetic[0];
        let Ey = geomagnetic[1];
        let Ez = geomagnetic[2];
        let Hx = Ey*Az - Ez*Ay;
        let Hy = Ez*Ax - Ex*Az;
        let Hz = Ex*Ay - Ey*Ax;
        let normH = Math.sqrt(Hx*Hx + Hy*Hy + Hz*Hz);
        if (normH < 0.1) {
            // device is close to free fall (or in space?), or close to
            // magnetic north pole. Typical values are  > 100.
            return false;
        }
        let invH = 1.0 / normH;
        Hx *= invH;
        Hy *= invH;
        Hz *= invH;
        let invA = 1.0 / Math.sqrt(Ax*Ax + Ay*Ay + Az*Az);
        Ax *= invA;
        Ay *= invA;
        Az *= invA;
        let Mx = Ay*Hz - Az*Hy;
        let My = Az*Hx - Ax*Hz;
        let Mz = Ax*Hy - Ay*Hx;
        if (R != null) {
            if (R.length == 9) {
                R[0] = Hx;     R[1] = Hy;     R[2] = Hz;
                R[3] = Mx;     R[4] = My;     R[5] = Mz;
                R[6] = Ax;     R[7] = Ay;     R[8] = Az;
            } else if (R.length == 16) {
                R[0]  = Hx;    R[1]  = Hy;    R[2]  = Hz;   R[3]  = 0;
                R[4]  = Mx;    R[5]  = My;    R[6]  = Mz;   R[7]  = 0;
                R[8]  = Ax;    R[9]  = Ay;    R[10] = Az;   R[11] = 0;
                R[12] = 0;     R[13] = 0;     R[14] = 0;    R[15] = 1;
            }
        }
        if (I != null) {
            // compute the inclination matrix by projecting the geomagnetic
            // vector onto the Z (gravity) and X (horizontal component
            // of geomagnetic vector) axes.
            let invE = 1.0 / Math.sqrt(Ex*Ex + Ey*Ey + Ez*Ez);
            let c = (Ex*Mx + Ey*My + Ez*Mz) * invE;
            let s = (Ex*Ax + Ey*Ay + Ez*Az) * invE;
            if (I.length == 9) {
                I[0] = 1;     I[1] = 0;     I[2] = 0;
                I[3] = 0;     I[4] = c;     I[5] = s;
                I[6] = 0;     I[7] =-s;     I[8] = c;
            } else if (I.length == 16) {
                I[0] = 1;     I[1] = 0;     I[2] = 0;
                I[4] = 0;     I[5] = c;     I[6] = s;
                I[8] = 0;     I[9] =-s;     I[10]= c;
                I[3] = I[7] = I[11] = I[12] = I[13] = I[14] = 0;
                I[15] = 1;
            }
        }
        return true;
  }
  getOrientation(R, values) {
      /*
        * 4x4 (length=16) case:
        *   /  R[ 0]   R[ 1]   R[ 2]   0  \
        *   |  R[ 4]   R[ 5]   R[ 6]   0  |
        *   |  R[ 8]   R[ 9]   R[10]   0  |
        *   \      0       0       0   1  /
        *
        * 3x3 (length=9) case:
        *   /  R[ 0]   R[ 1]   R[ 2]  \
        *   |  R[ 3]   R[ 4]   R[ 5]  |
        *   \  R[ 6]   R[ 7]   R[ 8]  /
        *
        */
      if (R.length == 9) {
          values[0] = Math.atan2(R[1], R[4]);
          values[1] = Math.asin(-R[7]);
          values[2] = Math.atan2(-R[6], R[8]);
      } else {
          values[0] = Math.atan2(R[1], R[5]);
          values[1] = Math.asin(-R[9]);
          values[2] = Math.atan2(-R[8], R[10]);
      }
      return values;
  }
}
```

注: iOS的加速度计的值与Android的值是不一样的，iOS的加速度值读出来要乘去重力加速度 9.81。



### 根据电子落盘与步数推算变更的位置

我们可以通过电子罗盘获得当前设备所面向的方向，并且我们每一次加速度计变化都会计算步数统计，所以我们每一个周期 把这周期内变化的步数。

![image-20200827124722136](https://yohox.oss-cn-shenzhen.aliyuncs.com/img/image-20200827124722136.png)

则易得以下结果：
$$
\begin{align*}
& D = step * 70 \\
& x = x_{0} + D * sin(Degree) \\
& y = y_{0} + D * cos(Degree)
\end{align*}
$$
其中70为推算人每一步所走的距离（因为有误差所以调大了点）

D为 人位移的距离

x为 人新的x坐标

y为 人新的y坐标

代码:

```javascript
setInterval(() => {
      if(this.data.step == 0){
        //this.render()
        return
      }
      let _step = this.data.step - this.data.lastStep
      let x = this.data.x
      let y = this.data.y
      let stepCm = _step * 70
      this.data.lastStep = this.data.step
      y += stepCm * Math.cos(toRadin(this.data.direction))
      x += stepCm * Math.sin(toRadin(this.data.direction))
      if(x < meIconWidth / 2 || y < meIconHeight/ 2){
        return
      }
      this.data.y = y
      this.data.x = x
      this.setData({
        x,
        y
      })
      this.render()
    }, 10)
```



# IBeacon的布置

iBeacon布置方法有两种方法一种是矩阵法，一种是蜂窝法， 矩阵法相对于蜂窝法的密度更高，并且我在任何一个点都可以获得至少四个IBeacon的信号（这也是为什么我只截取前4个IBeacon的原理），蜂窝法则是信号利用率更高的一种排布方法。

### 矩阵法

![image-20200827123741729](https://yohox.oss-cn-shenzhen.aliyuncs.com/img/image-20200827123741729.png)

### 蜂窝法

![img](https://yoho-1302523027.cos.ap-nanjing.myqcloud.com/img/271011070784933.png)

# 总结

因为IBeacon定位存在延时性，所以我们最终的方案是初始化阶段利用IBeacon提供一个相对准确的绝对位置，然后利用惯性推算步数，电子罗盘推算方向推算用户接下来与绝对位置相差的距离和角度即可实现导航功能。

![image-20200826223125330](https://yohox.oss-cn-shenzhen.aliyuncs.com/img/image-20200826223125330.png)

# 参考

https://github.com/megagao/IndoorPos

https://github.com/GXW19961104/Indoor-Positioning

[https://refined-x.com/2019/03/30/%E5%BE%AE%E4%BF%A1%E5%B0%8F%E7%A8%8B%E5%BA%8FiBeacon%E6%B5%8B%E8%B7%9D%E5%8F%8A%E7%A8%B3%E5%AE%9A%E7%A8%8B%E5%BA%8F%E7%9A%84%E5%AE%9E%E7%8E%B0/](https://refined-x.com/2019/03/30/微信小程序iBeacon测距及稳定程序的实现/)