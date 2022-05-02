# Resolving Fyne/v2 Data Races using Functional Options

Author: Changkun Ou (hi@changkun.de)

## Background

Since fyne/v2 (actually a little bit before v2), the current rendering loop is separated from the event loop, executed in parallel. This decision introduces the performance leap but also introduces a whole collection of challenges for the fyne code base to be concurrent-safe. For instance, the current official demo application cannot be executed without encountering data races between the rendering loop and event loop. When building the application with the `-race` flag:

```
cd cmd/hello && go build -race
./hello
```

Click the "Hello" button will trigger the following verbose data race with 5 different error:

<details><summary>$ ./hello</summary><br>
2022/05/02 14:22:26   At: /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/window_desktop.go:198
==================
WARNING: DATA RACE
Read at 0x00c0000ca2d8 by goroutine 9:
  fyne.io/fyne/v2/internal/painter/gl.(*painter).drawRectangle()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/draw.go:103 +0x75
  fyne.io/fyne/v2/internal/painter/gl.(*painter).drawObject()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/draw.go:145 +0x324
  fyne.io/fyne/v2/internal/painter/gl.(*painter).Paint()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/painter.go:88 +0x88
  fyne.io/fyne/v2/internal/driver/glfw.(*glCanvas).paint.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/canvas.go:287 +0x2be
  fyne.io/fyne/v2/internal/driver/common.(*Canvas).walkTree.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/common/canvas.go:438 +0x482
  fyne.io/fyne/v2/internal/driver.walkObjectTree()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/util.go:169 +0x2bb
  fyne.io/fyne/v2/internal/driver.walkObjectTree.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/util.go:176 +0x451
  fyne.io/fyne/v2/internal/driver.walkObjectTree()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/util.go:190 +0x460
  fyne.io/fyne/v2/internal/driver.walkObjectTree.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/util.go:176 +0x451
  fyne.io/fyne/v2/internal/driver.walkObjectTree()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/util.go:190 +0x460
  fyne.io/fyne/v2/internal/driver.WalkVisibleObjectTree()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/util.go:134 +0x7d
  fyne.io/fyne/v2/internal/driver/common.(*Canvas).walkTree()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/common/canvas.go:459 +0x1f0
  fyne.io/fyne/v2/internal/driver/common.(*Canvas).WalkTrees()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/common/canvas.go:371 +0x67
  fyne.io/fyne/v2/internal/driver/glfw.(*glCanvas).paint()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/canvas.go:301 +0x12a
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).repaintWindow.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop.go:202 +0xbc
  fyne.io/fyne/v2/internal/driver/glfw.(*window).RunWithContext()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/window.go:903 +0x5d
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).repaintWindow()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop.go:193 +0x6c
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).drawSingleFrame()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop.go:101 +0x1c5
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).startDrawThread.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop.go:249 +0x256

Previous write at 0x00c0000ca2d8 by goroutine 29:
  fyne.io/fyne/v2/widget.newButtonTapAnimation.func1()
      /Users/changkun/dev/changkun.de/fyne/widget/button.go:431 +0x1b2
  fyne.io/fyne/v2/internal/animation.(*Runner).tickAnimation()
      /Users/changkun/dev/changkun.de/fyne/internal/animation/runner.go:131 +0x5bd
  fyne.io/fyne/v2/internal/animation.(*Runner).runAnimations.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/animation/runner.go:75 +0x2d1

Goroutine 9 (running) created at:
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).startDrawThread()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop.go:225 +0x18e
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).initGLFW.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop_desktop.go:28 +0x65
  sync.(*Once).doSlow()
      /Users/changkun/goes/go/src/sync/once.go:68 +0x101
  sync.(*Once).Do()
      /Users/changkun/goes/go/src/sync/once.go:59 +0x46
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).initGLFW()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop_desktop.go:16 +0x5d
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).createWindow.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/window.go:937 +0x6e
  fyne.io/fyne/v2/internal/driver/glfw.runOnMain()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop.go:60 +0x1b9
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).createWindow()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/window.go:936 +0x1ae
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).CreateWindow()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/window.go:928 +0x45
  fyne.io/fyne/v2/app.(*fyneApp).NewWindow()
      /Users/changkun/dev/changkun.de/fyne/app/app.go:63 +0x57
  main.main()
      /Users/changkun/dev/changkun.de/fyne/cmd/hello/main.go:12 +0x46

