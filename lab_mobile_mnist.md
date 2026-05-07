# Лабораторная работа №7
## Мобильная разработка: Распознавание рукописных цифр с использованием камеры и модели MNIST_TPU на устройстве

---

**Дисциплина:** Разработка мобильных приложений  
**Платформа:** Android (Java / Kotlin)  
**Инструменты:** Android Studio, TensorFlow Lite, CameraX  
**Модель:** MNIST_TPU.h5 → конвертированная в `.tflite`  
**Продолжительность:** 4 академических часа  

---

## Цели и задачи

**Цель работы:** Разработать Android-приложение, которое с помощью камеры телефона фотографирует рукописную цифру, а затем распознаёт её локально на устройстве с использованием нейронной сети MNIST, не передавая данные в интернет.

**Задачи:**
1. Ознакомиться с архитектурой сверточной нейронной сети MNIST.
2. Конвертировать модель `MNIST_TPU.h5` в формат TensorFlow Lite (`.tflite`).
3. Настроить работу с камерой через библиотеку CameraX.
4. Интегрировать модель в приложение и выполнить предобработку изображения.
5. Реализовать вывод результата распознавания в UI.

---

## Теоретическая часть

### 1. Архитектура MNIST

MNIST (Modified National Institute of Standards and Technology) — классический датасет из 70 000 изображений рукописных цифр (0–9), размером 28×28 пикселей в оттенках серого.

Модель `MNIST_TPU.h5` обучена на этом датасете и содержит следующие слои:

```
Input (28×28×1)
  → Conv2D(32 фильтра, 3×3, ReLU)
  → MaxPooling2D(2×2)
  → Conv2D(64 фильтра, 3×3, ReLU)
  → MaxPooling2D(2×2)
  → Flatten
  → Dense(128, ReLU)
  → Dropout(0.5)
  → Dense(10, Softmax)   ← 10 классов (цифры 0–9)
```

**Выход модели:** вектор из 10 значений вероятностей. Индекс максимального значения — распознанная цифра.

### 2. TensorFlow Lite

TensorFlow Lite — облегчённая версия TensorFlow для запуска нейронных сетей **непосредственно на мобильных устройствах** без подключения к серверу. Преимущества:
- Низкая задержка (inference < 10 мс на современных устройствах)
- Конфиденциальность данных — изображение не покидает устройство
- Работа оффлайн

### 3. CameraX

CameraX — Jetpack-библиотека для работы с камерой на Android. Используется `ImageCapture` use case для захвата одиночного кадра в виде `Bitmap`.

---

## Практическая часть

### Шаг 1. Конвертация модели MNIST_TPU.h5 → TFLite

Выполните конвертацию на компьютере (Python + TensorFlow):

```python
import tensorflow as tf

# Загрузка обученной Keras-модели
model = tf.keras.models.load_model("MNIST_TPU.h5")

# Создание конвертера
converter = tf.lite.TFLiteConverter.from_keras_model(model)

# Опциональная оптимизация (квантизация для ускорения на телефоне)
converter.optimizations = [tf.lite.Optimize.DEFAULT]

# Конвертация
tflite_model = converter.convert()

# Сохранение файла
with open("mnist.tflite", "wb") as f:
    f.write(tflite_model)

print("Конвертация завершена. Размер модели:", len(tflite_model), "байт")
```

> **Результат:** файл `mnist.tflite` размером ~80–150 КБ.

---

### Шаг 2. Создание проекта в Android Studio

1. Откройте **Android Studio** → **New Project** → **Empty Activity**.
2. Параметры проекта:
   - Name: `MnistCameraApp`
   - Package: `com.lab.mnistcamera`
   - Language: `Kotlin`
   - Minimum SDK: **API 21 (Android 5.0)**

---

### Шаг 3. Настройка зависимостей (build.gradle)

Откройте файл `app/build.gradle` и добавьте:

```groovy
android {
    ...
    aaptOptions {
        // Запрет на сжатие файла модели
        noCompress "tflite"
    }
}

dependencies {
    // TensorFlow Lite
    implementation 'org.tensorflow:tensorflow-lite:2.13.0'
    implementation 'org.tensorflow:tensorflow-lite-support:0.4.4'

    // CameraX
    def camerax_version = "1.3.1"
    implementation "androidx.camera:camera-core:$camerax_version"
    implementation "androidx.camera:camera-camera2:$camerax_version"
    implementation "androidx.camera:camera-lifecycle:$camerax_version"
    implementation "androidx.camera:camera-view:$camerax_version"

    // Coroutines
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3'
}
```

Синхронизируйте проект: **File → Sync Project with Gradle Files**.

