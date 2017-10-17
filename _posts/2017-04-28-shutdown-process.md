---
layout: post
title: 关机流程
categories: [Android]
description: 分析android从按下power键到关机的过程
keywords: Android, Kernel, shutdown
---

## 简介  
系统关机一般有以下几种情况：  
1.	正常关机：长按power键，系统弹出关机菜单，选择关机按钮，系统关机；  
2.	强制关机：长按power键，直至系统掉电；  
3.	异常关机：断开电源、系统崩溃。  

## 正常关机  
### 内核接收power键触发  
按下power键，内核接收到该消息后，把该事件上报给上层，上层会判断power被按下的时间长度，做出不同的响应，如：短按power时关闭显示进入休眠，长按power的时候弹出关机选择菜单。  
现在，我们来看看power键的原理图，如下：  
![rk3399-power-key1](https://linjc.github.io/images/posts/rockchip/rk3399-power-key1.jpg)  
![rk3399-power-key2](https://linjc.github.io/images/posts/rockchip/rk3399-power-key1.jpg)  
从原理图可知，当我们按下POWER_KEY的时候，拉低了PWR_KEY_L，因此给主控的GPIO0_A5脚一个中断信号。我们从源码分析：  
kernel/arch/arm64/boot/dts/rockchip/rk3399-android.dtsi，我们可以看到如此定义：  
```
 114
 115     rk_key: rockchip-key {
 116         compatible = "rockchip,key";
 117         status = "okay";
 118
 119         io-channels = <&saradc 1>;
		 ......
 133         power-key {
 134             gpios = <&gpio0 5 GPIO_ACTIVE_LOW>;
 135             linux,code = <116>;
 136             label = "power";
 137             gpio-key,wakeup;
 138         };
     ......
```
设备树rockchip-key定义了板子上的所有key，驱动通过compatible来匹配这里的定义，其中,power-key就是power键的相关定义，”gpio0 5”即”GPIO0_A5”。  
在驱动实现kernel/drivers/input/keyboard/rk_keys.c中的probe函数，通过rk_keys_parse_dt解析dts文件定义的信息，给符合” button->type == TYPE_GPIO”的power键申请中断，中断函数为：`static irqreturn_t keys_isr(int irq, void *dev_id)`，函数中的keys_irs，通过input_event把中断上报给framworks层。
### Framworks层接收power键触发
在frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java中，interceptKeyBeforeQueueing监听KeyEvent.KEYCODE_POWER事件：  
```
 5850             case KeyEvent.KEYCODE_POWER: {
 5851                 result &= ~ACTION_PASS_TO_USER;
 5852                 isWakeKey = false; // wake-up will be handled separately
 5853                 if (down) {
 5854                     interceptPowerKeyDown(event, interactive);
 5855                 } else {
 5856                     interceptPowerKeyUp(event, interactive, canceled);
 5857                 }
 5858                 break;
 5859             }
```
按下power键时，进入interceptPowerKeyDown，松开时，进入interceptPowerKeyUp；至于长按，那就得具体看interceptPowerKeyDown的实现了。具体实现：  
```
 1025     private void interceptPowerKeyDown(KeyEvent event, boolean interactive) {
 1026         // Hold a wake lock until the power key is released.
 1027         if (!mPowerKeyWakeLock.isHeld()) {
 1028             mPowerKeyWakeLock.acquire(); // 持有锁
 1029         }
 1030
 1031         // Cancel multi-press detection timeout.
 1032         if (mPowerKeyPressCounter != 0) {  // 多次按下，只处理一次
 1033             mHandler.removeMessages(MSG_POWER_DELAYED_PRESS);
 1034         }       
 1035
 1036         // Detect user pressing the power button in panic when an application has
 1037         // taken over the whole screen.
 1038         boolean panic = mImmersiveModeConfirmation.onPowerKeyDown(interactive,
 1039                 SystemClock.elapsedRealtime(), isImmersiveMode(mLastSystemUiFlags),
 1040                 isNavBarEmpty(mLastSystemUiFlags));
 1041         if (panic) {
 1042             mHandler.post(mHiddenNavPanic);
 1043         }
 1044
 1045         // Latch power key state to detect screenshot chord.
 1046         if (interactive && !mScreenshotChordPowerKeyTriggered
 1047                 && (event.getFlags() & KeyEvent.FLAG_FALLBACK) == 0) {
 1048             mScreenshotChordPowerKeyTriggered = true;
 1049             mScreenshotChordPowerKeyTime = event.getDownTime();
 1050             interceptScreenshotChord();
 1051         }
 1052
 1053         // Stop ringing or end call if configured to do so when power is pressed.
 1054         TelecomManager telecomManager = getTelecommService();
 1055         boolean hungUp = false;
 1056         if (telecomManager != null) { //来电时按power-key，取消响铃，
 1057             if (telecomManager.isRinging()) {
 1058                 // Pressing Power while there's a ringing incoming
 1059                 // call should silence the ringer.
 1060                 telecomManager.silenceRinger();
 1061             } else if ((mIncallPowerBehavior
 1062                     & Settings.Secure.INCALL_POWER_BUTTON_BEHAVIOR_HANGUP) != 0
 1063                     && telecomManager.isInCall() && interactive) {
 1064                 // Otherwise, if "Power button ends call" is enabled,
 1065                 // the Power button will hang up any current active call.
 1066                 hungUp = telecomManager.endCall();
 1067             }
 1068         }
 1069
 1070         GestureLauncherService gestureService = LocalServices.getService(
 1071                 GestureLauncherService.class);
 1072         boolean gesturedServiceIntercepted = false;
 1073         if (gestureService != null) {
 1074             gesturedServiceIntercepted = gestureService.interceptPowerKeyDown(event, interactive,
 1075                     mTmpBoolean);
 1076             if (mTmpBoolean.value && mGoingToSleep) {
 1077                 mCameraGestureTriggeredDuringGoingToSleep = true;
 1078             }
 1079         }
 1081         // If the power key has still not yet been handled, then detect short
 1082         // press, long press, or multi press and decide what to do.
 1083         mPowerKeyHandled = hungUp || mScreenshotChordVolumeDownKeyTriggered
 1084                 || mScreenshotChordVolumeUpKeyTriggered || gesturedServiceIntercepted;
 1085         if (!mPowerKeyHandled) {  //powerKeyHandled处理
 1086             if (interactive) { //未休眠
 1087                 // When interactive, we're already awake.
 1088                 // Wait for a long press or for the button to be released to decide what to do.
 1089                 if (hasLongPressOnPowerBehavior()) { //是否支持长按键
 1090                     Message msg = mHandler.obtainMessage(MSG_POWER_LONG_PRESS);
 1091                     msg.setAsynchronous(true);
 1092                     mHandler.sendMessageDelayed(msg, //调用sendMessageDelayed方法发送消息，最终会在PolicyHandler的handleMessage进行处理该消息
 1093                             ViewConfiguration.get(mContext).getDeviceGlobalActionKeyTimeout());
 1094                 }
 1095             } else {  //其他情况
 1096                 wakeUpFromPowerKey(event.getDownTime());
 1097
 1098                 if (mSupportLongPressPowerWhenNonInteractive && hasLongPressOnPowerBehavior()) {
 1099                     Message msg = mHandler.obtainMessage(MSG_POWER_LONG_PRESS);
 1100                     msg.setAsynchronous(true);
 1101                     mHandler.sendMessageDelayed(msg,
 1102                             ViewConfiguration.get(mContext).getDeviceGlobalActionKeyTimeout());
 1103                     mBeganFromNonInteractive = true;
 1104                 } else {
 1105                     final int maxCount = getMaxMultiPressPowerCount();
 1106
 1107                     if (maxCount <= 1) {
 1108                         mPowerKeyHandled = true;
 1109                     } else {
 1110                         mBeganFromNonInteractive = true;
 1111                     }
 1112                 }
 1113             }
 1114         }
 1115     }
```
handleMessage函数：  
```
  694         public void handleMessage(Message msg) {
  695             switch (msg.what) {
  ......
  736                 case MSG_POWER_LONG_PRESS:
  737                     powerLongPress();//处理函数进入powerLongPress,上下文中处理不同按键的逻辑
  738                     break;
  ......
```
于是跳转到：  
```
1162     private void powerLongPress() {
1163         final int behavior = getResolvedLongPressOnPowerBehavior();//这里获取操作方式
1164         switch (behavior) {
1165         case LONG_PRESS_POWER_NOTHING://什么都不做
1166             break;
1167         case LONG_PRESS_POWER_GLOBAL_ACTIONS://确认后关机
1168             mPowerKeyHandled = true;
1169             if (!performHapticFeedbackLw(null, HapticFeedbackConstants.LONG_PRESS, false)) {
1170                 performAuditoryFeedbackForAccessibilityIfNeed();
1171             }
1172             if("vr".equals(android.os.SystemProperties.get("ro.target.product","unknown"))){
1173                 Intent intent = new Intent("com.rockchip.vr.action.power");
1174                 mContext.sendBroadcast(intent);
1175             }else{
1176                showGlobalActionsInternal();
1177             }
1178             
1179             //showGlobalActionsInternal();
1180             break;
1181         case LONG_PRESS_POWER_SHUT_OFF:
1182         case LONG_PRESS_POWER_SHUT_OFF_NO_CONFIRM://直接关机(工厂模式测试时)
1183             mPowerKeyHandled = true;
1184             performHapticFeedbackLw(null, HapticFeedbackConstants.LONG_PRESS, false);
1185             sendCloseSystemWindows(SYSTEM_DIALOG_REASON_GLOBAL_ACTIONS);
1186             mWindowManagerFuncs.shutdown(behavior == LONG_PRESS_POWER_SHUT_OFF);
1187             break;
1188         }   
1189     }
```
从上面的代码，我们知道长按power按有三种操作方式：什么都不做、确认后再关机、直接关机。而决定这个操作的是getResolvedLongPressOnPowerBehavior()，getResolvedLongPressOnPowerBehavior()的原型：  
```
1208     private int getResolvedLongPressOnPowerBehavior() {
1209         if (FactoryTest.isLongPressOnPowerOffEnabled()) {
1210             return LONG_PRESS_POWER_SHUT_OFF_NO_CONFIRM;
1211         }
1212         return mLongPressOnPowerBehavior;
1213     }
```
再看isLongPressOnPowerOffEnabled():  
在frameworks/base/core/java/android/os/FactoryTest.java中的定义：  
```
46     public static boolean isLongPressOnPowerOffEnabled() {
47         return SystemProperties.getInt("factory.long_press_power_off", 0) != 0;
48     }
```
所以如果想要直接关机，可以添加factory.long_press_power_off属性，如果要其他模式，那就得再看mLongPressOnPowerBehavior，  
```
1512         mLongPressOnPowerBehavior = mContext.getResources().getInteger(
1513                 com.android.internal.R.integer.config_longPressOnPowerBehavior);
```
在frameworks/base/core/res/res/values/config.xml中可以看到：  
```
740     <integer name="config_longPressOnPowerBehavior">1</integer>
```
这里默认是1(即LONG_PRESS_POWER_GLOBAL_ACTIONS)，如果想要执行LONG_PRESS_POWER_NOTHING，把上面的config_longPressOnPowerBehavior设为0即可。  
再看关机菜单，关机菜单在这个函数实现showGlobalActionsInternal():  
```
1273     void showGlobalActionsInternal() {
1274         sendCloseSystemWindows(SYSTEM_DIALOG_REASON_GLOBAL_ACTIONS); //发送由于全局关机动作的原因
1275         if (mGlobalActions == null) {
1276             mGlobalActions = new GlobalActions(mContext, mWindowManagerFuncs);
1277         }
1278         final boolean keyguardShowing = isKeyguardShowingAndNotOccluded();
1279         mGlobalActions.showDialog(keyguardShowing, isDeviceProvisioned()); //显示关机对话框
1280         if (keyguardShowing) {
1281             // since it took two seconds of long press to bring this up,
1282             // poke the wake lock so they have some time to see the dialog.
1283             mPowerManager.userActivity(SystemClock.uptimeMillis(), false);
1284         }
1285     }
```
showDialog函数的定义在：frameworks/base/services/core/java/com/android/server/policy/GlobalActions.java；  
```
171     public void showDialog(boolean keyguardShowing, boolean isDeviceProvisioned) {
172         mKeyguardShowing = keyguardShowing;
173         mDeviceProvisioned = isDeviceProvisioned;
174         if (mDialog != null) {
175             mDialog.dismiss();
176             mDialog = null;
177             // Show delayed, so that the dismiss of the previous dialog completes
178             mHandler.sendEmptyMessage(MESSAGE_SHOW);
179         } else {
180             handleShow();
181         }
182     }
```
这里handleShow()才是对话框的实际操作实现，handleShow的原型：  
```
196     private void handleShow() {
197         awakenIfNecessary();
198         mDialog = createDialog(); //创建全局关机对话框
199         prepareDialog(); //更新各个模式如静音、飞行
200
201         // If we only have 1 item and it's a simple press action, just do this action.
202         if (mAdapter.getCount() == 1
203                 && mAdapter.getItem(0) instanceof SinglePressAction
204                 && !(mAdapter.getItem(0) instanceof LongPressAction)) {
205             ((SinglePressAction) mAdapter.getItem(0)).onPress(); //进入onPress
206         } else {
207             WindowManager.LayoutParams attrs = mDialog.getWindow().getAttributes();
208             attrs.setTitle("GlobalActions");
209             mDialog.getWindow().setAttributes(attrs);
210             mDialog.show();
211             mDialog.getWindow().getDecorView().setSystemUiVisibility(View.STATUS_BAR_DISABLE_EXPAND);
212         }
213     }
```
在该方法中，首先调用awakenIfNecessary方法进行了屏幕唤醒，然后调用createDialog()创建全局关机对话框，当对话框创建完成后，调用prepareDialog方法进行keyguard窗口风格样式的设置，最后会进行我们全局关机对匡样式进行判断，如果是只有一个item，则会通过onPress方法进行处理，否则会进行系统UI显示的设置。此处我们进入creatDialog方法进行创建全局关机对话框。  
onPress的实现方法：  
```
375         @Override
376         public void onPress() {
377             // shutdown by making sure radio and power are handled accordingly.
378             mWindowManagerFuncs.shutdown(false /* confirm */);
379         }
```
shutdown函数定义在：frameworks/base/services/core/java/com/android/server/power/ShutdownThread.java  
```
132     public static void shutdown(final Context context, String reason, boolean confirm) {
133         mReboot = false;
134         mRebootSafeMode = false;
135         mReason = reason;
136         shutdownInner(context, confirm);
137     }
```
再看：shutdownInner（context, confirm);  
```
139     static void shutdownInner(final Context context, boolean confirm) {
140         // ensure that only one thread is trying to power down.
141         // any additional calls are just returned                                                                                                                                                
142         synchronized (sIsStartedGuard) {
143             if (sIsStarted) {
144                 Log.d(TAG, "Request to shutdown already running, returning.");
145                 return;
146             }
147         }       
148                 
149         final int longPressBehavior = context.getResources().getInteger(
150                         com.android.internal.R.integer.config_longPressOnPowerBehavior);
151         final int resourceId = mRebootSafeMode
152                 ? com.android.internal.R.string.reboot_safemode_confirm
153                 : (longPressBehavior == 2
154                         ? com.android.internal.R.string.shutdown_confirm_question
155                         : com.android.internal.R.string.shutdown_confirm);
156             
157         Log.d(TAG, "Notifying thread to start shutdown longPressBehavior=" + longPressBehavior);
158             
159         if (confirm) { //显示关机确认框
160             final CloseDialogReceiver closer = new CloseDialogReceiver(context);
161             if (sConfirmDialog != null) {
162                 sConfirmDialog.dismiss();
163             }       
164             sConfirmDialog = new AlertDialog.Builder(context)
165                     .setTitle(mRebootSafeMode
166                             ? com.android.internal.R.string.reboot_safemode_title
167                             : com.android.internal.R.string.power_off)
168                     .setMessage(resourceId)
169                     .setPositiveButton(com.android.internal.R.string.yes, new DialogInterface.OnClickListener() {
170                         public void onClick(DialogInterface dialog, int which) {
171                             beginShutdownSequence(context);
172                         }
173                     })
174                     .setNegativeButton(com.android.internal.R.string.no, null)
175                     .create();
176             closer.dialog = sConfirmDialog;
177             sConfirmDialog.setOnDismissListener(closer);
178             sConfirmDialog.getWindow().setType(WindowManager.LayoutParams.TYPE_KEYGUARD_DIALOG);
179             sConfirmDialog.show();
180         } else { //不显示关机确认框
181             beginShutdownSequence(context);
182         }
183     }
```
以上可以，无论有没有显示关机确认框，都会调用beginShutdownSequence函数进入关机流程，再看beginShutdownSequence：
```
243     private static void beginShutdownSequence(Context context) {
244         synchronized (sIsStartedGuard) {
245             if (sIsStarted) {
246                 Log.d(TAG, "Shutdown sequence already running, returning.");
247                 return;
248             }
249             sIsStarted = true;
250         }
251
252         // Throw up a system dialog to indicate the device is rebooting / shutting down.
253         ProgressDialog pd = new ProgressDialog(context);
254
255         // Path 1: Reboot to recovery for update
256         //   Condition: mReason == REBOOT_RECOVERY_UPDATE
257         //
258         //  Path 1a: uncrypt needed
259         //   Condition: if /cache/recovery/uncrypt_file exists but
260         //              /cache/recovery/block.map doesn't.
261         //   UI: determinate progress bar (mRebootHasProgressBar == True)
262         //
263         // * Path 1a is expected to be removed once the GmsCore shipped on
264         //   device always calls uncrypt prior to reboot.
265         //
266         //  Path 1b: uncrypt already done
267         //   UI: spinning circle only (no progress bar)
268         //
269         // Path 2: Reboot to recovery for factory reset
270         //   Condition: mReason == REBOOT_RECOVERY
271         //   UI: spinning circle only (no progress bar)
272         //
273         // Path 3: Regular reboot / shutdown
274         //   Condition: Otherwise
275         //   UI: spinning circle only (no progress bar)
276         if (PowerManager.REBOOT_RECOVERY_UPDATE.equals(mReason)) { //重启进入recovery升级系统
277             // We need the progress bar if uncrypt will be invoked during the
278             // reboot, which might be time-consuming.
279             mRebootHasProgressBar = RecoverySystem.UNCRYPT_PACKAGE_FILE.exists()
280                     && !(RecoverySystem.BLOCK_MAP_FILE.exists());
281             pd.setTitle(context.getText(com.android.internal.R.string.reboot_to_update_title));
282             if (mRebootHasProgressBar) {
283                 pd.setMax(100);
284                 pd.setProgress(0);
285                 pd.setIndeterminate(false);
286                 pd.setProgressNumberFormat(null);
287                 pd.setProgressStyle(ProgressDialog.STYLE_HORIZONTAL);
288                 pd.setMessage(context.getText(
289                             com.android.internal.R.string.reboot_to_update_prepare));
290             } else {
291                 pd.setIndeterminate(true);
292                 pd.setMessage(context.getText(
293                             com.android.internal.R.string.reboot_to_update_reboot));
294             }   
295         } else if (PowerManager.REBOOT_RECOVERY.equals(mReason)) { //重启进入recovery恢复出厂设置
296             // Factory reset path. Set the dialog message accordingly.
297             pd.setTitle(context.getText(com.android.internal.R.string.reboot_to_reset_title));
298             pd.setMessage(context.getText(
299                         com.android.internal.R.string.reboot_to_reset_message));
300             pd.setIndeterminate(true);
301         } else {  //正常重启或关机
302             pd.setTitle(context.getText(com.android.internal.R.string.power_off));
303             pd.setMessage(context.getText(com.android.internal.R.string.shutdown_progress));
304             pd.setIndeterminate(true);
305         }
306         pd.setCancelable(false);
307         pd.getWindow().setType(WindowManager.LayoutParams.TYPE_KEYGUARD_DIALOG);
308             
309         pd.show();
310             
311         sInstance.mProgressDialog = pd;
312         sInstance.mContext = context;
313         sInstance.mPowerManager = (PowerManager)context.getSystemService(Context.POWER_SERVICE);
314             
315         // make sure we never fall asleep again
316         sInstance.mCpuWakeLock = null;
317         try {
318             sInstance.mCpuWakeLock = sInstance.mPowerManager.newWakeLock(
319                     PowerManager.PARTIAL_WAKE_LOCK, TAG + "-cpu");
320             sInstance.mCpuWakeLock.setReferenceCounted(false);
321             sInstance.mCpuWakeLock.acquire();
322         } catch (SecurityException e) {
323             Log.w(TAG, "No permission to acquire wake lock", e);
324             sInstance.mCpuWakeLock = null;
325         }
326
327         // also make sure the screen stays on for better user experience
328         sInstance.mScreenWakeLock = null;
329         if (sInstance.mPowerManager.isScreenOn()) {
330             try {
331                 sInstance.mScreenWakeLock = sInstance.mPowerManager.newWakeLock(
332                         PowerManager.FULL_WAKE_LOCK, TAG + "-screen");
333                 sInstance.mScreenWakeLock.setReferenceCounted(false);
334                 sInstance.mScreenWakeLock.acquire();
335             } catch (SecurityException e) {
336                 Log.w(TAG, "No permission to acquire wake lock", e);
337                 sInstance.mScreenWakeLock = null;
338             }
339         }
340         
341         // start the thread that initiates shutdown
342         sInstance.mHandler = new Handler() {
343         };      
344         sInstance.start(); //进入到ShutdownThread线程的run
345     }
```
run的具体实现：  
```
358     public void run() {
359         BroadcastReceiver br = new BroadcastReceiver() {
360             @Override public void onReceive(Context context, Intent intent) {
361                 // We don't allow apps to cancel this, so ignore the result.
362                 actionDone();
363             }
364         };
365
366         /*
367          * Write a system property in case the system_server reboots before we
368          * get to the actual hardware restart. If that happens, we'll retry at
369          * the beginning of the SystemServer startup.
370          */
371         {
372             String reason = (mReboot ? "1" : "0") + (mReason != null ? mReason : "");
373             SystemProperties.set(SHUTDOWN_ACTION_PROPERTY, reason); //记录关机原因
374         }
375
376         /*
377          * If we are rebooting into safe mode, write a system property
378          * indicating so.
379          */
380         if (mRebootSafeMode) { //如果是安全模式关机，写属性"persist.sys.safemode"
381             SystemProperties.set(REBOOT_SAFEMODE_PROPERTY, "1");
382         }
383
384         Log.i(TAG, "Sending shutdown broadcast..."); //发送广播
385
386         // First send the high-level shut down broadcast.
387         mActionDone = false;
388         Intent intent = new Intent(Intent.ACTION_SHUTDOWN);
389         intent.addFlags(Intent.FLAG_RECEIVER_FOREGROUND);
390         mContext.sendOrderedBroadcastAsUser(intent,
391                 UserHandle.ALL, null, br, mHandler, 0, null, null);
392
393         final long endTime = SystemClock.elapsedRealtime() + MAX_BROADCAST_TIME;
394         synchronized (mActionDoneSync) {
395             while (!mActionDone) {
396                 long delay = endTime - SystemClock.elapsedRealtime();
397                 if (delay <= 0) {
398                     Log.w(TAG, "Shutdown broadcast timed out");
399                     break;
400                 } else if (mRebootHasProgressBar) {
401                     int status = (int)((MAX_BROADCAST_TIME - delay) * 1.0 *
402                             BROADCAST_STOP_PERCENT / MAX_BROADCAST_TIME);
403                     sInstance.setRebootProgress(status, null);
404                 }
405                 try {
406                     mActionDoneSync.wait(Math.min(delay, PHONE_STATE_POLL_SLEEP_MSEC));
407                 } catch (InterruptedException e) {
408                 }
409             }
410         }
411         if (mRebootHasProgressBar) {
412             sInstance.setRebootProgress(BROADCAST_STOP_PERCENT, null);
413         }
414
415         Log.i(TAG, "Shutting down activity manager...");//关闭activity manager
416
417         final IActivityManager am =
418             ActivityManagerNative.asInterface(ServiceManager.checkService("activity"));
419         if (am != null) {
420             try {
421                 am.shutdown(MAX_BROADCAST_TIME);
422             } catch (RemoteException e) {
423             }
424         }
425         if (mRebootHasProgressBar) {
426             sInstance.setRebootProgress(ACTIVITY_MANAGER_STOP_PERCENT, null);
427         }
428
429         Log.i(TAG, "Shutting down package manager...");//关闭package manager
430
431         final PackageManagerService pm = (PackageManagerService)
432             ServiceManager.getService("package");
433         if (pm != null) {
434             pm.shutdown();
435         }
436         if (mRebootHasProgressBar) {
437             sInstance.setRebootProgress(PACKAGE_MANAGER_STOP_PERCENT, null);
438         }
439         
440         // Shutdown radios.
441         shutdownRadios(MAX_RADIO_WAIT_TIME); //关闭通信相关
442         if (mRebootHasProgressBar) {
443             sInstance.setRebootProgress(RADIO_STOP_PERCENT, null);
444         }   
445         
446         // Shutdown MountService to ensure media is in a safe state
447         IMountShutdownObserver observer = new IMountShutdownObserver.Stub() {
448             public void onShutDownComplete(int statusCode) throws RemoteException {
449                 Log.w(TAG, "Result code " + statusCode + " from MountService.shutdown");
450                 actionDone();
451             }
452         };
453         
454         Log.i(TAG, "Shutting down MountService");
455                 
456         // Set initial variables and time out time.
457         mActionDone = false;
458         final long endShutTime = SystemClock.elapsedRealtime() + MAX_SHUTDOWN_WAIT_TIME;
459         synchronized (mActionDoneSync) {
460             try {
461                 final IMountService mount = IMountService.Stub.asInterface(
462                         ServiceManager.checkService("mount"));
463                 if (mount != null) {
464                     mount.shutdown(observer);//关闭挂载服务
465                 } else {
466                     Log.w(TAG, "MountService unavailable for shutdown");
467                 }
468             } catch (Exception e) {
469                 Log.e(TAG, "Exception during MountService shutdown", e);
470             }       
471             while (!mActionDone) {
472                 long delay = endShutTime - SystemClock.elapsedRealtime();
473                 if (delay <= 0) {
474                     Log.w(TAG, "Shutdown wait timed out");
475                     break;
476                 } else if (mRebootHasProgressBar) {
477                     int status = (int)((MAX_SHUTDOWN_WAIT_TIME - delay) * 1.0 *
478                             (MOUNT_SERVICE_STOP_PERCENT - RADIO_STOP_PERCENT) /
479                             MAX_SHUTDOWN_WAIT_TIME);
480                     status += RADIO_STOP_PERCENT;
481                     sInstance.setRebootProgress(status, null);
482                 }
483                 try {
484                     mActionDoneSync.wait(Math.min(delay, PHONE_STATE_POLL_SLEEP_MSEC));
485                 } catch (InterruptedException e) {
486                 }   
487             }
488         }       
489         if (mRebootHasProgressBar) {
490             sInstance.setRebootProgress(MOUNT_SERVICE_STOP_PERCENT, null);
491                 
492             // If it's to reboot to install an update and uncrypt hasn't been
493             // done yet, trigger it now.
494             uncrypt();
495         }
496             
497         rebootOrShutdown(mContext, mReboot, mReason);//进入rebootOrShutdown
498     }
```
再看rebootOrShutdown  
```
643     public static void rebootOrShutdown(final Context context, boolean reboot, String reason) {
644         if (reboot) { //重启
645             Log.i(TAG, "Rebooting, reason: " + reason);
646             PowerManagerService.lowLevelReboot(reason);
647             Log.e(TAG, "Reboot failed, will attempt shutdown instead");
648             reason = null;
649         } else if (SHUTDOWN_VIBRATE_MS > 0 && context != null) { //关机前是否振动
650             // vibrate before shutting down
651             Vibrator vibrator = new SystemVibrator(context);
652             try {
653                 vibrator.vibrate(SHUTDOWN_VIBRATE_MS, VIBRATION_ATTRIBUTES);
654             } catch (Exception e) {
655                 // Failure to vibrate shouldn't interrupt shutdown.  Just log it.
656                 Log.w(TAG, "Failed to vibrate during shutdown.", e);
657             }
658
659             // vibrator is asynchronous so we need to wait to avoid shutting down too soon.
660             try {
661                 Thread.sleep(SHUTDOWN_VIBRATE_MS);
662             } catch (InterruptedException unused) {
663             }
664         }
665
666         // Shutdown power
667         Log.i(TAG, "Performing low-level shutdown...");
668         PowerManagerService.lowLevelShutdown(reason); //进入lowLevelShutdown
669     }
```
lowLevelShutdown的实现在frameworks/base/services/core/java/com/android/server/power/PowerManagerService.java  
```
2795     public static void lowLevelShutdown(String reason) {
2796         if (reason == null) {
2797             reason = "";
2798         }
2799         SystemProperties.set("sys.powerctl", "shutdown," + reason);
2800     }
```
lowLevelShutdown只是通过设置系统属性sys.powerctl为shutdown来实现系统关机。我们也可以通过adb命令通过修改Android系统的属性进行关机，指令：`adb shell setprop sys.powerctl shutdown`  
接下来就是kernel层的关机了。  
### Kernel层关机
通过分析按下power键到确认关机，经过上层的一系列处理，最后只是用SystemProperties.set设置了一个属性，下面分析从设置这个属性到kernel关机是怎么处理的。  
sys.powerctl属性触发开关在/init.rc中定义：  
```
on property:sys.powerctl=*                                                      
        powerctl ${sys.powerctl}
```
`on property:sys.powerctl=*`表示把sys.powerctl设置为任何值都会触发`powerctl ${sys.powerctl}`，${sys.powerctl}表示sys.powerctl设置的值。  
在system/core/init/builtins.cpp中：  
```
994 BuiltinFunctionMap::Map& BuiltinFunctionMap::map() const {
995     constexpr std::size_t kMax = std::numeric_limits<std::size_t>::max();
996     static const Map builtin_functions = {
    ......
1020         {"powerctl",                {1,     1,    do_powerctl}},
    ......
```
可知，powerctl对应的操作是do_powerctl，do_powerctl的具体实现为：  
```
695 static int do_powerctl(const std::vector<std::string>& args) {                                                                                                                                  
696     const char* command = args[1].c_str();
697     int len = 0;
698     unsigned int cmd = 0;
699     const char *reboot_target = "";
700     void (*callback_on_ro_remount)(const struct mntent*) = NULL;
701
702     if (strncmp(command, "shutdown", 8) == 0) { //关机
703         cmd = ANDROID_RB_POWEROFF;
704         len = 8;
705     } else if (strncmp(command, "reboot", 6) == 0) { //重启
706         cmd = ANDROID_RB_RESTART2;
707         len = 6;
708     } else {
709         ERROR("powerctl: unrecognized command '%s'\n", command);
710         return -EINVAL;
711     }
712
713     if (command[len] == ',') {
714         if (cmd == ANDROID_RB_POWEROFF &&
715             !strcmp(&command[len + 1], "userrequested")) {
716             // The shutdown reason is PowerManager.SHUTDOWN_USER_REQUESTED.
717             // Run fsck once the file system is remounted in read-only mode.
718             callback_on_ro_remount = unmount_and_fsck;
719         } else if (cmd == ANDROID_RB_RESTART2) {
720             reboot_target = &command[len + 1];
721         }
722     } else if (command[len] != '\0') {
723         ERROR("powerctl: unrecognized reboot target '%s'\n", &command[len]);
724         return -EINVAL;
725     }
726
727     std::string timeout = property_get("ro.build.shutdown_timeout");
728     unsigned int delay = 0;
729
730     if (android::base::ParseUint(timeout.c_str(), &delay) && delay > 0) {
731         Timer t;
732         // Ask all services to terminate.
733         ServiceManager::GetInstance().ForEachService(
734             [] (Service* s) { s->Terminate(); });
735
736         while (t.duration() < delay) {
737             ServiceManager::GetInstance().ReapAnyOutstandingChildren();
738         
739             int service_count = 0;
740             ServiceManager::GetInstance().ForEachService(
741                 [&service_count] (Service* s) {
742                     // Count the number of services running.
743                     // Exclude the console as it will ignore the SIGTERM signal
744                     // and not exit.
745                     // Note: SVC_CONSOLE actually means "requires console" but
746                     // it is only used by the shell.
747                     if (s->pid() != 0 && (s->flags() & SVC_CONSOLE) == 0) {
748                         service_count++;
749                     }
750                 });
751                     
752             if (service_count == 0) {
753                 // All terminable services terminated. We can exit early.
754                 break;  
755             }       
756         
757             // Wait a bit before recounting the number or running services.
758             usleep(kTerminateServiceDelayMicroSeconds);
759         }
760         NOTICE("Terminating running services took %.02f seconds", t.duration());
761     }       
762         
763     return android_reboot_with_callback(cmd, 0, reboot_target,
764                                         callback_on_ro_remount);
765 }
```
可以看到，最后调用的是android_reboot_with_callback，下面看它的实现system/core/libcutils/android_reboot.c  
```
212 int android_reboot_with_callback(
213     int cmd, int flags __unused, const char *arg,
214     void (*cb_on_remount)(const struct mntent*))
215 {
216     int ret;
217     remount_ro(cb_on_remount);
218     switch (cmd) {
219         case ANDROID_RB_RESTART:
220             ret = reboot(RB_AUTOBOOT);
221             break;
222
223         case ANDROID_RB_POWEROFF:
224             ret = reboot(RB_POWER_OFF);
225             break;
226
227         case ANDROID_RB_RESTART2:
228             ret = syscall(__NR_reboot, LINUX_REBOOT_MAGIC1, LINUX_REBOOT_MAGIC2,
229                            LINUX_REBOOT_CMD_RESTART2, arg);
230             break;
231
232         default:
233             ret = -1;
234     }
235
236     return ret;
237 }
```
关机的时候cmd应该是ANDROID_RB_POWEROFF，因此执行reboot函数，reboot函数的实现在bionic/libc/bionic/reboot.cpp  
```
34 int reboot(int mode) {
35   return __reboot(LINUX_REBOOT_MAGIC1, LINUX_REBOOT_MAGIC2, mode, NULL);
36 }
```
`__reboot`函数的实现在`bionic/libc/arch-arm64/syscalls/__reboot.S`  
```
5 ENTRY(__reboot)
6     mov     x8, __NR_reboot
7     svc     #0
8
9     cmn     x0, #(MAX_ERRNO + 1)
10     cneg    x0, x0, hi
11     b.hi    __set_errno_internal
12
13     ret
14 END(__reboot)
15 .hidden __reboot
```
这里，`__NR_reboot`就是内核的入口，它的定义在`bionic/libc/kernel/uapi/asm-generic/unistd.h`  
```
#define __NR_reboot 142
```
这里，刚好对应内核的`kernel/include/uapi/asm-generic/unistd.h`
```
423 #define __NR_reboot 142
424 __SYSCALL(__NR_reboot, sys_reboot)
```
可以看到`__NR_reboot`映射到`sys_reboot`，它的定义在`kernel/include/linux/syscalls.h`  
```
315 asmlinkage long sys_reboot(int magic1, int magic2, unsigned int cmd,
316                 void __user *arg);
```
它的原型在`kernel/kernel/reboot.c`中的`SYSCALL_DEFINE4`：  
```
280 SYSCALL_DEFINE4(reboot, int, magic1, int, magic2, unsigned int, cmd,
281         void __user *, arg)
282 {   
283     struct pid_namespace *pid_ns = task_active_pid_ns(current);
284     char buffer[256];
285     int ret = 0;
286     
287     /* We only trust the superuser with rebooting the system. */
288     if (!ns_capable(pid_ns->user_ns, CAP_SYS_BOOT))
289         return -EPERM;
290             
291     /* For safety, we require "magic" arguments. */
292     if (magic1 != LINUX_REBOOT_MAGIC1 ||
293             (magic2 != LINUX_REBOOT_MAGIC2 &&
294             magic2 != LINUX_REBOOT_MAGIC2A &&
295             magic2 != LINUX_REBOOT_MAGIC2B &&
296             magic2 != LINUX_REBOOT_MAGIC2C))
297         return -EINVAL;
298     
299     /*
300      * If pid namespaces are enabled and the current task is in a child
301      * pid_namespace, the command is handled by reboot_pid_ns() which will
302      * call do_exit().
303      */
304     ret = reboot_pid_ns(pid_ns, cmd);
305     if (ret)
306         return ret;
307         
308     /* Instead of trying to make the power_off code look like
309      * halt when pm_power_off is not set do it the easy way.
310      */
311     if ((cmd == LINUX_REBOOT_CMD_POWER_OFF) && !pm_power_off)
312         cmd = LINUX_REBOOT_CMD_HALT;
313
314     mutex_lock(&reboot_mutex);
315     switch (cmd) {
316     case LINUX_REBOOT_CMD_RESTART:
317         kernel_restart(NULL);
318         break;
319         
320     case LINUX_REBOOT_CMD_CAD_ON:
321         C_A_D = 1;
322         break;
323         
324     case LINUX_REBOOT_CMD_CAD_OFF:
325         C_A_D = 0;
326         break;
327     
328     case LINUX_REBOOT_CMD_HALT:
329         kernel_halt();
330         do_exit(0);
331         panic("cannot halt");
332     
333     case LINUX_REBOOT_CMD_POWER_OFF:
334         kernel_power_off();
335         do_exit(0);
336         break;
337         
338     case LINUX_REBOOT_CMD_RESTART2:
339         ret = strncpy_from_user(&buffer[0], arg, sizeof(buffer) - 1);
340         if (ret < 0) {
341             ret = -EFAULT;
342             break;
343         }
344         buffer[sizeof(buffer) - 1] = '\0';
345         
346         kernel_restart(buffer);
347         break;
348         
349 #ifdef CONFIG_KEXEC_CORE
350     case LINUX_REBOOT_CMD_KEXEC:
351         ret = kernel_kexec();
352         break;
353 #endif
354     
355 #ifdef CONFIG_HIBERNATION
356     case LINUX_REBOOT_CMD_SW_SUSPEND:
357         ret = hibernate();
358         break;
359 #endif
360
361     default:
362         ret = -EINVAL;
363         break;
364     }   
365     mutex_unlock(&reboot_mutex);
366     return ret;
367 }
```
至于它是怎么从`sys_reboot`映射到`SYSCALL_DEFINE4`，这主要是通过宏定义实现的呢？  
看`kernel/include/linux/syscalls.h`  
```
182 #define SYSCALL_DEFINE1(name, ...) SYSCALL_DEFINEx(1, _##name, __VA_ARGS__)
183 #define SYSCALL_DEFINE2(name, ...) SYSCALL_DEFINEx(2, _##name, __VA_ARGS__)
184 #define SYSCALL_DEFINE3(name, ...) SYSCALL_DEFINEx(3, _##name, __VA_ARGS__)
185 #define SYSCALL_DEFINE4(name, ...) SYSCALL_DEFINEx(4, _##name, __VA_ARGS__)
186 #define SYSCALL_DEFINE5(name, ...) SYSCALL_DEFINEx(5, _##name, __VA_ARGS__)
187 #define SYSCALL_DEFINE6(name, ...) SYSCALL_DEFINEx(6, _##name, __VA_ARGS__)
188
189 #define SYSCALL_DEFINEx(x, sname, ...)              \
190     SYSCALL_METADATA(sname, x, __VA_ARGS__)         \
191     __SYSCALL_DEFINEx(x, sname, __VA_ARGS__)
192
193 #define __PROTECT(...) asmlinkage_protect(__VA_ARGS__)
194 #define __SYSCALL_DEFINEx(x, name, ...)                 \
195     asmlinkage long sys##name(__MAP(x,__SC_DECL,__VA_ARGS__))   \
196         __attribute__((alias(__stringify(SyS##name))));     \
197     static inline long SYSC##name(__MAP(x,__SC_DECL,__VA_ARGS__));  \
198     asmlinkage long SyS##name(__MAP(x,__SC_LONG,__VA_ARGS__));  \
199     asmlinkage long SyS##name(__MAP(x,__SC_LONG,__VA_ARGS__))   \
200     {                               \
201         long ret = SYSC##name(__MAP(x,__SC_CAST,__VA_ARGS__));  \
202         __MAP(x,__SC_TEST,__VA_ARGS__);             \
203         __PROTECT(x, ret,__MAP(x,__SC_ARGS,__VA_ARGS__));   \
204         return ret;                     \
205     }                               \
206     static inline long SYSC##name(__MAP(x,__SC_DECL,__VA_ARGS__))
```
因此：  
`SYSCALL_DEFINE4(reboot, int, magic1, int, magic2, unsigned int, cmd, void __user *, arg)`等同于：  
`asmlinkage long sys_reboot(int magic1, int magic2, unsigned int cmd, void __user *arg);`  
这里有很多case，我们只关注关机对应的：  
```
333     case LINUX_REBOOT_CMD_POWER_OFF:
334         kernel_power_off();
335         do_exit(0);
336         break;
```
kernel_power_off的原型：  
```
257 void kernel_power_off(void)
258 {
259     kernel_shutdown_prepare(SYSTEM_POWER_OFF);
260     if (pm_power_off_prepare)
261         pm_power_off_prepare();
262     migrate_to_reboot_cpu();
263     syscore_shutdown();
264     pr_emerg("Power down\n");
265     kmsg_dump(KMSG_DUMP_POWEROFF);
266     machine_power_off();
267 }
268 EXPORT_SYMBOL_GPL(kernel_power_off);
```

参考：
http://m.blog.csdn.net/article/details?id=51354632  
http://blog.csdn.net/zsj100213/article/details/49121161