Goroutine 29 (running) created at:
  fyne.io/fyne/v2/internal/animation.(*Runner).runAnimations()
      /Users/changkun/dev/changkun.de/fyne/internal/animation/runner.go:67 +0xda
  fyne.io/fyne/v2/internal/animation.(*Runner).Start()
      /Users/changkun/dev/changkun.de/fyne/internal/animation/runner.go:27 +0x349
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).StartAnimation()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/animation.go:6 +0x4a
  fyne.io/fyne/v2.(*Animation).Start()
      /Users/changkun/dev/changkun.de/fyne/animation.go:58 +0x41
  fyne.io/fyne/v2/widget.(*Button).tapAnimation()
      /Users/changkun/dev/changkun.de/fyne/widget/button.go:214 +0x84
  fyne.io/fyne/v2/widget.(*Button).Tapped()
      /Users/changkun/dev/changkun.de/fyne/widget/button.go:190 +0x44
  fyne.io/fyne/v2/internal/driver/glfw.(*window).mouseClickedHandleTapDoubleTap.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/window.go:634 +0x4d
  fyne.io/fyne/v2/internal/driver/common.(*Window).RunEventQueue()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/common/window.go:35 +0x76
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).createWindow.func1.1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/window.go:942 +0x39
==================
==================
WARNING: DATA RACE
Read at 0x00c0003ce61c by goroutine 9:
  runtime.racereadrange()
      <autogenerated>:1 +0x1b
  image.(*Uniform).RGBA()
      /Users/changkun/goes/go/src/image/names.go:29 +0xdb
  fyne.io/fyne/v2/internal/painter/gl.(*painter).imgToTexture()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/painter.go:203 +0xbe
  fyne.io/fyne/v2/internal/painter/gl.(*painter).newGlRectTexture()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/texture.go:98 +0x124
  fyne.io/fyne/v2/internal/painter/gl.(*painter).newGlRectTexture-fm()
      <autogenerated>:1 +0x4d
  fyne.io/fyne/v2/internal/painter/gl.(*painter).getTexture()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/texture.go:25 +0x5c
  fyne.io/fyne/v2/internal/painter/gl.(*painter).drawTextureWithDetails()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/draw.go:15 +0xc4
  fyne.io/fyne/v2/internal/painter/gl.(*painter).drawRectangle()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/draw.go:106 +0x311
  fyne.io/fyne/v2/internal/painter/gl.(*painter).drawObject()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/draw.go:145 +0x324
  fyne.io/fyne/v2/internal/painter/gl.(*painter).Paint()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/painter.go:88 +0x88
  fyne.io/fyne/v2/internal/driver/glfw.(*glCanvas).paint.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/canvas.go:287 +0x2be
  fyne.io/fyne/v2/internal/driver/common.(*Canvas).walkTree.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/common/canvas.go:438 +0x482
  fyne.io/fyne/v2/internal/driver.walkObjectTree()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/util.go:169 +0x2bb
  fyne.io/fyne/v2/internal/driver.walkObjectTree.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/util.go:176 +0x451
  fyne.io/fyne/v2/internal/driver.walkObjectTree()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/util.go:190 +0x460
  fyne.io/fyne/v2/internal/driver.walkObjectTree.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/util.go:176 +0x451
  fyne.io/fyne/v2/internal/driver.walkObjectTree()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/util.go:190 +0x460
  fyne.io/fyne/v2/internal/driver.WalkVisibleObjectTree()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/util.go:134 +0x7d
  fyne.io/fyne/v2/internal/driver/common.(*Canvas).walkTree()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/common/canvas.go:459 +0x1f0
  fyne.io/fyne/v2/internal/driver/common.(*Canvas).WalkTrees()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/common/canvas.go:371 +0x67
  fyne.io/fyne/v2/internal/driver/glfw.(*glCanvas).paint()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/canvas.go:301 +0x12a
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).repaintWindow.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop.go:202 +0xbc
  fyne.io/fyne/v2/internal/driver/glfw.(*window).RunWithContext()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/window.go:903 +0x5d
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).repaintWindow()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop.go:193 +0x6c
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).drawSingleFrame()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop.go:101 +0x1c5
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).startDrawThread.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop.go:249 +0x256

Previous write at 0x00c0003ce61c by goroutine 29:
  fyne.io/fyne/v2/widget.newButtonTapAnimation.func1()
      /Users/changkun/dev/changkun.de/fyne/widget/button.go:431 +0x13c
  fyne.io/fyne/v2/internal/animation.(*Runner).tickAnimation()
      /Users/changkun/dev/changkun.de/fyne/internal/animation/runner.go:131 +0x5bd
  fyne.io/fyne/v2/internal/animation.(*Runner).runAnimations.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/animation/runner.go:75 +0x2d1

Goroutine 9 (running) created at:
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).startDrawThread()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop.go:225 +0x18e
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).initGLFW.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop_desktop.go:28 +0x65
  sync.(*Once).doSlow()
      /Users/changkun/goes/go/src/sync/once.go:68 +0x101
  sync.(*Once).Do()
      /Users/changkun/goes/go/src/sync/once.go:59 +0x46
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).initGLFW()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop_desktop.go:16 +0x5d
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).createWindow.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/window.go:937 +0x6e
  fyne.io/fyne/v2/internal/driver/glfw.runOnMain()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop.go:60 +0x1b9
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).createWindow()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/window.go:936 +0x1ae
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).CreateWindow()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/window.go:928 +0x45
  fyne.io/fyne/v2/app.(*fyneApp).NewWindow()
      /Users/changkun/dev/changkun.de/fyne/app/app.go:63 +0x57
  main.main()
      /Users/changkun/dev/changkun.de/fyne/cmd/hello/main.go:12 +0x46

