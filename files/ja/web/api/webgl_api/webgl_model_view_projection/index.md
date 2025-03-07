---
title: WebGL モデル ビュー 射影
slug: Web/API/WebGL_API/WebGL_model_view_projection
---

{{DefaultAPISidebar("WebGL")}}

この記事では、[WebGL](/ja/docs/Web/API/WebGL_API) プロジェクト内でデータを取得し、それを適切な空間に投影して画面に表示する方法について説明します。並進、拡縮、回転行列を使用した基本的な行列計算の知識があることを前提としています。3D シーンを構成するときに通常使用される中心的な 3 つの行列である、モデル、ビュー、射影行列について説明します。

> **メモ:** This article is also available as an [MDN content kit](https://github.com/gregtatum/mdn-model-view-projection). It also uses a collection of [utility functions](https://github.com/gregtatum/mdn-webgl) available under the `MDN` global object.

## モデル、ビュー、射影行列

WebGL の空間内の点とポリゴンの個々の変換は、並進、拡縮、回転などの基本的な変換行列によって処理されます。 これらの行列は、複雑な 3D シーンの描画に役立つように、一緒に構成し、特別な方法でグループ化できます。これらの構成された行列は、最終的に元のモデルデータを**クリップ空間**と呼ばれる特別な座標空間に移動します。これは 2 ユニットの幅の立方体で、中心が (0,0,0) 、対角が (-1,-1,-1) から (1,1,1) になります。このクリップ空間は 2 次元平面に圧縮され、画像へラスタライズされます。

以下で説明する最初の行列は**モデル行列**です。これは、元のモデルデータを取得して 3 次元ワールド空間内で移動する方法を定義します。 **射影行列**は、ワールド空間座標をクリップ空間座標に変換するために使用されます。 一般的に使用される射影行列である**透視投影射影行列**は、3D 仮想世界の視聴者の代理として機能する一般的なカメラの*効果*を模倣するために使用されます。 **ビュー行列**は、変更されるカメラの位置をシミュレートし、シーン内のオブジェクトを移動して視聴者が現在何を見られるかを変更します。

以下のセクションでは、モデル、ビュー、射影行列の背景にある考え方と実装について詳説します。 これらの行列は、画面上でデータを移動するための根幹であり、個々のフレームワークやエンジンを超える概念です。

## クリップ空間

WebGL プログラムでは、通常、データは自分の座標系で GPU にアップロードされ、次に頂点シェーダーがそれらの点を**クリップ空間**と呼ばれる特別な座標系に変換します。クリップ空間の外側にあるデータは切り取られ、描画されません。ただし、三角形がこのスペースの境界を跨ぐ場合は、新しい三角形に分割され、クリップスペースにある新しい三角形の部分のみが残ります。

![A 3d graph showing clip space in WebGL.](clip_space_graph.svg)

上の図は、全ての点が収まる必要のあるクリップ空間を視覚化したものです。これは、各辺が 2 の立方体であり、片方の角が (-1,-1,-1) にあり、対角が (1,1,1) にあります。立方体の中心は点 (0,0,0) です。 クリップ空間に使用されるこの 8 立方メートルの座標系は、正規化デバイス座標（NDC）と呼ばれます。WebGL コードを調べて作業している間、その用語を時々耳にするかもしれません。

このセクションでは、データをクリップ空間座標系に直接配置する仕組みを説明します。 通常、任意の座標系にあるモデルデータが使用され、その後、行列を使用して変換され、モデル座標がクリップ空間座標系に変換されます。この例では、クリップ空間がどのように機能するかを最も簡単に説明する為、単純に (-1, -1, -1) から (1,1,1) までの範囲のモデル座標を使用します。以下のコードは、画面上に正方形を描く為に 2 つの三角形を作成します。正方形の Z 深度は、正方形が同じ空間を共有するときに何が上に描画されるかを決定します。小さい Z 値は大きい Z 値の上にレンダリングされます。

### WebGLBox の例

この例では、画面上に 2D ボックスを描画するカスタム `WebGLBox` オブジェクトを作成します。

> **メモ:** The code for each WebGLBox example is available in this [github repo](https://github.com/gregtatum/mdn-model-view-projection/tree/master/lessons) and is organized by section. In addition there is a JSFiddle link at the bottom of each section.

#### WebGLBox コンストラクタ

コンストラクターは次のようになります。

```js
function WebGLBox() {
  // Setup the canvas and WebGL context
  this.canvas = document.getElementById("canvas");
  this.canvas.width = window.innerWidth;
  this.canvas.height = window.innerHeight;
  this.gl = MDN.createContext(canvas);

  var gl = this.gl;

  // Setup a WebGL program, anything part of the MDN object is defined outside of this article
  this.webglProgram = MDN.createWebGLProgramFromIds(
    gl,
    "vertex-shader",
    "fragment-shader",
  );
  gl.useProgram(this.webglProgram);

  // Save the attribute and uniform locations
  this.positionLocation = gl.getAttribLocation(this.webglProgram, "position");
  this.colorLocation = gl.getUniformLocation(this.webglProgram, "color");

  // Tell WebGL to test the depth when drawing, so if a square is behind
  // another square it won't be drawn
  gl.enable(gl.DEPTH_TEST);
}
```

#### WebGLBox 描画

次に、画面上にボックスを描画するメソッドを作成します。

```js
WebGLBox.prototype.draw = function (settings) {
  // Create some attribute data; these are the triangles that will end being
  // drawn to the screen. There are two that form a square.

  var data = new Float32Array([
    //Triangle 1
    settings.left,
    settings.bottom,
    settings.depth,
    settings.right,
    settings.bottom,
    settings.depth,
    settings.left,
    settings.top,
    settings.depth,

    //Triangle 2
    settings.left,
    settings.top,
    settings.depth,
    settings.right,
    settings.bottom,
    settings.depth,
    settings.right,
    settings.top,
    settings.depth,
  ]);

  // Use WebGL to draw this onto the screen.

  // Performance Note: Creating a new array buffer for every draw call is slow.
  // This function is for illustration purposes only.

  var gl = this.gl;

  // Create a buffer and bind the data
  var buffer = gl.createBuffer();
  gl.bindBuffer(gl.ARRAY_BUFFER, buffer);
  gl.bufferData(gl.ARRAY_BUFFER, data, gl.STATIC_DRAW);

  // Setup the pointer to our attribute data (the triangles)
  gl.enableVertexAttribArray(this.positionLocation);
  gl.vertexAttribPointer(this.positionLocation, 3, gl.FLOAT, false, 0, 0);

  // Setup the color uniform that will be shared across all triangles
  gl.uniform4fv(this.colorLocation, settings.color);

  // Draw the triangles to the screen
  gl.drawArrays(gl.TRIANGLES, 0, 6);
};
```

シェーダーは GLSL で記述されたコードの一部であり、データポイントを取得して最終的に画面に描画します。便宜上、これらのシェーダーは、カスタム関数 `MDN.createWebGLProgramFromIds()` を介してプログラムに取り込まれる要素 {{htmlelement("script")}} に格納されます。この関数は、これらのチュートリアル用に作成された [ユーティリティ関数群](https://github.com/gregtatum/mdn-webgl) の一部であり、ここでは詳しく説明しません。この関数は、いくつかの GLSL ソースコードを取得して WebGL プログラムにコンパイルする基本を処理します。関数は 3 つのパラメーターを取ります。プログラムをレンダリングするコンテキスト、頂点シェーダーを含む要素の ID {{htmlelement("script")}}、フラグメントシェーダーを含む要素の ID {{htmlelement("script")}} です。頂点シェーダーは頂点を配置し、フラグメントシェーダーは各ピクセルに色を付けます。

最初に、画面上で頂点を移動させる頂点シェーダーを見てみましょう。

```glsl
// The individual position vertex
attribute vec3 position;

void main() {
  // the gl_Position is the final position in clip space after the vertex shader modifies it
  gl_Position = vec4(position, 1.0);
}
```

次に、データを実際にピクセルにラスタライズするために、フラグメントシェーダーはピクセルごとに全てを評価し、一つの色を設定します。GPU は、レンダリングする必要があるピクセルごとにシェーダー関数を呼び出します。シェーダーの仕事は、そのピクセルに使用する色を返すことです。

```glsl
precision mediump float;
uniform vec4 color;

void main() {
  gl_FragColor = color;
}
```

これらの設定が含まれているので、クリップ空間座標を使用して画面に直接描画します。

```js
var box = new WebGLBox();
```

最初に中央に赤いボックスを描きます。

```js
box.draw({
  top: 0.5, // x
  bottom: -0.5, // x
  left: -0.5, // y
  right: 0.5, // y

  depth: 0, // z
  color: [1, 0.4, 0.4, 1], // red
});
```

次に、緑色のボックスを赤いボックスの上部に描画します。

```js
box.draw({
  top: 0.9, // x
  bottom: 0, // x
  left: -0.9, // y
  right: 0.9, // y

  depth: 0.5, // z
  color: [0.4, 1, 0.4, 1], // green
});
```

最後に、クリッピングが実際に行われていることを示すために、このボックスは完全にクリップ空間の外側にあるため、描画されません。深さが -1.0 から 1.0 の範囲外です。

```js
box.draw({
  top: 1, // x
  bottom: -1, // x
  left: -1, // y
  right: 1, // y

  depth: -1.5, // z
  color: [0.4, 0.4, 1, 1], // blue
});
```

#### 結果

[JSFiddle で表示](https://jsfiddle.net/mff99yu5)

![The results of drawing to clip space using WebGL.](part1.png)

#### 演習

この時点で役立つ演習は、コードを変更してボックスをクリップ空間内で移動し、点がクリップ空間内でどのようにクリップされ、移動されるかを感じ取ることです。背景を持つボックス状のスマイルのような絵を描いてみてください。

## 斉次座標

前のクリップ空間の頂点シェーダのメインラインは、このコードを含んでいました。

```js
gl_Position = vec4(position, 1.0);
```

変数 `position` は、 `draw()` メソッドで定義され、属性としてシェーダーに渡されました。これは 3 次元の点ですが、パイプラインを介して渡されることになる変数 `gl_Position` は実際には 4 次元です。すなわち、 `(x, y, z)` の代わりに `(x, y, z, w)` となっています。`z` の後には文字がないため、慣例により、この 4 番目の次元には `w` というラベルが付いています。 上記の例では、 `w` 座標は 1.0 に設定されています。

明らかな疑問は、「なぜ余分な次元があるのか？」です。この追加により、3D データを操作するための多くの優れた手法が可能になることが分かります。この追加された次元により、遠近法の概念が座標系に導入されます。それを配置すると、3D 座標を 2D 空間にマッピングできます。これにより、2 本の平行線が遠くに離れるときに交差するようになります。値 `w` は、座標の他のコンポーネントの除数として使用されるため、 `x`、`y`、`z`の真の値は、`x/w`、`y/w`、`z/w`として計算されます（そして、`w`も `w/w`で 1 になる）。

A three dimensional point is defined in a typical Cartesian coordinate system. The added fourth dimension changes this point into a [homogeneous coordinate](https://en.wikipedia.org/wiki/homogeneous_coordinates). It still represents a point in 3D space and it can easily be demonstrated how to construct this type of coordinate through a pair of simple functions.

```js
function cartesianToHomogeneous(point)
  let x = point[0];
  let y = point[1];
  let z = point[2];

  return [x, y, z, 1];
}

function homogeneousToCartesian(point) {
  let x = point[0];
  let y = point[1];
  let z = point[2];
  let w = point[3];

  return [x/w, y/w, z/w];
}
```

As previously mentioned and shown in the functions above, the w component divides the x, y, and z components. When the w component is a non-zero real number then homogeneous coordinate easily translates back into a normal point in Cartesian space. Now what happens if the w component is zero? In JavaScript the value returned would be as follows.

```js
homogeneousToCartesian([10, 4, 5, 0]);
```

This evaluates to: `[Infinity, Infinity, Infinity]`.

This homogeneous coordinate represents some point at infinity. This is a handy way to represent a ray shooting off from the origin in a specific direction. In addition to a ray, it could also be thought of as a representation of a directional vector. If this homogeneous coordinate is multiplied against a matrix with a translation then the translation is effectively stripped out.

When numbers are extremely large (or extremely small) on computers they begin to become less and less precise because there are only so many ones and zeros that are used to represent them. The more operations that are done on larger numbers, the more and more errors accumulate into the result. When dividing by w, this can effectively increase the precision of very large numbers by operating on two potentially smaller, less error-prone numbers.

The final benefit of using homogeneous coordinates is that they fit very nicely for multiplying against 4x4 matrices. A vertex must match at least one of the dimensions of a matrix in order to be multiplied against it. The 4x4 matrix can be used to encode a variety of useful transformations. In fact, the typical perspective projection matrix uses the division by the w component to achieve its transformation.

The clipping of points and polygons from clip space actually happens after the homogeneous coordinates have been transformed back into Cartesian coordinates (by dividing by w). This final space is known as **normalized device coordinates** or NDC.

To start playing with this idea the previous example can be modified to allow for the use of the `w` component.

```js
//Redefine the triangles to use the W component
var data = new Float32Array([
  //Triangle 1
  settings.left,
  settings.bottom,
  settings.depth,
  settings.w,
  settings.right,
  settings.bottom,
  settings.depth,
  settings.w,
  settings.left,
  settings.top,
  settings.depth,
  settings.w,

  //Triangle 2
  settings.left,
  settings.top,
  settings.depth,
  settings.w,
  settings.right,
  settings.bottom,
  settings.depth,
  settings.w,
  settings.right,
  settings.top,
  settings.depth,
  settings.w,
]);
```

Then the vertex shader uses the 4 dimensional point passed in.

```js
attribute vec4 position;

void main() {
  gl_Position = position;
}
```

First, we draw a red box in the middle, but set W to 0.7. As the coordinates get divided by 0.7 they will all be enlarged.

```js
box.draw({
  top: 0.5, // y
  bottom: -0.5, // y
  left: -0.5, // x
  right: 0.5, // x
  w: 0.7, // w - enlarge this box

  depth: 0, // z
  color: [1, 0.4, 0.4, 1], // red
});
```

Now, we draw a green box up top, but shrink it by setting the w component to 1.1

```js
box.draw({
  top: 0.9, // y
  bottom: 0, // y
  left: -0.9, // x
  right: 0.9, // x
  w: 1.1, // w - shrink this box

  depth: 0.5, // z
  color: [0.4, 1, 0.4, 1], // green
});
```

This last box doesn't get drawn because it's outside of clip space. The depth is outside of the -1.0 to 1.0 range.

```js
box.draw({
  top: 1, // y
  bottom: -1, // y
  left: -1, // x
  right: 1, // x
  w: 1.5, // w - Bring this box into range

  depth: -1.5, // z
  color: [0.4, 0.4, 1, 1], // blue
});
```

### The results

[View on JSFiddle](https://jsfiddle.net/mff99yu)

![The results of using homogeneous coordinates to move the boxes around in WebGL.](part2.png)

### Exercises

- Play around with these values to see how it affects what is rendered on the screen. Note how the previously clipped blue box is brought back into range by setting its w component.
- Try creating a new box that is outside of clip space and bring it back in by dividing by w.

## Model transform

Placing points directly into clip space is of limited use. In real world applications, you don't have all your source coordinates already in clip space coordinates. So most of the time, you need to transform the model data and other coordinates into clip space. The humble cube is an easy example of how to do this. Cube data consists of vertex positions, the colors of the faces of the cube, and the order of the vertex positions that make up the individual polygons (in groups of 3 vertices to construct the triangles composing the cube's faces). The positions and colors are stored in GL buffers, sent to the shader as attributes, and then operated upon individually.

Finally a single model matrix is computed and set. This matrix represents the transformations to be performed on every point making up the model in order to move it into the correct space, and to perform any other needed transforms on each point in the model. This applies not just to each vertex, but to every single point on every surface of the model as well.

In this case, for every frame of the animation a series of scale, rotation, and translation matrices move the data into the desired spot in clip space. The cube is the size of clip space (-1,-1,-1) to (1,1,1) so it will need to be shrunk down in order to not fill the entirety of clip space. This matrix is sent directly to the shader, having been multiplied in JavaScript beforehand.

The following code sample defines a method on the `CubeDemo` object that will create the model matrix. It uses custom functions to create and multiply matrices as defined in the [MDN WebGL](https://github.com/gregtatum/mdn-webgl) shared code. The new function looks like this:

```js
CubeDemo.prototype.computeModelMatrix = function (now) {
  //Scale down by 50%
  var scale = MDN.scaleMatrix(0.5, 0.5, 0.5);

  // Rotate a slight tilt
  var rotateX = MDN.rotateXMatrix(now * 0.0003);

  // Rotate according to time
  var rotateY = MDN.rotateYMatrix(now * 0.0005);

  // Move slightly down
  var position = MDN.translateMatrix(0, -0.1, 0);

  // Multiply together, make sure and read them in opposite order
  this.transforms.model = MDN.multiplyArrayOfMatrices([
    position, // step 4
    rotateY, // step 3
    rotateX, // step 2
    scale, // step 1
  ]);
};
```

In order to use this in the shader it must be set to a uniform location. The locations for the uniforms are saved in the `locations` object shown below:

```js
this.locations.model = gl.getUniformLocation(webglProgram, "model");
```

And finally the uniform is set to that location. This hands off the matrix to the GPU.

```js
gl.uniformMatrix4fv(
  this.locations.model,
  false,
  new Float32Array(this.transforms.model),
);
```

In the shader, each position vertex is first transformed into a homogeneous coordinate (a `vec4` object), and then multiplied against the model matrix.

```glsl
gl_Position = model * vec4(position, 1.0);
```

> **メモ:** In JavaScript, matrix multiplication requires a custom function, while in the shader it is built into the language with the simple \* operator.

### The results

[View on JSFiddle](https://jsfiddle.net/5jofzgsh)

![Using a model matrix](part3.png)

At this point the w value of the transformed point is still 1.0. The cube still doesn't have any perspective. The next section will take this setup and modify the w values to provide some perspective.

### Exercises

- Shrink down the box using the scale matrix and position it in different places within clip space.
- Try moving it outside of clip space.
- Resize the window and watch as the box skews out of shape.
- Add a `rotateZ` matrix.

## Divide by W

An easy way to start getting some perspective on our model of the cube is to take the Z coordinate and copy it over to the w coordinate. Normally when converting a cartesian point to homogeneous it becomes `(x,y,z,1)`, but we're going to set it to something like `(x,y,z,z)`. In reality we want to make sure that z is greater than 0 for points in view, so we'll modify it slightly by changing the value to `((1.0 + z) * scaleFactor)`. This will take a point that is normally in clip space (-1 to 1) and move it into a space more like (0 to 1) depending on what the scale factor is set to. The scale factor changes the final w value to be either higher or lower overall.

The shader code looks like this.

```js
// First transform the point
vec4 transformedPosition = model * vec4(position, 1.0);

// How much effect does the perspective have?
float scaleFactor = 0.5;

// Set w by taking the z value which is typically ranged -1 to 1, then scale
// it to be from 0 to some number, in this case 0-1.
float w = (1.0 + transformedPosition.z) * scaleFactor;

// Save the new gl_Position with the custom w component
gl_Position = vec4(transformedPosition.xyz, w);
```

### The results

[View on JSFiddle](https://jsfiddle.net/vk9r8h2c)

![Filling the W component and creating some projection.](part4.png)

See that small dark blue triangle? That's an additional face added to our object because the rotation of our shape has caused that corner to extend outside clip space, thus causing the corner to be clipped away. See [Perspective projection matrix](#perspective_projection_matrix) below for an introduction to how to use more complex matrices to help control and prevent clipping.

### Exercise

If that sounds a little abstract, open up the vertex shader and play around with the scale factor and watch how it shrinks vertices more towards the surface. Completely change the w component values for really trippy representations of space.

In the next section we'll take this step of copying Z into the w slot and turn it into a matrix.

## Simple projection

The last step of filling in the w component can actually be accomplished with a simple matrix. Start with the identity matrix:

```js
var identity = [1, 0, 0, 0, 0, 1, 0, 0, 0, 0, 1, 0, 0, 0, 0, 1];

MDN.multiplyPoint(identity, [2, 3, 4, 1]);
//> [2, 3, 4, 1]
```

Then move the last column's 1 up one space.

```js
var copyZ = [1, 0, 0, 0, 0, 1, 0, 0, 0, 0, 1, 1, 0, 0, 0, 0];

MDN.multiplyPoint(copyZ, [2, 3, 4, 1]);
//> [2, 3, 4, 4]
```

However in the last example we performed `(z + 1) * scaleFactor`:

```
var scaleFactor = 0.5;

var simpleProjection = [
  1, 0, 0, 0,
  0, 1, 0, 0,
  0, 0, 1, scaleFactor,
  0, 0, 0, scaleFactor,
];

MDN.multiplyPoint(simpleProjection, [2, 3, 4, 1]);
//> [2, 3, 4, 2.5]
```

Breaking it out a little further we can see how this works:

```js
var x = 2 * 1 + 3 * 0 + 4 * 0 + 1 * 0;
var y = 2 * 0 + 3 * 1 + 4 * 0 + 1 * 0;
var z = 2 * 0 + 3 * 0 + 4 * 1 + 1 * 0;
var w = 2 * 0 + 3 * 0 + 4 * scaleFactor + 1 * scaleFactor;
```

The last line could be simplified to:

```js
w = 4 * scaleFactor + 1 * scaleFactor;
```

Then factoring out the scaleFactor, we get this:

```js
w = (4 + 1) * scaleFactor;
```

Which is exactly the same as the `(z + 1) * scaleFactor` that we used in the previous example.

In the box demo, an additional `computeSimpleProjectionMatrix()` method is added. This is called in the `draw()` method and has the scale factor passed to it. The result should be identical to the last example:

```js
CubeDemo.prototype.computeSimpleProjectionMatrix = function (scaleFactor) {
  this.transforms.projection = [
    1,
    0,
    0,
    0,
    0,
    1,
    0,
    0,
    0,
    0,
    1,
    scaleFactor,
    0,
    0,
    0,
    scaleFactor,
  ];
};
```

Although the result is identical, the important step here is in the vertex shader. Rather than modifying the vertex directly, it gets multiplied by an additional **[projection matrix](#projection_matrix)**, which (as the name suggests) projects 3D points onto a 2D drawing surface:

```glsl
// Make sure to read the transformations in reverse order
gl_Position = projection * model * vec4(position, 1.0);
```

### The results

[View on JSFiddle](https://jsfiddle.net/zwyLLcbw)

![A simple projection matrix](part5.png)

## The viewing frustum

Before we move on to covering how to compute a perspective projection matrix, we need to introduce the concept of the **[viewing frustum](https://en.wikipedia.org/wiki/viewing_frustum)** (also known as the **view frustum**). This is the region of space whose contents are visible to the user at the current time. It's the 3D region of space defined by the field of view and the distances specified as the nearest and farthest content that should be rendered.

While rendering, we need to determine which polygons need to be rendered in order to represent the scene. This is what the viewing frustum defines. But what's a frustum in the first place?

A [frustum](https://en.wikipedia.org/wiki/frustum) is the 3D solid that results from taking any solid and slicing off two sections of it using two parallel planes. Consider our camera, which is viewing an area that starts immediately in front of its lens and extends off into the distance. The viewable area is a four-sided pyramid with its peak at the lens, its four sides corresponding to the extents of its peripheral vision range, and its base at the farthest distance it can see, like this:

![A depiction of the entire viewing area of a camera. This area is a four-sided pyramid with its peak at the lens and its base at the world's maximum viewable distance.](fullcamerafov.svg)

If we simply used this to determine the polygons to be rendered each frame, our renderer would need to render every polygon within this pyramid, all the way off into infinity, including also polygons that are very close to the lens—likely too close to be useful (and certainly including things that are so close that a real human wouldn't be able to focus on them in the same setting).

So the first step in reducing the number of polygons we need to compute and render, we turn this pyramid into the viewing frustum. The two planes we'll use to chop away vertices in order to reduce the polygon count are the **near clipping plane** and the **far clipping plane**.

In WebXR, the near and far clipping planes are defined by specifying the distance from the lens to the closest point on a plane which is perpendicular to the viewing direction. Anything closer to the lens than the near clipping plane or farther from it than the far clipping plane is removed. This results in the viewing frustum, which looks like this:

![A depiction of the camera's view frustum; the near and far planes have removed part of the volume, reducing the polygon count.](camera_view_frustum.svg)

The set of objects to be rendered for each frame is essentially created by starting with the set of all objects in the scene. Then any objects which are _entirely_ outside the viewing frustum are removed from the set. Next, objects which partially extrude outside the viewing frustum are clipped by dropping any polygons which are entirely outside the frustum, and by clipping the polygons which cross outside the frustrum so that they no longer exit it.

Once that's been done, we have the largest set of polygons which are entirely within the viewing frustum. This list is usually further reduced using processes like [back-face_culling](https://en.wikipedia.org/wiki/back-face_culling)}} (removing polygons whose back side is facing the camera) and occlusion culling using [hidden-surface determination](https://en.wikipedia.org/wiki/hidden-surface_determination)}} (removing polygons which can't be seen because they're entirely blocked by polygons that are closer to the lens).

## Perspective projection matrix

Up to this point, we've built up our own 3D rendering setup, step by step. However the current code as we've built it has some issues. For one, it gets skewed whenever we resize our window. Another is that our simple projection doesn't handle a wide range of values for the scene data. Most scenes don't work in clip space. It would be helpful to define what distance is relevant to the scene so that precision isn't lost in converting the numbers. Finally it's very helpful to have a fine-tuned control over what points get placed inside and outside of clip space. In the previous examples the corners of the cube occasionally get clipped.

The **perspective projection matrix** is a type of projection matrix that accomplishes all of these requirements. The math also starts to get a bit more involved and won't be fully explained in these examples. In short, it combines dividing by w (as done with the previous examples) with some ingenious manipulations based on [similar triangles](https://en.wikipedia.org/wiki/Similarity_%28geometry%29). If you want to read a full explanation of the math behind it check out some of the following links:

- [OpenGL Projection Matrix](http://www.songho.ca/opengl/gl_projectionmatrix.html)
- [Perspective Projection](http://ogldev.atspace.co.uk/www/tutorial12/tutorial12.html)
- [Trying to understand the math behind the perspective projection matrix in WebGL](http://stackoverflow.com/questions/28286057/trying-to-understand-the-math-behind-the-perspective-matrix-in-webgl/28301213#28301213)

One important thing to note about the perspective projection matrix used below is that it flips the z axis. In clip space the z+ goes away from the viewer, while with this matrix it comes towards the viewer.

The reason to flip the z axis is that the clip space coordinate system is a left-handed coordinate system (wherein the z-axis points away from the viewer and into the screen), while the convention in mathematics, physics and 3D modeling, as well as for the view/eye coordinate system in OpenGL, is to use a right-handed coordinate system (z-axis points out of the screen towards the viewer) . More on that in the relevant Wikipedia articles: [Cartesian coordinate system](https://en.wikipedia.org/wiki/Cartesian_coordinate_system#Orientation_and_handedness), [Right-hand rule](https://en.wikipedia.org/wiki/Right-hand_rule).

Let's take a look at a `perspectiveMatrix()` function, which computes the perspective projection matrix.

```js
MDN.perspectiveMatrix = function (
  fieldOfViewInRadians,
  aspectRatio,
  near,
  far,
) {
  var f = 1.0 / Math.tan(fieldOfViewInRadians / 2);
  var rangeInv = 1 / (near - far);

  return [
    f / aspectRatio,
    0,
    0,
    0,
    0,
    f,
    0,
    0,
    0,
    0,
    (near + far) * rangeInv,
    -1,
    0,
    0,
    near * far * rangeInv * 2,
    0,
  ];
};
```

The four parameters into this function are:

- `fieldOfviewInRadians`
  - : An angle, given in radians, indicating how much of the scene is visible to the viewer at once. The larger the number is, the more is visible by the camera. The geometry at the edges becomes more and more distorted, equivalent to a wide angle lens. When the field of view is larger, the objects typically get smaller. When the field of view is smaller, then the camera can see less and less in the scene. The objects are distorted much less by perspective and objects seem much closer to the camera
- `aspectRatio`
  - : The scene's aspect ratio, which is equivalent to its width divided by its height. In these examples, that's the window's width divided by the window height. The introduction of this parameter finally solves the problem wherein the model gets warped as the canvas is resized and reshaped.
- `nearClippingPlaneDistance`
  - : A positive number indicating the distance into the screen to a plane which is perpendicular to the floor, nearer than which everything gets clipped away. This is mapped to -1 in clip space, and should not be set to 0.
- `farClippingPlaneDistance`
  - : A positive number indicating the distance to the plane beyond which geometry is clipped away. This is mapped to 1 in clip space. This value should be kept reasonably close to the distance of the geometry in order to avoid precision errors creeping in while rendering.

In the latest version of the box demo, the `computeSimpleProjectionMatrix()` method has been replaced with the `computePerspectiveMatrix()` method.

```js
CubeDemo.prototype.computePerspectiveMatrix = function () {
  var fieldOfViewInRadians = Math.PI * 0.5;
  var aspectRatio = window.innerWidth / window.innerHeight;
  var nearClippingPlaneDistance = 1;
  var farClippingPlaneDistance = 50;

  this.transforms.projection = MDN.perspectiveMatrix(
    fieldOfViewInRadians,
    aspectRatio,
    nearClippingPlaneDistance,
    farClippingPlaneDistance,
  );
};
```

The shader code is identical to the previous example:

```js
gl_Position = projection * model * vec4(position, 1.0);
```

Additionally (not shown), the position and scale matrices of the model have been changed to take it out of clip space and into the larger coordinate system.

### The results

[View on JSFiddle](https://jsfiddle.net/Lzxw7e1q)

![A true perspective matrix](part6.png)

### Exercises

- Experiment with the parameters of the perspective projection matrix and the model matrix.
- Swap out the perspective projection matrix to use [orthographic projection](https://en.wikipedia.org/wiki/orthographic_projection). In the MDN WebGL shared code you'll find the `MDN.orthographicMatrix()`. This can replace the `MDN.perspectiveMatrix()` function in `CubeDemo.prototype.computePerspectiveMatrix()`.

## View matrix

While some graphics libraries have a virtual camera that can be positioned and pointed while composing a scene, OpenGL (and by extension WebGL) does not. This is where the **view matrix** comes in. Its job is to translate, rotate, and scale the objects in the scene so that they are located in the right place relative to the viewer given the viewer's position and orientation.

### Simulating a camera

This makes use of one of the fundamental facets of Einstein's special relativity theory: the principle of reference frames and relative motion says that, from the perspective of a viewer, you can simulate changing the position and orientation of the viewer by applying the opposite change to the objects in the scene. Either way, the result appears to be identical to the viewer.

Consider a box sitting on a table and a camera resting on the table one meter away, pointed at the box, the front of which is pointed toward the camera. Then consider moving the camera away from the box until it's two meters away (by adding a meter to the camera's Z position), then sliding it 10 centimeters to the its left. The box recedes from the camera by that amount and slides to the right slightly, thereby appearing smaller to the camera and exposing a small amount of its left side to the camera.

Now let's reset the scene, placing the box back in its starting point, with the camera two meters from, and directly facing, the box. This time, however, the camera is locked down on the table and cannot be moved or turned. This is what working in WebGL is like. So how do we simulate moving the camera through space?

Instead of moving the camera backward and to the left, we apply the inverse transform to the box: we move the _box_ backward one meter, and then 10 centimeters to its right. The result, from the perspective of each of the two objects, is identical.

**<<< insert image(s) here >>>**

The final step in all of this is to create the **view matrix**, which transforms the objects in the scene so they're positioned to simulate the camera's current location and orientation. Our code as it stands can move the cube around in world space and project everything to have perspective, but we still can't move the camera.

Imagine shooting a movie with a physical camera. You have the freedom to place the camera essentially anywhere you wish, and to aim the camera in whichever direction you choose. To simulate this in 3D graphics, we use a view matrix to simulate the position and rotation of that physical camera.

Unlike the model matrix, which directly transforms the model vertices, the view matrix moves an abstract camera around. In reality, the vertex shader is still only moving the models while the "camera" stays in place. In order for this to work out correctly, the inverse of the transform matrix must be used. The inverse matrix essentially reverses a transformation, so if we move the camera view forward, the inverse matrix causes the objects in the scene to move back.

The following `computeViewMatrix()` method animates the view matrix by moving it in and out, and left and right.

```js
CubeDemo.prototype.computeViewMatrix = function (now) {
  var moveInAndOut = 20 * Math.sin(now * 0.002);
  var moveLeftAndRight = 15 * Math.sin(now * 0.0017);

  // Move the camera around
  var position = MDN.translateMatrix(moveLeftAndRight, 0, 50 + moveInAndOut);

  // Multiply together, make sure and read them in opposite order
  var matrix = MDN.multiplyArrayOfMatrices([
    // Exercise: rotate the camera view
    position,
  ]);

  // Inverse the operation for camera movements, because we are actually
  // moving the geometry in the scene, not the camera itself.
  this.transforms.view = MDN.invertMatrix(matrix);
};
```

The shader now uses three matrices.

```glsl
gl_Position = projection * view * model * vec4(position, 1.0);
```

After this step, the GPU pipeline will clip the out of range vertices, and send the model down to the fragment shader for rasterization.

### The results

[View on JSFiddle](https://jsfiddle.net/86fd797g)

![The view matrix](part7.png)

### Relating the coordinate systems

At this point it would be beneficial to take a step back and look at and label the various coordinate systems we use. First off, the cube's vertices are defined in **model space**. To move the model around the scene. these vertices need to be converted into **world space** by applying the model matrix.

model space → model matrix → world space

The camera hasn't done anything yet, and the points need to be moved again. Currently they are in world space, but they need to be moved to **view space** (using the view matrix) in order to represent the camera placement.

world space → view matrix → view space

Finally a **projection** (in our case the perspective projection matrix) needs to be added in order to map the world coordinates into clip space coordinates.

view space → projection matrix → clip space

### Exercise

- Move the camera around the scene.
- Add some rotation matrices to the view matrix to look around.
- Finally, track the mouse's position. Use 2 rotation matrices to have the camera look up and down based on where the user's mouse is on the screen.

## See also

- [WebGL](/ja/docs/Web/API/WebGL_API)
- [3D projection](https://en.wikipedia.org/wiki/3D_projection)
