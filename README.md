# WebView for JetBrains Compose Multiplatform

[![Maven Central](https://img.shields.io/maven-central/v/io.github.kevinnzou/compose-webview-multiplatform.svg)](https://search.maven.org/artifact/io.github.kevinnzou/compose-webview-multiplatform)

<img src="media/cmm-webview-sample.png" height="550">

This library can be considered as the Multiplatform version of [Accompanist Web library](https://github.com/google/accompanist/tree/main/web). 
It provides the basic WebView functionalities for JetBrains Compose Multiplatform, which supports loading URLs, HTML, and post data. Currently, it supports the platforms of Android, iOS, and Desktop.

The Android implementation of this library relies on the web module from the [Accompanist Library](https://github.com/google/accompanist/tree/main/web). However, it has been deprecated in version 0.33.1-alpha. 
Thus I created a fork of it and used it as the base for this lirbary. If you just want to use the WebView in Jetpack Compose, please visit this repo: https://github.com/KevinnZou/compose-webview.

The iOS implementation of this library relies on [WKWebView](https://developer.apple.com/documentation/webkit/wkwebview).

The Desktop implementation of this library relies on [JavaFX WebView](https://docs.oracle.com/javase/8/javafx/api/javafx/scene/web/WebView.html).


## Basic Usage
To use this widget there are two key APIs that are needed: *WebView*, which provides the layout, and *rememberWebViewState(url)* which provides some remembered state including the URL to display.

The basic usage is as follows:
```kotlin
val state = rememberWebViewState("https://example.com")

WebView(state)
```
This will display a WebView in your Compose layout that shows the URL provided.

## WebView State
This library provides a *WebViewState* class as a state holder to hold the state for the WebView. 
```kotlin
public class WebViewState(webContent: WebContent) {
    public var lastLoadedUrl: String? by mutableStateOf(null)
        internal set

    /**
     *  The content being loaded by the WebView
     */
    public var content: WebContent by mutableStateOf(webContent)

    /**
     * Whether the WebView is currently [LoadingState.Loading] data in its main frame (along with
     * progress) or the data loading has [LoadingState.Finished]. See [LoadingState]
     */
    public var loadingState: LoadingState by mutableStateOf(LoadingState.Initializing)
        internal set

    /**
     * Whether the webview is currently loading data in its main frame
     */
    public val isLoading: Boolean
        get() = loadingState !is Finished

    /**
     * The title received from the loaded content of the current page
     */
    public var pageTitle: String? by mutableStateOf(null)
        internal set

    /**
     * the favicon received from the loaded content of the current page
     */
    public var pageIcon: Bitmap? by mutableStateOf(null)
        internal set

    /**
     * A list for errors captured in the last load. Reset when a new page is loaded.
     * Errors could be from any resource (iframe, image, etc.), not just for the main page.
     * For more fine grained control use the OnError callback of the WebView.
     */
    public val errorsForCurrentRequest: SnapshotStateList<WebViewError> = mutableStateListOf()

    /**
     * The saved view state from when the view was destroyed last. To restore state,
     * use the navigator and only call loadUrl if the bundle is null.
     * See WebViewSaveStateSample.
     */
    public var viewState: Bundle? = null
        internal set

    // We need access to this in the state saver. An internal DisposableEffect or AndroidView
    // onDestroy is called after the state saver and so can't be used.
    internal var webView by mutableStateOf<WebView?>(null)
}
```
It can be created using the *rememberWebViewState* function, which can be remembered across Compositions.
```kotlin
val state = rememberWebViewState("https://github.com/KevinnZou/compose-webview-multiplatform")

/**
 * Creates a WebView state that is remembered across Compositions.
 *
 * @param url The url to load in the WebView
 * @param additionalHttpHeaders Optional, additional HTTP headers that are passed to [WebView.loadUrl].
 *                              Note that these headers are used for all subsequent requests of the WebView.
 */
@Composable
public fun rememberWebViewState(
    url: String,
    additionalHttpHeaders: Map<String, String> = emptyMap()
): WebViewState =
// Rather than using .apply {} here we will recreate the state, this prevents
    // a recomposition loop when the webview updates the url itself.
    remember {
        WebViewState(
            WebContent.Url(
                url = url,
                additionalHttpHeaders = additionalHttpHeaders
            )
        )
    }.apply {
        this.content = WebContent.Url(
            url = url,
            additionalHttpHeaders = additionalHttpHeaders
        )
    }
```
Developers can use the *WebViewState* to get the loading information of the WebView, such as the loading progress, the loading status, and the URL of the current page.
```kotlin
Column {
    val state = rememberWebViewState("https://github.com/KevinnZou/compose-webview-multiplatform")

    Text(text = "${state.pageTitle}")
    val loadingState = state.loadingState
    if (loadingState is LoadingState.Loading) {
        LinearProgressIndicator(
            progress = loadingState.progress,
            modifier = Modifier.fillMaxWidth()
        )
    }
    WebView(
        state
    )
}
```

## WebView Navigator
This library provides a *WebViewNavigator* class to control over the navigation of a WebView from outside the composable. E.g.for performing a back navigation in response to the user clicking the "up" button in a TopAppBar.
It can be used to load a new URL, reload the current URL, and go back and forward in the history.

```kotlin
class WebViewNavigator(private val coroutineScope: CoroutineScope) {

    /**
     * True when the web view is able to navigate backwards, false otherwise.
     */
    var canGoBack: Boolean by mutableStateOf(false)
        internal set

    /**
     * True when the web view is able to navigate forwards, false otherwise.
     */
    var canGoForward: Boolean by mutableStateOf(false)
        internal set

    fun loadUrl(url: String, additionalHttpHeaders: Map<String, String> = emptyMap()) {}

    fun loadHtml(
        html: String,
        baseUrl: String? = null,
        mimeType: String? = null,
        encoding: String? = "utf-8",
        historyUrl: String? = null
    ) {
    }

    fun postUrl(
        url: String,
        postData: ByteArray
    ) {
    }

    /**
     * Navigates the webview back to the previous page.
     */
    fun navigateBack() {}

    /**
     * Navigates the webview forward after going back from a page.
     */
    fun navigateForward() {}

    /**
     * Reloads the current page in the webview.
     */
    fun reload() {}

    /**
     * Stops the current page load (if one is loading).
     */
    fun stopLoading() {}
}
```
It can be created using the *rememberWebViewNavigator* function, which can be remembered across Compositions.
```kotlin
val navigator = rememberWebViewNavigator()

@Composable
public fun rememberWebViewNavigator(
    coroutineScope: CoroutineScope = rememberCoroutineScope()
): WebViewNavigator = remember(coroutineScope) { WebViewNavigator(coroutineScope) }
```
Developers can use the *WebViewNavigator* to control the navigation of the WebView.
```kotlin
val navigator = rememberWebViewNavigator()

Column {
    val state = rememberWebViewState("https://example.com")
    val navigator = rememberWebViewNavigator()

    TopAppBar(
        title = { Text(text = "WebView Sample") },
        navigationIcon = {
            if (navigator.canGoBack) {
                IconButton(onClick = { navigator.navigateBack() }) {
                    Icon(
                        imageVector = Icons.Default.ArrowBack,
                        contentDescription = "Back"
                    )
                }
            }
        }
    )
    Text(text = "${state.pageTitle}")
    val loadingState = state.loadingState
    if (loadingState is LoadingState.Loading) {
        LinearProgressIndicator(
            progress = loadingState.progress,
            modifier = Modifier.fillMaxWidth()
        )
    }
    WebView(
        state = state,
        navigator = navigator
    )
}
```

## API
The complete API of this library is as follows:
```kotlin
/**
 *
 * A wrapper around the Android View WebView to provide a basic WebView composable.
 *
 * @param state The webview state holder where the Uri to load is defined.
 * @param modifier A compose modifier
 * @param captureBackPresses Set to true to have this Composable capture back presses and navigate
 * the WebView back.
 * @param navigator An optional navigator object that can be used to control the WebView's
 * navigation from outside the composable.
 * @param onCreated Called when the WebView is first created.
 * @param onDispose Called when the WebView is destroyed.
 * @sample sample.BasicWebViewSample
 */
@Composable
fun WebView(
    state: WebViewState,
    modifier: Modifier = Modifier,
    captureBackPresses: Boolean = true,
    navigator: WebViewNavigator = rememberWebViewNavigator(),
    onCreated: () -> Unit = {},
    onDispose: () -> Unit = {},
)
```

## Example
A simple example would be like this:
```kotlin
@Composable
internal fun WebViewSample() {
    MaterialTheme {
        val webViewState = rememberWebViewState("https://github.com/KevinnZou/compose-webview-multiplatform")
        Column(Modifier.fillMaxSize()) {
            val text = webViewState.let {
                "${it.pageTitle ?: ""} ${it.loadingState} ${it.lastLoadedUrl ?: ""}"
            }
            Text(text)
            WebView(
                state = webViewState,
                modifier = Modifier.fillMaxSize()
            )
        }

    }
}
```
For full example, please refers to [BasicWebViewSample](https://github.com/KevinnZou/compose-webview-multiplatform/blob/main/webview/src/commonMain/kotlin/sample/BasicWebViewSample.kt)

## Download

[![Maven Central](https://img.shields.io/maven-central/v/io.github.kevinnzou/compose-webview-multiplatform.svg)](https://search.maven.org/artifact/io.github.kevinnzou/compose-webview-multiplatform)

```groovy
repositories {
    mavenCentral()
}

dependencies {
    implementation "io.github.kevinnzou:compose-webview-multiplatform:1.0.0"
}
```