Goroutine 29 (running) created at:
  fyne.io/fyne/v2/internal/animation.(*Runner).runAnimations()
      /Users/changkun/dev/changkun.de/fyne/internal/animation/runner.go:67 +0xda
  fyne.io/fyne/v2/internal/animation.(*Runner).Start()
      /Users/changkun/dev/changkun.de/fyne/internal/animation/runner.go:27 +0x349
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).StartAnimation()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/animation.go:6 +0x4a
  fyne.io/fyne/v2.(*Animation).Start()
      /Users/changkun/dev/changkun.de/fyne/animation.go:58 +0x41
  fyne.io/fyne/v2/widget.(*Button).tapAnimation()
      /Users/changkun/dev/changkun.de/fyne/widget/button.go:214 +0x84
  fyne.io/fyne/v2/widget.(*Button).Tapped()
      /Users/changkun/dev/changkun.de/fyne/widget/button.go:190 +0x44
  fyne.io/fyne/v2/internal/driver/glfw.(*window).mouseClickedHandleTapDoubleTap.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/window.go:634 +0x4d
  fyne.io/fyne/v2/internal/driver/common.(*Window).RunEventQueue()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/common/window.go:35 +0x76
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).createWindow.func1.1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/window.go:942 +0x39
==================
==================
WARNING: DATA RACE
Read at 0x00c0003ce61d by goroutine 9:
  runtime.racereadrange()
      <autogenerated>:1 +0x1b
  image.(*Uniform).RGBA()
      /Users/changkun/goes/go/src/image/names.go:29 +0xdb
  fyne.io/fyne/v2/internal/painter/gl.(*painter).imgToTexture()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/painter.go:203 +0xbe
  fyne.io/fyne/v2/internal/painter/gl.(*painter).newGlRectTexture()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/texture.go:98 +0x124
  fyne.io/fyne/v2/internal/painter/gl.(*painter).newGlRectTexture-fm()
      <autogenerated>:1 +0x4d
  fyne.io/fyne/v2/internal/painter/gl.(*painter).getTexture()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/texture.go:25 +0x5c
  fyne.io/fyne/v2/internal/painter/gl.(*painter).drawTextureWithDetails()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/draw.go:15 +0xc4
  fyne.io/fyne/v2/internal/painter/gl.(*painter).drawRectangle()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/draw.go:106 +0x311
  fyne.io/fyne/v2/internal/painter/gl.(*painter).drawObject()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/draw.go:145 +0x324
  fyne.io/fyne/v2/internal/painter/gl.(*painter).Paint()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/painter.go:88 +0x88
  fyne.io/fyne/v2/internal/driver/glfw.(*glCanvas).paint.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/canvas.go:287 +0x2be
  fyne.io/fyne/v2/internal/driver/common.(*Canvas).walkTree.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/common/canvas.go:438 +0x482
  fyne.io/fyne/v2/internal/driver.walkObjectTree()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/util.go:169 +0x2bb
  fyne.io/fyne/v2/internal/driver.walkObjectTree.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/util.go:176 +0x451
  fyne.io/fyne/v2/internal/driver.walkObjectTree()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/util.go:190 +0x460
  fyne.io/fyne/v2/internal/driver.walkObjectTree.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/util.go:176 +0x451
  fyne.io/fyne/v2/internal/driver.walkObjectTree()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/util.go:190 +0x460
  fyne.io/fyne/v2/internal/driver.WalkVisibleObjectTree()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/util.go:134 +0x7d
  fyne.io/fyne/v2/internal/driver/common.(*Canvas).walkTree()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/common/canvas.go:459 +0x1f0
  fyne.io/fyne/v2/internal/driver/common.(*Canvas).WalkTrees()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/common/canvas.go:371 +0x67
  fyne.io/fyne/v2/internal/driver/glfw.(*glCanvas).paint()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/canvas.go:301 +0x12a
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).repaintWindow.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop.go:202 +0xbc
  fyne.io/fyne/v2/internal/driver/glfw.(*window).RunWithContext()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/window.go:903 +0x5d
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).repaintWindow()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop.go:193 +0x6c
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).drawSingleFrame()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop.go:101 +0x1c5
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).startDrawThread.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop.go:249 +0x256

Previous write at 0x00c0003ce61d by goroutine 29:
  fyne.io/fyne/v2/widget.newButtonTapAnimation.func1()
      /Users/changkun/dev/changkun.de/fyne/widget/button.go:431 +0x151
  fyne.io/fyne/v2/internal/animation.(*Runner).tickAnimation()
      /Users/changkun/dev/changkun.de/fyne/internal/animation/runner.go:131 +0x5bd
  fyne.io/fyne/v2/internal/animation.(*Runner).runAnimations.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/animation/runner.go:75 +0x2d1

Goroutine 9 (running) created at:
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).startDrawThread()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop.go:225 +0x18e
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).initGLFW.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop_desktop.go:28 +0x65
  sync.(*Once).doSlow()
      /Users/changkun/goes/go/src/sync/once.go:68 +0x101
  sync.(*Once).Do()
      /Users/changkun/goes/go/src/sync/once.go:59 +0x46
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).initGLFW()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop_desktop.go:16 +0x5d
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).createWindow.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/window.go:937 +0x6e
  fyne.io/fyne/v2/internal/driver/glfw.runOnMain()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop.go:60 +0x1b9
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).createWindow()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/window.go:936 +0x1ae
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).CreateWindow()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/window.go:928 +0x45
  fyne.io/fyne/v2/app.(*fyneApp).NewWindow()
      /Users/changkun/dev/changkun.de/fyne/app/app.go:63 +0x57
  main.main()
      /Users/changkun/dev/changkun.de/fyne/cmd/hello/main.go:12 +0x46

