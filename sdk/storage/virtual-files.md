IAN LAKE: A computer file is a resource for storing information. Almost all the files you're used to working with are openable files-- files with a direct representation in bytes. But there's actually another kind of file-- a virtual file, a new part of the Storage Access Framework added in API 24. Now, on Android, a traditional file is known as an openable file. This comes from the fact that you can open a file stream and get directly at the bytes that make up the file via something like openFileDescriptor or openInputStream. And in most cases when you're using something like getContent or ActionOpenDocument, this is exactly what you'd want. So way back in API 1, Android added CATEGORY_OPENABLE. You'd add this to your intent, and then you could ensure that what you got back could be opened. Virtual files will appear, but they won't be selectable. If you're already using that category, never get a virtual file, and you don't need to worry about them. But if you don't include CATEGORY_OPENABLE, the user will be able to select both openable files and virtual files. In both cases, you'll be able to get the document's metadata and use other documents provider APIs, such as getting a thumbnail, if supported. But beyond that, you'll need to handle virtual files a bit differently. You can determine if you have a virtual document by using a method like this, which looks at the column flags for the URI. That FLAG_VIRTUAL_DOCUMENT is the indicator which marks a virtual file. With this information, you can also take advantage of one of API 24's other new features, alternate MIME types for the Storage Access Framework. Think of this as on-the-fly transcoding or exporting a virtual file into an openable file, such as a PDF file. GetStreamTypes determines what openable MIME types are available-- say, checking if we can get an image from the file. Then use openTypedAssetFileDescriptor and createInputStream to get an input stream for the file. This is really useful if you're just wanting a one-time export of a file, say, to send as an email attachment. But for local viewing and editing of the file, you'll need to rely on the app that provided the virtual file in the first place. In those cases, the virtual file serves more just as a link to a file, allowing you to use ACTION_VIEW intent to open the file in the native app, which would know how to interact with whatever crazy internal format or database or cloud storage system the document might have under the hood. So for developers interested in using virtual files in your documents provider, you want to make sure you add the VIRTUAL_DOCUMENT flag to each virtual document. Then make sure you have an activity that supports ACTION_VIEW for the MIME types you use on those virtual documents. That way, there's at least one app that can render the file. And if you want to provide an alternate MIME type that is openable, which is strongly recommended, override getDocumentStreamTypes and openTypedDocument. Just make sure to call the super implementations for non-virtual files. This ensures that every client app can call it, whether explicitly checking whether it's a virtual file or not. So make sure that you're explicitly including CATEGORY_OPENABLE if you want to ensure openable files, and handling virtual files correctly in every other case. Check out the links in the description for all the details as you continue to build better apps. [MUSIC PLAYING]