[MUSIC PLAYING] JEFF SHARKEY: Hey, everyone. My name is Jeff Sharkey, and I'm a software engineer on the Android Framework Team. And today, we're going to be digging into files on Android and all the various places that Android gives you to store those files. So today, we'll dig into you starting by looking at some common locations that Android offers. The first broad category is internal storage, and this storage can be classified as safe and secure, because it's something the Android OS protects. It's part of the application sandbox model that we offer. You've probably encountered some of these directories before, like Context.getFilesDir. It's a great default location to store things. One that's slightly different is getCacheDir. We'll dig into that a little bit later today, but one thing to note is that files that you store in that location, the disk space is not counted against your application. And the reason for that is that Android reserves the right to go in there and delete some of those files if the user needs that disk space elsewhere. So it's a trade-off. It's a two way street. Another directory getNoBackupfilesDir, this can be useful if you have things like cloud messaging tokens that you want to renew when your app migrates between devices. If the device goes through a backup and restore phase, those files that you store in that directory, they won't be carried across the backup. So it be useful for that. Finally, getCodeCache directory. This is a great place to store things like jitted code or optimized code, and the OS will do two things. It will delete contents inside of the directory under two conditions-- either when your application is updated via Play Store or whenever the OS itself receives an upgrade. Say from the O release to the P release, it will clear the contents of that directory. So that's a summary of internal storage. Next, the other broad category of locations is external storage, and when we think of external storage, it's more of a shared area. And it's unprotected and the reason I just mentioned that is data that you store in that location, you may write it there. But other applications can request the storage permission on Android, and they may write that data or modify it without you knowing about it. And so it's something we definitely discourage storing sensitive contents in that location. If you do need to store data out there, consider finding a way to prove to verify the integrity of that data if you need to trust it. The directories here, files are similar to internal storage, cached are the same way. External media dirs, that you save in there will be scanned by media stored on the device. So it's a good place to store photos or videos that you want to be included in the user's Gallery application to be scanned and included there. GetObbDirs, Obb stands for Opaque binary blobs, and these are large files typically used for game developers that are delivered through Google Play. Data that you store in those locations, they're counted towards your app's code size instead of its data size. So so far, we've talked about broadly internal storage and external storage, and these are all great places for you to store data that belongs to your app. But you might find yourself creating data that belongs to the user, that the user may want to store in a different location. And that's a great place to use the storage Access Framework, and there's two intents that work great there-- Intent.ACTION_OPEN_DOCUMENT and CREATE_DOCUMENT. These have been around in the platform since the KitKat release, and you can think of them as an open and a save dialog box for the user. It really offers a great experience, because the user has control over exactly where those files are stored on the device. It gives them the freedom to choose any of those locations. It also opens the door for cloud storage providers. You don't have to integrate a cloud provider SDK into your application. You just simply launch the intent, and the user can select where they want that file to be stored. There's some great talks that have dug into this elsewhere, so I'd encourage you to search around online. There's some great content that digs more in depth. So we talked about some basic locations. Let's do a deep dive on two specific advanced locations today. One that we'll dig into is Direct Boot, and the second we'll dig into is Cache Data. So first, Direct Boot, which you may not have encountered before. And first, it's worth starting out like we built the Direct Boot feature in the Android N release to solve an important problem. Before the N release, when we encrypted an Android device, and if the user rebooted that device, no apps could run until after the user had entered their credentials-- a pin, pattern, or password. So what we did in the N releases, we created two storage areas that they're still encrypted, but they're encrypted with two different keys. And we call these areas the device protected area and the credential protected area. The device protected area becomes available by virtue of the device proving that it hasn't been tampered with. So when the device boots up, there is dm-verity verifies that the device hasn't been tampered with. By virtue of proving that, it unlocks that device protected storage. Then later, when the user enters that pin, pattern, or password, the credential protected storage becomes available for applications to use. If you haven't encountered these APIs before, rest assured that all of your data by default as an app developer is always stored in credential protected area. But if you find a place where you'd like to run before the user has unlocked their device, that's where it might be useful to store small bits of information out in that device protected area, so that your app can be useful while the device is locked. So then you might ask the question, how do you gain access to that storage area? The credential protected area, as I mentioned, context.getFilesDir offers that credential protected area. There's a method on context, and you can see it down here at the bottom of the screen, create DeviceProtectedStorageContext. It's a little bit of a mouthful. What it does is it returns another different context where the file APIs, referring to internal storage on that return to context, point at the device protected storage. So let's take a look at some code examples of how you might integrate with those APIs. One of the first things you'll need to do is if you want to become Direct Boot aware is to think about what data you want to keep on credential protected storage or migrate out to device protected storage. So a lot of you will be writing code like this to decide during that initial upgrade step how you want to migrate data back and forth. The first thing you'll probably do when starting your application is ask is the user currently unlocked? Has that pin, pattern, or password been offered? So the User Manager class offers, you can check is the current user unlocked? Assuming that they are unlocked, that means you have access to both the device and credential storage. And here we can see there's two move methods that are offered as helpers. You can move shared preferences between two locations, and you can also move databases back and forth. The reason we provide these helper methods is oftentimes shared preferences or databases can actually be made up of multiple files on disk, and some of that data may also be cached in memory. So by calling these helper methods to move that data around, we ensure that all the data gets moved and that any in-memory caches are invalidated along the way. When you think about data that you would want to migrate, one thing we say is only move the data you think you need to provide that user experience while the device is locked. So things like if you're building an alarm lock app, you'd probably move the user's next alarm time out into that device protected area to make sure the alarm clock would go off if the user's device is currently locked. Another strategy that we've seen used, if you have auth tokens to talk to a server, we've actually recommended people create a second type of auth token for your cloud server. One, auth token is a full, rich, full-access token, which you'd probably use today. We'd recommend keeping that in the credential protected area and creating a second much more limited in-scope auth token and only storing that limited in-scope auth token out in the device protected area. Maybe that auth token, when it talks to your server back end, is only able to return the fact that the user has three unread messages, and maybe it can't do any operations beyond that. So it helps you deliver the experience of getting the user's attention when you need to but without being able to access any of the additional richer information in your cloud. OK, so we've talked if it's unlocked, we can migrate data back and forth. The else clause here, we'd recommend that you register. There's a runtime broadcast that's sent, an action user unlocked, and that would allow you to then run code when the user enters that pin, pattern, or password. The middle code snippet here, those move methods, they work both directions. So if you accidentally move some data out into the device protected area, you can also move it right back into the credential area. And another tidbit, another API that might be useful at the bottom of the screen there, if you're ever wondering if a particular file is going to be encrypted at rest by the operating system, you can quickly check for that as well. There's a StorageManager isEncrypted API, and that can be useful if you're trying to decide if you want to roll your own encryption or rely on the encryption at rest that the OS provides. All right, so a second deep dive area that we'll dig into is cached data on the OS, and this is typically data that you can regenerate or redownload later on, if it happens to be deleted. And I mentioned earlier that this is a two way street. The OS won't count the data that you use in that area against your app, but at the same time, we reserve the right to go in there and delete some of that data if the user needs the disk space for something else that they're doing. And something that we improved in the O release is we rewrote the algorithms used internally. One of the biggest questions we got from you, from developers, was how much cache space is appropriate to use? Can I use 500 meg? Is that too much? Is 50 meg too much? So now we offer explicit guidance. There's an API on Storage Manager to figure out a cache quota for your application, a reasonable amount that the device thinks is reasonable for your app to use. And the nice thing is the OS adjusts that value over time. So if the user spends a lot of time in your application, we're going to increase that number to give you more cache space to work with, so that you can offer a better user experience to your users. Another thing that we did in the O release, we rewrote the internal implementation of how that data is cleared. Before the O release, we would literally list all of the cache files on the OS, sort them by modified time, and delete the oldest files. And you can imagine that there were ways you could gamify that system by setting your modified time out to like the year 2038. So we fixed that, and so now in O and future releases, the OS will delete data from apps that are most over their quota first. So what this means, if your app stays right around the cache quota that the OS has recommended, you can be pretty confident that your data will be there and will remain available, even as the user starts filling up their disk. So let's look again at some code snippets, like how do we use this in practice? So again going to Storage Manager, if you're integrating with a common class, say like Disk LRU cache, it's pretty easy to connect the two things together. You can ask the US for that recommended cache quota bytes and plug it straight into the disk color you cache to help it trim how much size it's using. If you have multiple types of caches, it's up to you to decide how you want to fractionally account or distribute that cache amongst here, the inside, internals of your app. The second code snippet there, if you're rolling some of your own caching, I'd point out there's a method called getCacheSizeBytes. This is a fast way to ask the question, how much cache space your app is currently using. It's an optimized call that will turn very quickly, and it's faster than you having to go iterate over your own disk usage to try to figure out how much space you're using. Another feature that we added in the O release which is covered at the bottom half of this slide is the ability to have cache behaviors, and we heard this from developers that it can be useful. You may download multiple files that really should be treated as a unit or a group. One concrete example is downloading say a movie file and a subtitles file that goes along with it. If you store both of those in the cache directory, if one of them gets deleted, the other file isn't really useful and valuable. So the cache behavior offers you a way to indicate, to tell us is the operating system, that if we need disk space, we should delete both of those things at the same time. All right, so we talked about some of the common storage locations. Let's switch gears and talk about how we can work together, both the OS helping you as developers. And one of the biggest things that we offered in the O release is the ability to help you get the disk space that you need. Before the O release, if you wanted to do a large download, let's say one gigabyte in size. And if you'd looked just at the free disk space, you may only see 500 megabytes were free, and it may look like that download was impossible. But there's a new API in the O release, where the OS will offer to go and delete cache files belonging to other applications to help free up the disk space for that operation to succeed for your application. If there still isn't enough disk space, there's new intents that you can launch to help get the user involved, to help them pick items and different things that they can do to help free up that disk space. So how do we use this API? Here's a snapshot of this. The very top of the slide, this is maybe the way that you've been writing code today. You'll just do a pretty simple check. You have a download size you're comparing it against. Java.io.File.getUsableSpace how much space is there? That's the operation that may look like it wouldn't be possible to succeed. But if you convert your code to using the rest with a code snippet on this slide, if you call the StorageManager.g etAllocatableBytes API, that will return not just the free space, it will also include space that the OS is willing to go free up on your behalf from other applications' cache data. So in this case, it may look like if we have enough space inside of that if block, we can actually open that file output stream. And now there's an API call allocate bytes, and what this will do is it will go actually claim that disk space, free application. Deep underneath, it'll use the F allocate system call to ensure that those blocks belong to your application, and that you do have the space that's guaranteed to belong to you. So that could be a useful API to use. And the else block here, if there still wasn't enough space, we now offer some great intents for you to ask the user for help to come along and free up information, free up the disk space. Sharing content, we've covered this a little bit before. How can we work together there? Please always use content URIs when you're sharing between applications. Never use file URIs, and the reason is that receiving application, they may not have the permissions that they need to directly access the files on disk. If you use content URIs, the OS can manage the dynamic permissions to give the receiving app, to make sure that they can open the content. If you find yourself in this position, File Provider in the support library is a great way to convert between the two with a single line of code every place to convert from file to content. And over the years, because this is an important thing to pay attention to, we built strict mode APIs to help you track down to detect these places in your app. You can detect places where you might be accidentally sharing file URIs, and now, in a more recent release, you can also detect places where you're sharing a content URI, and you might be forgetting the flag_grant_read or the flag_grant_write to go along with that intent. So those can be two APIs that are helpful to look at. Native code is another area to think about. We'd strongly recommend that you look at opening files up in higher level managed code, up in Java or Kotlin, and passing down the already open file descriptor, the integer, down into native code. And the reason for that is opening in a managed code gives the OS the opportunity to notice and inspect and correct things. In particular, it can look for strict mode violations. If you open a managed code, you may notice that the thread that you're currently running on is important to the user, may block or cause jank in your app, and we're going to start using this more in future Android releases. So this is why we want to strongly encourage, open the file in Java or the higher level language, pass the integer down to native code. Don't pass the file path itself across the JNI boundary. And a quick code snippet of what that looks like, you can pretty quickly do this with ParcelFileDescriptor. You can open a particular file on disk, maybe for read/write in this case, and there's a method called detachfd that will return that integer. It's just an int that is ready for you to pass across a JNI boundary as a jint. So that's the recommended design going forward. Another note that might be at a trick or a tip that could be useful. If you find yourself just jumping across JNI to do a handful of system calls, you might go look at Android.system.OS. There are several dozen POSIX syscalls ready for you to use up in Java today. We only added that a couple of releases ago. So you may be able to find that you can do those handful of syscalls purely in Java, and you may be able to get rid of the JNI and the native code in your application. So take a look at that. And finally, working with media, we'd really recommend that you use Media Store if you're looking to find the photos or videos that the user has on their device. And you might be tempted to go and build your own index of whatever media you find, but that can be pretty wasteful, both of CPU and battery for the user. And another note is we're actively working on improving Media Store and really adding functionality there, so stay tuned over the next couple releases. Another note is open files, open the content of that media, through Content Resolver. You may have noticed that there are columns across the operating system called underscore data and they return a raw file system path. Over time, you may have noticed that a handful of those underscore data columns have been deprecated in previous releases, and just to expect that's going to continue. So you'll notice more of those underscore data columns becoming deprecated over time. We really want to encourage people to move towards content URIs as a best practice. So yeah, thank you for your time, being able to dig into some of the nitty gritty areas of storage, and I'll be available in the Q&A section afterwards, if you have questions for me. Thanks for your time. [MUSIC PLAYING]