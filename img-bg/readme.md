# canvas 操作图像记录：自适应+提取背景色+灰度处理

> 用个小 demo 记录一下，如何在 canvas 上操作图像。

- 绘制图像，并自适应水平垂直居中
- 图像灰度处理
- 提取图像的主题色：平均值法（单色背景）、最多色值法（双色背景）

![demo](./img-bg/demo.png)

[点击在线体验](https://esnail.github.io/canvas-demos/img-bg/index.html)

### 绘制图像，并自适应水平垂直居中

#### 绘制图像

利用的是 canvas 的 api [`drawImage`](https://developer.mozilla.org/zh-cn/docs/web/api/canvasrenderingcontext2d/drawimage)

```js
void ctx.drawImage(image, sx, sy, sWidth, sHeight, dx, dy, dWidth, dHeight);
```

参数理解：

![参数理解](./Canvas_drawimage.jpg)

#### 自适应水平垂直居中

在 canvas 上绘制图像，会存在这么几种情况：

- 图像的长宽都比canvas的长宽大
- 图像的长比canvas的大
- 图像的宽比canvas的大
- 图像的长宽都比canvas的小

所以在自适应时需要根据这几种情况分别处理，利用宽高比，并且保证图像的宽高比始终一致。

其次水平垂直居中，利用的是 css 中处理水平垂直居中的方案：

```
top = (box.height - div.height) / 2
left = (box.width - div.width) / 2
```

于是就有了以下的方法：
```js
// 计算图片居中绘制到画布上时 的宽高及起点坐标位置
function calculate(canvasWidth, canvasHeight, imgWidth, imgHeight) {
  let x = 0;
  let y = 0;

  const canvasWHRadio = canvasWidth / canvasHeight
  const imgWHRadio = imgWidth / imgHeight
  
  if (imgWidth < canvasWidth && imgHeight < canvasHeight) {
    x = (canvasWidth - imgWidth) * 0.5
    y = (canvasHeight - imgHeight) * 0.5
  } else if (imgWHRadio > canvasWHRadio) {
    imgHeight = canvasWidth / imgWHRadio
    imgWidth = canvasWidth
    y = (canvasHeight - imgHeight) * 0.5
  } else {
    imgWidth = canvasHeight * imgWHRadio
    imgHeight = canvasHeight
    x = (canvasWidth - imgWidth) * 0.5
  }

  return {
    x,
    y,
    width: imgWidth,
    height: imgHeight
  }
}
```

### 图像数据的处理

要对图像进行处理，比如灰度化，提取颜色等。都是在图像数据上进行处理的。

获取图像数据，要用 canvas 提供的 api [`getImageData`](https://developer.mozilla.org/zh-CN/docs/Web/API/CanvasRenderingContext2D/getImageData)

```
ImageData ctx.getImageData(sx, sy, sw, sh);
```

获取到的数据中，包含获取到的矩形图像的 `width`、`height`、`data`。要处理的就是 data 了。

data 是一个大的类数组，类型是[Uint8ClampedArray（8位无符号整型固定数组）](https://developer.mozilla.org/zh-cn/docs/web/javascript/reference/global_objects/uint8clampedarray)，限定了数组值在[0-255]。其中，每 **4** 位表示一个 rgba 值。分别对应 r(红)、g(绿)、b(蓝)、a(透明度)。


### 图像灰度处理

#### 公式

RGB图转灰度图经典的心理学公式：**Gray = R*0.299 + G*0.587 + B*0.114**

人眼对绿色的敏感度最高，对红色的敏感度次之，对蓝色的敏感度最低，因此使用不同的权重将得到比较合理的灰度图像。

```js
function getGrayColor (r, g, b) {
  // 心理学灰度公式： Gray = R*0.299 + G*0.587 + B*0.114
  // 考虑精度：Gray = (R*299 + G*587 + B*114) / 1000
  // 考虑精度 + 速度：Gray = (R*38 + G*75 + B*15) >> 7
  return (r * 38 + g * 75 + b * 15) >> 7
}
```

公式各种变体参考：[从RGB色转为灰度色算法](https://blog.csdn.net/u013314786/article/details/80543447?utm_medium=distribute.pc_relevant.none-task-blog-baidujs_title-7&spm=1001.2101.3001.4242)

#### 平均值

求出 rgb 的平均值，并把这个平均值赋给 rgb。处理出来的灰度图可能会比较生硬，没有公式法处理出来的灰度图柔和。

```js
function getGrayColorByAvg (r, g, b) {
  // 平均值法
  const avg = (r + g + b) / 3
  return avg
}
```

#### 图像数据绘制到 canvas 上

处理好图像数据了，灰度处理，再将图像数据绘制到 canvas 上，利用的是 [`putImageData`](https://developer.mozilla.org/zh-CN/docs/Web/API/CanvasRenderingContext2D/putImageData)

```js
void ctx.putImageData(imagedata, dx, dy, dirtyX, dirtyY, dirtyWidth, dirtyHeight);
```

#### 灰度处理方法

```js
function imgGray () {
  const {ctx, drawImgW, drawImgH, drawImgX, drawImgY} = global;
  let imgData = ctx.getImageData(drawImgX, drawImgY, drawImgW, drawImgH);
  global.imgData = imgData;

  let copyImgData = new ImageData(new Uint8ClampedArray([...imgData.data]), imgData.width, imgData.height)

  for (let i=0; i<copyImgData.data.length; i+=4) {
    const R = copyImgData.data[i];
    const G = copyImgData.data[i+1];
    const B = copyImgData.data[i+2];
    const gray = getGrayColor(R, G, B)
    copyImgData.data[i] = gray;
    copyImgData.data[i+1] = gray;
    copyImgData.data[i+2] = gray;
  }

  ctx.putImageData(copyImgData, drawImgX, drawImgY);
}
```

### 提取图像的主题色：平均值法（单色背景）、最多色值法（双色背景）

图像的主题色有什么用呢？   
用处之一就是作为图像的背景色，当图像没加载出来之前，可以先用主题色填充。或者让图像的容器填充图像的背景色填补空白部分，让图像观感体验更好。

关于提取图像的主题色，其实是门深奥的技术。    
主要是这么几种：颜色量化算法（中为切分法、八叉树法）、聚类算法、颜色建模。详情可参考[图像主题色提取算法](https://github.com/benhowdle89/grade)。    
这些算法比较复杂，下面介绍的是比较简单粗暴的。

#### rgba 二维数组

```js
const perChunkSize = 4;
const imgRgbaData = Array.from(imgData.data).reduce((rgba, item, index) => {
  const subIndex = Math.floor(index / perChunkSize);
  if (!rgba[subIndex]) {
      rgba[subIndex] = []
  }
  rgba[subIndex].push(item)
  return rgba;
}, [])
```

#### 平均值法（单色背景）

提取图像的主题色，最简单的方法是将图像数据的所有 r、g、b 值加起来，再除以图像的面积，求其平均值。

该方法的缺点在于：无法计算透明背景的主色调，主色调会被png图片透明区域的大小所影响。优点就是简单明了，方便快捷。

主题色求出来了，互补色也比较简单。就是用 255 - 主色调。即用 255 分别减去主色调的 r，g，b 的值分别得到一个新的 r，g，b 的值作为互补色调。

互补色有什么用呢？   
用处之一就是，填充文字的颜色，让文字显示正常。文字的颜色和主题色背景的颜色互斥(互补)时，会比较容易进入眼睛被看到。

```js
function getColorByAvg (imgRgbaData, sizes) {
  // 主色，平均值。将图片每一个像素点的r，g，b通道的值分别累加，然后分别用累加的r，g，b的值除以图片总像素点的个数，分别得到一个平均的r，g，b值并作为图片主色调的rgb值
  const mainColor = {
      r: 0,
      g: 0,
      b: 0
  }
  imgRgbaData.forEach(rgba => {
      mainColor.r += rgba[0]
      mainColor.g += rgba[1]
      mainColor.b += rgba[2]
  })

  const area = sizes.width * sizes.height
  mainColor.r = mainColor.r / area | 0
  mainColor.g = mainColor.g / area | 0
  mainColor.b = mainColor.b / area | 0

  // 互补色，255 - 主色调。用255分别减去主色调的r，g，b的值分别得到一个新的r，g，b的值作为互补色调
  const reverseColor = {
      r: 255 - mainColor.r,
      g: 255 - mainColor.g,
      b: 255 - mainColor.b
  }

  return {
    bgColor: `rgb(${mainColor.r}, ${mainColor.g}, ${mainColor.b})`,
    txtColor: `rgb(${reverseColor.r}, ${reverseColor.g}, ${reverseColor.b})`
  }
}
```

#### 最多色值法（渐变背景）

这种方法比较复杂一些。统计出每种颜色被使用到的次数，再根据次数降序排序，根据灰度值降序排序。取出第1个和第10个最为渐变色。

互补色利用灰度公式，比中间值 125 大的为白色，反之为黑色。

该方案借鉴的是[grade.js](https://benhowdle89.github.io/grade/)

```js
function get2ColorByCount (imgRgbaData) {
  const filterData = imgRgbaData.filter(rgba => rgba.slice(0, 3).every(val => val > 0 && val < 255))
  // 统计每一种颜色的使用次数
  const countData = filterData.reduce((obj, rgba, index) => {
      const key = rgba.join('|')
      obj[key] = obj[key] ? ++(obj[key]) : 1;
      return obj
  }, {});

  let sortData = Object.keys(countData).map(key => {
      const rgba = key.split('|');
      const gray = getGrayColor(rgba[0], rgba[1], rgba[2])
      return {
          rgba,
          count: countData[key],
          gray
      }
  })
  sortData = sortData.sort((a, b) => a.count - b.count).reverse()
  sortData = sortData.slice(0, 10).sort((a, b) => a.brightness - b.brightness).reverse()

  const start = sortData[0].rgba
  const end = sortData[sortData.length - 1].rgba

  const rgb = [(start[0] / 2 + end[0] / 2) | 0, (start[1] / 2 + end[1] / 2) | 0, (start[2] / 2 + end[2] / 2) | 0]
  const color = getGrayColor(rgb[0], rgb[1], rgb[2]) > 255 / 2 ? '#000' : '#fff'

  return {
    bgColor: [`rgb(${start[0]}, ${start[1]}, ${start[2]})`, `rgb(${end[0]}, ${end[1]}, ${end[2]})`],
    txtColor: color
  }
}
```

#### 中位切分法实现

[color-thief](https://lokeshdhakar.com/projects/color-thief/)

不仅提取出了主题色，还提取出了互补色，配色。可以说非常厉害了。

以上就是记录的全部了。