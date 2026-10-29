# PhotoBoothPrototype-Public
Android digital photo booth application
#!/bin/bash

# ایجاد ساختار پوشه‌ها
mkdir -p RestaurantPhotoBooth/android-app/app/src/main/java/com/restaurant/photobooth
mkdir -p RestaurantPhotoBooth/android-app/app/src/main/res/layout
mkdir -p RestaurantPhotoBooth/android-app/app/src/main/res/values
mkdir -p RestaurantPhotoBooth/server/public
mkdir -p RestaurantPhotoBooth/.github/workflows

cd RestaurantPhotoBooth

# ---------------------------------------------------------
# ۱. فایل‌های سرور (Node.js)
# ---------------------------------------------------------

cat << 'EOF' > server/package.json
{
  "name": "restaurant-photobooth-server",
  "version": "1.0.0",
  "description": "Backend server for Restaurant PhotoBooth with auto-deletion after 7 days",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "cors": "^2.8.5",
    "dotenv": "^16.4.5",
    "express": "^4.19.2",
    "multer": "^1.4.5-lts.1",
    "uuid": "^9.0.1"
  }
}
EOF

cat << 'EOF' > server/.env.example
PORT=3000
BASE_URL=https://your-server-domain.com
EOF

cat << 'EOF' > server/server.js
const express = require('express');
const multer = require('multer');
const path = require('path');
const fs = require('fs');
const { v4: uuidv4 } = require('uuid');
const cors = require('cors');
require('dotenv').config();

const app = express();
const PORT = process.env.PORT || 3000;
const BASE_URL = process.env.BASE_URL || `http://localhost:${PORT}`;

app.use(cors());
app.use(express.json());
app.use('/uploads', express.static(path.join(__dirname, 'uploads')));
app.use(express.static(path.join(__dirname, 'public')));

const uploadDir = path.join(__dirname, 'uploads');
if (!fs.existsSync(uploadDir)) {
    fs.mkdirSync(uploadDir, { recursive: true });
}

const storage = multer.diskStorage({
    destination: (req, file, cb) => {
        cb(null, uploadDir);
    },
    filename: (req, file, cb) => {
        const uniqueSuffix = Date.now() + '-' + Math.round(Math.random() * 1E9);
        cb(null, uniqueSuffix + '.jpg');
    }
});

const upload = multer({
    storage: storage,
    limits: { fileSize: 5 * 1024 * 1024 },
    fileFilter: (req, file, cb) => {
        if (file.mimetype === 'image/jpeg' || file.mimetype === 'image/png') {
            cb(null, true);
        } else {
            cb(new Error('Nur JPEG und PNG sind erlaubt!'), false);
        }
    }
});

const metadataFile = path.join(__dirname, 'metadata.json');

function getMetadata() {
    if (!fs.existsSync(metadataFile)) return {};
    try {
        return JSON.parse(fs.readFileSync(metadataFile, 'utf8'));
    } catch (e) {
        return {};
    }
}

function saveMetadata(data) {
    fs.writeFileSync(metadataFile, JSON.stringify(data, null, 2));
}

app.post('/api/upload', upload.array('photos', 4), (req, res) => {
    try {
        if (!req.files || req.files.length === 0) {
            return res.status(400).json({ success: false, error: 'Keine Fotos hochgeladen.' });
        }

        const sessionId = uuidv4();
        const photoUrls = req.files.map(file => `${BASE_URL}/uploads/${file.filename}`);
        const expiresAt = Date.now() + (7 * 24 * 60 * 60 * 1000);

        const metadata = getMetadata();
        metadata[sessionId] = {
            photos: photoUrls,
            filenames: req.files.map(file => file.filename),
            expiresAt: expiresAt
        };
        saveMetadata(metadata);

        const galleryUrl = `${BASE_URL}/gallery.html?session=${sessionId}`;
        res.json({
            success: true,
            galleryUrl: galleryUrl,
            expiresAt: new Date(expiresAt).toISOString()
        });
    } catch (error) {
        res.status(500).json({ success: false, error: error.message });
    }
});