Goroutine 29 (running) created at:
  fyne.io/fyne/v2/internal/animation.(*Runner).runAnimations()
      /Users/changkun/dev/changkun.de/fyne/internal/animation/runner.go:67 +0xda
  fyne.io/fyne/v2/internal/animation.(*Runner).Start()
      /Users/changkun/dev/changkun.de/fyne/internal/animation/runner.go:27 +0x349
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).StartAnimation()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/animation.go:6 +0x4a
  fyne.io/fyne/v2.(*Animation).Start()
      /Users/changkun/dev/changkun.de/fyne/animation.go:58 +0x41
  fyne.io/fyne/v2/widget.(*Button).tapAnimation()
      /Users/changkun/dev/changkun.de/fyne/widget/button.go:214 +0x84
  fyne.io/fyne/v2/widget.(*Button).Tapped()
      /Users/changkun/dev/changkun.de/fyne/widget/button.go:190 +0x44
  fyne.io/fyne/v2/internal/driver/glfw.(*window).mouseClickedHandleTapDoubleTap.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/window.go:634 +0x4d
  fyne.io/fyne/v2/internal/driver/common.(*Window).RunEventQueue()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/common/window.go:35 +0x76
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).createWindow.func1.1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/window.go:942 +0x39
==================
==================
WARNING: DATA RACE
Read at 0x00c0003ce61e by goroutine 9:
  runtime.racereadrange()
      <autogenerated>:1 +0x1b
  image.(*Uniform).RGBA()
      /Users/changkun/goes/go/src/image/names.go:29 +0xdb
  fyne.io/fyne/v2/internal/painter/gl.(*painter).imgToTexture()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/painter.go:203 +0xbe
  fyne.io/fyne/v2/internal/painter/gl.(*painter).newGlRectTexture()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/texture.go:98 +0x124
  fyne.io/fyne/v2/internal/painter/gl.(*painter).newGlRectTexture-fm()
      <autogenerated>:1 +0x4d
  fyne.io/fyne/v2/internal/painter/gl.(*painter).getTexture()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/texture.go:25 +0x5c
  fyne.io/fyne/v2/internal/painter/gl.(*painter).drawTextureWithDetails()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/draw.go:15 +0xc4
  fyne.io/fyne/v2/internal/painter/gl.(*painter).drawRectangle()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/draw.go:106 +0x311
  fyne.io/fyne/v2/internal/painter/gl.(*painter).drawObject()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/draw.go:145 +0x324
  fyne.io/fyne/v2/internal/painter/gl.(*painter).Paint()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/painter.go:88 +0x88
  fyne.io/fyne/v2/internal/driver/glfw.(*glCanvas).paint.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/canvas.go:287 +0x2be
  fyne.io/fyne/v2/internal/driver/common.(*Canvas).walkTree.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/common/canvas.go:438 +0x482
  fyne.io/fyne/v2/internal/driver.walkObjectTree()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/util.go:169 +0x2bb
  fyne.io/fyne/v2/internal/driver.walkObjectTree.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/util.go:176 +0x451
  fyne.io/fyne/v2/internal/driver.walkObjectTree()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/util.go:190 +0x460
  fyne.io/fyne/v2/internal/driver.walkObjectTree.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/util.go:176 +0x451
  fyne.io/fyne/v2/internal/driver.walkObjectTree()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/util.go:190 +0x460
  fyne.io/fyne/v2/internal/driver.WalkVisibleObjectTree()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/util.go:134 +0x7d
  fyne.io/fyne/v2/internal/driver/common.(*Canvas).walkTree()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/common/canvas.go:459 +0x1f0
  fyne.io/fyne/v2/internal/driver/common.(*Canvas).WalkTrees()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/common/canvas.go:371 +0x67
  fyne.io/fyne/v2/internal/driver/glfw.(*glCanvas).paint()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/canvas.go:301 +0x12a
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).repaintWindow.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop.go:202 +0xbc
  fyne.io/fyne/v2/internal/driver/glfw.(*window).RunWithContext()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/window.go:903 +0x5d
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).repaintWindow()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop.go:193 +0x6c
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).drawSingleFrame()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop.go:101 +0x1c5
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).startDrawThread.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop.go:249 +0x256

Previous write at 0x00c0003ce61e by goroutine 29:
  fyne.io/fyne/v2/widget.newButtonTapAnimation.func1()
      /Users/changkun/dev/changkun.de/fyne/widget/button.go:431 +0x167
  fyne.io/fyne/v2/internal/animation.(*Runner).tickAnimation()
      /Users/changkun/dev/changkun.de/fyne/internal/animation/runner.go:131 +0x5bd
  fyne.io/fyne/v2/internal/animation.(*Runner).runAnimations.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/animation/runner.go:75 +0x2d1

Goroutine 9 (running) created at:
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).startDrawThread()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop.go:225 +0x18e
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).initGLFW.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop_desktop.go:28 +0x65
  sync.(*Once).doSlow()
      /Users/changkun/goes/go/src/sync/once.go:68 +0x101
  sync.(*Once).Do()
      /Users/changkun/goes/go/src/sync/once.go:59 +0x46
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).initGLFW()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop_desktop.go:16 +0x5d
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).createWindow.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/window.go:937 +0x6e
  fyne.io/fyne/v2/internal/driver/glfw.runOnMain()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop.go:60 +0x1b9
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).createWindow()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/window.go:936 +0x1ae
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).CreateWindow()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/window.go:928 +0x45
  fyne.io/fyne/v2/app.(*fyneApp).NewWindow()
      /Users/changkun/dev/changkun.de/fyne/app/app.go:63 +0x57
  main.main()
      /Users/changkun/dev/changkun.de/fyne/cmd/hello/main.go:12 +0x46

