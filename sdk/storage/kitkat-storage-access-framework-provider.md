Before KitKat, you may have seen or implemented something like this.

```kotlin
val intent = Intent(Intent.ACTION_GET_CONTENT).apply { type = "file/*" }
startActivityForResult(intent, REQUEST_CODE)
```

```xml
<intent-filter>
    <action android:name="android.intent.action.GET_CONTENT"/>
    <category android:name="android.intent.category.OPENABLE"/>
    <category android:name="android.intent.category.DEFAULT"/>
    <data android:mimeType="*/*"/>
</intent-filter>

<intent-filter>
    <action android:name="android.intent.action.PICK"/>
    <category android:name="android.intent.category.DEFAULT"/>
</intent-filter>
```

You had `ACTION_GET_CONTENT` or `ACTION_PICK`, and you declared one or more intent filters in an activity in your manifest. And this is what it looked like. You would pick one app, and you would get one file back in your original application. 

In KitKat, we've introduced a beautiful new picker UI owned by the system. You can browse content from all apps, not just one. You can see recent files across all apps, and you can search within an app. You can edit and save files in place. And it supports traditional file hierarchies, but it's flexible enough for tag-based cloud storage. And finally, it supports multiple user accounts, or transient roots, like a USB storage provider, where it should only show up when the drive is plugged in. 

In the storage API, we've introduced two new intents, `OPEN_DOC` and `CREATE_DOC`. GET_CONTENT and PICK still work. And we've got a new permission, `MANAGE_DOCUMENTS`, **which only the system can hold**. So how to implement this.

```xml
<provider
    android:authorities="program.playio.provider"
    android:name="program.playio.provider.DocumentProvider"
    android:grantUriPermissions="true"
    android:exported="true"
    android:permission="android.permission.MANAGE_DOCUMENTS">

    <intent-filter>
        <action android:name="android.content.action.DOCUMENTS_PROVIDER"/>
    </intent-filter>
</provider>
```

1. First of all, in your *manifest*, set your target SDK to 19. 
2. Next, add your provider. You provider name in the manifest is the name of your Java class for your content provider. Mine is called My Cloud Provider. 
3. Name your *authority* your *package* named plus provider. 
4. Your provider should be **exported** because you want other applications to see it, namely the system. 
5. Also, add the `MANAGE_DOCUMENTS` permission. By default, a provider is available to everyone. Adding the `MANAGE_DOCUMENTS` permission restricts your provider to the system, **important for security**. 
6. Also, we've added intent filters to providers. Make sure yours has the `DOCUMENTS_PROVIDER` action so it shows up when the system searches for document providers. 

Here's a quick picture of the flow. As you can see, providers and clients don't interact directly, just as before. A client requests permission to interact with files, read, edit, create, et cetera. The system picker goes to each registered provider and shows the user the matching content roots. One more note before we get into the provider. Usually with content providers, you have to make your own contract document. In this case, it's done for you. All the constants for fields you might return and a lot of other really useful methods are in the class DocumentsContract. Here, these are the columns we're going to return in the cursor when we're queried for documents or the root. 

How you implement a provider. You must extend DocumentsProvider. It's an abstract class, and it has a minimum of four methods you must implement yourself. They're called in this order, `queryRoots`, followed by `queryChildDocuments`, and finally, possibly either `queryDocument` or `openDocument`. There's many more, but we're going to start with the most simple case, which is just supporting openDoc. 

Here's an example of what happens when the provider first queries the roots of all documented providers. The projection you see is an argument, just represents the specific fields the caller wants to get back. `ResolveRootProjection` is a method that returns either those fields or the full projection if the caller passed in null. So here, we're creating a new cursor, and we're adding one row to it, one root, a top level directory, like gallery or drive. Most applications will have one root. You might have more than one in the case of, say, multiple user accounts. In that case, just add a second row to the cursor. The one thing that's not given here is GetDocumentId for a file. Your implementation is going to depend on how you structure your file storage. What's important is that every file, including directories, have exactly one unique ID. Other apps might hold onto this ID, and it's an explicit part of the contract that it won't change. Here's what shows up when you query children. This method gets called when you choose an application's root in the Picker UI. It gets the child documents of a directory. It can be called at every level in the file hierarchy, not just the root. This sample implementation is very simple. It makes a new cursor with the requested columns, and then it adds information about every immediate child in the parent directory to the cursor. IncludeFile is very similar to what we just saw for the root. It adds the file display name, MIME type, size, and so forth. A child can be an image, another directory, any type of file. So queryDocument. One or both of queryDocument or openDocument will get called when the user selects a document. QueryDocument returns the same information that was passed in queryChildDocuments, but just that specific file, just the one of them. OpenDocument returns a parcel file descriptor, which another application can use to stream data. You can see that we're setting the access mode. And the system takes care of issuing URI permission grants for us. Those first four are enough to get your content provider up and running. But there are a lot more methods you can override. Recent documents, search, add thumbnails to your images. Your implementation of these may vary significantly, depending on what kind of back end you're running. I'm not going to go over them all here, but there's a sample implementation of each of these methods in the source code I'm posting. One more thing, security is often a large issue when you're sharing documents. Suppose you 're a password-protected cloud storage service, and you want to make sure the user is logged in before you start sharing their files. I'm assuming you have some existing method of authenticating the user. If not, I recommend Google Plus Integration. In any case, you're starting. Your user is not logged in. What you do in your content provider is to return zero roots, that is, an empty root cursor. This might be important if your user had been logged in previously and you had been providing a root full of documents. Now you don't want to. The other step to this is to call getContentResolver.notifyChange. Remember the documents contract? We're using it to make this URI. This tells the system to query the roots of our provider again, which will return a different value now because of the IF statement we just saw. So sample code and slides will be available in-- look at the description of the video for a link. Thanks for watching, and let's share some documents.