app.get('/api/gallery/:sessionId', (req, res) => {
    const { sessionId } = req.params;
    const metadata = getMetadata();
    const session = metadata[sessionId];

    if (!session) {
        return res.status(404).json({ success: false, error: 'Galerie nicht gefunden oder abgelaufen.' });
    }

    if (Date.now() > session.expiresAt) {
        session.filenames.forEach(filename => {
            const filePath = path.join(uploadDir, filename);
            if (fs.existsSync(filePath)) fs.unlinkSync(filePath);
        });
        delete metadata[sessionId];
        saveMetadata(metadata);
        return res.status(410).json({ success: false, error: 'Diese Galerie ist abgelaufen und wurde gelöscht.' });
    }

    res.json({
        success: true,
        photos: session.photos,
        expiresAt: session.expiresAt
    });
});

setInterval(() => {
    const metadata = getMetadata();
    const now = Date.now();
    let updated = false;

    for (const sessionId in metadata) {
        const session = metadata[sessionId];
        if (now > session.expiresAt) {
            session.filenames.forEach(filename => {
                const filePath = path.join(uploadDir, filename);
                if (fs.existsSync(filePath)) {
                    fs.unlinkSync(filePath);
                }
            });
            delete metadata[sessionId];
            updated = true;
        }
    }
    if (updated) {
        saveMetadata(metadata);
    }
}, 60 * 60 * 1000);

app.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
});
EOF

