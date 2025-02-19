---
title: Adding 2D content to a WebGL context
slug: Web/API/WebGL_API/Tutorial/Adding_2D_content_to_a_WebGL_context
page-type: guide
---

{{DefaultAPISidebar("WebGL")}} {{PreviousNext("Web/API/WebGL_API/Tutorial/Getting_started_with_WebGL", "Web/API/WebGL_API/Tutorial/Using_shaders_to_apply_color_in_WebGL")}}

Once you've successfully [created a WebGL context](/en-US/docs/Web/API/WebGL_API/Tutorial/Getting_started_with_WebGL), you can start rendering into it. A simple thing we can do is draw an untextured square plane, so let's start there.

The complete source code for this project is [available on GitHub](https://github.com/mdn/dom-examples/tree/main/webgl-examples/tutorial/sample2).

## Including the glMatrix library

This project uses the [glMatrix](https://glmatrix.net/) library to perform its matrix operations, so you will need to include that in your project. We're loading a copy from a CDN.

> [!NOTE]
> Update your "index.html" so it looks like this:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <title>WebGL Demo</title>
    <script
      src="https://cdnjs.cloudflare.com/ajax/libs/gl-matrix/2.8.1/gl-matrix-min.js"
      integrity="sha512-zhHQR0/H5SEBL3Wn6yYSaTTZej12z0hVZKOv3TwCUXT1z5qeqGcXJLLrbERYRScEDDpYIJhPC1fk31gqR783iQ=="
      crossorigin="anonymous"
      defer></script>
    <script src="webgl-demo.js" type="module"></script>
  </head>

  <body>
    <canvas id="gl-canvas" width="640" height="480"></canvas>
  </body>
</html>
```

## Drawing the scene

The most important thing to understand before we get started is that even though we're only rendering a square plane object in this example, we're still drawing in 3D space. It's just we're drawing a square and we're putting it directly in front of the camera perpendicular to the view direction. We need to define the shaders that will create the color for our simple scene as well as draw our object. These will establish how the square plane appears on the screen.

### The shaders

A **shader** is a program, written using the [OpenGL ES Shading Language](https://registry.khronos.org/OpenGL/specs/es/3.2/GLSL_ES_Specification_3.20.pdf) (**GLSL**), that takes information about the vertices that make up a shape and generates the data needed to render the pixels onto the screen: namely, the positions of the pixels and their colors.

There are two shader functions run when drawing WebGL content: the **vertex shader** and the **fragment shader**. You write these in GLSL and pass the text of the code into WebGL to be compiled for execution on the GPU. Together, a set of vertex and fragment shaders is called a **shader program**.

Let's take a quick look at the two types of shader, with the example in mind of drawing a 2D shape into the WebGL context.

#### Vertex shader

Each time a shape is rendered, the vertex shader is run for each vertex in the shape. Its job is to transform the input vertex from its original coordinate system into the **[clip space](/en-US/docs/Web/API/WebGL_API/WebGL_model_view_projection#clip_space)** coordinate system used by WebGL, in which each axis has a range from -1.0 to 1.0, regardless of aspect ratio, actual size, or any other factors.

The vertex shader must perform the needed transforms on the vertex's position, make any other adjustments or calculations it needs to make on a per-vertex basis, then return the transformed vertex by saving it in a special variable provided by GLSL, called `gl_Position`.

The vertex shader can, as needed, also do things like determine the coordinates within the face's texture of the {{Glossary("texel")}} to apply to the vertex, apply the normals to determine the lighting factor to apply to the vertex, and so on. This information can then be stored in [varyings](/en-US/docs/Web/API/WebGL_API/Data#varyings) or [attributes](/en-US/docs/Web/API/WebGL_API/Data#attributes) as appropriate to be shared with the fragment shader.

Our vertex shader below receives vertex position values from an attribute we define called `aVertexPosition`. That position is then multiplied by two 4x4 matrices we provide called `uProjectionMatrix` and `uModelViewMatrix`; `gl_Position` is set to the result. For more info on projection and other matrixes [you might find this article useful](https://webglfundamentals.org/webgl/lessons/webgl-3d-perspective.html).

> [!NOTE]
> Add this code to your `main()` function:

```js
// Vertex shader program
const vsSource = `
    attribute vec4 aVertexPosition;
    uniform mat4 uModelViewMatrix;
    uniform mat4 uProjectionMatrix;
    void main() {
      gl_Position = uProjectionMatrix * uModelViewMatrix * aVertexPosition;
    }
  `;
```

It's worth noting that we're using a `vec4` attribute for the vertex position, which doesn't actually use a 4-component vector; that is, it could be handled as a `vec2` or `vec3` depending on the situation. But when we do our math, we will need it to be a `vec4`, so rather than convert it to a `vec4` every time we do math, we'll just use a `vec4` from the beginning. This eliminates operations from every calculation we do in our shader. Performance matters.

In this example, we're not computing any lighting at all, since we haven't yet applied any to the scene. That will come later, in the example [Lighting in WebGL](/en-US/docs/Web/API/WebGL_API/Tutorial/Lighting_in_WebGL). Note also the lack of any work with textures here; that will be added in [Using textures in WebGL](/en-US/docs/Web/API/WebGL_API/Tutorial/Using_textures_in_WebGL).

#### Fragment shader

The **fragment shader** is called once for every pixel on each shape to be drawn, after the shape's vertices have been processed by the vertex shader. Its job is to determine the color of that pixel by figuring out which texel (that is, the pixel from within the shape's texture) to apply to the pixel, getting that texel's color, then applying the appropriate lighting to the color. The color is then returned to the WebGL layer by storing it in the special variable `gl_FragColor`. That color is then drawn to the screen in the correct position for the shape's corresponding pixel.

In this case, we're returning white every time, since we're just drawing a white square, with no lighting in use.

> [!NOTE]
> Add this code to your `main()` function:

```js
const fsSource = `
    void main() {
      gl_FragColor = vec4(1.0, 1.0, 1.0, 1.0);
    }
  `;
```

### Initializing the shaders

Now that we've defined the two shaders we need to pass them to WebGL, compile them, and link them together. The code below creates the two shaders by calling `loadShader()`, passing the type and source for the shader. It then creates a program, attaches the shaders and links them together. If compiling or linking fails the code displays an alert.

> [!NOTE]
> Add these two functions to your "webgl-demo.js" script:

```js
//
// Initialize a shader program, so WebGL knows how to draw our data
//
function initShaderProgram(gl, vsSource, fsSource) {
  const vertexShader = loadShader(gl, gl.VERTEX_SHADER, vsSource);
  const fragmentShader = loadShader(gl, gl.FRAGMENT_SHADER, fsSource);

  // Create the shader program

  const shaderProgram = gl.createProgram();
  gl.attachShader(shaderProgram, vertexShader);
  gl.attachShader(shaderProgram, fragmentShader);
  gl.linkProgram(shaderProgram);

  // If creating the shader program failed, alert

  if (!gl.getProgramParameter(shaderProgram, gl.LINK_STATUS)) {
    alert(
      `Unable to initialize the shader program: ${gl.getProgramInfoLog(
        shaderProgram,
      )}`,
    );
    return null;
  }

  return shaderProgram;
}

//
// creates a shader of the given type, uploads the source and
// compiles it.
//
function loadShader(gl, type, source) {
  const shader = gl.createShader(type);

  // Send the source to the shader object

  gl.shaderSource(shader, source);

  // Compile the shader program

  gl.compileShader(shader);

  // See if it compiled successfully

  if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
    alert(
      `An error occurred compiling the shaders: ${gl.getShaderInfoLog(shader)}`,
    );
    gl.deleteShader(shader);
    return null;
  }

  return shader;
}
```

The `loadShader()` function takes as input the WebGL context, the shader type, and the source code, then creates and compiles the shader as follows:

1. A new shader is created by calling {{domxref("WebGLRenderingContext.createShader", "gl.createShader()")}}.
2. The shader's source code is sent to the shader by calling {{domxref("WebGLRenderingContext.shaderSource", "gl.shaderSource()")}}.
3. Once the shader has the source code, it's compiled using {{domxref("WebGLRenderingContext.compileShader", "gl.compileShader()")}}.
4. To check to be sure the shader successfully compiled, the shader parameter `gl.COMPILE_STATUS` is checked. To get its value, we call {{domxref("WebGLRenderingContext.getShaderParameter", "gl.getShaderParameter()")}}, specifying the shader and the name of the parameter we want to check (`gl.COMPILE_STATUS`). If that's `false`, we know the shader failed to compile, so show an alert with log information obtained from the compiler using {{domxref("WebGLRenderingContext.getShaderInfoLog", "gl.getShaderInfoLog()")}}, then delete the shader and return `null` to indicate a failure to load the shader.
5. If the shader was loaded and successfully compiled, the compiled shader is returned to the caller.

> [!NOTE]
> Add this code to your `main()` function:

```js
// Initialize a shader program; this is where all the lighting
// for the vertices and so forth is established.
const shaderProgram = initShaderProgram(gl, vsSource, fsSource);
```

After we've created a shader program we need to look up the locations that WebGL assigned to our inputs. In this case we have one attribute and two uniforms. Attributes receive values from buffers. Each iteration of the vertex shader receives the next value from the buffer assigned to that attribute. [Uniforms](/en-US/docs/Web/API/WebGL_API/Data#uniforms) are similar to JavaScript global variables. They stay the same value for all iterations of a shader. Since the attribute and uniform locations are specific to a single shader program we'll store them together to make them easy to pass around

> [!NOTE]
> Add this code to your `main()` function:

```js
// Collect all the info needed to use the shader program.
// Look up which attribute our shader program is using
// for aVertexPosition and look up uniform locations.
const programInfo = {
  program: shaderProgram,
  attribLocations: {
    vertexPosition: gl.getAttribLocation(shaderProgram, "aVertexPosition"),
  },
  uniformLocations: {
    projectionMatrix: gl.getUniformLocation(shaderProgram, "uProjectionMatrix"),
    modelViewMatrix: gl.getUniformLocation(shaderProgram, "uModelViewMatrix"),
  },
};
```

## Creating the square plane

Before we can render our square plane, we need to create the buffer that contains its vertex positions and put the vertex positions in it.

We'll do that using a function we call `initBuffers()`, which we will implement in a separate [JavaScript module](/en-US/docs/Web/JavaScript/Guide/Modules). As we explore more advanced WebGL concepts, this module will be augmented to create more — and more complex — 3D objects.

> [!NOTE]
> Create a new file called "init-buffers.js", and give it the following contents:

```js
function initBuffers(gl) {
  const positionBuffer = initPositionBuffer(gl);

  return {
    position: positionBuffer,
  };
}

function initPositionBuffer(gl) {
  // Create a buffer for the square's positions.
  const positionBuffer = gl.createBuffer();

  // Select the positionBuffer as the one to apply buffer
  // operations to from here out.
  gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer);

  // Now create an array of positions for the square.
  const positions = [1.0, 1.0, -1.0, 1.0, 1.0, -1.0, -1.0, -1.0];

  // Now pass the list of positions into WebGL to build the
  // shape. We do this by creating a Float32Array from the
  // JavaScript array, then use it to fill the current buffer.
  gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(positions), gl.STATIC_DRAW);

  return positionBuffer;
}