Goroutine 29 (running) created at:
  fyne.io/fyne/v2/internal/animation.(*Runner).runAnimations()
      /Users/changkun/dev/changkun.de/fyne/internal/animation/runner.go:67 +0xda
  fyne.io/fyne/v2/internal/animation.(*Runner).Start()
      /Users/changkun/dev/changkun.de/fyne/internal/animation/runner.go:27 +0x349
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).StartAnimation()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/animation.go:6 +0x4a
  fyne.io/fyne/v2.(*Animation).Start()
      /Users/changkun/dev/changkun.de/fyne/animation.go:58 +0x41
  fyne.io/fyne/v2/widget.(*Button).tapAnimation()
      /Users/changkun/dev/changkun.de/fyne/widget/button.go:214 +0x84
  fyne.io/fyne/v2/widget.(*Button).Tapped()
      /Users/changkun/dev/changkun.de/fyne/widget/button.go:190 +0x44
  fyne.io/fyne/v2/internal/driver/glfw.(*window).mouseClickedHandleTapDoubleTap.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/window.go:634 +0x4d
  fyne.io/fyne/v2/internal/driver/common.(*Window).RunEventQueue()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/common/window.go:35 +0x76
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).createWindow.func1.1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/window.go:942 +0x39
==================
==================
WARNING: DATA RACE
Read at 0x00c0003ce61f by goroutine 9:
  runtime.racereadrange()
      <autogenerated>:1 +0x1b
  image.(*Uniform).RGBA()
      /Users/changkun/goes/go/src/image/names.go:29 +0xdb
  fyne.io/fyne/v2/internal/painter/gl.(*painter).imgToTexture()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/painter.go:203 +0xbe
  fyne.io/fyne/v2/internal/painter/gl.(*painter).newGlRectTexture()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/texture.go:98 +0x124
  fyne.io/fyne/v2/internal/painter/gl.(*painter).newGlRectTexture-fm()
      <autogenerated>:1 +0x4d
  fyne.io/fyne/v2/internal/painter/gl.(*painter).getTexture()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/texture.go:25 +0x5c
  fyne.io/fyne/v2/internal/painter/gl.(*painter).drawTextureWithDetails()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/draw.go:15 +0xc4
  fyne.io/fyne/v2/internal/painter/gl.(*painter).drawRectangle()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/draw.go:106 +0x311
  fyne.io/fyne/v2/internal/painter/gl.(*painter).drawObject()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/draw.go:145 +0x324
  fyne.io/fyne/v2/internal/painter/gl.(*painter).Paint()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/painter.go:88 +0x88
  fyne.io/fyne/v2/internal/driver/glfw.(*glCanvas).paint.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/canvas.go:287 +0x2be
  fyne.io/fyne/v2/internal/driver/common.(*Canvas).walkTree.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/common/canvas.go:438 +0x482
  fyne.io/fyne/v2/internal/driver.walkObjectTree()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/util.go:169 +0x2bb
  fyne.io/fyne/v2/internal/driver.walkObjectTree.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/util.go:176 +0x451
  fyne.io/fyne/v2/internal/driver.walkObjectTree()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/util.go:190 +0x460
  fyne.io/fyne/v2/internal/driver.walkObjectTree.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/util.go:176 +0x451
  fyne.io/fyne/v2/internal/driver.walkObjectTree()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/util.go:190 +0x460
  fyne.io/fyne/v2/internal/driver.WalkVisibleObjectTree()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/util.go:134 +0x7d
  fyne.io/fyne/v2/internal/driver/common.(*Canvas).walkTree()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/common/canvas.go:459 +0x1f0
  fyne.io/fyne/v2/internal/driver/common.(*Canvas).WalkTrees()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/common/canvas.go:371 +0x67
  fyne.io/fyne/v2/internal/driver/glfw.(*glCanvas).paint()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/canvas.go:301 +0x12a
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).repaintWindow.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop.go:202 +0xbc
  fyne.io/fyne/v2/internal/driver/glfw.(*window).RunWithContext()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/window.go:903 +0x5d
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).repaintWindow()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop.go:193 +0x6c
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).drawSingleFrame()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop.go:101 +0x1c5
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).startDrawThread.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop.go:249 +0x256

Previous write at 0x00c0003ce61f by goroutine 29:
  fyne.io/fyne/v2/widget.newButtonTapAnimation.func1()
      /Users/changkun/dev/changkun.de/fyne/widget/button.go:431 +0x17d
  fyne.io/fyne/v2/internal/animation.(*Runner).tickAnimation()
      /Users/changkun/dev/changkun.de/fyne/internal/animation/runner.go:131 +0x5bd
  fyne.io/fyne/v2/internal/animation.(*Runner).runAnimations.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/animation/runner.go:75 +0x2d1

