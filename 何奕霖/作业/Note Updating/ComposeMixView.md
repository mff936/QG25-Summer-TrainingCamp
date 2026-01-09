# Android 项目：Compose 与 View 混用快速指南

本项目演示了如何在一个新建的 Empty Views Activity 项目中，混合使用 Jetpack Compose 和传统 View，并成功运行到手机上。

---

## 步骤一：新建 Empty Views Activity 项目

1.  使用 Android Studio 新建项目，选择 **Empty Views Activity** 模板。
2.  选择 Kotlin 语言，设置合适的包名和最低 SDK。

---

## 步骤二：配置 Compose 支持

### 1. 在 `app/build.gradle.kts` 中启用 Compose

确保 `buildFeatures` 中启用了 `compose`，并设置了 `composeOptions`。

```kotlin
// app/build.gradle.kts
android {
    //...
    buildFeatures {
        compose = true
    }
    composeOptions {
        kotlinCompilerExtensionVersion = "1.5.11" // 版本需与Kotlin版本兼容
    }
}
```

### 2. 添加 Compose 依赖

在 `dependencies` 中，使用 `compose-bom` 来管理版本，并添加必要的 Compose 库。

```kotlin
// app/build.gradle.kts
dependencies {
    // ...
    implementation(platform("androidx.compose:compose-bom:2023.10.01"))
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.material3:material3")
    implementation("androidx.compose.ui:ui-tooling-preview")
    debugImplementation("androidx.compose.ui:ui-tooling")
    implementation(libs.androidx.activity.compose)
    implementation(libs.androidx.lifecycle.viewmodel.compose)
}
```

### 3. 确保仓库配置正确

在项目根目录的 `settings.gradle.kts` 中，确保 `dependencyResolutionManagement` 包含了 `google()` 和 `mavenCentral()`。

```kotlin
// settings.gradle.kts
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
    }
}
```

---

## 步骤三：在布局文件中添加 ComposeView

在你的 XML 布局文件（如 `res/layout/activity_main.xml`）中，像普通 View 一样添加 `ComposeView`。

```xml
<!-- res/layout/activity_main.xml -->
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <!-- 这是一个传统的View -->
    <Button
        android:id="@+id/normalBtn"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="正常按钮" />

    <!-- 这是Compose内容的容器 -->
    <androidx.compose.ui.platform.ComposeView
        android:id="@+id/compose_view"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1" />

</LinearLayout>
```

---

## 步骤四：在 Activity 中设置 ComposeView 内容

在 `MainActivity.kt` 的 `onCreate` 方法中，获取 `ComposeView` 并为其设置 Compose 内容。

```kotlin
// MainActivity.kt
import androidx.compose.ui.platform.ComposeView
import androidx.compose.ui.platform.ViewCompositionStrategy

class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val composeView : ComposeView = findViewById(R.id.compose_view)

        // 设置ComposeView内容
        composeView.apply {
            // 设置Compose的生命周期策略
            setViewCompositionStrategy(ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed)
            // 设置Compose UI内容
            setContent {
                SimpleList(listOf("Item 1", "Item 2", "Item 3", "Item 4"))
            }
        }
    }
}

// 定义一个Composable函数
@Composable
fun SimpleList(items: List<String>) {
    LazyColumn {
        items(items) { item ->
            Text(
                text = item,
                modifier = Modifier.fillMaxWidth().padding(16.dp)
            )
        }
    }
}
```

---

## 步骤五：同步 Gradle 并运行

1.  点击 Android Studio 右上角的 "Sync Now" 或大象图标，同步 Gradle。
2.  连接手机或启动模拟器，点击 "Run" 即可看到 Compose 和 View 混用的效果。

---

## 常见问题

-   **无法下载 Compose Compiler**：这是网络问题。请确保你的网络可以访问 Google Maven 仓库 (`https://dl.google.com/dl/android/maven2/`)。
-   **版本兼容问题**：Compose Compiler 的版本必须与 Kotlin 版本兼容。请参考 [官方兼容性列表](https://developer.android.com/jetpack/androidx/releases/compose-kotlin)。

