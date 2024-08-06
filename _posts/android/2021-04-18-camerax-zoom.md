---
title: "CameraX 줌 관련 사용 방법"

toc: true
toc_sticky: true

categories:
    - Android
tags:
    - Android
    - Kotlin
    - CameraX
---

처음으로 내 의지로 만든 프로그램도 무언가를 함에 있어 불편해서 직접 만들게 되었다. 이번에도 필요함에 의해 프로토타입의(?) 앱을 만들게 되었다. 그런데 개발 초보자가 다양한 API를 사용하기에는 어려운 부분들이 많았다. 
특히 확대, 축소 API를 구현하는데만 일주일이 걸렸다. 사실 해결하고 보니 하루도 안걸릴 문제였는데, 리스너를 등록하는 것은 꼭 `onCreate`에 들어가야 한다고 생각했기 때문이다.

# CameraX 
### 기본적인 코드
>https://codelabs.developers.google.com/codelabs/camerax-getting-started

이 곳을 보고 하면 된다. 구 버전의 API를 기준으로 작성하여 한두개 없는 메소드가 있는데 구글에 검색하면 스택오버플로우에 바로 뜬다.

## SeekBar를 이용한 줌 제어
>https://stackoverflow.com/questions/56057310/how-to-zoom-camera-using-android-camerax-api

이 곳을 참고하여 작성하였다.

처음에는 `cameraControl`를 `onCreat`로 가지고 와야한다고 생각했다. 그래서 전역변수니 뭐니 엄청 삽질을 많이 했지만 해결 방법은 단순했다.

```kotlin
private fun startCamera() {
        val cameraProviderFuture = ProcessCameraProvider.getInstance(this)

        cameraProviderFuture.addListener(Runnable {
            val cameraProvider: ProcessCameraProvider = cameraProviderFuture.get()

            val preview = Preview.Builder().build().also {
                it.setSurfaceProvider(viewFinder.surfaceProvider)
            }

            val cameraSelector = CameraSelector.DEFAULT_BACK_CAMERA

            try {
                cameraProvider.unbindAll()
                cameraProvider.bindToLifecycle(this, cameraSelector, preview)
                val cameraControl = cameraProvider.bindToLifecycle(
                    this, cameraSelector, preview, imageCapture
                )

                //줌 리스너 추
                seekBar.setOnSeekBarChangeListener(object : SeekBar.OnSeekBarChangeListener {
                    override fun onProgressChanged(
                        seekBar: SeekBar?,
                        progress: Int,
                        fromUser: Boolean
                    ) {
                        cameraControl.cameraControl.setLinearZoom(progress / 100.toFloat())
                    }

                    override fun onStartTrackingTouch(seekBar: SeekBar?) {}
                    override fun onStopTrackingTouch(seekBar: SeekBar?) {}
                })
            } catch (exc: Exception) {
                Log.e(TAG, "Use case binding faild", exc)
            }
        }, ContextCompat.getMainExecutor(this))

        imageCapture = ImageCapture.Builder().build()
    }
```
`SeekBarChangeListener`를 `cameraStart()`에 등록하는 것이다. 어차피 앱을 실행하면 `cameraStart()`가 실행될 것이고, 어차피 카메라를 사용할 때에만 사용하니 카메라 기능이 실행될 때 등록해버리는 것이다.