---

### Шаг 4. Добавление файла модели в проект

1. В Android Studio перейдите в дерево проекта (вид **Android**).
2. Раскройте `app/src/main/`.
3. Щёлкните правой кнопкой → **New → Directory** → назовите `assets`.
4. Скопируйте файл `mnist.tflite` в папку `app/src/main/assets/`.

---

### Шаг 5. Разрешения в AndroidManifest.xml

Откройте `app/src/main/AndroidManifest.xml` и добавьте до тега `<application>`:

```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-feature android:name="android.hardware.camera" android:required="true" />
```

---

### Шаг 6. Разметка интерфейса (activity_main.xml)

Замените содержимое файла `res/layout/activity_main.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:gravity="center"
    android:padding="24dp"
    android:background="#121212">

    <!-- Превью камеры -->
    <androidx.camera.view.PreviewView
        android:id="@+id/previewView"
        android:layout_width="300dp"
        android:layout_height="300dp"
        android:layout_marginBottom="24dp"/>

    <!-- Кнопка захвата -->
    <Button
        android:id="@+id/btnCapture"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="📷  Распознать цифру"
        android:textSize="18sp"
        android:paddingHorizontal="32dp"
        android:paddingVertical="14dp"
        android:backgroundTint="#3D5AFE"/>

    <!-- Отображение результата -->
    <TextView
        android:id="@+id/tvResult"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="32dp"
        android:text="Наведите камеру на цифру"
        android:textColor="#FFFFFF"
        android:textSize="22sp"
        android:gravity="center"/>

    <!-- Уверенность предсказания -->
    <TextView
        android:id="@+id/tvConfidence"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="8dp"
        android:textColor="#90CAF9"
        android:textSize="16sp"/>

</LinearLayout>
```

---

### Шаг 7. Класс-классификатор (MnistClassifier.kt)

Создайте новый файл `MnistClassifier.kt` в пакете `com.lab.mnistcamera`:

```kotlin
package com.lab.mnistcamera

import android.content.Context
import android.graphics.Bitmap
import android.graphics.Color
import org.tensorflow.lite.Interpreter
import java.io.FileInputStream
import java.nio.ByteBuffer
import java.nio.ByteOrder
import java.nio.channels.FileChannel

class MnistClassifier(context: Context) {

    private val interpreter: Interpreter

    // Размер входного изображения модели MNIST
    companion object {
        const val IMG_SIZE = 28
        const val NUM_CLASSES = 10
    }

    init {
        // Загрузка модели из assets
        val assetManager = context.assets
        val fileDescriptor = assetManager.openFd("mnist.tflite")
        val inputStream = FileInputStream(fileDescriptor.fileDescriptor)
        val fileChannel = inputStream.channel
        val startOffset = fileDescriptor.startOffset
        val declaredLength = fileDescriptor.declaredLength
        val modelBuffer = fileChannel.map(
            FileChannel.MapMode.READ_ONLY, startOffset, declaredLength
        )
        interpreter = Interpreter(modelBuffer)
    }

    /**
     * Распознаёт цифру на переданном Bitmap.
     * @return Pair(цифра: Int, уверенность: Float)
     */
    fun classify(bitmap: Bitmap): Pair<Int, Float> {
        val resized = Bitmap.createScaledBitmap(bitmap, IMG_SIZE, IMG_SIZE, true)
        val inputBuffer = preprocessBitmap(resized)

        // Выходной массив: [1][10] — вероятности для 10 классов
        val output = Array(1) { FloatArray(NUM_CLASSES) }

        interpreter.run(inputBuffer, output)

        val probabilities = output[0]
        val predictedClass = probabilities.indices.maxByOrNull { probabilities[it] } ?: -1
        val confidence = probabilities[predictedClass]

        return Pair(predictedClass, confidence)
    }

    /**
     * Предобработка изображения:
     * - Перевод в оттенки серого
     * - Инверсия (белая цифра на чёрном фоне, как в MNIST)
     * - Нормализация значений пикселей в диапазон [0.0, 1.0]
     */
    private fun preprocessBitmap(bitmap: Bitmap): ByteBuffer {
        // 4 байта на пиксель (float32), 1 канал (grayscale)
        val byteBuffer = ByteBuffer.allocateDirect(4 * IMG_SIZE * IMG_SIZE)
        byteBuffer.order(ByteOrder.nativeOrder())

        for (y in 0 until IMG_SIZE) {
            for (x in 0 until IMG_SIZE) {
                val pixel = bitmap.getPixel(x, y)
                // Яркость пикселя через стандартную формулу люминансности
                val r = Color.red(pixel)
                val g = Color.green(pixel)
                val b = Color.blue(pixel)
                val luminance = (0.299f * r + 0.587f * g + 0.114f * b)
                // Инверсия: в MNIST цифра белая, фон чёрный
                val normalized = (255f - luminance) / 255f
                byteBuffer.putFloat(normalized)
            }
        }
        return byteBuffer
    }

    fun close() {
        interpreter.close()
    }
}
```