export { initBuffers };
```

This routine is pretty simplistic given the basic nature of the scene in this example. It starts by calling the `gl` object's {{domxref("WebGLRenderingContext.createBuffer()", "createBuffer()")}} method to obtain a buffer into which we'll store the vertex positions. This is then bound to the context by calling the {{domxref("WebGLRenderingContext.bindBuffer()", "bindBuffer()")}} method.

Once that's done, we create a JavaScript array containing the position for each vertex of the square plane. This is then converted into an array of floats and passed into the `gl` object's {{domxref("WebGLRenderingContext.bufferData()", "bufferData()")}} method to establish the vertex positions for the object.

## Rendering the scene

Once the shaders are established, the locations are looked up, and the square plane's vertex positions put in a buffer, we can actually render the scene. We'll do this in a `drawScene()` function that, again, we'll implement in a separate JavaScript module.

> [!NOTE]
> Create a new file called "draw-scene.js", and give it the following contents:

```js
function drawScene(gl, programInfo, buffers) {
  gl.clearColor(0.0, 0.0, 0.0, 1.0); // Clear to black, fully opaque
  gl.clearDepth(1.0); // Clear everything
  gl.enable(gl.DEPTH_TEST); // Enable depth testing
  gl.depthFunc(gl.LEQUAL); // Near things obscure far things

  // Clear the canvas before we start drawing on it.

  gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);

  // Create a perspective matrix, a special matrix that is
  // used to simulate the distortion of perspective in a camera.
  // Our field of view is 45 degrees, with a width/height
  // ratio that matches the display size of the canvas
  // and we only want to see objects between 0.1 units
  // and 100 units away from the camera.

  const fieldOfView = (45 * Math.PI) / 180; // in radians
  const aspect = gl.canvas.clientWidth / gl.canvas.clientHeight;
  const zNear = 0.1;
  const zFar = 100.0;
  const projectionMatrix = mat4.create();

  // note: glMatrix always has the first argument
  // as the destination to receive the result.
  mat4.perspective(projectionMatrix, fieldOfView, aspect, zNear, zFar);

  // Set the drawing position to the "identity" point, which is
  // the center of the scene.
  const modelViewMatrix = mat4.create();

  // Now move the drawing position a bit to where we want to
  // start drawing the square.
  mat4.translate(
    modelViewMatrix, // destination matrix
    modelViewMatrix, // matrix to translate
    [-0.0, 0.0, -6.0],
  ); // amount to translate

  // Tell WebGL how to pull out the positions from the position
  // buffer into the vertexPosition attribute.
  setPositionAttribute(gl, buffers, programInfo);

  // Tell WebGL to use our program when drawing
  gl.useProgram(programInfo.program);

  // Set the shader uniforms
  gl.uniformMatrix4fv(
    programInfo.uniformLocations.projectionMatrix,
    false,
    projectionMatrix,
  );
  gl.uniformMatrix4fv(
    programInfo.uniformLocations.modelViewMatrix,
    false,
    modelViewMatrix,
  );

  {
    const offset = 0;
    const vertexCount = 4;
    gl.drawArrays(gl.TRIANGLE_STRIP, offset, vertexCount);
  }
}

