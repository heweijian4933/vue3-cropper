# Vue3-cropper

一个年轻的 Vue3 裁剪工具~。

🚀[在线 Demo](https://hubertroy.github.io/)


# 从零开始实现一个图片裁剪工具

cn/user/3403743729289704)

![从零开始实现一个图片裁剪工具](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4c63641c5e5142678488f4cb32424953~tplv-k3u1fbpfcp-zoom-crop-mark:3024:3024:3024:1702.png?)

从零开始的~~异世界~~裁剪工具

# 前言

本篇文章从一个 Canvas 小白的角度来实现一个完整的裁剪工具，实现到最后发现其实并没有涉及到很多 Canvas 内容，更多的还是实现思路。

裁剪工具目前有很多成熟的库，在大部分情况下也够用，不过偶尔也会有一些特定裁剪样式的需求这时候就需要自定义了，掌握一下还是有必要的~。

# 从一个刮刮乐说起

在实现裁剪工具之前先来看一个刮刮乐的实现，实现刮刮乐的思路很简单，底层一张图上面是一个灰色蒙版，鼠标(点击后)经过的地方将部分灰色蒙版去除。

去除用 CSS 其实并不困难，难得是任意坐标下连贯的去除，这点对于 CSS 来说并不好实现。这里会自然而然的想到用 Canvas 去做：

一些前置的 Canvas 理论知识：

```
save和restore用来存储画笔状态和还原画笔状态，因为我们操作的同一支画笔，不存储和还原可能会出现难以调试的BUG。

fillStyle可以设置当前画笔的颜色，支持透明度。

fillRect可以绘制一个矩形，x,y,width,height。

clearRect可以以一个矩形擦除某块区域，x,y,width,height。
```

1. 绘制底图：

```html
<style>
  #canvas {
    background: url(./gouzi.jpeg);
  }
</style>
<canvas id="canvas"></canvas>
<script>
  let canvas = document.querySelector("#canvas");
  let ctx = canvas.getContext("2d");
  canvas.width = 500;
  canvas.height = 500;
</script>
```

1. 绘制一层蒙版：

```javascript
const drawModal = () => {
  ctx.save();
  ctx.fillStyle = "rgba(0,0,0,0.5)";
  ctx.fillRect(0, 0, 500, 500);
  ctx.restore();
};
```

![1.jpg](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1ae13ac772e144fd849813de7a00a2c1~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.png?)

1. 鼠标移动时我们将对应的块擦除：

```javascript
const clearRect = (x, y, w, h) => {
  ctx.save();
  ctx.clearRect(x, y, w, h);
  ctx.restore();
};
canvas.addEventListener("mousemove", (e) => {
  const { offsetX, offsetY } = e;
  clearRect(offsetX, offsetY, 30, 30);
});
```

这样便可以实现一个超简易版的刮刮乐。

![2.jpg](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/34bbac5b40894eedbf4d8729d808199c~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.png?)

## 刮刮乐进阶

蒙版，鼠标擦除我们都实现了，美中不足的是擦除的块只能是以矩形的方式展示，无法支持任意形状。

而 Canvas 本身没有提供其他擦除的方法，一种有趣(和有用)的曲线救国的方式是用[混合](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.mozilla.org%2Fen-US%2Fdocs%2FWeb%2FAPI%2FCanvas_API%2FTutorial%2FCompositing)(CSS 中的 mix-blend-mode 也支持设置混合)，在所有的混合模式中，会发现`destination-out`刚好符合我们的需求，在此种混合模式下，已存在的图像只保留没有重叠的部分，刚好是一个擦除的效果。

写起来并不困难，只需要将上面 3.中的 clearRect 换成 fillRect：

```javascript
const clearRect = (x, y, w, h) => {
  ctx.save();
  ctx.globalCompositeOperation = "destination-out";
  ctx.fillStyle = "rgba(0,0,0)";
  ctx.fillRect(x, y, w, h);
  ctx.restore();
};
```

这里 fillStyle 并不重要，新的图像只会留下形状，不会留下内容。

这样用 fill 实现了一个和 clear 一致的擦除效果，现在只需要将 fillRect 换成 drawImage，就可以实现任意图形的擦除。