cat << 'EOF' > server/public/gallery.html
<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Restaurant Fotobox Galerie</title>
    <style>
        body { font-family: Arial, sans-serif; background: #f4f4f9; margin: 0; padding: 20px; text-align: center; }
        .container { max-width: 600px; margin: auto; background: white; padding: 20px; border-radius: 12px; box-shadow: 0 4px 10px rgba(0,0,0,0.1); }
        h1 { color: #333; font-size: 22px; }
        .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; margin-top: 20px; }
        img { width: 100%; border-radius: 8px; box-shadow: 0 2px 5px rgba(0,0,0,0.1); }
        .error { color: #d9534f; font-weight: bold; margin-top: 40px; }
        .btn { display: inline-block; margin-top: 20px; padding: 12px 24px; background: #0275d8; color: white; text-decoration: none; border-radius: 6px; font-weight: bold; }
        .timer { font-size: 14px; color: #666; margin-top: 15px; }
    </style>
</head>
<body>
    <div class="container" id="content">
        <h1>Ihre Fotos werden geladen...</h1>
    </div>

    <script>
        const urlParams = new URLSearchParams(window.location.search);
        const sessionId = urlParams.get('session');

        if (!sessionId) {
            document.getElementById('content').innerHTML = '<p class="error">Ungültiger Link.</p>';
        } else {
            fetch(`/api/gallery/${sessionId}`)
                .then(res => res.json())
                .then(data => {
                    if (!data.success) {
                        document.getElementById('content').innerHTML = `<p class="error">${data.error || 'Galerie abgelaufen.'}</p>`;
                        return;
                    }

                    let html = '<h1>Ihre Erinnerungsfotos aus dem Restaurant</h1><div class="grid">';
                    data.photos.forEach(photoUrl => {
                        html += `<div><img src="${photoUrl}" alt="Photo"><br><a href="${photoUrl}" download class="btn" style="padding:6px 12px; font-size:12px; margin-top:8px;">Herunterladen</a></div>`;
                    });
                    html += '</div>';

                    const expireDate = new Date(data.expiresAt).toLocaleDateString('de-DE');
                    html += `<div class="timer">Diese Fotos bleiben bis zum ${expireDate} auf dem Server und werden danach automatisch gelöscht.</div>`;

                    document.getElementById('content').innerHTML = html;
                })
                .catch(err => {
                    document.getElementById('content').innerHTML = '<p class="error">Verbindungsfehler zum Server.</p>';
                });
        }
    </script>
</body>
</html>
EOF

# ---------------------------------------------------------
# ۲. فایل‌های اپلیکیشن اندروید
# ---------------------------------------------------------

cat << 'EOF' > android-app/build.gradle.kts
plugins {
    id("com.android.application") version "8.2.0" apply false
    id("org.jetbrains.kotlin.android") version "1.9.0" apply false
}
EOF

cat << 'EOF' > android-app/settings.gradle.kts
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
    }
}
rootProject.name = "RestaurantPhotoBooth"
include(":app")
EOF

cat << 'EOF' > android-app/gradle.properties
org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
android.useAndroidX=true
android.enableJetifier=true
EOF

cat << 'EOF' > android-app/app/build.gradle.kts
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
}

android {
    namespace = "com.restaurant.photobooth"
    compileSdk = 34

    defaultConfig {
        applicationId = "com.restaurant.photobooth"
        minSdk = 24
        targetSdk = 34
        versionCode = 1
        versionName = "1.0"

        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            isMinifyEnabled = false
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
    kotlinOptions {
        jvmTarget = "17"
    }
    buildFeatures {
        viewBinding = true
    }
}

dependencies {
    implementation("androidx.core:core-ktx:1.12.0")
    implementation("androidx.appcompat:appcompat:1.6.1")
    implementation("com.google.android.material:material:1.11.0")
    implementation("androidx.constraintlayout:constraintlayout:2.1.4")

    val cameraxVersion = "1.3.1"
    implementation("androidx.camera:camera-core:$cameraxVersion")
    implementation("androidx.camera:camera-camera2:$cameraxVersion")
    implementation("androidx.camera:camera-lifecycle:$cameraxVersion")
    implementation("androidx.camera:camera-view:$cameraxVersion")

    implementation("com.google.zxing:core:3.5.2")
}
EOF

cat << 'EOF' > android-app/app/proguard-rules.pro
-keep class com.google.zxing.** { *; }
EOF

cat << 'EOF' > android-app/app/src/main/AndroidManifest.xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <uses-permission android:name="android.permission.CAMERA" />
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="Restaurant Fotobox"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.AppCompat.Light.NoActionBar">
        
        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:screenOrientation="landscape">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <activity
            android:name=".SettingsActivity"
            android:screenOrientation="landscape" />
    </application>

</manifest>
EOF

cat << 'EOF' > android-app/app/src/main/res/values/strings.xml
<resources>
    <string name="app_name">Restaurant Fotobox</string>
</resources>
EOF

cat << 'EOF' > android-app/app/src/main/res/values/colors.xml
<resources>
    <color name="black">#FF000000</color>
    <color name="white">#FFFFFFFF</color>
</resources>
EOF

cat << 'EOF' > android-app/app/src/main/res/layout/activity_main.xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="#111111">

    <androidx.camera.view.PreviewView
        android:id="@+id/viewFinder"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

    <Button
        android:id="@+id/btnStart"
        android:layout_width="280dp"
        android:layout_height="80dp"
        android:text="Fotografierung starten"
        android:textSize="20sp"
        android:backgroundTint="#4CAF50"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <TextView
        android:id="@+id/tvStatus"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text=""
        android:textColor="#FFFFFF"
        android:textSize="48sp"
        android:textStyle="bold"
        android:shadowColor="#000000"
        android:shadowDx="4"
        android:shadowDy="4"
        android:shadowRadius="8"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <LinearLayout
        android:id="@+id/layoutQrResult"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:background="#CC000000"
        android:gravity="center"
        android:orientation="vertical"
        android:visibility="gone"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent">

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Ihre Fotos sind bereit! QR-Code scannen:"
            android:textColor="#FFFFFF"
            android:textSize="22sp"
            android:textStyle="bold"
            android:layout_marginBottom="20dp" />

        <ImageView
            android:id="@+id/ivQrCode"
            android:layout_width="280dp"
            android:layout_height="280dp"
            android:background="#FFFFFF"
            android:padding="10dp" />

        <Button
            android:id="@+id/btnNewSession"
            android:layout_width="220dp"
            android:layout_height="60dp"
            android:layout_marginTop="20dp"
            android:backgroundTint="#2196F3"
            android:text="Neue Sitzung"
            android:textSize="18sp" />
    </LinearLayout>

    <Button
        android:id="@+id/btnSettings"
        android:layout_width="50dp"
        android:layout_height="50dp"
        android:layout_margin="16dp"
        android:text="⚙"
        android:textSize="20sp"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
EOF

cat << 'EOF' > android-app/app/src/main/res/layout/activity_settings.xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="32dp"
    android:gravity="center">

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Server-URL Einstellungen"
        android:textSize="24sp"
        android:textStyle="bold"
        android:layout_marginBottom="24dp" />

    <EditText
        android:id="@+id/etServerUrl"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="https://your-server-domain.com"
        android:inputType="textUri"
        android:padding="12dp"
        android:textSize="18sp" />

    <Button
        android:id="@+id/btnSaveSettings"
        android:layout_width="200dp"
        android:layout_height="60dp"
        android:layout_marginTop="24dp"
        android:backgroundTint="#4CAF50"
        android:text="Speichern"
        android:textSize="18sp" />
</LinearLayout>
EOF

cat << 'EOF' > android-app/app/src/main/java/com/restaurant/photobooth/SettingsActivity.kt
package com.restaurant.photobooth

import android.content.Context
import android.os.Bundle
import android.widget.Button
import android.widget.EditText
import androidx.appcompat.app.AppCompatActivity

class SettingsActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_settings)

        val etServerUrl = findViewById<EditText>(R.id.etServerUrl)
        val btnSave = findViewById<Button>(R.id.btnSaveSettings)

        val prefs = getSharedPreferences("BoothPrefs", Context.MODE_PRIVATE)
        etServerUrl.setText(prefs.getString("server_url", "https://your-server-domain.com"))

        btnSave.setOnClickListener {
            val url = etServerUrl.text.toString().trim()
            if (url.isNotEmpty()) {
                prefs.edit().putString("server_url", url).apply()
                finish()
            }
        }
    }
}
EOF