// Tell WebGL how to pull out the positions from the position
// buffer into the vertexPosition attribute.
function setPositionAttribute(gl, buffers, programInfo) {
  const numComponents = 2; // pull out 2 values per iteration
  const type = gl.FLOAT; // the data in the buffer is 32bit floats
  const normalize = false; // don't normalize
  const stride = 0; // how many bytes to get from one set of values to the next
  // 0 = use type and numComponents above
  const offset = 0; // how many bytes inside the buffer to start from
  gl.bindBuffer(gl.ARRAY_BUFFER, buffers.position);
  gl.vertexAttribPointer(
    programInfo.attribLocations.vertexPosition,
    numComponents,
    type,
    normalize,
    stride,
    offset,
  );
  gl.enableVertexAttribArray(programInfo.attribLocations.vertexPosition);
}

export { drawScene };
```

The first step is to clear the canvas to our background color; then we establish the camera's perspective. We set a field of view of 45°, with a width to height ratio that matches the display dimensions of our canvas. We also specify that we only want objects between 0.1 and 100 units from the camera to be rendered.

Then we establish the position of the square plane by loading the identity position and translating away from the camera by 6 units. After that, we bind the square's vertex buffer to the attribute the shader is using for `aVertexPosition` and we tell WebGL how to pull the data out of it. Finally we draw the object by calling the {{domxref("WebGLRenderingContext.drawArrays()", "drawArrays()")}} method.

Finally, let's call `initBuffers()` and `drawScene()`.

> [!NOTE]
> Add this code to the start of your "webgl-demo.js" file:

```js
import { initBuffers } from "./init-buffers.js";
import { drawScene } from "./draw-scene.js";
```

> [!NOTE]
> Add this code to the end of your `main()` function:

```js
// Here's where we call the routine that builds all the
// objects we'll be drawing.
const buffers = initBuffers(gl);