```javascript
const clearRect = (x, y, w, h) => {
  ctx.save();
  ctx.globalCompositeOperation = "destination-out";
  ctx.drawImage(img, x, y, w, h);
  ctx.restore();
};
```

![3.jpg](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/91343ec3b99543e19474e1ef36883114~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.png?)

可以看到绘制出了猛犸状的擦除，不要问我为什么选猛犸，因为猛犸它不上**BAN**。

# 实现图片裁剪

## 裁剪工具的实现思路

实现之前整理一波需求：

1. 绘制一层黑白透明底。
2. 用一个 Canvas 来绘制将要裁剪的图片，这里需要注意的点是图片可能会超过画板的大小，需要做一个缩放处理。
3. 用另一个 Canvas 绘制灰色蒙版以及挖洞，这里用一个 Canvas 加载底图和绘制蒙版这些也是可以的，用上面提到的混合，我还是分成两个了更加符合直觉。
4. 用 HTML 来绘制裁剪框的边框以及处理交互事件。

## 黑白透明底

透明的底一般都会用灰白相间的格子表示，可以直接放一张图，不过这并不能凸显出我们的逼格，用 CSS 实现也并不困难，这里用到 background 的相关属性。

我们知道 background 可以写渐变色，而且可以写很多个，background-size 和 background-position 也都可以单独指定每一个 background 的状态。

首先指定两个同样的渐变色：

```css
background: linear-gradient(
    45deg,
    #ccc 25%,
    transparent 25%,
    transparent 75%,
    #ccc 25%
  ), linear-gradient(45deg, #ccc 25%, transparent 25%, transparent 75%, #ccc 25%);
```

`linear-gradient(45deg, #ccc 25%, transparent 25%, transparent 75%, #ccc 25%)`这一段渐变可以生成一个在右上角和左下角是灰色，中间透明的渐变。

将 background 的大小设置为想要的块大小，这里我设置为 8px:

```css
background-size: 8px 8px;
```

因为 background 默认铺不满会重复的特性，会看到一张满是三角的格子。

![4.jpg](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e82ff6ce63bf4b4d9710023bac7f44c7~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.png?)

最后我们将第二个(第一个也行，只要符合预期即可)background 的 x 和 y 都移动`4px`，上一个三角与下一个三角会组合为一整个正方形，不断重复就生成了一个透明格子板。

```css
background-position: 0 0, 4px 4px;
```

![5.jpg](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/56991842f21b4b8babf2ea1eb0aa438b~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.png?)

## 缩放和绘制图片

上面提到绘制图片时我们需要将图片缩放至合适的画板大小，canvas 里并没有类似`object-fit: contain`的方便属性给我们用，只能自己写了，基本思路也很简单，图片的宽和高都必须小于画板的宽高，如果其中有一个不符合就继续缩小，直到都符合：

```javascript
const shrinkImage = ({ imageWidth, imageHeight, width, height, base = 1 }) => {
  // 根据height和width 返回一个合适的 imageWidth和imageHeight。

  if (imageWidth / base < width && imageHeight / base < height) {
    return {
      imageWidth: imageWidth / base,
      imageHeight: imageHeight / base,
    };
  }

  return shrinkImage({
    imageWidth,
    imageHeight,
    width,
    height,
    base: base + 0.1,
  });
};
```

一个简单的尾递归版本，除数的步幅可以看需求增减。

drawImage 有很多参数，这里我们全都用一遍：

```javascript
let baseWidth = 500;
let baseHeight = 500;
let img = new Image();
let canvasOne = document.querySelector("#canvas-one");
let basePaintParams = {
  baseOffsetX: 0,
  baseOffsetY: 0,
  w: 0,
  h: 0,
};
img.src = "./gouzi.jpeg";
img.onload = () => {
  let { imageWidth, imageHeight } = shrinkImage({
    imageWidth: img.width,
    imageHeight: img.height,
    width: baseWidth,
    height: baseHeight,
  });
  ctxOne.drawImage(
    img,
    0,
    0,
    img.width,
    img.height,
    (baseWidth - imageWidth) / 2,
    (baseHeight - imageHeight) / 2,
    imageWidth,
    imageHeight
  );
  basePaintParams = {
    baseOffsetX: (baseWidth - imageWidth) / 2,
    baseOffsetY: (baseHeight - imageHeight) / 2,
    w: imageWidth,
    h: imageHeight,
  };
};
```