---

### Шаг 8. Основная Activity (MainActivity.kt)

Замените содержимое `MainActivity.kt`:

```kotlin
package com.lab.mnistcamera

import android.Manifest
import android.content.pm.PackageManager
import android.graphics.BitmapFactory
import android.os.Bundle
import android.widget.Button
import android.widget.TextView
import android.widget.Toast
import androidx.appcompat.app.AppCompatActivity
import androidx.camera.core.*
import androidx.camera.lifecycle.ProcessCameraProvider
import androidx.camera.view.PreviewView
import androidx.core.app.ActivityCompat
import androidx.core.content.ContextCompat
import java.io.File
import java.util.concurrent.ExecutorService
import java.util.concurrent.Executors

class MainActivity : AppCompatActivity() {

    private lateinit var previewView: PreviewView
    private lateinit var btnCapture: Button
    private lateinit var tvResult: TextView
    private lateinit var tvConfidence: TextView

    private lateinit var classifier: MnistClassifier
    private lateinit var imageCapture: ImageCapture
    private lateinit var cameraExecutor: ExecutorService

    companion object {
        private const val REQUEST_CODE_PERMISSIONS = 10
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        previewView   = findViewById(R.id.previewView)
        btnCapture    = findViewById(R.id.btnCapture)
        tvResult      = findViewById(R.id.tvResult)
        tvConfidence  = findViewById(R.id.tvConfidence)

        // Инициализация классификатора
        classifier = MnistClassifier(this)
        cameraExecutor = Executors.newSingleThreadExecutor()

        // Запрос разрешения на камеру
        if (allPermissionsGranted()) {
            startCamera()
        } else {
            ActivityCompat.requestPermissions(
                this,
                arrayOf(Manifest.permission.CAMERA),
                REQUEST_CODE_PERMISSIONS
            )
        }

        btnCapture.setOnClickListener { takePhotoAndClassify() }
    }

    private fun startCamera() {
        val cameraProviderFuture = ProcessCameraProvider.getInstance(this)
        cameraProviderFuture.addListener({
            val cameraProvider = cameraProviderFuture.get()

            val preview = Preview.Builder().build().also {
                it.setSurfaceProvider(previewView.surfaceProvider)
            }

            imageCapture = ImageCapture.Builder()
                .setCaptureMode(ImageCapture.CAPTURE_MODE_MINIMIZE_LATENCY)
                .build()

            val cameraSelector = CameraSelector.DEFAULT_BACK_CAMERA

            try {
                cameraProvider.unbindAll()
                cameraProvider.bindToLifecycle(
                    this, cameraSelector, preview, imageCapture
                )
            } catch (e: Exception) {
                Toast.makeText(this, "Ошибка камеры: ${e.message}", Toast.LENGTH_SHORT).show()
            }
        }, ContextCompat.getMainExecutor(this))
    }

    private fun takePhotoAndClassify() {
        val photoFile = File(externalCacheDir, "capture.jpg")
        val outputOptions = ImageCapture.OutputFileOptions.Builder(photoFile).build()

        btnCapture.isEnabled = false
        tvResult.text = "Обработка…"

        imageCapture.takePicture(
            outputOptions,
            ContextCompat.getMainExecutor(this),
            object : ImageCapture.OnImageSavedCallback {

                override fun onImageSaved(output: ImageCapture.OutputFileResults) {
                    val bitmap = BitmapFactory.decodeFile(photoFile.absolutePath)
                    val (digit, confidence) = classifier.classify(bitmap)
                    val percent = (confidence * 100).toInt()

                    tvResult.text = "Распознано: $digit"
                    tvConfidence.text = "Уверенность: $percent%"
                    btnCapture.isEnabled = true
                }

                override fun onError(exception: ImageCaptureException) {
                    tvResult.text = "Ошибка захвата"
                    btnCapture.isEnabled = true
                }
            }
        )
    }

    private fun allPermissionsGranted() = ContextCompat.checkSelfPermission(
        this, Manifest.permission.CAMERA
    ) == PackageManager.PERMISSION_GRANTED

    override fun onRequestPermissionsResult(
        requestCode: Int, permissions: Array<String>, grantResults: IntArray
    ) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults)
        if (requestCode == REQUEST_CODE_PERMISSIONS) {
            if (allPermissionsGranted()) startCamera()
            else Toast.makeText(this, "Разрешение на камеру отклонено", Toast.LENGTH_LONG).show()
        }
    }

    override fun onDestroy() {
        super.onDestroy()
        classifier.close()
        cameraExecutor.shutdown()
    }
}
```

