# Google Sign-In and Google Drive Integration with Firebase

This guide explains how to integrate Google Sign-In with Firebase and use Google Drive API to upload data in a Jetpack Compose application.

## Prerequisites

1. **Firebase Project**: Create a Firebase project in the [Firebase Console](https://console.firebase.google.com/).
2. **Google API Console**: Set up Google Drive API in the [Google API Console](https://console.developers.google.com/).
3. **Android Studio**: Ensure you have Android Studio installed.

## Step 1: Configure Firebase and Google Sign-In

### 1.1. Set up Firebase in your project

1. Go to the Firebase Console.
2. Select your project and click on "Add app" to add an Android app.
3. Follow the instructions to download the `google-services.json` file.
4. Place the `google-services.json` file in the `app` directory of your Android project.

### 1.2. Add Firebase SDK to your project

Add the following dependencies to your `build.gradle` files.

Project-level `build.gradle` (usually in the root directory):**

```
plugins {
    /....
    id("com.google.gms.google-services") version "4.4.2" apply false
}
```

App-level `build.gradle` (usually in the app directory):

```
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.jetbrains.kotlin.android)
    id("com.google.gms.google-services") // this line
}

dependencies {
    implementation("com.google.firebase:firebase-bom:33.2.0")
    implementation("com.google.firebase:firebase-auth")
    implementation("com.google.android.gms:play-services-auth:21.2.0")

    implementation ("com.google.api-client:google-api-client-android:1.33.2")
    implementation ("com.google.api-client:google-api-client-gson:1.33.2")
    implementation ("com.google.apis:google-api-services-drive:v3-rev197-1.25.0")
}
```

### 1.3. Set up Google-Signin

1. In the Firebase Console, go to the Authentication section.
2. Enable the Google sign-in method.

## Step 2: Implement google sign in

### 2.1. Create a GoogleSignInClient


```
val googleSignInOptions =
            GoogleSignInOptions
                .Builder(GoogleSignInOptions.DEFAULT_SIGN_IN)
                .requestIdToken("Your web client here")
                .requestEmail()
                .requestScopes(Scope(Scopes.DRIVE_FILE))
                .build()
        val googleSignInClient = GoogleSignIn.getClient(this, googleSignInOptions)
        signInLauncher.launch(googleSignInClient.signInIntent)
}
```

### 2.2. Create a launcher to launch signin intent

```
signInLauncher =
            registerForActivityResult(ActivityResultContracts.StartActivityForResult()) { result ->
                handleSignInResult(result, this)
            }
```

## Step 3: Integrate Google Drive and upload data

3.1. Handle Sign in result

```
private fun handleSignInResult(
    result: ActivityResult,
    context: Context,
) {
    val data = result.data
    val task = GoogleSignIn.getSignedInAccountFromIntent(data)
    try {
        val account = task.getResult(ApiException::class.java)
        if (account != null) {
            Log.d("GoogleSignIn", "Account: ${account.email}")
            firebaseAuthWithGoogle(account.idToken!!, context)
        } else {
            Log.w("GoogleSignIn", "Account is null")
        }
    } catch (e: ApiException) {
        Log.w("GoogleSignIn", "Google sign-in failed", e)
    }
}
```

### 3.2. After successfully sign in with Google create a folder in drive

```
private fun firebaseAuthWithGoogle(
    idToken: String,
    activity: MainActivity,
) {
    val credential = GoogleAuthProvider.getCredential(idToken, null)
    FirebaseAuth
        .getInstance()
        .signInWithCredential(credential)
        .addOnCompleteListener(activity) { task ->
            if (task.isSuccessful) {
                val account = GoogleSignIn.getLastSignedInAccount(activity)
                account?.let {
                    // Run Google Drive operations on a background thread
                    CoroutineScope(Dispatchers.IO).launch {
                        try {
                            val googleAccountCredential =
                                GoogleAccountCredential.usingOAuth2(
                                    activity,
                                    listOf(Scopes.DRIVE_FILE),
                                )
                            googleAccountCredential.selectedAccount = it.account
                            val driveService =
                                Drive
                                    .Builder(
                                        NetHttpTransport(),
                                        GsonFactory(),
                                        googleAccountCredential,
                                    ).setApplicationName("Your project name")
                                    .build()

                            createFolderInGoogleDrive(driveService, activity)

                            withContext(Dispatchers.Main) {
                                Log.d("GoogleDrive", "Folder creation successful")
                            }
                        } catch (e: Exception) {
                            withContext(Dispatchers.Main) {
                                Log.e("GoogleDrive", "Error creating folder: ${e.message}")
                            }
                        }
                    }
                }
            } else {
                Log.w("GoogleSignIn", "signInWithCredential:failure", task.exception)
            }
        }
}

private fun createFolderInGoogleDrive(
    driveService: Drive,
    context: MainActivity,
) {
    val metadata = File()
    metadata.name = "Backup Folder"
    metadata.mimeType = "application/vnd.google-apps.folder"

    val folder =
        driveService
            .files()
            .create(metadata)
            .setFields("id")
            .execute()

    uploadPhotoToGoogleDrive(
        driveService,
        folder.id,
        "photo.jpg", //Any kind of file
        Uri.parse("android.resource://YourApplicationPackage/drawable/yourfilename"),
        context,
    )
    Log.d("GoogleDrive", "Folder ID: ${folder.id}")
}
```

### 3.3. Upload data to Drive

```
private fun uploadPhotoToGoogleDrive(
    driveService: Drive,
    folderId: String,
    fileName: String,
    imageUri: Uri,
    context: Context,
) {
    val fileMetadata = File()
    fileMetadata.name = fileName
    fileMetadata.parents = listOf(folderId) // Place the file in the specific folder
    fileMetadata.mimeType =
        "image/jpg" // You can use "image/png" or other types based on your image type

    // Open the image file as InputStream
    val inputStream = context.contentResolver.openInputStream(imageUri)
    val fileContent = InputStreamContent("image/jpg", inputStream) // Adjust MIME type accordingly

    // Upload the file
    val file =
        driveService
            .files()
            .create(fileMetadata, fileContent)
            .setFields("id")
            .execute()

    Log.d("GoogleDrive", "Photo uploaded with ID: ${file.id}")
}
```

Don't forget to enable Google drive api in Api console