// Draw the scene
drawScene(gl, programInfo, buffers);
```

The result should look like this:

{{EmbedGHLiveSample('dom-examples/webgl-examples/tutorial/sample2/index.html', 670, 510) }}

[View the complete code](https://github.com/mdn/dom-examples/tree/main/webgl-examples/tutorial/sample2) | [Open this demo on a new page](https://mdn.github.io/dom-examples/webgl-examples/tutorial/sample2/)

## Matrix utility operations

Matrix operations might seem complicated but [they are actually pretty simple if you take them one step at a time](https://webglfundamentals.org/webgl/lessons/webgl-2d-matrices.html). Generally people use a matrix library rather than writing their own. In our case we're using the popular [glMatrix library](https://glmatrix.net/).

### See also

- [Matrices](https://webglfundamentals.org/webgl/lessons/webgl-2d-matrices.html) on WebGLFundamentals
- [Matrices](https://mathworld.wolfram.com/Matrix.html) on Wolfram MathWorld
- [Matrix](<https://en.wikipedia.org/wiki/Matrix_(mathematics)>) on Wikipedia

{{PreviousNext("Web/API/WebGL_API/Tutorial/Getting_started_with_WebGL", "Web/API/WebGL_API/Tutorial/Using_shaders_to_apply_color_in_WebGL")}}