---

## Структура проекта

```
MnistCameraApp/
├── app/
│   ├── src/
│   │   └── main/
│   │       ├── assets/
│   │       │   └── mnist.tflite          ← конвертированная модель
│   │       ├── java/com/lab/mnistcamera/
│   │       │   ├── MainActivity.kt        ← логика UI + камера
│   │       │   └── MnistClassifier.kt     ← инференс модели
│   │       ├── res/
│   │       │   └── layout/
│   │       │       └── activity_main.xml  ← интерфейс
│   │       └── AndroidManifest.xml
│   └── build.gradle
└── build.gradle
```

---

## Тестирование

### Подготовка тестовых данных

Распечатайте или нарисуйте цифры от 0 до 9 на белом листе:
- Используйте **тёмный маркер** (толщина 3–5 мм) на **белой бумаге**
- Цифра должна занимать не менее 70% кадра

### Порядок тестирования

| № | Действие | Ожидаемый результат |
|---|----------|---------------------|
| 1 | Запустить приложение | Отображается превью камеры |
| 2 | Навести камеру на цифру «5» | Превью отображает лист бумаги |
| 3 | Нажать «Распознать цифру» | Кнопка блокируется, появляется «Обработка…» |
| 4 | Дождаться результата | Отображается «Распознано: 5», уверенность > 80% |
| 5 | Проверить все цифры 0–9 | Точность > 85% при хорошем освещении |

### Условия для лучшей точности

- Равномерное освещение без теней
- Камера параллельна плоскости листа
- Цифра по центру кадра
- Отсутствие посторонних объектов рядом с цифрой

---

## Задания для самостоятельного выполнения

**Обязательное задание (базовый уровень):**
Запустить приложение, протестировать все 10 цифр (0–9) и заполнить таблицу результатов:

| Цифра | Распознано как | Уверенность | Верно? |
|-------|---------------|-------------|--------|
| 0     |               |             |        |
| 1     |               |             |        |
| 2     |               |             |        |
| 3     |               |             |        |
| 4     |               |             |        |
| 5     |               |             |        |
| 6     |               |             |        |
| 7     |               |             |        |
| 8     |               |             |        |
| 9     |               |             |        |

**Дополнительное задание (+1 балл):**  
Добавить в интерфейс гистограмму вероятностей для всех 10 классов (используйте `BarChart` из библиотеки MPAndroidChart).

**Повышенное задание (+2 балла):**  
Реализовать режим реального времени: модель запускается автоматически каждые 500 мс без нажатия кнопки, используя `ImageAnalysis` use case из CameraX.

---

## Контрольные вопросы

1. Чем отличается формат `.h5` от `.tflite`? Почему необходима конвертация?
2. Для чего выполняется инверсия пикселей при предобработке изображения?
3. Что означает значение на выходе слоя Softmax? Как интерпретировать вектор вероятностей?
4. Какие есть способы ускорить инференс модели на мобильном устройстве?
5. Почему модель, обученная на датасете MNIST, может плохо распознавать цифры с реальной фотографии?
6. Что такое квантизация модели и как она влияет на точность и скорость работы?

---

## Критерии оценки

| Критерий | Баллы |
|----------|-------|
| Конвертация модели и добавление в проект | 2 |
| Корректная работа камеры (CameraX) | 2 |
| Правильная предобработка изображения | 2 |
| Отображение результата и уверенности | 2 |
| Заполненная таблица тестирования | 1 |
| Ответы на контрольные вопросы | 1 |
| **Итого (базовый уровень)** | **10** |
| Индивидуальное задание по варианту | **+3** |

---
**Формат:** A4, портретная ориентация

**Что сдаётся:**
- Готовый сгенерированный PDF-файл с результатом распознавания
- Исходный код функции генерации PDF
- Письменный ответ: зачем нужен `FileProvider` при передаче файлов между приложениями на Android 7+?

---

### Вариант 5 — Виджет на рабочем столе

> **Для студентов, у которых последняя цифра зачётки: 5**

Создать **App Widget** (виджет рабочего стола), который показывает последнюю распознанную цифру и время последнего распознавания. Виджет автоматически обновляется при каждом новом распознавании через `BroadcastReceiver`. Размер — 2×1 ячейки. Нажатие на виджет открывает главный экран приложения (PendingIntent).

