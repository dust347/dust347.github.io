# 道格拉斯算法-go语言实现


<!--more-->
## 1.算法概述
道格拉斯算法是一种曲线的点简化算法。

一般来说，在计算机中表示一条曲线往往使用若干个点来表示，将点连接后形成的折线来趋近想要表示的曲线。点的数量越多，就越贴合想要表示的曲线。

点的数量过多，就需要更多的空间去保存，同时，涉及传输或者计算，如将点的现象传输给前端或者用点生成复杂图像，往往需要更多的时间。

在不影响最终绘制效果的前提下，可以将曲线中的点进行适当删除。这个步骤就是点的简化。而在对精确度要求不高的情况下，可以简化更多。
## 2.算法步骤
算法的基本思路是:
- 假设要简化一条曲线，曲线的两个端点分别是A和B，容忍度为D，D为一个数值（距离），用于判断一个点是否要保留；

- 端点AB连接后得到的直线为AB;

- 遍历曲线上的所有点,计算所有点到AB的距离， 寻找曲线上离AB最远的点C。

- 假设C到AB的距离为dMax，将dMax与D相比;

- 如果dMax <D,则只保留AB两点，这个时候，认为曲线是一条直线;

- 如果dMax ≥D,保留对应的点C,并对AC和CB之间的曲线重复上述流程。


## 3.代码实现
```go
package Util

import (
	"math"
)

/*
* @author dust347
* @brief
*/
//point
type Point struct{
	X float64
	Y float64
}


//计算两点之间距离
func GetDistPt(x1, y1 float64, x2, y2 float64) float64 {
	return math.Sqrt((x1-x2)*(x1-x2) + (y1-y2)*(y1-y2))
}

//计算点到另外两点所确定的直线的距离
func GetDistPt2Line(linePtX1, linePtY1 float64, linePtX2, linePtY2 float64, x, y float64) float64 {
	//如果线的两个点一样，相当于计算两点之间距离
	if (linePtX1 == linePtX2) && (linePtY1 == linePtY2) {
		return GetDistPt(linePtX1, linePtY1, x, y)
	}

	d := (linePtY2*x-linePtY1*x + linePtX1*y-linePtX2*y + linePtX2*linePtY1-linePtX1*linePtY2) / math.Sqrt((linePtY2-linePtY1)*(linePtY2-linePtY1)+(linePtX2-linePtX1)*(linePtX2-linePtX1))
	return math.Abs(d)
}

//道格拉斯抽稀算法
func Douglas(vecPt []Point, tolerance float64) (vecPtRes []Point) {
	l := len(vecPt)
	if l <= 2 {			//小于两个点直接返回
		return vecPt
	}


	//用于标记哪些点需要保留
	m := make(map[int]bool)

	//两端点必保留
	m[0] = true
	m[l-1] = true
	douglas(&vecPt, 0, l-1, tolerance, &m)

	//返回数据
	for i, pt := range vecPt {
		if m[i] {
			vecPtRes = append(vecPtRes, pt)
		}
	}
	return
}

func douglas(vecPt *[]Point, l, r int, tolerance float64, m *map[int]bool) {
	if r - l <= 1 {
		return
	}

	//取两个端点的数据
	x1, y1 := pt2float((*vecPt)[l])
	x2, y2 := pt2float((*vecPt)[r])

	var maxDist float64 = -1		//记录最大距离
	var keepIndex int = -1			//记录要保留的点的index

	for i := l+1; i < r; i++ {
		x, y := pt2float((*vecPt)[i])
		d := GetDistPt2Line(x1, y1, x2, y2, x, y)

		if d > maxDist {
			maxDist = d
			keepIndex = i
		}
	}

	if maxDist > tolerance {
		(*m)[keepIndex] = true
		douglas(vecPt, l, keepIndex, tolerance, m)
		douglas(vecPt, keepIndex, r, tolerance, m)
	}
}

func pt2float(pt Point) (float64, float64) {
	return pt.X, pt.Y
}

```