Goroutine 9 (running) created at:
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).startDrawThread()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop.go:225 +0x18e
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).initGLFW.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop_desktop.go:28 +0x65
  sync.(*Once).doSlow()
      /Users/changkun/goes/go/src/sync/once.go:68 +0x101
  sync.(*Once).Do()
      /Users/changkun/goes/go/src/sync/once.go:59 +0x46
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).initGLFW()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop_desktop.go:16 +0x5d
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).createWindow.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/window.go:937 +0x6e
  fyne.io/fyne/v2/internal/driver/glfw.runOnMain()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/loop.go:60 +0x1b9
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).createWindow()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/window.go:936 +0x1ae
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).CreateWindow()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/window.go:928 +0x45
  fyne.io/fyne/v2/app.(*fyneApp).NewWindow()
      /Users/changkun/dev/changkun.de/fyne/app/app.go:63 +0x57
  main.main()
      /Users/changkun/dev/changkun.de/fyne/cmd/hello/main.go:12 +0x46

Goroutine 29 (running) created at:
  fyne.io/fyne/v2/internal/animation.(*Runner).runAnimations()
      /Users/changkun/dev/changkun.de/fyne/internal/animation/runner.go:67 +0xda
  fyne.io/fyne/v2/internal/animation.(*Runner).Start()
      /Users/changkun/dev/changkun.de/fyne/internal/animation/runner.go:27 +0x349
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).StartAnimation()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/animation.go:6 +0x4a
  fyne.io/fyne/v2.(*Animation).Start()
      /Users/changkun/dev/changkun.de/fyne/animation.go:58 +0x41
  fyne.io/fyne/v2/widget.(*Button).tapAnimation()
      /Users/changkun/dev/changkun.de/fyne/widget/button.go:214 +0x84
  fyne.io/fyne/v2/widget.(*Button).Tapped()
      /Users/changkun/dev/changkun.de/fyne/widget/button.go:190 +0x44
  fyne.io/fyne/v2/internal/driver/glfw.(*window).mouseClickedHandleTapDoubleTap.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/window.go:634 +0x4d
  fyne.io/fyne/v2/internal/driver/common.(*Window).RunEventQueue()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/common/window.go:35 +0x76
  fyne.io/fyne/v2/internal/driver/glfw.(*gLDriver).createWindow.func1.1()
      /Users/changkun/dev/changkun.de/fyne/internal/driver/glfw/window.go:942 +0x39
==================
Found 5 data race(s)
</details>

Since programs that have data races are usually considered as invalid programs (because it is difficult to discuss what would be an expected behavior), we should resolve the data races that appears in the fyne internal to guarantee a fyne program is data race free.

## Challenges

Resolve the data race in the current code base is not a simple task, and we have to tackle the following design goal:

1. **Internal data race**: The internal rendering process in fyne do not introduce any data races.
2. **External data race**: Public APIs that allows users to invoke should be concurrent-safe, especially for UI widgets.

To elaborate why these design goals are crucial, let's visit two simple examples.

## Example 1: Internal Data Races - From Animation

