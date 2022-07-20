# morden storage

Modern storage devices are the keepers. This includes photos of our loved ones and videos of our most memorable days. And with so many of us now coducting business with our phones and tablets, often important files are stored on our deivces, including finacial and legal documents. Android's amazing app developer community has created countless app to help us share our memories, edit our documents, and listen to music. But until recently, storage access on Android hadn't evolved to ensure that apps could get exactly the information users want to share and no more. Today we'll quickly go over the storage changes introduced with Android 10. And then we'll discuss in detail the new modifications we've added to improve the developer experience in Android 11. And we'll finish with some tips on migrating your app to use morden storage.

## Topics

- Review of Scoped storage
- New developer improvements
- Tips for migrating

## Scoped storage

Last year with Android 10, we introduced the concept of scoped storage. The idea is to organize shared storage into specific collections, and limit access to broad storage. These are the changed introduced for apps targeting Android 10.

1. Unrestricted access to your own app storage
2. Unrestricted media and downloads contributions
3. Runtime permission only gives read access to media
4. User confirmation required for modifying media
5. Location metadata gated by new permission

First, apps has unrestricted access to their own storage, both on internal memory and external volumes. Second, shared storage is divided into four organized collection -- pictures, videos, music, and downloads. And apps can contribute files to these organized collections without any permission. The storage runtime permission new only gives read access to shared pictures, videos, and music files. To access downloads or unorganized files the user must give specific access via the Document Picker. Deleting or modifying media files that were not created by your app now requires user confirmation. And lastly, apps must request a new permission to access photo location metadata.

While many apps have successfully migrated to scoped storage on Android 10. We recognized that we didn't sufficiently satisfy all of the use cases in that release. Therefore, for apps targeting Android 10, we included the option to opt out of scoped storage with a flag in the manifest called **requestLegacyExternalStorage**. Over the past year, we've heard feedback about the storage update from all kinds of developers. We know this change isn't easy for many apps. And we recognize the challenges involved with adapting to morden storage. In response, my team focused all our efforts in this release on adding improvements to Android 11, specifically to make the transition easier. And because we've made these changes, we now feel comfortable making scoped storage mandatory for all apps targeting Android 11. There won't be an option to opt out.

```
# App targeting Android 10
requestLegacyExternalStorage
```

So here's what's new on Android 11. As an alternatve to MediaStore APIs, apps can choose to use other APIs which rely on local file paths. This likely to reduce or eliminate the amount of code your have to change when upgrading to target Android 11. We've created new, eaiser to use APIs for modifying media files, and added the ability to make changes in bulk. For specific app that can verify that they require the ability to read me broadly we created a new special app access called All Files Acess. App storage is now private from other apps, including external app directories. Let's go through each of these in more detail.

1. Enabled File Path APIs
2. Bulk media modification APIs
3. All Files access
4. Private app storage

# storage on kitkat

In our last episode about the storage access framework, we saw the new system documents UI that lets the user of your app browse content from all apps on the device that share documents. Today, we're going to walk through how to launch the UI and how to handle the results in your app. There're more to it than there used to be.

New in Android KitKat are document editing, writing, and saving in place, and document creation and deletion. Here's a quick picture of the flow. As you can see, document providers and clients don't interact directly, just as before. A client requests permission to interact with files, read, edit, create, et cetera. The system picker goes to each registered provider and shows the user the matching content. Finally, the user selects a ducument, and the system grants the client app permissions just for that URI.

Let's talk about ACTION_OPEN_DOCUMENT. 

```java
private static final int READ_REQUEST_CODE = 1;

public void performFileSearch() {
  Intent intent = new Intent(Itent.ACTION_OPEN_DOCUMENT);
  intent.addCategory(Intent.CATEGORY_OPENABLE);
  intent.setType("image/*");
  startActivityForResult(intent, READ_REQUEST_CODE);
}
```

This isn't much different than what you may have done before. You're going to create an intent. CATEGORY_OPENABLE means we only want results that can be opened--basically documents--as opposed to a list of contacts or time zones. The type is the document MIME type you want. Here I'm asking for any type of image. And then launch your intent with a request code. This can be any value you like, but it should by unique within your app. And we've launched the file picker. Next the user selects a document, and it's back to your app.

Retrieving a URI is also the same as before.

```java
// Retrieving a Uri
@Override
public void onActivityResult(int requestCode, int resultCode, Intent resultData) {
  if (requestCode == READ_REQUEST_CODE && resultCode == Activity.RESULT_OK) {
    if (resultData != null) {
      Uri uri = resultData.getData();
      getMetaData(uri);
    }
  }
}
```

