createSubDecor

windowNoTitle == false mIsFloating == true

subDecor = abc_dialog_title_material

```xml
<androidx.appcompat.widget.FitWindowsLinearLayout
        xmlns:android="http://schemas.android.com/apk/res/android"
        android:layout_height="match_parent"
        android:layout_width="match_parent"
        android:orientation="vertical"
        android:fitsSystemWindows="true">

    <TextView
            android:id="@+id/title"
            style="?android:attr/windowTitleStyle"
            android:singleLine="true"
            android:ellipsize="end"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_gravity="start"
            android:textAlignment="viewStart"
            android:paddingLeft="?attr/dialogPreferredPadding"
            android:paddingRight="?attr/dialogPreferredPadding"
            android:paddingTop="@dimen/abc_dialog_padding_top_material"/>

    <include
            layout="@layout/abc_screen_content_include"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_weight="1"/>

</androidx.appcompat.widget.FitWindowsLinearLayout>

<merge xmlns:android="http://schemas.android.com/apk/res/android">

    <androidx.appcompat.widget.ContentFrameLayout
            android:id="@id/action_bar_activity_content"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:foregroundGravity="fill_horizontal|top"
            android:foreground="?android:attr/windowContentOverlay" />

</merge>

```

windowNoTitle == false mIsFloating == false
```xml
<androidx.appcompat.widget.ActionBarOverlayLayout
        xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:app="http://schemas.android.com/apk/res-auto"
        android:id="@+id/decor_content_parent"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:fitsSystemWindows="true">

    <include layout="@layout/abc_screen_content_include"/>

    <androidx.appcompat.widget.ActionBarContainer
            android:id="@+id/action_bar_container"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_alignParentTop="true"
            style="?attr/actionBarStyle"
            android:touchscreenBlocksFocus="true"
            android:gravity="top">

        <androidx.appcompat.widget.Toolbar
                android:id="@+id/action_bar"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                app:navigationContentDescription="@string/abc_action_bar_up_description"
                style="?attr/toolbarStyle"/>

        <androidx.appcompat.widget.ActionBarContextView
                android:id="@+id/action_context_bar"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:visibility="gone"
                android:theme="?attr/actionModeTheme"
                style="?attr/actionModeStyle"/>

    </androidx.appcompat.widget.ActionBarContainer>

</androidx.appcompat.widget.ActionBarOverlayLayout>
```