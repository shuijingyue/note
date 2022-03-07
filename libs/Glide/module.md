Loader的注册在Glide构造函数中

RegistryFactory.lazilyCreateAndInitializeRegistry -> createAndInitRegistry -> initializeDefaults
|->Registry -> append

glide如何监听view的可见性,释放资源
ViewTarget.addOnAttachStateListener