we get a callback in `onActivityResult`. We get a callback in onActivityResult. We check the request code matches, and the result came back OK. Now we don't get back the specific document directly, here. We get a URI pointing to it using intent getData. If you've done this before with getContent or pick intents, nothing's changed. And this might look familiar too.

```java
public void getMetaData(Uri uri) {
  Coursor cursor = getActivity().getContentResolver().query(uri, null, null, null, null, null);
  if (cursor != null && cursor.moveToFirst()) {
    String displayName = cursor.getString(cursor.getColumnIndex(OpenableColumns.DISPLAY_NAME));
    int sizeIndex = cour.getColumnIndex(OpenableColumns.Size)
    String size = null
    if (!cursor.isNull(sizeIndex)) {
      size = cursor.getString(sizeIndex);
    } else {
      size = "Unknown"
    }
  }
  cursor.close();
}
```

With a URI, we can get metadata about the image. Here this query applies to only one document, and we want all the rows, so that's what all those null are for. Here I'm getting its display name--note that this is not the same as its file name--and its size. You can't count on getting a reliable size for a document, because it might be stored remotely, for example. So always check that it isn't null before you try and get the size.

```java
// opening a bitmap
private Bitmap getBitmapFromUri(Uri uri) throw IOException {
  ParcelFileDescriptor parcelFileDescriptor = getContentResolver().openFileDescriptor(uri, "r");
  FileDescriptor fileDescriptor = parcelFileDescriptor.getFileDescritor();
  Bitmap bitmap = BitmapFactory.decodeFileDescriptor(fileDescriptor);
  parcelFileDescriptor.close();
  return bitmap;
}
```

But forget metadata. You probably want to open this image. Here's a shortcut. You can use `ContentResolver.openFileDescriptor`. You pass in the URI and the access mode you want, and you get back a parcelFileDescriptor, which is a wrapper around a FileDescriptor. For images, there's a handy method in BitmapFactory, `decodeFileDescriptor`. Don't do this on the UI thread. Do in the background using an async task. There's an example of this in the storage client sample code that we're posting. And finally, it's best practice to wrap the close in a try finally block, so it's guaranteed to close. And then you can set this image to an image view. And here it is.

```java
// Get an InputStream from a Uri
private String readTextFromUri(Uri uri) throw IOException {
  InputStream inputStream = getContentResolver().openInputStream(uri);
  BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream));
  StringBuilder stringBuilder = new StringBuilder();
  String line;
  while((line = reader.readLine()) != null) {
    stringBuilder.append(line);
  }
  fileInputStream.close();
  return stringBuilder.toString();
}
```

Now wait, you're saying, I want to do something else, or I don't have an image. Easy. You can  get an inputStream from the `ContentResolver.openInputStream` and do whatever you want with it. Here I'm reading the lines of my file into a string.

```java
// Get an OutputStream from a Uri
// Check for flag Document.FLAG_SUPPORTS_WRITE to enable/disable a "Save" button
private String writeTextToUri(Uri uri) throw IOException {
  OutputStream outputStream = getContentResolver().openOutputStream(uri);
  outputStream.write(("Some text").getBytes());
  outputStream.close();
}
```

So here's a new one. We weren't able to do this before. You can get an outputStream from the `ContentResolver`. By default it uses write mode. You want to ask for the least amount of access you need, so don't ask for read/write if all you need is write. When I'm done, I just let the document provider know I'm done by closing the stream, which you have to do anyway. Simple.

```java
// Mutiple documents
Intent intent = new Intent(Intent.ACTION_OPEN_DOCUMENT);
intent.addCategory(Intent.CATEGORY_OPENABLE);
intent.putExtra(Intent.EXTRA_ALLOW_MULTIPLE, true);
intent.setType("*/*");
if (mimeTypes != null) {
  intent.putExtra(Intent.EXTRA_MIME_TYPES, mimeTypes);
}
```

So we could open one document before. What about lots of documents at the same time? All you have to do let the user select mutiples is add `EXTRA_ALLOW_MULTIPLE` to you intent. If you want, you can specify mutiple MIME types.

```java
// Multiple document Uris
@Override
public void onActivityResult(int requestCode, int resultCode, Intent resultData) {
  if (requestCode == READ_REQUEST_CODE && resultCode == Activity.RESULT_OK) {
    if (resultData == null) {
      return;
    }
    if (resultData.getData != null) {
      handleSingleDocument(resultData);
    } else {
      ClipData clipData = resultData.getClipData();
      if (clipData != null) {
        uris = new Uri[clipData.getItemCount()];
        for (int i = 0; i < clipData.getItemCount(); i++) {
          uris[i] = clipData.getItemAt(i).getUri();
        }
      }
    }
  }
}
```