drawImage 中，第一个参数是原图像，二三个参数标识原图的剪裁 xy,四五个参数表示原图的剪裁宽高，六七个参数表示绘制到 Canvas 中的 xy 这里用 canvas 的宽高减去图像绘制的宽高再除以 2 得到的 xy 是居中的 xy，八和九则是绘制到 Canvas 中的宽高。

这里声明了一个`basePaintParams`的变量，现在用不到，后面绘制裁剪框时会用到，这里做一个初始化，默认和绘制的图像一致的 xy 宽高。

![6.jpg](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4d0bfa712d7e4aa583a53a13c48038d2~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.png?)

MDN 参考：[drawImage](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.mozilla.org%2Fzh-CN%2Fdocs%2FWeb%2FAPI%2FCanvasRenderingContext2D%2FdrawImage)

## 绘制蒙版和挖空

蒙版与挖空的具体实现可以用刮刮乐里的实现，这里实现一个矩形裁剪框，混合和 clearRect 都可以。

蒙版：

```javascript
let canvasTwo = document.querySelector("#canvas-two");

const paintModal = () => {
  ctxTwo.clearRect(0, 0, baseWidth, baseHeight);
  ctxTwo.fillStyle = "rgba(0,0,0,0.5)";
  ctxTwo.fillRect(0, 0, baseWidth, baseHeight);
};

paintModal();
```

![7.jpg](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fceabaacb25e4d1e916ef9cfeea7f2ab~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.png?)

挖空：

```javascript
const clipModal = () => {
  ctxTwo.save();
  ctxTwo.clearRect(
    basePaintParams.baseOffsetX,
    basePaintParams.baseOffsetY,
    basePaintParams.w,
    basePaintParams.h
  );
  ctxTwo.restore();
};

// 要等到img onload之后再绘制，或者提供一个初始化的参数。
clipModal();
```

![8.jpg](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5c1f4d5c09e941728a2470ee2ae41ec5~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.png?)

这里用 HTML+CSS 实现起来也并不困难，不必非要用 Canvas。

## 绘制裁剪框

具体的裁剪框我们用 HTML 来实现，绑定事件这些还是 HTML 来的舒服，常规形状的裁剪框 HTML 可以很轻松的编写，奇奇怪怪的形状也可以用`clip-path`来实现，暂时先不趟 Canvas 的浑水了。

```html
<style>
  .cropper-clip {
    position: absolute;
    cursor: all-scroll;
    border: 1px solid rgb(30, 158, 251);
  }
  .dot {
    width: 10px;
    height: 10px;
    border-radius: 50%;
    background: #1e9efb;
    position: absolute;
  }

  .topleft {
    top: -5px;
    left: -5px;
    cursor: nwse-resize;
  }

  .topright {
    top: -5px;
    right: -5px;
    cursor: nesw-resize;
  }

  .topcenter {
    top: -5px;
    left: 50%;
    transform: translateX(-50%);
    cursor: ns-resize;
  }

  .bottomcenter {
    bottom: -5px;
    left: 50%;
    transform: translateX(-50%);
    cursor: ns-resize;
  }

  .bottomleft {
    bottom: -5px;
    left: -5px;
    cursor: nesw-resize;
  }

  .bottomright {
    bottom: -5px;
    right: -5px;
    cursor: nwse-resize;
  }

  .leftcenter {
    top: 50%;
    transform: translateY(-50%);
    left: -5px;
    cursor: ew-resize;
  }

  .rightcenter {
    top: 50%;
    transform: translateY(-50%);
    right: -5px;
    cursor: ew-resize;
  }
</style>

<div class="cropper-clip">
  <div class="dot topleft"></div>
  <div class="dot topright"></div>
  <div class="dot topcenter"></div>
  <div class="dot bottomleft"></div>
  <div class="dot bottomcenter"></div>
  <div class="dot bottomright"></div>
  <div class="dot leftcenter"></div>
  <div class="dot rightcenter"></div>
</div>
```

