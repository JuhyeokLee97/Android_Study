# 왜 Application에서 `ImageLoaderFactory`를 구현할까?
Now In Android의 `NiaApplication`에서는 다음과 같이 Coil의 `ImageLoaderFactory`를 상속받고 있다.
``` kotlin
@HiltAndroidApp
class NiaApplication : Application(), ImageLoaderFactory {
    @Inject
    lateinit var imageLoader: dagger.Lazy<ImageLoader>
    ...
}
```

과연 왜 NiaApplication에서는 `ImageLoaderFactory`를 상속받아서 `newImageLoader()` 함수를 override 했을까?

## 문제 상황
Coil 이미지 로딩 라이브러리를 사용할 때, 기본 설정만으로는 **SVG 파일**을 로드할 수 없다. Now In Android 앱에서는 뉴스 아이콘 등에 SVG 포맷을 사용하므로 **커스텀 설정**이 필요한 것이다.

### Coil에서 SVG 파일을 로드하는 방법
1. `"io.coil-kt.coil3:coil-svg:$version"` 의존성 추가
2. Coil의 ImageLoader를 생성하고 components에 `SvgDecoder.Factory()` 전달

## Coil의 초기화 매커니즘

### Coil이 ImageLoader를 가져오는 방법
Jetpack Compose에서 이미지를 로드할 때, 내부적으로 Coil은 다음과 같이 ImageLoader를 생성한다.

``` kotlin
// Coil.kt
fun imageLoader(context: Context): ImageLoader {
    return imageLoader ?: newImageLoader(context)
}

private fun newImageLoader(context: Context): ImageLoader {
    // Check again in case imageLoader was just set.
    imageLoader?.let { return it }

    // Create a new ImageLoader.
    val newImageLoader = imageLoaderFactory?.newImageLoader()
        ?: (context.applicationContext as? ImageLoaderFactory)?.newImageLoader()
        ?: ImageLoader(context)
    imageLoaderFactory = null
    imageLoader = newImageLoader
    return newImageLoader
}
```

Coil은 Application이 `ImageLoaderFactory`를 구현했는지 확인하고 구현된 것이 없으면 기본 `ImageLoader`를 사용한다.

## Now In Android의 해결 방법
### 1. Application에서 ImageLoaderFactory 구현
``` kotlin
// NiaApplication.kt
@HiltAndroidApp
class NiaApplication : Application(), ImageLoaderFactory {
    @Inject
    lateinit var imageLoader: dagger.Lazy<ImageLoader>

    override fun newImageLoader(): ImageLoader = imageLoader.get()
}
```

### 2. Dagger로 커스텀 ImageLoader 제공

```kotlin
// NetworkModule.kt
@Provides
@Singleton
fun imageLoader(
    okHttpCallFactory: dagger.Lazy<Call.Factory>,
    @ApplicationContext application: Context,
): ImageLoader = ImageLoader.Builder(application)
    .callFactory { okHttpCallFactory.get() }
    .components { add(SvgDecoder.Factory()) }  // ← SVG 지원!
    .respectCacheHeaders(false)
    .apply {
        if (BuildConfig.DEBUG) {
            logger(DebugLogger())
        }
    }
    .build()
```

## 동작 흐름

```
1. 앱에서 AsyncImage 사용
   AsyncImage(model = "icon.svg")
   ↓
2. Coil이 ImageLoader 필요
   ↓
3. Application 확인
   NiaApplication이 ImageLoaderFactory인가? → Yes
   ↓ 
4. 커스텀 ImageLoader 반환
   app.newImageLoader()
   ↓
5. Dagger가 제공한 ImageLoader 사용
   - SVG 디코더 포함
   - 커스텀 캐싱 정책
   - 디버그 로깅
   ↓
6. SVG 이미지 로드 성공
```

## 마무리
**왜 Application에서 ImageLoaderFactory를 구현하나?**

1. **Coil의 자동 초기화**: Coil이 Application을 확인하여 커스텀 ImageLoader를 자동으로 사용
2. **전역 설정**: 앱 전체에서 일관된 이미지 로딩 설정 적용
3. **SVG 지원**: SvgDecoder를 추가하여 SVG 파일 로드 가능

**결과:**
- 모든 Composable에서 자동으로 SVG 로드 가능
- 설정 코드 중복 없음