cat << 'EOF' > android-app/app/src/main/java/com/restaurant/photobooth/MainActivity.kt
package com.restaurant.photobooth

import android.Manifest
import android.content.Context
import android.content.Intent
import android.content.pm.PackageManager
import android.graphics.Bitmap
import android.os.Bundle
import android.os.Handler
import android.os.Looper
import android.view.View
import android.widget.Button
import android.widget.ImageView
import android.widget.TextView
import android.widget.Toast
import androidx.appcompat.app.AppCompatActivity
import androidx.camera.core.*
import androidx.camera.lifecycle.ProcessCameraProvider
import androidx.camera.view.PreviewView
import androidx.core.app.ActivityCompat
import androidx.core.content.ContextCompat
import com.google.zxing.BarcodeFormat
import com.google.zxing.qrcode.QRCodeWriter
import org.json.JSONObject
import java.io.File
import java.net.HttpURLConnection
import java.net.URL
import java.util.concurrent.ExecutorService
import java.util.concurrent.Executors

class MainActivity : AppCompatActivity() {

    private lateinit var viewFinder: PreviewView
    private lateinit var btnStart: Button
    private lateinit var btnSettings: Button
    private lateinit var tvStatus: TextView
    private lateinit var layoutQrResult: View
    private lateinit var ivQrCode: ImageView
    private lateinit var btnNewSession: Button

    private var imageCapture: ImageCapture? = null
    private lateinit var cameraExecutor: ExecutorService
    private val capturedPhotoFiles = mutableListOf<File>()
    private val handler = Handler(Looper.getMainLooper())

    companion object {
        private const val REQUEST_CODE_PERMISSIONS = 10
        private val REQUIRED_PERMISSIONS = arrayOf(Manifest.permission.CAMERA)
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        viewFinder = findViewById(R.id.viewFinder)
        btnStart = findViewById(R.id.btnStart)
        btnSettings = findViewById(R.id.btnSettings)
        tvStatus = findViewById(R.id.tvStatus)
        layoutQrResult = findViewById(R.id.layoutQrResult)
        ivQrCode = findViewById(R.id.ivQrCode)
        btnNewSession = findViewById(R.id.btnNewSession)

        cameraExecutor = Executors.newSingleThreadExecutor()

        if (allPermissionsGranted()) {
            startCamera()
        } else {
            ActivityCompat.requestPermissions(this, REQUIRED_PERMISSIONS, REQUEST_CODE_PERMISSIONS)
        }

        btnStart.setOnClickListener {
            startPhotoSession()
        }

        btnSettings.setOnClickListener {
            startActivity(Intent(this, SettingsActivity::class.java))
        }

        btnNewSession.setOnClickListener {
            layoutQrResult.visibility = View.GONE
            btnStart.visibility = View.VISIBLE
            capturedPhotoFiles.clear()
        }
    }