一组很容易写的框，一共八个点，分布在四周，框的位置和大小在设置蒙版和挖空时一起给上：

```javascript
const drawClipDiv = () => {
  let cropperClip = document.querySelector(".cropper-clip");
  cropperClip.style.width = `${Math.abs(basePaintParams.w)}px`;
  cropperClip.style.height = `${Math.abs(basePaintParams.h)}px`;
  cropperClip.style.left = `${basePaintParams.baseOffsetX}px`;
  cropperClip.style.top = `${basePaintParams.baseOffsetY}px`;
};
```

`Math.abs`用来适配改变裁剪框大小时越过线的情况，如果不做可以不加(本文没有实现这个)。

现在已经将裁剪的骨骼描绘完整了，只要后续改变`basePaintParams`中的坐标和大小参数，重新绘制蒙版挖空以及裁剪框即可。

![9.jpg](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9159f12641a74efea37ac31279e2e216~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.png?)

## 裁剪框的移动与伸缩

上面我们已经绘制出了基本骨骼，最后只要让裁剪框可以跟随鼠标动起来就可以了。

首先我们现在的 HTML 结构是这样的:

```HTML
<div class="cropper">
  <div class="cropper-clip">
    <div class="dot topleft"></div>
    <div class="dot topright"></div>
    <div class="dot topcenter"></div>
    <div class="dot bottomleft"></div>
    <div class="dot bottomcenter"></div>
    <div class="dot bottomright"></div>
    <div class="dot leftcenter"></div>
    <div class="dot rightcenter"></div>
  </div>
</div>
```

我们给`cropper`注册一个`mousemove`事件，这样比较容易处理。

裁剪框体和保存某个点按下，需要额外的变量标识。

```javascript
let dotType = "";
let dotDown = false;
let clipDown = false;

const registerEvents = () => {
  const register = (_class) => {
    let node = document.querySelector(`.${_class}`);
    node.addEventListener("mousedown", (e) => {
      dotDown = true;
      dotType = _class;
    });

    node.addEventListener("mouseup", () => {
      dotDown = false;
    });
  };

  register("topcenter");
  register("bottomcenter");
  register("leftcenter");
  register("rightcenter");
  register("bottomright");
  register("bottomleft");
  register("topright");
  register("topleft");

  let clip = document.querySelector(".cropper-clip");
  clip.addEventListener("mousedown", () => {
    clipDown = true;
  });

  clip.addEventListener("mouseup", () => {
    clipDown = false;
  });
};
```

核心`mousemove`的事件整体思路从一条边出发，只要确定了上下和左右的运动策略，其他点进行组合即可：

```javascript
const cardMouseMove = (e) => {
  if (!dotDown && !clipDown) {
    return;
  }

  const { offsetX, offsetY } = e;

  const topYChange = () => {
    if (e.target === canvasTwo) {
      if (offsetY < basePaintParams.baseOffsetY) {
        basePaintParams.h += basePaintParams.baseOffsetY - offsetY;
        basePaintParams.baseOffsetY = offsetY;
      }
    } else if (e.target === clip) {
      basePaintParams.h -= offsetY;
      basePaintParams.baseOffsetY += offsetY;
    }
  };

  const bottomYChange = () => {
    if (e.target === canvasTwo) {
      if (offsetY > basePaintParams.baseOffsetY) {
        basePaintParams.h = offsetY - basePaintParams.baseOffsetY;
      }
    } else if (e.target === clip) {
      basePaintParams.h = offsetY;
    }
  };

  const leftXChange = () => {
    if (e.target === canvasTwo) {
      if (offsetX < basePaintParams.baseOffsetX) {
        basePaintParams.w += basePaintParams.baseOffsetX - offsetX;
        basePaintParams.baseOffsetX = offsetX;
      }
    } else if (e.target === clip) {
      basePaintParams.w -= offsetX;
      basePaintParams.baseOffsetX += offsetX;
    }
  };

  const rightXChange = () => {
    if (e.target === canvasTwo) {
      if (offsetX > basePaintParams.baseOffsetX) {
        basePaintParams.w = offsetX - basePaintParams.baseOffsetX;
      }
    } else if (e.target === clip) {
      basePaintParams.w = offsetX;
    }
  };
};
```

