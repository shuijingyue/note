

ActivityStarter::startActivityInner
  ActivityStarter::recycleTask
    ActivityStarter::complyActivityFlags

FLAG_ACTIVITY_FORWARD_RESULT

A -> B -> C
A::startActivityForResult -> B::startActivity(intent) -> C::setResult(OK)

AndroidManifest.xml

taskAffinity 默认包名

ActivityStartSupervisor::resolveActivity

task管理 android 11之前 TaskRecord