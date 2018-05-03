---
layout: post
title: RunLoop 源码学习笔记
date: 2018-04-14 15:54:22 +0800
comments: true
categories: 
---

RunLoop 是个老生常谈的话题了，一直以来对 RunLoop 的认识还停留在 [深入理解RunLoop](https://blog.ibireme.com/2015/05/18/runloop/)（ YY 大神） 等业内好文当中，对于自个来说仍有些知识盲区：

* 网上有文章用伪代码提到 `CheckIfExistMessagesInMainDispatchQueue();` ，代码看似提供了一次执行 main queue 的机会，为什么有这样的设计？
* 怎么理解在 kCFRunLoopBeforeSources 和 kCFRunLoopAfterWaiting 之间来检测是否卡顿？
* 我们能在代码中看到 `USE_DISPATCH_SOURCE_FOR_TIMERS` 这样的宏，到底 GCD 的 timer 与 RunLoop 有关吗？
* 能让 RunLoop 退出的几种方式

<!--more-->

为了搞懂自己没理解清楚的问题，我下载了苹果开源的 CoreFoudation 来一窥究竟（ [https://github.com/apple/swift-corelibs-foundation/](https://github.com/apple/swift-corelibs-foundation/)）。

### 主函数 __CFRunLoopRun 

RunLoop 的事件循环机制主要由 `__CFRunLoopRun` 函数实现，其代码如下：

```c
static int32_t __CFRunLoopRun(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFTimeInterval seconds, Boolean stopAfterHandle, CFRunLoopModeRef previousMode) {
  ...
  
#if __HAS_DISPATCH__
    // 获取主线程消息端口，用于 main queue 事件的派发
    __CFPort dispatchPort = CFPORT_NULL;
    Boolean libdispatchQSafe = pthread_main_np() && ((HANDLE_DISPATCH_ON_BASE_INVOCATION_ONLY && NULL == previousMode) || (!HANDLE_DISPATCH_ON_BASE_INVOCATION_ONLY && 0 == _CFGetTSD(__CFTSDKeyIsInGCDMainQ)));
    if (libdispatchQSafe && (CFRunLoopGetMain() == rl) && CFSetContainsValue(rl->_commonModes, rlm->_name)) 
      // 还需检查当前 RunLoopMode 是否在 _commonModes 的set集合中
      dispatchPort = _dispatch_get_main_queue_port_4CF();
#endif

#if USE_DISPATCH_SOURCE_FOR_TIMERS
    // 初始 GCD timer
    mach_port_name_t modeQueuePort = MACH_PORT_NULL;
    if (rlm->_queue) {
        modeQueuePort = _dispatch_runloop_root_queue_get_port_4CF(rlm->_queue);
        if (!modeQueuePort) {
            CRASH("Unable to get port for run loop mode queue (%d)", -1);
        }
    }
#endif

// 设置超时，即 - runUntilDate: 设置的参数，- run 方法的超时时间是无限大
#if __HAS_DISPATCH__
    dispatch_source_t timeout_timer = NULL;
#endif
    struct __timeout_context *timeout_context = (struct __timeout_context *)malloc(sizeof(*timeout_context));
    if (seconds <= 0.0) { // instant timeout
        seconds = 0.0;
        timeout_context->termTSR = 0ULL;
    } else if (seconds <= TIMER_INTERVAL_LIMIT) {
#if __HAS_DISPATCH__
	dispatch_queue_t queue = pthread_main_np() ? __CFDispatchQueueGetGenericMatchingMain() : __CFDispatchQueueGetGenericBackground();
	timeout_timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
        dispatch_retain(timeout_timer);
	timeout_context->ds = timeout_timer;
#endif
	timeout_context->rl = (CFRunLoopRef)CFRetain(rl);
	timeout_context->termTSR = startTSR + __CFTimeIntervalToTSR(seconds);
#if __HAS_DISPATCH__
	dispatch_set_context(timeout_timer, timeout_context); // source gets ownership of context
	dispatch_source_set_event_handler_f(timeout_timer, __CFRunLoopTimeout);
        dispatch_source_set_cancel_handler_f(timeout_timer, __CFRunLoopTimeoutCancel);
        uint64_t ns_at = (uint64_t)((__CFTSRToTimeInterval(startTSR) + seconds) * 1000000000ULL);
        dispatch_source_set_timer(timeout_timer, dispatch_time(1, ns_at), DISPATCH_TIME_FOREVER, 1000ULL);
        dispatch_resume(timeout_timer);
#endif
    } else { // infinite timeout
        seconds = 9999999999.0;
        timeout_context->termTSR = UINT64_MAX;
    }
    
    ...
```

再往下就是 RunLoop 的主体 do-while 代码：

```c
do {
      voucher_mach_msg_state_t voucherState = VOUCHER_MACH_MSG_STATE_UNCHANGED;
      voucher_t voucherCopy = NULL;
      uint8_t msg_buffer[3 * 1024];
      
      mach_msg_header_t *msg = NULL;
      mach_port_t livePort = MACH_PORT_NULL;
      
      __CFPortSet waitSet = rlm->_portSet;

      __CFRunLoopUnsetIgnoreWakeUps(rl);

      // observer 收到 beforeTimers 的通知，RunLoop 即将处理 timer
      if (rlm->_observerMask & kCFRunLoopBeforeTimers) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeTimers);
      // observer 收到 beforeSources 的通知，RunLoop 即将处理 source0 事件
      if (rlm->_observerMask & kCFRunLoopBeforeSources) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeSources);

      // RunLoop 执行 block
	    __CFRunLoopDoBlocks(rl, rlm);

      // RunLoop 处理 source0 事件
      Boolean sourceHandledThisLoop = __CFRunLoopDoSources0(rl, rlm, stopAfterHandle);
      if (sourceHandledThisLoop) {
          // RunLoop 执行 block
          __CFRunLoopDoBlocks(rl, rlm);
	   }
	   
      ...
```

source0 事件可以是 UIEvent，如用户点击按钮

![source0-image](https://user-images.githubusercontent.com/5633917/38467502-048498ac-3b6c-11e8-90fe-e0b620300f65.png)

__CFRunLoopDoBlocks 可以是执行 block 或调用 performSelector 方法。

### CheckIfExistMessagesInMainDispatchQueue() 实现分析

```c
#if __HAS_DISPATCH__
        // 使用 didDispatchPortLastTime 变量配合检查有没有 main queue 需要执行的
        if (MACH_PORT_NULL != dispatchPort && !didDispatchPortLastTime) { 
            msg = (mach_msg_header_t *)msg_buffer;
            if (__CFRunLoopServiceMachPort(dispatchPort, &msg, sizeof(msg_buffer), &livePort, 0, &voucherState, NULL)) {
                goto handle_msg; // 执行 main queue 并将 didDispatchPortLastTime 设为 true
            }
        }
#endif

        didDispatchPortLastTime = false;
        
        ...
```

上面代码是 main queue 可能被本次 RunLoop 执行的一个机会，可以看到 if 语句里还加入了 didDispatchPortLastTime 这个变量，该变量作用很像是获取上次 RunLoop 有没有执行过 main queue 的标志，假如 `handle_msg` 执行了 main queue， didDispatchPortLastTime 会被设为 true，这样在下次 RunLoop ，!didDispatchPortLastTime 为 false，不会直接跳转执行 `handle_msg`。

但假如 `handle_msg` 执行了其他的分支（比如 timer），那么本次 RunLoop 将不再执行 main queue 了（即便有），来到下次 RunLoop 时，由于!didDispatchPortLastTime 为 true，如果有 main queue 的代码要执行，就会直接跳转到 `handle_msg` 处理 main queue，略过 RunLoop 休眠等代码。

为此我通过简单的代码配合 CoreFoundation 源码来验证。

![161523259381_ pic](https://user-images.githubusercontent.com/5633917/38560281-de4cdd46-3d07-11e8-841f-4187b3674130.jpg)

我在 timer 的回调里添加了执行 main queue 的方法，这么做的原因是想模拟下本次 RunLoop 没有执行 main queue 的情况。运行代码，timer 的回调执行完之后就进入到下一个 RunLoop 了，接着和预期的一致，来到了 `goto handle_msg;` 直接跳转去执行 main queue。

![151523259187_ pic](https://user-images.githubusercontent.com/5633917/38560508-60a2939e-3d08-11e8-85a9-8b33deca79e1.jpg)

对于第一个疑惑，可以看出系统之所以这么做是为了确保加进来的 main queue 能获得快速执行和跳过界面更新和休眠提升效率。

### 利用 RunLoop 实现卡顿检测的原因

接下来 RunLoop 准备休眠

```c
// 通知 RunLoop 的线程即将进入休眠
if (!poll && (rlm->_observerMask & kCFRunLoopBeforeWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeWaiting);
	__CFRunLoopSetSleeping(rl);
	// do not do any user callouts after this point (after notifying of sleeping)

        // Must push the local-to-this-activation ports in on every loop
        // iteration, as this mode could be run re-entrantly and we don't
        // want these ports to get serviced.
#if __HAS_DISPATCH__
        __CFPortSetInsert(dispatchPort, waitSet);
#endif
        
	__CFRunLoopModeUnlock(rlm);
	__CFRunLoopUnlock(rl);
```

`__CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeWaiting);` 这一步实际上系统还会进行界面更新的操作，为验证我子类化并复写了 UIView 的 -drawRect: 

首先添加 `__CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__` 的符号断点

![171523352314_ pic](https://user-images.githubusercontent.com/5633917/38566863-57cb35be-3d17-11e8-8742-301c5221cb81.jpg)

直至找到 callout 为 `_ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv()` 的 observer

![191523352370_ pic](https://user-images.githubusercontent.com/5633917/38567061-d10aded4-3d17-11e8-8ce2-132187878bbc.jpg)

```c
static void __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(CFRunLoopObserverCallBack func, CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info) {
    if (func) {
        func(observer, activity, info);
    }
    asm __volatile__(""); // thwart tail-call optimization
}
```

结合源码由上图可知：
rdi 表示第一个参数，即 func 为 QuartzCore:CA::Transaction::observer_callback
rdx 表示第三个参数，十进制值为 32，十六进制表示是 0x0000000000000020，二进制简单对应为 0b100000，而这正好是 `kCFRunLoopBeforeWaiting = (1UL << 5),` 的值，所以得出结论 activity 为 kCFRunLoopBeforeWaiting。

再往下看堆栈，最终 UIView 将会调用 -drawRect: 方法

![211523354018_ pic](https://user-images.githubusercontent.com/5633917/38567632-53054626-3d19-11e8-8670-f5f9bd38800e.jpg)

从 kCFRunLoopBeforeSources 为起点到 kCFRunLoopBeforeWaiting 休眠前，这其中处理了大量的工作————执行 block，处理 source0，更新界面...做完这些之后 RunLoop 就休眠了，直到 RunLoop 被 timer、source、libdispatch 唤醒，唤醒后会发送休眠结束的 kCFRunLoopAfterWaiting 通知。我们知道屏幕的刷新率是 60fps，即 1/60s ≈ 16ms，假如一次 RunLoop 超过了这个时间，UI 线程有可能出现了卡顿，BeforeSources 到 AfterWaiting 可以粗略认为是一次 RunLoop 的起止。至于其他状态，譬如 BeforeWaiting，它在更新完界面之后有可能休眠了，此时 APP 已是不活跃的状态，不太可能造成卡顿；而 kCFRunLoopExit，它在 RunLoop 退出之后触发，主线程的 RunLoop 除了换 mode 又不太可能主动退出，这也不能用作卡顿检测。

### RunLoop 开始休眠

```c
/** 
  这个 do-while 并不是死循环，__CFRunLoopServiceMachPort 是实际上使线程休眠的函数
  在 __CFRunLoopServiceMachPort 函数内部调用 mach_msg(...) 实现休眠
  USE_DISPATCH_SOURCE_FOR_TIMERS 在 iOS 系统上为 0
*/
#if USE_DISPATCH_SOURCE_FOR_TIMERS
        do {
            msg = (mach_msg_header_t *)msg_buffer;
            
            __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort, poll ? 0 : TIMEOUT_INFINITY, &voucherState, &voucherCopy);
            
            if (modeQueuePort != MACH_PORT_NULL && livePort == modeQueuePort) {
                // Drain the internal queue. If one of the callout blocks sets the timerFired flag, break out and service the timer.
                while (_dispatch_runloop_root_queue_perform_4CF(rlm->_queue));
                if (rlm->_timerFired) {
                    // Leave livePort as the queue port, and service timers below
                    rlm->_timerFired = false;
                    break;
                } else {
                    if (msg && msg != (mach_msg_header_t *)msg_buffer) free(msg);
                }
            } else {
                // Go ahead and leave the inner loop.
                break;
            }
        } while (1);
#else
        msg = (mach_msg_header_t *)msg_buffer;
        __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort, poll ? 0 : TIMEOUT_INFINITY, &voucherState, &voucherCopy);
#endif
```

线程休眠之后就不干活了，直到被 timer、source、libdispatch 唤醒。

```c
// 在唤醒之后发送 kCFRunLoopAfterWaiting 通知
	__CFRunLoopUnsetSleeping(rl);
	if (!poll && (rlm->_observerMask & kCFRunLoopAfterWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopAfterWaiting);

```

### `handle_msg` 事件源处理及 timer 分析

```c
handle_msg:;
        __CFRunLoopSetIgnoreWakeUps(rl);
        
        if (MACH_PORT_NULL == livePort) {
            CFRUNLOOP_WAKEUP_FOR_NOTHING();
            // handle nothing
        } else if (livePort == rl->_wakeUpPort) {
            CFRUNLOOP_WAKEUP_FOR_WAKEUP();
            // do nothing on Mac OS
        }
#if USE_DISPATCH_SOURCE_FOR_TIMERS
        else if (modeQueuePort != MACH_PORT_NULL && livePort == modeQueuePort) {
            CFRUNLOOP_WAKEUP_FOR_TIMER();
            if (!__CFRunLoopDoTimers(rl, rlm, mach_absolute_time())) {
                // Re-arm the next timer, because we apparently fired early
                __CFArmNextTimerInMode(rlm, rl);
            }
        }
#endif
#if USE_MK_TIMER_TOO
        else if (rlm->_timerPort != MACH_PORT_NULL && livePort == rlm->_timerPort) {
        // 处理 NSTimer 或 CFRunLoopTimer 的回调
            CFRUNLOOP_WAKEUP_FOR_TIMER();
            
            if (!__CFRunLoopDoTimers(rl, rlm, mach_absolute_time())) {
                // Re-arm the next timer
                __CFArmNextTimerInMode(rlm, rl);
            }
        }
#endif
#if __HAS_DISPATCH__
        else if (livePort == dispatchPort) {
        // 执行 main queue 的回调
            CFRUNLOOP_WAKEUP_FOR_DISPATCH();
            __CFRunLoopModeUnlock(rlm);
            __CFRunLoopUnlock(rl);
            _CFSetTSD(__CFTSDKeyIsInGCDMainQ, (void *)6, NULL);

            __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);
            _CFSetTSD(__CFTSDKeyIsInGCDMainQ, (void *)0, NULL);
            __CFRunLoopLock(rl);
            __CFRunLoopModeLock(rlm);
            sourceHandledThisLoop = true;
            didDispatchPortLastTime = true;
        }
#endif
        else {
        // 处理 source1 事件
            CFRUNLOOP_WAKEUP_FOR_SOURCE();
            // Despite the name, this works for windows handles as well
            CFRunLoopSourceRef rls = __CFRunLoopModeFindSourceForMachPort(rl, rlm, livePort);
            if (rls) {
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
		mach_msg_header_t *reply = NULL;
		sourceHandledThisLoop = __CFRunLoopDoSource1(rl, rlm, rls, msg, msg->msgh_size, &reply) || sourceHandledThisLoop;
		if (NULL != reply) {
		    (void)mach_msg(reply, MACH_SEND_MSG, reply->msgh_size, 0, MACH_PORT_NULL, 0, MACH_PORT_NULL);
		    CFAllocatorDeallocate(kCFAllocatorSystemDefault, reply);
		}
#endif
	    }
            
        } 
```

代码还是比较直观，一次 RunLoop 仅能执行一类事件源，我们注意到有两个处理 timer 的地方，其中第一处是

```c
#if USE_DISPATCH_SOURCE_FOR_TIMERS
        else if (modeQueuePort != MACH_PORT_NULL && livePort == modeQueuePort) {
            CFRUNLOOP_WAKEUP_FOR_TIMER();
            if (!__CFRunLoopDoTimers(rl, rlm, mach_absolute_time())) {
                // Re-arm the next timer, because we apparently fired early
                __CFArmNextTimerInMode(rlm, rl);
            }
        }
#endif
```

乍看这很像是 `dispatch_source_t` 的 timer 回调，而我在实际的调试中当创建
`dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, dispatch_get_main_queue());` 
触发的 timer 回调并不是从这里面回调出来的，而是来自 main queue 的 block。如果使用创建队列的方式生成 timer，而回调又来自 GCD 的内部。上面已经说过 `USE_DISPATCH_SOURCE_FOR_TIMERS` 在 iOS 环境下是 0，那么如果 GCD 的 timer 依赖 RunLoop，我们怎又能在 iOS 上使用 GCD 的 tiemr。

大胆的猜测：GCD 的 tiemr 不依赖 RunLoop，GCD 实现了一套与 mk_timer 不同的机制。RunLoop 在这可能只是想表达下也能通过 GCD 实现 timer。

线程休眠后，当 NSTimer 或 CFRunLoopTimer 的时间到了，则由内核的 mk_timer 驱动

![3c3e0e4f-4243-4b6b-bd7b-56c61625ee1f-2208-0000023d12c0dd01_tmp](https://user-images.githubusercontent.com/5633917/38627763-5334e362-3de2-11e8-9f05-1ea6d1c0cd95.JPG)

追溯堆栈

![4e5d88d2-3995-48dc-a02a-a4e2d18c77ae-2208-0000023d0a7abe8c_tmp](https://user-images.githubusercontent.com/5633917/38628196-609918d8-3de3-11e8-88bd-a144360a507f.JPG)

### RunLoop 退出的几种方式

```c
// 再执行一下 block 或 performSelector 的代码
__CFRunLoopDoBlocks(rl, rlm);
        
  /** 
    进入以下任一 case 后就会退出 RunLoop 
    结束就会调用 if (currentMode->_observerMask & kCFRunLoopExit ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);
  */
	if (sourceHandledThisLoop && stopAfterHandle) {
	    retVal = kCFRunLoopRunHandledSource;
  } else if (timeout_context->termTSR < mach_absolute_time()) {
      // 自身超时时间到了
      retVal = kCFRunLoopRunTimedOut;
	} else if (__CFRunLoopIsStopped(rl)) {
	    // 被外部调用 CFRunLoopStop 停止了
       __CFRunLoopUnsetStopped(rl);
	    retVal = kCFRunLoopRunStopped;
	} else if (rlm->_stopped) {
	    // 被 _CFRunLoopStopMode 停止
	    rlm->_stopped = false;
	    retVal = kCFRunLoopRunStopped;
	} else if (__CFRunLoopModeIsEmpty(rl, rlm, previousMode)) { // 检查上一个 mode 有没有执行完所有事件源
	    retVal = kCFRunLoopRunFinished;
	}

    } while (0 == retVal);
```

解释下 `_CFRunLoopStopMode` 函数，该函数作用是通过停止当前 RunLoop 执行的某个 mode，对于当前 RunLoop 不包含的 mode，调用这个函数并不会产生效果，查找有没有包含 mode 的大致代码如下：

```c
struct __CFRunLoopMode srlm;
memset(&srlm, 0, sizeof(srlm));
_CFRuntimeSetInstanceTypeIDAndIsa(&srlm, __kCFRunLoopModeTypeID);
srlm._name = modeName;
// 寻找当前 modes 中有没有包含指定 modeName
rlm = (CFRunLoopModeRef)CFSetGetValue(rl->_modes, &srlm);
if (NULL != rlm) {
	__CFRunLoopModeLock(rlm);
	return rlm;
}
```

我们知道当 RunLoop 切换 mode 时，RunLoop 将会退出，并重新指定 mode 再进入，我猜测系统内部通过调用了 `_CFRunLoopStopMode` 来达到切换 mode 的效果。

## 总结
通过此次对 RunLoop 的学习，我对 RunLoop 的运行机制终于有了一些概念。在学习过程中也抛出了一些新的疑惑————譬如对系统怎样换 mode 感到好奇，希望以后能想办法搞懂。

参考链接

[深入理解RunLoop](https://blog.ibireme.com/2015/05/18/runloop/)

[iOS线下分享《RunLoop》by 孙源@sunnyxx](http://v.youku.com/v_show/id_XODgxODkzODI0.html)

[iOS 事件处理机制与图像渲染过程](http://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=400417748&idx=1&sn=0c5f6747dd192c5a0eea32bb4650c160&scene=0)

[检测iOS的APP性能的一些方法](https://github.com/ming1016/study/wiki/%E6%A3%80%E6%B5%8BiOS%E7%9A%84APP%E6%80%A7%E8%83%BD%E7%9A%84%E4%B8%80%E4%BA%9B%E6%96%B9%E6%B3%95)