这里需要注意的是，我们注册的事件在父节点上，所以事件的 target 有可能是我们绘制的蒙版(在放大的时候)，也有可能是裁剪框(缩小的时候)，这时候的`offset`会有所不同。

之后我们只需要根据 8 个点所期望改变的 xy 调用不同的方法即可:

```javascript
const dotRunMap = {
  topcenter: () => {
    topYChange();
  },
  bottomcenter: () => {
    bottomYChange();
  },
  leftcenter: () => {
    leftXChange();
  },
  rightcenter: () => {
    rightXChange();
  },
  bottomright: () => {
    rightXChange();
    bottomYChange();
  },
  bottomleft: () => {
    leftXChange();
    bottomYChange();
  },
  topright: () => {
    rightXChange();
    topYChange();
  },
  topleft: () => {
    leftXChange();
    topYChange();
  },
};

if (dotDown) {
  dotRunMap[dotType]();
}

// 如果没有点按下，说明是在移动，这里没有做边缘限制，会出圈。
if (clipDown && !dotDown) {
  basePaintParams.baseOffsetX = basePaintParams.baseOffsetX + e.movementX;
  basePaintParams.baseOffsetY = basePaintParams.baseOffsetY + e.movementY;
}

drawClipDiv();
paintModal();
clipModal();
```

不要忘了重新绘制裁剪框，蒙版和挖空。

![1.gif](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1536f0279e8f4238b5479456bd63fc20~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.png?)

## 预览与保存

预览需要用到另一个 Canvas 来接盘，这里主要用到[getImageData](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.mozilla.org%2Fzh-CN%2Fdocs%2FWeb%2FAPI%2FCanvasRenderingContext2D%2FgetImageData)和[putImageData](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.mozilla.org%2Fzh-CN%2Fdocs%2FWeb%2FAPI%2FCanvasRenderingContext2D%2FputImageData)，参数大部分都一样，需要注意的坑点是如果是个跨域图片，Canvas 默认会认为受到污染，这时需要设置一下原来 image 请求时的跨域设置，更具体可以参考一下[这篇文章](https://juejin.cn/post/6844903765422653454)。

```javascript
<canvas id="clipImage" />;
const save = () => {
  let bgCtx = canvasOne;

  let imageData = bgCtx.getImageData(
    basePaintParams.w < 0
      ? basePaintParams.baseOffsetX + basePaintParams.w
      : basePaintParams.baseOffsetX,
    basePaintParams.h < 0
      ? basePaintParams.baseOffsetY + basePaintParams.h
      : basePaintParams.baseOffsetY,
    Math.abs(basePaintParams.w),
    Math.abs(basePaintParams.h)
  );

  let targetCanvas = clipImage.getContext("2d");
  clipImage.width = basePaintParams.w;
  clipImage.height = basePaintParams.h;
  targetCanvas.putImageData(
    imageData,
    0,
    0,
    0,
    0,
    basePaintParams.w,
    basePaintParams.h
  );
};
```

保存上是一个比较常规的做法，把 Canvas 先转成 dataURL 的 base64 形式，然后再转换为 Blob，下载这个 Blob 即可。

```javascript
const dataURItoBlob = async (url) => await (await fetch(url)).blob();
const saveData = (function () {
  const a = document.createElement("a");
  document.body.appendChild(a);
  return function (blob, fileName) {
    let url = window.URL.createObjectURL(blob);
    a.href = url;
    a.download = fileName;
    a.click();
    window.URL.revokeObjectURL(url);
  };
})();

const exportImg = async () => {
  let MIME_TYPE = "image/png";
  let imgURL = clipImage.toDataURL(MIME_TYPE);
  let res = await dataURItoBlob(imgURL);
  saveData(res, "data.png");
};
```

# 总结

这样一个基础功能完备的裁剪工具就完成啦，核心裁剪部分代码并不多，用到的 Canvas 内容也不多，整体很适合来熟悉一波 Canvas，有了骨骼之后其他的功能都可以沿着这个脉络进行扩展。

这里封装了一个 Vue3 的版本，有需要可以在[Github](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2FHuberTRoy%2Fvue3-cropper)上找到~。