    private fun allPermissionsGranted() = REQUIRED_PERMISSIONS.all {
        ContextCompat.checkSelfPermission(baseContext, it) == PackageManager.PERMISSION_GRANTED
    }

    override fun onRequestPermissionsResult(requestCode: Int, permissions: Array<String>, grantResults: IntArray) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults)
        if (requestCode == REQUEST_CODE_PERMISSIONS) {
            if (allPermissionsGranted()) {
                startCamera()
            } else {
                Toast.makeText(this, "Kamerazugriff verweigert.", Toast.LENGTH_SHORT).show()
                finish()
            }
        }
    }

    private fun startCamera() {
        val cameraProviderFuture = ProcessCameraProvider.getInstance(this)
        cameraProviderFuture.addListener({
            val cameraProvider: ProcessCameraProvider = cameraProviderFuture.get()
            val preview = Preview.Builder().build().also {
                it.setSurfaceProvider(viewFinder.surfaceProvider)
            }

            imageCapture = ImageCapture.Builder().build()
            val cameraSelector = CameraSelector.DEFAULT_FRONT_CAMERA

            try {
                cameraProvider.unbindAll()
                cameraProvider.bindToLifecycle(this, cameraSelector, preview, imageCapture)
            } catch (exc: Exception) {
                Toast.makeText(this, "Kamera konnte nicht gestartet werden", Toast.LENGTH_SHORT).show()
            }
        }, ContextCompat.getMainExecutor(this))
    }

    private fun startPhotoSession() {
        btnStart.visibility = View.GONE
        capturedPhotoFiles.clear()
        runCountdown(5) {
            capturePhotosSequence(1)
        }
    }

    private fun runCountdown(seconds: Int, onFinished: () -> Unit) {
        if (seconds > 0) {
            tvStatus.text = seconds.toString()
            handler.postDelayed({ runCountdown(seconds - 1, onFinished) }, 1000)
        } else {
            tvStatus.text = ""
            onFinished()
        }
    }

    private fun capturePhotosSequence(photoIndex: Int) {
        if (photoIndex > 4) {
            uploadPhotos()
            return
        }

        tvStatus.text = "Foto $photoIndex von 4"
        takePhoto { file ->
            if (file != null) {
                capturedPhotoFiles.add(file)
            }
            if (photoIndex < 4) {
                handler.postDelayed({
                    capturePhotosSequence(photoIndex + 1)
                }, 3000)
            } else {
                capturePhotosSequence(5)
            }
        }
    }

    private fun takePhoto(callback: (File?) -> Unit) {
        val imageCapture = imageCapture ?: return callback(null)
        val photoFile = File(externalMediaDirs.firstOrNull(), "${System.currentTimeMillis()}.jpg")
        val outputOptions = ImageCapture.OutputFileOptions.Builder(photoFile).build()

        imageCapture.takePicture(outputOptions, ContextCompat.getMainExecutor(this), object : ImageCapture.OnImageSavedCallback {
            override fun onError(exc: ImageCaptureException) {
                callback(null)
            }
            override fun onImageSaved(output: ImageCapture.OutputFileResults) {
                callback(photoFile)
            }
        })
    }

    private fun uploadPhotos() {
        tvStatus.text = "Fotos werden hochgeladen..."
        Thread {
            try {
                val prefs = getSharedPreferences("BoothPrefs", Context.MODE_PRIVATE)
                val serverUrl = prefs.getString("server_url", "https://your-server-domain.com") ?: "https://your-server-domain.com"
                val boundary = "Boundary-" + System.currentTimeMillis()
                val url = URL("$serverUrl/api/upload")
                val connection = url.openConnection() as HttpURLConnection
                connection.doOutput = true
                connection.requestMethod = "POST"
                connection.setRequestProperty("Content-Type", "multipart/form-data; boundary=$boundary")

                val outputStream = connection.outputStream
                val writer = outputStream.bufferedWriter(Charsets.UTF_8)

                for (file in capturedPhotoFiles) {
                    writer.write("--$boundary\r\n")
                    writer.write("Content-Disposition: form-data; name=\"photos\"; filename=\"${file.name}\"\r\n")
                    writer.write("Content-Type: image/jpeg\r\n\r\n")
                    writer.flush()

                    file.inputStream().use { input ->
                        input.copyTo(outputStream)
                    }
                    outputStream.flush()
                    writer.write("\r\n")
                }
                writer.write("--$boundary--\r\n")
                writer.flush()
                writer.close()

                val responseCode = connection.responseCode
                if (responseCode == HttpURLConnection.HTTP_OK) {
                    val response = connection.inputStream.bufferedReader().readText()
                    val json = JSONObject(response)
                    val galleryUrl = json.getString("galleryUrl")

                    handler.post {
                        tvStatus.text = ""
                        showQrCode(galleryUrl)
                    }
                } else {
                    handler.post { retryUpload("Serverfehler: $responseCode") }
                }
            } catch (e: Exception) {
                handler.post { retryUpload("Netzwerkfehler. Bitte Internetverbindung prüfen.") }
            }
        }.start()
    }

    private fun retryUpload(errorMsg: String) {
        tvStatus.text = "$errorMsg\nErneuter Versuch..."
        handler.postDelayed({ uploadPhotos() }, 4000)
    }

    private fun showQrCode(url: String) {
        try {
            val bitmap = generateQRCode(url, 400, 400)
            ivQrCode.setImageBitmap(bitmap)
            layoutQrResult.visibility = View.VISIBLE
        } catch (e: Exception) {
            Toast.makeText(this, "Fehler beim Erstellen des QR-Codes", Toast.LENGTH_LONG).show()
            btnStart.visibility = View.VISIBLE
        }
    }

    private fun generateQRCode(text: String, width: Int, height: Int): Bitmap {
        val bitMatrix = QRCodeWriter().encode(text, BarcodeFormat.QR_CODE, width, height)
        val bmp = Bitmap.createBitmap(width, height, Bitmap.Config.RGB_565)
        for (x in 0 until width) {
            for (y in 0 until height) {
                bmp.setPixel(x, y, if (bitMatrix[x, y]) android.graphics.Color.BLACK else android.graphics.Color.WHITE)
            }
        }
        return bmp
    }

    override fun onDestroy() {
        super.onDestroy()
        cameraExecutor.shutdown()
    }
}
EOF

# ---------------------------------------------------------
# ۳. گیت‌هاب اکشنز (ساخت خودکار APK)
# ---------------------------------------------------------

cat << 'EOF' > .github/workflows/build-apk.yml
name: Build Android APK

on:
  push:
    branches: [ "main", "master" ]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Set up JDK 17
      uses: actions/setup-java@v4
      with:
        distribution: 'temurin'
        java-version: '17'

    - name: Setup Android SDK
      uses: android-actions/setup-android@v3

    - name: Grant execute permission for gradlew
      run: |
        cd android-app
        chmod +x gradlew

    - name: Build APK
      run: |
        cd android-app
        ./gradlew assembleDebug

    - name: Upload APK Artifact
      uses: actions/upload-artifact@v4
      with:
        name: photobooth-apk
        path: android-app/app/build/outputs/apk/debug/app-debug.apk
EOF

cd ..
echo "پروژه با موفقیت ایجاد شد! تمام فایل‌ها در پوشه RestaurantPhotoBooth قرار دارند."


mahdi salek <mahdisalek98@gmail.com> schrieb am Do., 29. Okt. 2026, 18:11:
https://share.gemini.google/jD3uAKhK5T25
 
