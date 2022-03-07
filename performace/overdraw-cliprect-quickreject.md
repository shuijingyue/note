[MUSIC PLAYING] COLT MCANLIS: For modern mobile applications, one of the easiest performance traps to fall into is drawing elements too many times. But the tools that are available to help you detect and fix this problem for standard views frequently don't help when you're using highly customized views. My name is Colt McAnlis. And the trick to fast performance involves updating your code to use a few simple APIs that help the hardware spend less time drawing. Now, if you remember, overdraw occurs when the hardware spin cycle's drawing pixels on the screen that don't end up contributing to the final image-- for example, when the Nav Drawer slides out in front of your current activity. See, we still have to draw all those original items under the Nav Drawer, which aren't actually visible to the final image and thus wasted time. Now, the Android system will try to reduce overdraw as much as it can by trying to avoid drawing any items that are completely hidden by another opaque surface. Now, for example, all those labels underneath the Nav Drawer won't actually waste processing time. But sadly, this technique doesn't extend itself to complex custom views where you're overriding the on draw method. In these cases, the underlying system doesn't have insight into how you're drawing your content, which makes it really hard for it to know what to avoid. But you can help the system get a better visibility into this process by utilizing the canvas.cliprect API. This function allows you to define the drawable boundaries for your given view. Only the stuff inside the clipping rectangle will actually be drawn. Everything else will be ignored. This API can be really helpful for custom views with lots of overlapping components. For example, if you've got a set of stacked UI cards, you can determine what part of the current card is hidden by the one on top of it and set your cliprect bounds to ignore that area entirely. What's nice here is that cliprect helps save performance on the CPU and on the GPU. See, on the CPU side, each one of those canvas draw commands have a bit of overhead associated with them once they're actually submitted to OpenGL ES to draw. So any draw commands that lie entirely outside the cliprect won't be submitted to the hardware and thus won't produce that overhead. Now, anything that's partially intersecting the cliprect will still be drawn, which is why, on the GPU side, cliprect will help define an exclusion rectangle that allows the GPU at a per pixel level to avoid coloring anything that's actually clipped. In addition to cliprect, you can also use the quickreject API, which will allow you to test for cliprect intersection inside of your on draw function. If you've got some part of your view that takes up a lot of processing time, quickreject can help tell you if it's outside the cliprect and let you skip all of that processing altogether. And don't forget that you can see the impact of overdraw right on your device with the help of the Show GPU Overdraw tool. This tool visualize how much overdraw is happening in your app by tinting the pixels different colors. Lots of red is really bad in this regard. If you're looking to know more about overdraw and validations and how it affects your performance, make sure to check out the rest of the Android performance patterns resources on the web. Oh, and don't forget to join our Google+ community for more great tips and tricks. So keep calm, profile your code, and always remember Perf Matters. [MUSIC PLAYING]