This time we get back clip data. It's then stored in `intent.getClipData`, and you can get the URIs using `clipData.getItems` and then `item.getUri`. Note that you still have to check intent getData, the same as a single URI, because if the user picks just one document, it doesn't matter if you allowed multiple selection, it's still coming back the first way, in `intent.getData`. So you have to check both.

```java
// create document
// New Intent ACTION_CREATE_DOCUMENT
private void createFile(String mimeType, String fileName) {
  Intent intent = new Intent(Intennt.ACTION_CREATE_DOCUMENT);
  intent.setType("text/plain");
  intent.putExtra(Intent.EXTRA_TITLE, fileName);
  startActivityForResult(intent, CREATE_REQUEST_CODE);
}
```

Creating a document is also new in Android KitKat, and it's really straightforward. You give your intent a MIME type, a file name, and you launch it with a unique request code. The rest is taken care of for you. You get back its URI in `onActivityResult`, and that way, you can continue to write to it or whatever else you want.

```java
// Delete documents
// Check for flag Document.FLAG_SUPPORTS_DELETE
// Enable/disable "delete" button
DocumentsConstract.deleteDocument(getContentResovler(), uri);
```

And deleting a document is even easier. You can't launch an intent to delete a document, but if you have its URI, which you would if you've opened or created it, then you can ask to deleted it, and `DocumentsContact` does this for you. Again, in a document's metadata, you can check document column flags. If that contains support delete, you'll know whether to enable or disable your Delete button or Menu option.

```java
// Persisted permissions
// Use ContentResolver.getPersistedUriPermissions() to check for freshest data
// Documents can be delete, app removed, etc.
@Override
public void onActivityResult(int requestCode, int resultCode, Intent intent) {
  final int takeFlags = intent.getFlags() & Intent.FLAG_GRANT_READ_URI_PERMISSION | Intent.FLAG_GRANT_WRITE_URI_PERMISSION;
  getContentResolver().takePersistableUriPermission(urim takeFlags);
}
```

One more thing to mention, when you open a file for reading or writing, the system gives you a URI permisson grant for that file. It lasts until your device restarts. However, you might want to access that file again directly from your applications. Say you're an image editing app, and you want to show the user the last five images they've edited. If the device is turned of in the meantime, you won't have access to those. You could send the user back to the document picker, but that's far from ideal. Instead, you can persit the permissions the system gave you. Now they'll last no matter whether the phone is turned off or on. The system won't do this automatically for you. Your app has to explicitly request that the permissions be persisted. This is a security measure, though, so it's a good thing. One last note, you may have saved the most revent URIs your app accessed, but you should still always use `ContentResolver.getPersistedUriPermissions` to check that you have the freshest data. Other applications could delete documents, apps could be removed, so make this check to make sure your data's right.

# Android 4.4 KitKat Storage Access Framework: Provider

Before KitKat, you may have seen or implemented something like this.

```java
Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
intent.setType("file/*");
startActivityForResult(intent, RESULT_CODE);
```

```xml
<intent-filter>
  <action android:name="android.intent.action.GET_CONTENT" />
  <category android:name="android.intent.categrory.OPENABLE" />
  <category android:name="android.intent.categrory.DEFAULT" />
  <data android:mimeType="*/*"/>
</intent-filter>
```

**registerForActivityResult**

```kotlin
private fun performFileSearch(imageBitmap: MutableState<ImageBitmap>) {
    val intent = Intent(Intent.ACTION_OPEN_DOCUMENT)
    intent.addCategory(Intent.CATEGORY_OPENABLE)
    intent.type = "image/*"
    mSearchImageActivityResultLauncher.launch(intent)
    mBitmap?.let {
        imageBitmap.value = it.asImageBitmap()
    }
}

private val mSearchImageActivityResultLauncher =
    registerForActivityResult(ActivityResultContracts.StartActivityForResult()) {
    if (it.resultCode == RESULT_OK) {
        if (it.data != null && it.data!!.data != null) {
            it.data!!.data!!.let {
                val parcelFileDescriptor = contentResolver.openFileDescriptor(it, "r")
                val fileDescriptor= parcelFileDescriptor?.fileDescriptor
                mBitmap = BitmapFactory.decodeFileDescriptor(fileDescriptor)
            }
        }
    }
}
```