The internal data race can be reproduced in from the [hello](https://github.com/fyne-io/fyne/blob/develop/cmd/hello/main.go) example:

```go
package main

import (
	"fyne.io/fyne/v2/app"
	"fyne.io/fyne/v2/container"
	"fyne.io/fyne/v2/widget"
)

func main() {
	a := app.New()
	w := a.NewWindow("Hello")

	hello := widget.NewLabel("Hello Fyne!")
	w.SetContent(container.NewVBox(
		hello,
		widget.NewButton("Hi!", func() {
			hello.SetText("Welcome :)")
		}),
	))

	w.ShowAndRun()
}
```

In this example, the program creates a `Label` widget and may be changed whenever a user clicks the "Hi!" button. The label alert is done via a registered call back function passed into the `widget.NewButton` function.

However, the actual reported data race proposes something different:

```
WARNING: DATA RACE
Read at 0x00c0000ca2d8 by goroutine 9:
  fyne.io/fyne/v2/internal/painter/gl.(*painter).drawRectangle()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/draw.go:103 +0x75
  fyne.io/fyne/v2/internal/painter/gl.(*painter).drawObject()
      /Users/changkun/dev/changkun.de/fyne/internal/painter/gl/draw.go:145 +0x324
  ...

Previous write at 0x00c0000ca2d8 by goroutine 29:
  fyne.io/fyne/v2/widget.newButtonTapAnimation.func1()
      /Users/changkun/dev/changkun.de/fyne/widget/button.go:431 +0x1b2
  fyne.io/fyne/v2/internal/animation.(*Runner).tickAnimation()
      /Users/changkun/dev/changkun.de/fyne/internal/animation/runner.go:131 +0x5bd
  fyne.io/fyne/v2/internal/animation.(*Runner).runAnimations.func1()
      /Users/changkun/dev/changkun.de/fyne/internal/animation/runner.go:75 +0x2d1
```

In `internal/painter/gl/draw.go`, a read behavior reports on this if statement:

```go
func (p *painter) drawRectangle(rect *canvas.Rectangle, pos fyne.Position, frame fyne.Size) {
	if (rect.FillColor == color.Transparent || rect.FillColor == nil) && (rect.StrokeColor == color.Transparent || rect.StrokeColor == nil || rect.StrokeWidth == 0) {
		return
	}
	...
}
```

and in `widget/button.go`, a write behavior reports the `bg.FillColor` is racing with the above read:

```go
func newButtonTapAnimation(bg *canvas.Rectangle, w fyne.Widget) *fyne.Animation {
	return fyne.NewAnimation(canvas.DurationStandard, func(done float32) {
		...
		bg.FillColor = &color.NRGBA{R: uint8(r), G: uint8(g), B: uint8(bb), A: fade}
		canvas.Refresh(bg)
	})
}
```

A little bit of background regarding fyne's animation system: widgets animations are cached in `internal/cache.renderers`. When a widget is created, its renderer will be cached into this map lazily. When a user clicks the button, the background click animation is then driven by a shared `canvas.Rectangle`, which is reused for a `fyne.Animation`. When the animation starts to render, it ticks the animation in a separate goroutine.

```go
func (r *Runner) runAnimations() {
	draw := time.NewTicker(time.Second / 60)

	go func() {
		for done := false; !done; {
			<-draw.C
			r.animationMutex.Lock()
			oldList := r.animations
			r.animationMutex.Unlock()
			newList := make([]*anim, 0, len(oldList))
			for _, a := range oldList {
				if !a.isStopped() && r.tickAnimation(a) { // <- HERE
					newList = append(newList, a)
				}
...
```

This means our current implementation of animation accesses widget properties in a separate thread, which creates potential data races between the animation goroutine (in a different thread), and the current rendering thread.

To address this type of internal data race, we have multiple different solutions:

1. Putting every property with internal locks, be careful about all implementation requirements.
2. Replicate the rendering state for every animation frame, this will require us to design a mechanism to copy the state of an arbitrary widget (ideally using atomics, but also very complex to make it correct).
3. Redesign the animation system (in this example).

### Example 2: Misuse or Difficult to Use Correctly from the Call side

Apart from internal data races, another design goal for addressing data races is to guarantee that users won't be able to race with our rendering thread. If a user manipulates a widget property in a separate goroutine, there is no way for Fyne's runtime to resolve this data race. For example, in the following snippet, the user creates a collection of cells that updates their background color in a separate goroutine by manipulating the public field of a `canvas.Rectangle`:


```go
package main

import (
	"image/color"
	"log"
	"time"

	"fyne.io/fyne/v2"
	"fyne.io/fyne/v2/app"
	"fyne.io/fyne/v2/canvas"
	"fyne.io/fyne/v2/container"
	"fyne.io/fyne/v2/layout"
)

func main() {
	win := app.New().NewWindow("Some Window")
	w, h := 90, 25
	cells := make([]fyne.CanvasObject, 0, w*h)
	for i := 0; i < w*h; i++ {
		cells = append(cells, canvas.NewRectangle(color.RGBA{R: 255, G: 255, B: 255, A: 128}))
	}
	container := container.New(layout.NewGridLayout(h), cells...)
	colors := [3]color.RGBA{
		{R: 20, G: 30, B: 100, A: 128},
		{R: 100, G: 20, B: 30, A: 128},
		{R: 30, G: 100, B: 20, A: 128},
	}

	go func() {
		c := 0
		tk := time.NewTicker(100 * time.Millisecond)
		for range tk.C {
			for i := 0; i < w; i++ {
				for j := 0; j < h; j++ {
					obj := container.Objects[i*h+j].(*canvas.Rectangle)
					obj.FillColor = colors[c] // Chaging property in an external goroutine
					obj.Refresh()
				}
			}
			log.Println("still refreshing")
			c = (c + 1) % 3
		}
	}()
	win.SetContent(container)
	win.ShowAndRun()
}
```

The `canvas.Rectangle`'s `FillColor` property will be accessed by the rendering thread for sure (at least a read behavior). But as we can notice here, when the user is concurrently writing into this field, we can't properly prevent data race, even by replicating the widget properties (because a copy of the widget properties is still a read operation).

With the above two examples, unlike single-threaded JavaScript DOM rendering in a browser, we could summarize the overall technical goal for designing concurrent-safe UI framework APIs:

1. Customizable widget properties should return the latest observed state for a read operation (e.g. `rect.FillColor == nil`)
2. Customizable widget properties should guarantee it is concurrent-safe when changing one or a list of properties.

If we implement the first goal, this will be beneficial for users and automatically resolves the need for our rendering thread, which will read all changed widgets per frame.

Implementing the second goal will be beneficial and complement the first goal so that we don't have to worry about questions like "is it safe to read the property?" It will also leave us room to design future mechanisms that need to utilize Go's concurrent nature.


## Proposed Design


> This design introduces massive breaking changes, which can only be done in fyne/v3. It might also not be a wise choice if we maintain v2 and v3 together as they would expectedly work differently.

As we have discussed, the general goals and technical goals. **I propose using functional options** and redesigning the `widget`/`canvas`/etc related package. The general idea behind functional options can refer to these previous articles (originally by [Rob Pike](https://commandcenter.blogspot.com/2014/01/self-referential-functions-and-design.html), and advertised by [Dave Cheney](https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis).

Take `widget.Button` as an example. The proposed design will lead us to do the following:

1. Remove all public fields of user-customizable widgets, renderable objects, etc.
2. Define widget-specific options for every widget
3. Provide concurrent-safe getters to allow users to fetch widget properties at any goroutine.

```go
package widget

type Button struct {
    // Contains unexported fields
}

func NewButton(opts ...ButtonOption) *Button { ... }

// Define widget option, and Options method to update the options.
type ButtonOption func(b *Button)
func (b *Button) Options(opts ...ButtonOption) { ... }

// Possible Button options to configure the properties of a button
func ButtonText(s string) ButtonOption { return func(b *Button) { ... } }
func ButtonTap(f func())  ButtonOption { return func(b *Button) { ... } }

// Concurrent-safe getters
func Text() string { ... }
func Icon() fyne.Resource { ... }
...
```

### Advantages

The following elaborates on a few advantages, but not limited to, of switching to a functional option pattern.

#### Avoid multiple constructors

With functional option design, we could avoid APIs like this:

```go
func NewButton(label string, tapped func()) *Button
func NewButtonWithIcon(label string, icon fyne.Resource, tapped func()) *Button
```

When we offered Button options like `func ButtonTapped(f func()) ButtonOption`, `func ButtonIcon(icon fyne.Resource) ButtonOption`, etc. They can be used as a parameter of the `NewButton(opts ...Option)`. There is only a single constructor for creating a button.

Similar examples can be found in other widgets, such as `NewCheck`, `NewCheckWithData`, `NewEntry`, `NewPasswordEntry`, etc.

### Sometimes backward compatible

Using functional options requires us to add an `...` parameter into the function parameter list.
For example, in the `canvas.NewRectangle`, a rectangle's constructor is:

```go
func NewRectangle(color color.Color) *Rectangle // before
```

Luckily, it is not required to provide `...` arguments, for the new signature:

```go
func NewRectangle(c color.Color, opts ...RectangleOption) *Rectangle // after
```

Code like this will continue to work:

```go
canvas.NewRectangle(color.Black)
```

### Batch Update

Functional options not only unlock us the ability to have a unified API for the constructor, but it is also beneficial for us to update a batch of properties that only requires locking the widget one time. For example, passing just one option to the `Options` method will behave the same as a setter (acquire the lock once). However, if we give 100 options, it still only needs to acquire the lock once, but setters will have to lock the button 100 times.

```go
// Options updates the button properties.
func (r *Button) Options(opts ...ButtonOption) {
	// This gives us the opportunity to update any given widget property with a single unified API.
	// When there are multiple given options, we could just lock the widget once, rather than
	// invoke N different setters that each require a write lock for the entire widget.
	r.mu.Lock()
	defer r.mu.Unlock()

	for _, opt := range opts {
		opt(r)
	}
}
```

### Potential Downside

There are several potential downsides, but I do not yet know how to evaluate these downsides, they are just hypothetical in my mind.

#### Verbose Usage

The first identified down side for using `Option` pattern is that the API usage may be verbose, such as:

```go
s.track.FillColor = theme.ShadowColor()                         // Before
s.track.Options(canvas.RectangleFillColor(theme.ShadowColor())) // After
```

This type of embarassing may be resolved by separating different widgets into different sub-packages,
such as:

```go
package rect

type Rect struct { ... }
type Option func(r *Rect)

func New(opts ...Option) *Rect {...}

func (r *Rect) Options(opts ...Option) { ... }
func FillColor(c color.Color) Option { ... }
```

On the call side, it may remain simple enough:

```
s.track.FillColor = theme.ShadowColor()              // Before
s.track.Options(rect.FillColor(theme.ShadowColor())) // After
```

However, we don't know if the separation of widgets is feasible and whether it will introduce circular imports or not.

#### Require Additional Documentation

Another downside of switching to `Option` is that the affordance of an option is comparably little. According to the past experiences of functional option patterns, the user reports that unless with good documentation, people are generally difficult to understand the possible options that could be used for a given function that ends with `...Option` in its parameter list.

#### Maybe Deadlock?

If a rendering thread and a property setter both waiting for a lock that they own, we may encounter deadlock. Although this may be resolved by using conditional variables, it introduces another level of complexity in the implementation.

### Future Changes

As Go's generics are out already for a release, we should also consider how generics may change the functional option pattern. I have already investigated this and wrote an additional article about this, see https://golang.design/research/generic-option/.

Considering the design of the current generics is in a very early stage, I would suggest we forget about generics.

## Implementation

I prototyped a very minimum implementation to use the Options pattern for the `canvas.Rectangle`. See https://github.com/changkun/fyne/commit/46a9b1bc76071d8df00ec08f172ce8dbd2dd014a.

This prototype fixes all data races in the `hello` example.

Note that this change is only for discussion and to understand the potential impact on the existing fyne code base.

## Prior art

Before writing this proposal, I've privately tackled different attempts.

### Why not just use sync.Mutex and lock every property?

Adding mutex to widgets in order to fix the internal data races. However, as we elaborated in the background section, this attempt cannot resolve the data race on the user side.

### Why not just use atomics as much as possible? Why not just getter/setter for every property?

Use atomics as much as possible and add a locked getter/setter to every property. However, it turns out, that having a getter/setter for every different property is very costly, especially when we wanted to update a batch of properties for a widget. In this case, a setter will require to lock a widget once, and configuring N different properties will invoke the lock's Lock N times. It seems that having an `Options(opt ...Option)` that can configure the widget properties at once is more wisely. See previous discussions regarding the advantages of functional option patterns.

### Early attempts

These early attempts can be found in these places:

- A chain of commits: https://github.com/changkun/fyne/tree/data-race
- Another chain of commits: https://github.com/changkun/fyne/commits/race-draft
- A series of discussions and observations: https://github.com/fyne-io/fyne/issues/2509
- Many many locally reverted (overnight) changes, which are gone after causing a huge mess...