**Технологии:** `AppWidgetProvider`, `RemoteViews`, `BroadcastReceiver`, `SharedPreferences`

**Что сдаётся:**
- Скриншот рабочего стола с виджетом после нескольких распознаваний
- Исходный код `AppWidgetProvider` и `appwidget_info.xml`
- Письменный ответ: какие ограничения Android накладывает на частоту обновления виджетов?

---

### Вариант 6 — Голосовое озвучивание результата

> **Для студентов, у которых последняя цифра зачётки: 6**

После успешного распознавания приложение **озвучивает результат голосом** на русском языке: _«Распознана цифра пять, уверенность девяносто два процента»_. Добавить кнопку включения/выключения звука (иконка динамика в тулбаре). Реализовать очередь: если несколько распознаваний подряд — не перебивать предыдущее, а ставить в очередь.

**Технологии:** `TextToSpeech` API, `Locale("ru", "RU")`, utterance ID queue

**Что сдаётся:**
- Видеозапись (screen recording) **со звуком**, демонстрирующая озвучивание трёх разных цифр
- Исходный код инициализации TTS и функции озвучивания
- Письменный ответ: что произойдёт, если на устройстве не установлен русский голосовой движок TTS?

---

### Вариант 7 — Сравнение двух изображений

> **Для студентов, у которых последняя цифра зачётки: 7**

Реализовать режим «Сравнение»: пользователь делает **два снимка** разных рукописных цифр. Приложение показывает оба изображения рядом, результаты распознавания для каждого, а также **групповую гистограмму вероятностей** обоих предсказаний на одном графике (две серии столбцов разных цветов). Вывести итог: «Цифры совпадают» или «Цифры различаются».

**Технологии:** MPAndroidChart (`GroupedBarChart`), двойной `ImageCapture`

**Что сдаётся:**
- Скриншот экрана сравнения с двумя цифрами и гистограммой
- Исходный код формирования данных для `GroupedBarChart`
- Письменный ответ: как по гистограмме определить, насколько «уверена» модель в своём ответе?

---

### Вариант 8 — Встроенная рисовалка (DrawView)

> **Для студентов, у которых последняя цифра зачётки: 8**

Добавить альтернативный режим ввода — **холст для рисования пальцем**. Пользователь рисует цифру на экране (белый фон, чёрная линия), нажимает «Распознать» — приложение берёт `Bitmap` с холста и передаёт в модель. Реализовать кнопки «Очистить» и «Отменить последний штрих» (undo через стек `Path`). Переключение между камерой и рисовалкой — через `BottomNavigationView`.

**Технологии:** Custom View (`Canvas`, `Path`), `BottomNavigationView`, стек отмены (`ArrayDeque`)

**Что сдаётся:**
- Скриншоты обоих режимов с результатами распознавания
- Исходный код `DrawView` (полный класс)
- Письменный ответ: почему толщина линии на холсте влияет на точность распознавания моделью MNIST?

---

### Вариант 9 — Статистика и точность

> **Для студентов, у которых последняя цифра зачётки: 9**

Реализовать экран **«Моя статистика»**. Приложение накапливает историю распознаваний и отображает: общее число распознаваний, среднюю уверенность модели (%), круговую диаграмму частоты распознавания каждой цифры (0–9), матрицу ошибок 10×10 (какие цифры путаются чаще всего) в виде тепловой карты. Данные сохраняются между запусками.

**Технологии:** MPAndroidChart (`PieChart`), кастомный `GridView` для тепловой карты, `SharedPreferences` или SQLite  
**Минимум:** ≥20 распознаваний, каждая цифра встречается ≥2 раза

**Что сдаётся:**
- Скриншот экрана статистики с заполненной матрицей ошибок
- Исходный код построения матрицы
- Письменный ответ: что такое матрица ошибок (confusion matrix) и как её читать применительно к задаче классификации цифр?

---

## Список литературы

1. Android Developers — CameraX: https://developer.android.com/training/camerax  
2. TensorFlow Lite — Android Quickstart: https://www.tensorflow.org/lite/android/quickstart  
3. LeCun Y. et al. «Gradient-Based Learning Applied to Document Recognition» — IEEE, 1998  
4. TensorFlow Lite Model Optimization Toolkit: https://www.tensorflow.org/lite/performance/model_optimization  
5. Kotlin Coroutines Guide: https://kotlinlang.org/docs/coroutines-guide.html  

---

*Лабораторная работа подготовлена для курса «Разработка мобильных приложений»*  
*Версия: 1.0 | Платформа: Android API 21+*
