```kotlin
@SuppressLint("WrongConstant")
private fun setNavigationBarThemeMode(@Suppress("SameParameterValue") nightMode: Boolean) {
    Log.d(TAG, "${Build.VERSION.SDK_INT}")
    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.R) {
        var systemUiVisibility = window.decorView.systemUiVisibility
        if (nightMode) {
            systemUiVisibility = systemUiVisibility and View.SYSTEM_UI_FLAG_LIGHT_NAVIGATION_BAR
        } else {
            systemUiVisibility = systemUiVisibility or View.SYSTEM_UI_FLAG_LIGHT_NAVIGATION_BAR.inv()
        }
        window.decorView.systemUiVisibility = systemUiVisibility
    } else {
        val controller = window.insetsController
        controller?.let {
            Log.d(TAG, "systemBar: ${it.systemBarsAppearance}")
            Log.d(TAG, "systemBar: ${it.systemBarsBehavior}")
            if (systemBarVisibility) {
                it.hide(
                    WindowInsets.Type.navigationBars()
                            or WindowInsets.Type.statusBars()
                            or WindowInsets.Type.captionBar()
                )
            } else {

            }
            Log.d(TAG, "systemBarAppearance: " + controller.systemBarsAppearance)
            Log.d(TAG, "systemBarBehavior: " + controller.systemBarsBehavior)
        }
    }
}
```