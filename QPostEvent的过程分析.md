##Qt中的postEvent过程解析

用法 app->postEvent(QObject *, QEvent*);

####1.传递给QCoreApplication的postEvent(QObject *, QEvent*, int NormalEventPriority)；
<details>
<summary>postEvent()</summary>
```
void postEvent(QObject *receiver, QEvent*event, int priority)
{
   
   if (receiver == 0) {
        qWarning("QCoreApplication::postEvent: Unexpected null receiver");
        delete event;
        return;
    }

//判断object的的threadData是否为空
    QThreadData * volatile * pdata = &receiver->d_func()->threadData;
    QThreadData *data = *pdata;
    if (!data) {
        // posting during destruction? just delete the event to prevent a leak
        delete event;
        return;
    }

    // 加锁
    data->postEventList.mutex.lock();

    /如果object被移到了另外一个线程中，跟新threadData数据
    while (data != *pdata) {
        data->postEventList.mutex.unlock();

        data = *pdata;
        if (!data) {
            // posting during destruction? just delete the event to prevent a leak
            delete event;
            return;
        }

        data->postEventList.mutex.lock();
    }

    QMutexUnlocker locker(&data->postEventList.mutex);

    // 判断事件是不是压缩事件（DeferredDelete， quit）类型，如果是，把多个事件合并为一个
    if (receiver->d_func()->postedEvents
        && self && self->compressEvent(event, receiver, &data->postEventList)) {
        return;
    }

    if (event->type() == QEvent::DeferredDelete && data == QThreadData::current()) {
        //记录下DeferredDelete所在事件循环
        event->d = reinterpret_cast<QEventPrivate *>(quintptr(data->loopLevel));
    }

    //将事件添加到事件队列，通过QScopedPointer释放事件。
    QScopedPointer<QEvent> eventDeleter(event);
    data->postEventList.addEvent(QPostEvent(receiver, event, priority));
    eventDeleter.take();
    event->posted = true;
    ++receiver->d_func()->postedEvents;
    data->canWait = false;
    locker.unlock();

//启动事件调度
    if (data->eventDispatcher)
        data->eventDispatcher->wakeUp();
}
```
</details>
####2.一般到这边就可能看不到wakeUp()里面的东西了，因为发现wakeUp()是纯虚函数。这时候就要从QCoreApplication的初始化里面开始看了，发现里面有一段初始化分配器的代码
```
    Q_Q(QCoreApplication);
#if defined(Q_OS_SYMBIAN)
    eventDispatcher = new QEventDispatcherSymbian(q);
#elif defined(Q_OS_UNIX)
#  if defined(Q_OS_BLACKBERRY)
    eventDispatcher = new QEventDispatcherBlackberry(q);
#  else
#  if !defined(QT_NO_GLIB)
    if (qgetenv("QT_NO_GLIB").isEmpty() && QEventDispatcherGlib::versionSupported())
        eventDispatcher = new QEventDispatcherGlib(q);
    else
#  endif
        eventDispatcher = new QEventDispatcherUNIX(q);
#  endif
#elif defined(Q_OS_WIN)
    eventDispatcher = new QEventDispatcherWin32(q);
#else
#  error "QEventDispatcher not yet ported to this platform"
#endif
```

###Linux： QEventDispatcherGlib
linux下基本就会有GLib， 所以Linux一般会掉用QEventDispatcherGlib（）

```
void QEventDispatcherGlib::wakeUp()
{
    Q_D(QEventDispatcherGlib);
    d->postEventSource->serialNumber.ref();
    g_main_context_wakeup(d->mainContext);
}

```
直接把事件交给GLibC处理，处理了好之后，直接调用postEventSourceDispatch()处理这些事件
```
static gboolean postEventSourceDispatch(GSource *s, GSourceFunc, gpointer)
{
    GPostEventSource *source = reinterpret_cast<GPostEventSource *>(s);
    source->lastSerialNumber = source->serialNumber;
    QCoreApplication::sendPostedEvents();
    source->d->runTimersOnceWithNormalPriority();
    return true; // i dunno, george...
}
```

###没有Glib QEventDispatcherUNIX

```
void QEventDispatcherUNIX::wakeUp()
{
    Q_D(QEventDispatcherUNIX);
    if (d->wakeUps.testAndSetAcquire(0, 1)) {
        char c = 0;
        qt_safe_write( d->thread_pipe[1], &c, 1 );
    }
}

static inline qint64 qt_safe_write(int fd, void *data, qint64 maxlen)
{
    qint64 ret = 0;
    EINTR_LOOP(ret, QT_WRITE(fd, data, maxlen));
    return ret;
}

#define EINTR_LOOP(var, cmd)                    \
    do {                                        \
        var = cmd;                              \
    } while (var == -1 && errno == EINTR)
```
不理解最后这几段代码。真个post过程到此结束了，最后猜测用signal来回调函数




####windows： QEventDispatcherWin32
```
void QEventDispatcherWin32::wakeUp()
{
    Q_D(QEventDispatcherWin32);
    d->serialNumber.ref();
    if (d->internalHwnd && d->wakeUps.testAndSetAcquire(0, 1)) {
        // post a WM_QT_SENDPOSTEDEVENTS to this thread if there isn't one already pending
        PostMessage(d->internalHwnd, WM_QT_SENDPOSTEDEVENTS, 0, 0);
    }
}
```
通过PostMessage把消息加到windows队列里面去，查看类中会发现有两个回调函数
   ##### friend LRESULT QT_WIN_CALLBACK qt_internal_proc(HWND hwnd, UINT message, WPARAM wp, LPARAM lp);
   ##### friend LRESULT QT_WIN_CALLBACK qt_GetMessageHook(int, WPARAM, LPARAM);

我们挨个分析，首先qt_GetMessageHook（），函数名为获得windows的信息的钩子函数。我们的事件极大可能是通过这个函数回调
```
LRESULT QT_WIN_CALLBACK qt_GetMessageHook(int code, WPARAM wp, LPARAM lp)
{
//当事件被移除队列
    if (wp == PM_REMOVE) {
        QEventDispatcherWin32 *q = qobject_cast<QEventDispatcherWin32 *>(QAbstractEventDispatcher::instance());
        Q_ASSERT(q != 0);
        if (q) {
            MSG *msg = (MSG *) lp;
            QEventDispatcherWin32Private *d = q->d_func();
            int localSerialNumber = d->serialNumber;
            static const UINT mask = inputTimerMask();
            if (HIWORD(GetQueueStatus(mask)) == 0) {
                // no more input or timer events in the message queue, we can allow posted events to be sent normally now
                if (d->sendPostedEventsWindowsTimerId != 0) {
                    // stop the timer to send posted events, since we now allow the WM_QT_SENDPOSTEDEVENTS message
                    KillTimer(d->internalHwnd, d->sendPostedEventsWindowsTimerId);
                    d->sendPostedEventsWindowsTimerId = 0;
                }
                (void) d->wakeUps.fetchAndStoreRelease(0);
                if (localSerialNumber != d->lastSerialNumber
                    // if this message IS the one that triggers sendPostedEvents(), no need to post it again
                    && (msg->hwnd != d->internalHwnd
                        || msg->message != WM_QT_SENDPOSTEDEVENTS)) {
                    PostMessage(d->internalHwnd, WM_QT_SENDPOSTEDEVENTS, 0, 0);
                }
            } else if (d->sendPostedEventsWindowsTimerId == 0
                       && localSerialNumber != d->lastSerialNumber) {
                // start a special timer to continue delivering posted events while
                // there are still input and timer messages in the message queue
                d->sendPostedEventsWindowsTimerId = SetTimer(d->internalHwnd,
                                                             SendPostedEventsWindowsTimerId,
                                                             0, // we specify zero, but Windows uses USER_TIMER_MINIMUM
                                                             NULL);
                // we don't check the return value of SetTimer()... if creating the timer failed, there's little
                // we can do. we just have to accept that posted events will be starved
            }
        }
    }
#ifdef Q_OS_WINCE
    Q_UNUSED(code);
    return 0;
#else
    return CallNextHookEx(0, code, wp, lp);
#endif
}
```

下面是qt_internal_proc(), Qt内部事件处理函数，这个主要windows内部回调，分发消息，处理非窗口消息
```
LRESULT QT_WIN_CALLBACK qt_internal_proc(HWND hwnd, UINT message, WPARAM wp, LPARAM lp)
{
    if (message == WM_NCCREATE)
        return true;

    MSG msg;
    msg.hwnd = hwnd;
    msg.message = message;
    msg.wParam = wp;
    msg.lParam = lp;
    QCoreApplication *app = QCoreApplication::instance();
    long result;
    if (!app) {
        if (message == WM_TIMER)
            KillTimer(hwnd, wp);
        return 0;
    } else if (app->filterEvent(&msg, &result)) {
        return result;
    }

#ifdef GWLP_USERDATA
    QEventDispatcherWin32 *q = (QEventDispatcherWin32 *) GetWindowLongPtr(hwnd, GWLP_USERDATA);
#else
    QEventDispatcherWin32 *q = (QEventDispatcherWin32 *) GetWindowLong(hwnd, GWL_USERDATA);
#endif
    QEventDispatcherWin32Private *d = 0;
    if (q != 0)
        d = q->d_func();
    //下面 WM_QT_SOCKETNOTIFIER socket Qt 事件底层的处理机制、处理网络的消息事件
    if (message == WM_QT_SOCKETNOTIFIER) {
        // socket notifier message
        int type = -1;
        switch (WSAGETSELECTEVENT(lp)) {//在非阻塞模式下利用socket事件的消息机制，Server端与Client端之间的通信处于异步状态下
        case FD_READ:             //socket 文件描述符 read 、有数据到达时发生
        case FD_CLOSE:            //socket 文件描述符关闭断开连接 、套接口关闭时发生
        case FD_ACCEPT:          //socket 文件描述符 接收连接 、 作为客户端连接成功时发生
            type = 0;
            break;
        case FD_WRITE:   //socket 文件描述符写 、有数据发送时产生
        case FD_CONNECT:
            type = 1;
            break;
        case FD_OOB:   //socket 文件描述符收到数据 、 收到外带数据时发生
            type = 2;
            break;
        }
        if (type >= 0) {
            Q_ASSERT(d != 0);
            QSNDict *sn_vec[3] = { &d->sn_read, &d->sn_write, &d->sn_except };
            QSNDict *dict = sn_vec[type];

            QSockNot *sn = dict ? dict->value(wp) : 0;
            if (sn) {
                QEvent event(QEvent::SockAct);
                QCoreApplication::sendEvent(sn->obj, &event);
            }
        }
        return 0;
    } else if (message == WM_QT_SENDPOSTEDEVENTS //  处理 Qt sendPostEvent()发送事件，这个包含了大部分Qt的事件，特殊情况除外
               // we also use a Windows timer to send posted events when the message queue is full
               || (message == WM_TIMER
                   && d->sendPostedEventsWindowsTimerId != 0
                   && wp == (uint)d->sendPostedEventsWindowsTimerId)) {
        int localSerialNumber = d->serialNumber;
        if (localSerialNumber != d->lastSerialNumber) {
            d->lastSerialNumber = localSerialNumber;
            QCoreApplicationPrivate::sendPostedEvents(0, 0, d->threadData); //sendEvent（同步调用）
        }
        return 0;
    } else if (message == WM_TIMER) { //定时器超时
        Q_ASSERT(d != 0);
        d->sendTimerEvent(wp);
        return 0;
    }

    return DefWindowProc(hwnd, message, wp, lp); //继续让windows处理分发其他消息
```


windows底层处理好消息之后通过回调QtWndProc，这里主要是用来处理分发QWidget的消息，窗口消息
<details>
<summary>QtWndProc()</summary>
```
extern "C" LRESULT QT_WIN_CALLBACK QtWndProc(HWND hwnd, UINT message, WPARAM wParam, LPARAM lParam)
{
    bool result = true;
    QEvent::Type evt_type = QEvent::None;
    QETWidget *widget = 0;

        // there is no need to process pakcets from tablet unless
        // it is actually on the tablet, a flag to let us know...
        int nPackets;        // the number of packets we get from the queue

    long res = 0;
    if (!qApp)                                // unstable app state
#ifndef QT_NO_IM
        RETURN(QWinInputContext::DefWindowProc(hwnd,message,wParam,lParam))
#else
        return res;
#endif // QT_NO_IM
    QScopedLoopLevelCounter loopLevelCounter(QThreadData::get2(qApp->thread()));

#if 0
    // make sure we update widgets also when the user resizes
    if (inLoop && qApp->loopLevel())
        qApp->sendPostedEvents(0, QEvent::Paint);
#endif

    inLoop = true;

    MSG msg;
    msg.hwnd = hwnd;                                // create MSG structure
    msg.message = message;                        // time and pt fields ignored
    msg.wParam = wParam;
    msg.lParam = lParam;
    msg.pt.x = GET_X_LPARAM(lParam);
    msg.pt.y = GET_Y_LPARAM(lParam);
    // If it's a non-client-area message the coords are screen coords, otherwise they are
    // client coords.
#ifndef Q_WS_WINCE
    if (message < WM_NCMOUSEMOVE || message > WM_NCMBUTTONDBLCLK)
#endif
        ClientToScreen(msg.hwnd, &msg.pt);

    /*
    // sometimes the autograb is not released, so the clickevent is sent
    // to the wrong window. We ignore this for now, because it doesn't
    // cause any problems.
    if (msg.message == WM_LBUTTONDOWN || msg.message == WM_RBUTTONDOWN || msg.message == WM_MBUTTONDOWN) {
        HWND handle = WindowFromPoint(msg.pt);
        if (msg.hwnd != handle) {
            msg.hwnd = handle;
            hwnd = handle;
        }
    }
    */

#if defined(QT_NON_COMMERCIAL)
    QT_NC_WNDPROC
#endif

    // send through app filter
    if (qApp->filterEvent(&msg, &res))
        return res;

    // close any opened ime candidate window (enabled only on a popup widget)
    if (imeParentWnd  && QApplication::activePopupWidget()
        && (message == WM_MBUTTONDOWN || message == WM_XBUTTONDOWN
        || message == WM_LBUTTONDOWN || message == WM_RBUTTONDOWN
#ifndef Q_WS_WINCE
        || message == WM_NCMBUTTONDOWN || message == WM_NCLBUTTONDOWN
        || message == WM_NCRBUTTONDOWN)) {
#else
                                      )) {
#endif
            ::SendMessage(imeParentWnd, WM_IME_ENDCOMPOSITION, 0, 0);
    }

    switch (message) {
#ifndef Q_WS_WINCE
#ifndef QT_NO_SESSIONMANAGER
    case WM_QUERYENDSESSION: {
        if (sm_smActive) // bogus message from windows
            RETURN(true);

        sm_smActive = true;
        sm_blockUserInput = true; // prevent user-interaction outside interaction windows
        sm_cancel = false;
        if (qt_session_manager_self)
            qApp->commitData(*qt_session_manager_self);
        if (lParam & ENDSESSION_LOGOFF) {
            _flushall();
        }
        RETURN(!sm_cancel);
    }
    case WM_ENDSESSION: {
        sm_smActive = false;
        sm_blockUserInput = false;
        bool endsession = (bool) wParam;

        // we receive the message for each toplevel window included internal hidden ones,
        // but the aboutToQuit signal should be emitted only once.
        QApplicationPrivate *qAppPriv = QApplicationPrivate::instance();
        if (endsession && !qAppPriv->aboutToQuitEmitted) {
            qAppPriv->aboutToQuitEmitted = true;
            int index = QApplication::staticMetaObject.indexOfSignal("aboutToQuit()");
            qApp->qt_metacall(QMetaObject::InvokeMetaMethod, index,0);
            // since the process will be killed immediately quit() has no real effect
            QApplication::quit();
        }

        RETURN(0);
    }
#endif
    case WM_DISPLAYCHANGE:
        if (QApplication::type() == QApplication::Tty)
            break;
        if (qt_desktopWidget) {
            qt_desktopWidget->move(GetSystemMetrics(76), GetSystemMetrics(77));
            QSize sz(GetSystemMetrics(78), GetSystemMetrics(79));
            if (sz == qt_desktopWidget->size()) {
                 // a screen resized without changing size of the virtual desktop
                QResizeEvent rs(sz, qt_desktopWidget->size());
                QApplication::sendEvent(qt_desktopWidget, &rs);
            } else {
                qt_desktopWidget->resize(sz);
            }
        }
        break;
#endif

    case WM_SETTINGCHANGE:
#ifdef Q_WS_WINCE
        // CE SIP hide/show
        if (qt_desktopWidget && wParam == SPI_SETSIPINFO) {
            QResizeEvent re(QSize(0, 0), QSize(0, 0)); // Calculated by QDesktopWidget
            QApplication::sendEvent(qt_desktopWidget, &re);
            break;
        }
#endif
        // ignore spurious XP message when user logs in again after locking
        if (QApplication::type() == QApplication::Tty)
            break;
        if (QApplication::desktopSettingsAware() && wParam != SPI_SETWORKAREA) {
            widget = (QETWidget*)QWidget::find(hwnd);
            if (widget) {
                if (wParam == SPI_SETNONCLIENTMETRICS)
                    widget->markFrameStrutDirty();
            }
        }
        else if (qt_desktopWidget && wParam == SPI_SETWORKAREA) {
            qt_desktopWidget->move(GetSystemMetrics(76), GetSystemMetrics(77));
            QSize sz(GetSystemMetrics(78), GetSystemMetrics(79));
            if (sz == qt_desktopWidget->size()) {
                 // a screen resized without changing size of the virtual desktop
                QResizeEvent rs(sz, qt_desktopWidget->size());
                QApplication::sendEvent(qt_desktopWidget, &rs);
            } else {
                qt_desktopWidget->resize(sz);
            }
        }

        if (wParam == SPI_SETFONTSMOOTHINGTYPE) {
            qt_win_read_cleartype_settings();
            foreach (QWidget *w, QApplication::topLevelWidgets()) {
                if (!w->isVisible())
                    continue;
                ((QETWidget *) w)->forceUpdate();
            }
        }

        break;
    case WM_SYSCOLORCHANGE:
        if (QApplication::type() == QApplication::Tty)
            break;
        if (QApplication::desktopSettingsAware()) {
            widget = (QETWidget*)QWidget::find(hwnd);
            if (widget && !widget->parentWidget())
                qt_set_windows_color_resources();
        }
        break;

    case WM_LBUTTONDOWN:
    case WM_MBUTTONDOWN:
    case WM_RBUTTONDOWN:
    case WM_XBUTTONDOWN:
    case WM_LBUTTONDBLCLK:
    case WM_RBUTTONDBLCLK:
    case WM_MBUTTONDBLCLK:
    case WM_XBUTTONDBLCLK:
        if (qt_win_ignoreNextMouseReleaseEvent)
            qt_win_ignoreNextMouseReleaseEvent = false;
        break;

    case WM_LBUTTONUP:
    case WM_MBUTTONUP:
    case WM_RBUTTONUP:
    case WM_XBUTTONUP:
        if (qt_win_ignoreNextMouseReleaseEvent) {
            qt_win_ignoreNextMouseReleaseEvent = false;
            if (qt_button_down && qt_button_down->internalWinId() == autoCaptureWnd) {
                releaseAutoCapture();
                qt_button_down = 0;
            }

            RETURN(0);
        }
        break;

    default:
        break;
    }

    if (!widget)
        widget = (QETWidget*)QWidget::find(hwnd);
    if (!widget)                                // don't know this widget
        goto do_default;

    if (app_do_modal)        {                        // modal event handling
        int ret = 0;
        if (!qt_try_modal(widget, &msg, ret))
            RETURN(ret);
    }

    res = 0;
    if (widget->winEvent(&msg, &res))                // send through widget filter
        RETURN(res);

    if (qt_is_translatable_mouse_event(message)) {
        if (QApplication::activePopupWidget() != 0) { // in popup mode
            POINT curPos = msg.pt;
            QWidget* w = QApplication::widgetAt(curPos.x, curPos.y);
            if (w)
                widget = (QETWidget*)w;
        }

        if (!qt_tabletChokeMouse) {
            result = widget->translateMouseEvent(msg);        // mouse event
#if defined(Q_WS_WINCE) && !defined(QT_NO_CONTEXTMENU)
            if (message == WM_LBUTTONDOWN && widget != QApplication::activePopupWidget()) {
                QWidget* alienWidget = widget;
                if ((alienWidget != QApplication::activePopupWidget()) && (alienWidget->contextMenuPolicy() != Qt::PreventContextMenu)) {
                    QPoint pos(GET_X_LPARAM(lParam), GET_Y_LPARAM(lParam));
                    QPoint globalPos(msg.pt.x, msg.pt.y);
                    // In case we are using Alien, then the widget to
                    // send the context menu event is a different one
                    if (!alienWidget->testAttribute(Qt::WA_NativeWindow) && !alienWidget->testAttribute(Qt::WA_PaintOnScreen)) {
                        alienWidget = QApplication::widgetAt(globalPos);
                        if (alienWidget)
                            pos = alienWidget->mapFromGlobal(globalPos);
                    }
                    if (alienWidget) {
                        SHRGINFO shrg;
                        shrg.cbSize = sizeof(shrg);
                        shrg.hwndClient = hwnd;
                        shrg.ptDown.x = GET_X_LPARAM(lParam);
                        shrg.ptDown.y = GET_Y_LPARAM(lParam);
                        shrg.dwFlags = SHRG_RETURNCMD | SHRG_NOANIMATION;
#ifndef QT_NO_GESTURES
                        resolveAygLibs();
                        if (ptrRecognizeGesture && (ptrRecognizeGesture(&shrg) == GN_CONTEXTMENU)) {
                            if (QApplication::activePopupWidget())
                                QApplication::activePopupWidget()->close();
                            QContextMenuEvent e(QContextMenuEvent::Mouse, pos, globalPos);
                            result = qt_sendSpontaneousEvent(alienWidget, &e);
                        }
#endif // QT_NO_GESTURES
                    }
                }
            }
#endif
        } else {
            // Sometimes we only get a WM_MOUSEMOVE message
            // and sometimes we get both a WM_MOUSEMOVE and
            // a WM_LBUTTONDOWN/UP, this creates a spurious mouse
            // press/release event, using the PeekMessage
            // will help us fix this.  This leaves us with a
            // question:
            //    This effectively kills using the mouse AND the
            //    tablet simultaneously, well creates wacky input.
            //    Is this going to be a problem? (probably not)
            bool next_is_button = false;
            bool is_mouse_move = (message == WM_MOUSEMOVE);
            if (is_mouse_move) {
                MSG msg1;
                if (PeekMessage(&msg1, msg.hwnd, WM_MOUSEFIRST,
                                WM_MOUSELAST, PM_NOREMOVE))
                    next_is_button = (msg1.message == WM_LBUTTONUP
                                       || msg1.message == WM_LBUTTONDOWN);
            }
            if (!is_mouse_move || (is_mouse_move && !next_is_button))
                qt_tabletChokeMouse = false;
        }
    } else {
        switch (message) {
        case WM_TOUCH:
            result = QApplicationPrivate::instance()->translateTouchEvent(msg);
            break;
        case WM_KEYDOWN:                        // keyboard event
        case WM_SYSKEYDOWN:
            qt_keymapper_private()->updateKeyMap(msg);
            // fall-through intended
        case WM_KEYUP:
        case WM_SYSKEYUP:
#if Q_OS_WINCE_WM
        case WM_HOTKEY:
            if(HIWORD(msg.lParam) == VK_TBACK) {
                const bool hotKeyDown = !(LOWORD(msg.lParam) & MOD_KEYUP);
                msg.lParam = 0x69 << 16;
                msg.wParam = VK_BACK;
                if (hotKeyDown) {
                    msg.message = WM_KEYDOWN;
                    qt_keymapper_private()->updateKeyMap(msg);
                } else {
                    msg.message = WM_KEYUP;
                }
            }
            // fall-through intended
#endif
        case WM_IME_CHAR:
        case WM_IME_KEYDOWN:
        case WM_CHAR: {
            MSG msg1;
            bool anyMsg = PeekMessage(&msg1, msg.hwnd, 0, 0, PM_NOREMOVE);
            if (anyMsg && msg1.message == WM_DEADCHAR) {
                result = true; // consume event since there is a dead char next
                break;
            }
            QWidget *g = QWidget::keyboardGrabber();
            if (g && qt_get_tablet_widget() && hwnd == qt_get_tablet_widget()->winId()) {
                // if we get an event for the internal tablet widget,
                // then don't send it to the keyboard grabber, but
                // send it to the widget itself (we don't use it right
                // now, just in case).
                g = 0;
            }
            if (g)
                widget = (QETWidget*)g;
            else if (QApplication::activePopupWidget())
                widget = (QETWidget*)QApplication::activePopupWidget()->focusWidget()
                       ? (QETWidget*)QApplication::activePopupWidget()->focusWidget()
                       : (QETWidget*)QApplication::activePopupWidget();
            else if (QApplication::focusWidget())
                widget = (QETWidget*)QApplication::focusWidget();
            else if (!widget || widget->internalWinId() == GetFocus()) // We faked the message to go to exactly that widget.
                widget = (QETWidget*)widget->window();
            if (widget->isEnabled())
                result = sm_blockUserInput
                            ? true
                            : qt_keymapper_private()->translateKeyEvent(widget, msg, g != 0);
            break;
        }
        case WM_SYSCHAR:
            result = true;                        // consume event
            break;

        case WM_MOUSEWHEEL:
        case WM_MOUSEHWHEEL:
            result = widget->translateWheelEvent(msg);
            break;

        case WM_APPCOMMAND:
            {
                uint cmd = GET_APPCOMMAND_LPARAM(lParam);
                uint uDevice = GET_DEVICE_LPARAM(lParam);
                uint dwKeys = GET_KEYSTATE_LPARAM(lParam);

                int state = translateButtonState(dwKeys, QEvent::KeyPress, 0);

                switch (uDevice) {
                case FAPPCOMMAND_KEY:
                    {
                        int key = 0;

                        switch(cmd) {
                        case APPCOMMAND_BASS_BOOST:
                            key = Qt::Key_BassBoost;
                            break;
                        case APPCOMMAND_BASS_DOWN:
                            key = Qt::Key_BassDown;
                            break;
                        case APPCOMMAND_BASS_UP:
                            key = Qt::Key_BassUp;
                            break;
                        case APPCOMMAND_TREBLE_DOWN:
                            key = Qt::Key_TrebleDown;
                            break;
                        case APPCOMMAND_TREBLE_UP:
                            key = Qt::Key_TrebleUp;
                            break;
                        case APPCOMMAND_HELP:
                            key = Qt::Key_Help;
                            break;
                        case APPCOMMAND_FIND:
                            key = Qt::Key_Search;
                            break;
                        default:
                            break;
                        }
                        if (key) {
                            bool res = false;
                            QWidget *g = QWidget::keyboardGrabber();
                            if (g)
                                widget = (QETWidget*)g;
                            else if (QApplication::focusWidget())
                                widget = (QETWidget*)QApplication::focusWidget();
                            else
                                widget = (QETWidget*)widget->window();
                            if (widget->isEnabled()) {
                                res = QKeyMapper::sendKeyEvent(widget, g != 0, QEvent::KeyPress, key,
                                                               Qt::KeyboardModifier(state),
                                                               QString(), false, 0, 0, 0, 0);
                            }
                            if (res)
                                return true;
                        }
                    }
                    break;

                default:
                    break;
                }

                result = false;
            }
            break;

#ifndef Q_WS_WINCE
        case WM_NCHITTEST:
            if (widget->isWindow()) {
                QPoint pos = widget->mapFromGlobal(QPoint(GET_X_LPARAM(lParam), GET_Y_LPARAM(lParam)));
                // don't show resize-cursors for fixed-size widgets
                QRect fs = widget->frameStrut();
                if (!widget->isMinimized()) {
                    if (widget->minimumHeight() == widget->maximumHeight()) {
                        if (pos.y() < -(fs.top() - fs.left()))
                            return HTCAPTION;
                        if (pos.y() >= widget->height())
                            return HTBORDER;
                    }
                    if (widget->minimumWidth() == widget->maximumWidth() && (pos.x() < 0 || pos.x() >= widget->width()))
                        return HTBORDER;
                }
            }

            result = false;
            break;
#endif

        case WM_SYSCOMMAND: {
#ifndef Q_WS_WINCE
            bool window_state_change = false;
            Qt::WindowStates oldstate = Qt::WindowStates(widget->dataPtr()->window_state);
            // MSDN:In WM_SYSCOMMAND messages, the four low-order bits of the wParam parameter are
            // used internally by the system. To obtain the correct result when testing the value of
            // wParam, an application must combine the value 0xFFF0 with the wParam value by using
            // the bitwise AND operator.
            switch(0xfff0 & wParam) {
            case SC_CONTEXTHELP:
#ifndef QT_NO_WHATSTHIS
                QWhatsThis::enterWhatsThisMode();
#endif
                DefWindowProc(hwnd, WM_NCPAINT, 1, 0);
                break;
#if defined(QT_NON_COMMERCIAL)
                QT_NC_SYSCOMMAND
#endif
            case SC_MINIMIZE:
                window_state_change = true;
                widget->dataPtr()->window_state |= Qt::WindowMinimized;
                if (widget->isVisible()) {
                    QHideEvent e;
                    qt_sendSpontaneousEvent(widget, &e);
                    widget->hideChildren(true);
                    const QString title = widget->windowIconText();
                    if (!title.isEmpty())
                        widget->setWindowTitle_helper(title);
                }
                result = false;
                break;
            case SC_MAXIMIZE:
                if(widget->isWindow())
                    widget->topData()->normalGeometry = widget->geometry();
            case SC_RESTORE:
                window_state_change = true;
                if ((0xfff0 & wParam) == SC_MAXIMIZE)
                    widget->dataPtr()->window_state |= Qt::WindowMaximized;
                else if (!widget->isMinimized())
                    widget->dataPtr()->window_state &= ~Qt::WindowMaximized;

                if (widget->isMinimized()) {
                    widget->dataPtr()->window_state &= ~Qt::WindowMinimized;
                    widget->showChildren(true);
                    QShowEvent e;
                    qt_sendSpontaneousEvent(widget, &e);
                    const QString title = widget->windowTitle();
                    if (!title.isEmpty())
                        widget->setWindowTitle_helper(title);
                }
                result = false;
                break;
            default:
                result = false;
                break;
            }

            if (window_state_change) {
                QWindowStateChangeEvent e(oldstate);
                qt_sendSpontaneousEvent(widget, &e);
            }
#endif // #ifndef Q_OS_WINCE

            break;
        }

        case WM_SETTINGCHANGE:
            if ( QApplication::type() == QApplication::Tty )
                break;

            if (!msg.wParam) {
#ifdef Q_WS_WINCE
                // On Windows CE, lParam parameter is a constant, not a char pointer.
                if (msg.lParam == INI_INTL) {
#else
                QString area = QString::fromWCharArray((wchar_t*)msg.lParam);
                if (area == QLatin1String("intl")) {
#endif
                    QLocalePrivate::updateSystemPrivate();
                    if (!widget->testAttribute(Qt::WA_SetLocale))
                        widget->dptr()->setLocale_helper(QLocale(), true);
                    QEvent e(QEvent::LocaleChange);
                    QApplication::sendEvent(qApp, &e);
                }
            }
            else if (msg.wParam == SPI_SETICONTITLELOGFONT) {
                if (QApplication::desktopSettingsAware()) {
                    widget = (QETWidget*)QWidget::find(hwnd);
                    if (widget && !widget->parentWidget()) {
                        qt_set_windows_font_resources();
                    }
                }
            }
            else if (msg.wParam == SPI_SETNONCLIENTMETRICS) {
                widget = (QETWidget*)QWidget::find(hwnd);
                if (widget && !widget->parentWidget()) {
                    qt_set_windows_updateScrollBar(widget);
                    QEvent e(QEvent::LayoutRequest);
                    QApplication::sendEvent(widget, &e);
                }
        }

            break;

        case WM_PAINT:                                // paint event
        case WM_ERASEBKGND:                        // erase window background
            result = widget->translatePaintEvent(msg);
            break;

#ifndef Q_WS_WINCE
        case WM_ENTERSIZEMOVE:
            autoCaptureWnd = hwnd;
            break;
        case WM_EXITSIZEMOVE:
            autoCaptureWnd = 0;
            break;
#endif
        case WM_MOVE:                                // move window
        case WM_SIZE:                                // resize window
            result = widget->translateConfigEvent(msg);
            break;

        case WM_ACTIVATEAPP:
            if (wParam == FALSE) {
                QApplication::setActiveWindow(0);
                // Another application was activated while our popups are open,
                // then close all popups.  In case some popup refuses to close,
                // we give up after 1024 attempts (to avoid an infinite loop).
                int maxiter = 1024;
                QWidget *popup;
                while ((popup=QApplication::activePopupWidget()) && maxiter--)
                    popup->close();
            }
            break;

        case WM_ACTIVATE:
            if ( QApplication::type() == QApplication::Tty )
                break;

            if (ptrWTOverlap && ptrWTEnable) {
                // cooperate with other tablet applications, but when
                // we get focus, I want to use the tablet...
                if (qt_tablet_context && GET_WM_ACTIVATE_STATE(wParam, lParam)) {
                    if (ptrWTEnable(qt_tablet_context, true))
                        ptrWTOverlap(qt_tablet_context, true);
                }
            }
            if (QApplication::activePopupWidget() && LOWORD(wParam) == WA_INACTIVE &&
                QWidget::find((HWND)lParam) == 0) {
                // Another application was activated while our popups are open,
                // then close all popups.  In case some popup refuses to close,
                // we give up after 1024 attempts (to avoid an infinite loop).
                int maxiter = 1024;
                QWidget *popup;
                while ((popup=QApplication::activePopupWidget()) && maxiter--)
                    popup->close();
            }

            if (LOWORD(wParam) != WA_INACTIVE) {
                // WM_ACTIVATEAPP handles the "true" false case, as this is only when the application
                // loses focus. Doing it here would result in the widget getting focus to not know
                // where it got it from; it would simply get a 0 value as the old focus widget.
#ifdef Q_WS_WINCE
                {
#ifdef Q_WS_WINCE_WM
                    // On Windows mobile we do not receive WM_SYSCOMMAND / SC_MINIMIZE messages.
                    // Thus we have to unset the minimized state explicitly. We must do this for all
                    // top-level widgets, because we get the HWND of a random widget here.
                    foreach (QWidget* tlw, QApplication::topLevelWidgets()) {
                        if (tlw->isMinimized())
                            tlw->setWindowState(tlw->windowState() & ~Qt::WindowMinimized);
                    }
#else
                    // On Windows CE we do not receive WM_SYSCOMMAND / SC_MINIMIZE messages.
                    // Thus we have to unset the minimized state explicitly.
                    if (widget->windowState() & Qt::WindowMinimized)
                        widget->setWindowState(widget->windowState() & ~Qt::WindowMinimized);
#endif  // Q_WS_WINCE_WM

#else
                if (!(widget->windowState() & Qt::WindowMinimized)) {
#endif
                    // Ignore the activate message send by WindowsXP to a minimized window
#ifdef Q_WS_WINCE_WM
                    if  (widget->windowState() & Qt::WindowFullScreen)
                        qt_wince_hide_taskbar(widget->winId());
#endif
                    qApp->winFocus(widget, true);
                    // reset any window alert flashes
                    alert_widget(widget, -1);
                }
            }

            // Windows tries to activate a modally blocked window.
            // This happens when restoring an application after "Show Desktop"
            if (app_do_modal && LOWORD(wParam) == WA_ACTIVE) {
                QWidget *top = 0;
                if (!QApplicationPrivate::tryModalHelper(widget, &top) && top && widget != top) {
                    if (top->isVisible()) {
                        top->activateWindow();
                    } else {
                        // This is the case when native file dialogs are shown
                        QWidget *p = (top->parentWidget() ? top->parentWidget()->window() : 0);
                        if (p && p->isVisible())
                            p->activateWindow();
                    }
                }
            }
            break;

#ifndef Q_WS_WINCE
            case WM_MOUSEACTIVATE:
                if (widget->window()->windowType() == Qt::Tool) {
                    QWidget *w = widget;
                    if (!w->window()->focusWidget()) {
                        while (w && (w->focusPolicy() & Qt::ClickFocus) == 0) {
                            if (w->isWindow()) {
                                QWidget *fw = w;
                                while ((fw = fw->nextInFocusChain()) != w && fw->focusPolicy() == Qt::NoFocus)
                                    ;
                                if (fw != w)
                                   break;
                                QWidget *pw = w->parentWidget();
                                while (pw) {
                                    pw = pw->window();
                                    if (pw && pw->isVisible() && pw->focusWidget()) {
                                        Q_ASSERT(pw->testAttribute(Qt::WA_WState_Created));
                                        SetWindowPos(pw->internalWinId(), HWND_TOP, 0, 0, 0, 0, SWP_NOSIZE | SWP_NOMOVE);
                                        break;
                                    }
                                    pw = pw->parentWidget();
                                }
                                RETURN(MA_NOACTIVATE);
                            }
                            w = w->parentWidget();
                        }
                    }
                }
                RETURN(MA_ACTIVATE);
                break;
#endif
            case WM_SHOWWINDOW:
                if (lParam == SW_PARENTOPENING) {
                    if (widget->testAttribute(Qt::WA_WState_Hidden))
                        RETURN(0);
                }
                if (widget->isWindow() && widget->testAttribute(Qt::WA_WState_Visible)
                    && !widget->testWindowState(Qt::WindowMinimized)) {
                    if (lParam == SW_PARENTOPENING) {
                        QShowEvent e;
                        qt_sendSpontaneousEvent(widget, &e);
                        widget->showChildren(true);
                    } else if (lParam == SW_PARENTCLOSING) {
                        QHideEvent e;
                        qt_sendSpontaneousEvent(widget, &e);
                        widget->hideChildren(true);
                    }
                }
                if  (!wParam && autoCaptureWnd == widget->internalWinId())
                    releaseAutoCapture();
                result = false;
                break;

        case WM_PALETTECHANGED:                        // our window changed palette
            if (QColormap::hPal() && (WId)wParam == widget->internalWinId())
                RETURN(0);                        // otherwise: FALL THROUGH!
            // FALL THROUGH
        case WM_QUERYNEWPALETTE:                // realize own palette
            if (QColormap::hPal()) {
                Q_ASSERT(widget->testAttribute(Qt::WA_WState_Created));
                HDC hdc = GetDC(widget->internalWinId());
                HPALETTE hpalOld = SelectPalette(hdc, QColormap::hPal(), FALSE);
                uint n = RealizePalette(hdc);
                if (n)
                    InvalidateRect(widget->internalWinId(), 0, TRUE);
                SelectPalette(hdc, hpalOld, TRUE);
                RealizePalette(hdc);
                ReleaseDC(widget->internalWinId(), hdc);
                RETURN(n);
            }
            break;
        case WM_CLOSE:                                // close window
            widget->translateCloseEvent(msg);
            RETURN(0);                                // always handled

        case WM_DESTROY:                        // destroy window
            if (hwnd == curWin) {
                QWidget *enter = QWidget::mouseGrabber();
                if (enter == widget)
                    enter = 0;
                QApplicationPrivate::dispatchEnterLeave(enter, widget);
                curWin = enter ? enter->effectiveWinId() : 0;
                qt_last_mouse_receiver = enter;
            }
            if (widget == popupButtonFocus)
                popupButtonFocus = 0;
            result = false;
            break;

#ifndef Q_WS_WINCE
        case WM_WINDOWPOSCHANGING:
            {
                result = false;
                if (widget->isWindow()) {
                    WINDOWPOS *winPos = (WINDOWPOS *)lParam;
                    if (widget->layout() && widget->layout()->hasHeightForWidth()
                        && !(winPos->flags & (SWP_NOCOPYBITS | SWP_NOSIZE))) {
                        QRect fs = widget->frameStrut();
                        QRect rect = widget->geometry();
                        QRect newRect = QRect(winPos->x + fs.left(),
                                              winPos->y + fs.top(),
                                              winPos->cx - fs.left() - fs.right(),
                                              winPos->cy - fs.top() - fs.bottom());

                        QSize newSize = QLayout::closestAcceptableSize(widget, newRect.size());

                        int dh = newSize.height() - newRect.height();
                        int dw = newSize.width() - newRect.width();
                        if (!dw && ! dh)
                            break; // Size OK

                        if (rect.y() != newRect.y()) {
                            newRect.setTop(newRect.top() - dh);
                        } else {
                            newRect.setBottom(newRect.bottom() + dh);
                        }

                        if (rect.x() != newRect.x()) {
                            newRect.setLeft(newRect.left() - dw);
                        } else {
                            newRect.setRight(newRect.right() + dw);
                        }

                        winPos->x = newRect.x() - fs.left();
                        winPos->y = newRect.y() - fs.top();
                        winPos->cx = newRect.width() + fs.left() + fs.right();
                        winPos->cy = newRect.height() + fs.top() + fs.bottom();

                        RETURN(0);
                    }
                    if (widget->windowFlags() & Qt::WindowStaysOnBottomHint) {
                        winPos->hwndInsertAfter = HWND_BOTTOM;
                    }
                }
            }
            break;

        case WM_GETMINMAXINFO:
            if (widget->xtra()) {
                MINMAXINFO *mmi = (MINMAXINFO *)lParam;
                QWExtra *x = widget->xtra();
                QRect fs = widget->frameStrut();
                if ( x->minw > 0 )
                    mmi->ptMinTrackSize.x = x->minw + fs.right() + fs.left();
                if ( x->minh > 0 )
                    mmi->ptMinTrackSize.y = x->minh + fs.top() + fs.bottom();
                qint32 maxw = (x->maxw >= x->minw) ? x->maxw : x->minw;
                qint32 maxh = (x->maxh >= x->minh) ? x->maxh : x->minh;
                if ( maxw < QWIDGETSIZE_MAX ) {
                    mmi->ptMaxTrackSize.x = maxw + fs.right() + fs.left();
                    // windows with title bar have an implicit size limit of 112 pixels
                    if (widget->windowFlags() & Qt::WindowTitleHint)
                        mmi->ptMaxTrackSize.x = qMax<long>(mmi->ptMaxTrackSize.x, 112);
                }
                if ( maxh < QWIDGETSIZE_MAX )
                    mmi->ptMaxTrackSize.y = maxh + fs.top() + fs.bottom();
                RETURN(0);
            }
            break;

#ifndef QT_NO_CONTEXTMENU
            case WM_CONTEXTMENU:
            {
                // it's not VK_APPS or Shift+F10, but a click in the NC area
                if (lParam != (int)0xffffffff) {
                    result = false;
                    break;
                }

                QWidget *fw = QWidget::keyboardGrabber();
                if (!fw) {
                    if (QApplication::activePopupWidget())
                        fw = (QApplication::activePopupWidget()->focusWidget()
                                                  ? QApplication::activePopupWidget()->focusWidget()
                                                  : QApplication::activePopupWidget());
                    else if (QApplication::focusWidget())
                        fw = QApplication::focusWidget();
                    else if (widget)
                        fw = widget->window();
                }
                if (fw && fw->isEnabled()) {
                    QPoint pos = fw->inputMethodQuery(Qt::ImMicroFocus).toRect().center();
                    QContextMenuEvent e(QContextMenuEvent::Keyboard, pos, fw->mapToGlobal(pos),
                                      qt_win_getKeyboardModifiers());
                    result = qt_sendSpontaneousEvent(fw, &e);
                }
            }
            break;
#endif
#endif

#ifndef QT_NO_IM
        case WM_IME_STARTCOMPOSITION:
        case WM_IME_ENDCOMPOSITION:
        case WM_IME_COMPOSITION: {
            QWidget *fw = QApplication::focusWidget();
            QWinInputContext *im = fw ? qobject_cast<QWinInputContext *>(fw->inputContext()) : 0;
            if (fw && im) {
                if(message == WM_IME_STARTCOMPOSITION)
                    result = im->startComposition();
                else if (message == WM_IME_ENDCOMPOSITION)
                    result = im->endComposition();
                else if (message == WM_IME_COMPOSITION)
                    result = im->composition(lParam);
            }
            break;
        }
        case WM_IME_REQUEST: {
            QWidget *fw = QApplication::focusWidget();
            QWinInputContext *im = fw ? qobject_cast<QWinInputContext *>(fw->inputContext()) : 0;
            if (fw && im) {
                if(wParam == IMR_RECONVERTSTRING) {
                    int ret = im->reconvertString((RECONVERTSTRING *)lParam);
                    if (ret == -1) {
                        result = false;
                    } else {
                        return ret;
                    }
                } else if (wParam == IMR_CONFIRMRECONVERTSTRING) {
                    RETURN(TRUE);
                } else {
                    // in all other cases, call DefWindowProc()
                    result = false;
                }
            }
            break;
        }
#endif // QT_NO_IM
#ifndef Q_WS_WINCE
        case WM_CHANGECBCHAIN:
        case WM_DRAWCLIPBOARD:
#endif
        case WM_RENDERFORMAT:
        case WM_RENDERALLFORMATS:
#ifndef QT_NO_CLIPBOARD
        case WM_DESTROYCLIPBOARD:
            if (qt_clipboard) {
                QClipboardEvent e(reinterpret_cast<QEventPrivate *>(&msg));
                qt_sendSpontaneousEvent(qt_clipboard, &e);
                RETURN(0);
            }
            result = false;
            break;
#endif //QT_NO_CLIPBOARD
#ifndef QT_NO_ACCESSIBILITY
        case WM_GETOBJECT:
            {
#if !defined(Q_OS_WINCE)
                /* On Win64, lParam can be 0x00000000fffffffc or 0xfffffffffffffffc (!),
                   but MSDN says that lParam should be converted to a DWORD
                   before its compared against OBJID_CLIENT
                */
                const DWORD dwObjId = (DWORD)lParam;
                // Ignoring all requests while starting up
                if (QApplication::startingUp() || QApplication::closingDown() || dwObjId != OBJID_CLIENT) {
                    result = false;
                    break;
                }

                typedef LRESULT (WINAPI *PtrLresultFromObject)(REFIID, WPARAM, LPUNKNOWN);
                static PtrLresultFromObject ptrLresultFromObject = 0;
                static bool oleaccChecked = false;
                if (!oleaccChecked) {
                    QSystemLibrary oleacclib(QLatin1String("oleacc"));
                    ptrLresultFromObject = (PtrLresultFromObject)oleacclib.resolve("LresultFromObject");
                    oleaccChecked = true;
                }
                if (ptrLresultFromObject) {
                    QAccessibleInterface *acc = QAccessible::queryAccessibleInterface(widget);
                    if (!acc) {
                        result = false;
                        break;
                    }

                    // and get an instance of the IAccessibile implementation
                    IAccessible *iface = qt_createWindowsAccessible(acc);
                    res = ptrLresultFromObject(IID_IAccessible, wParam, iface);  // ref == 2
                    iface->Release(); // the client will release the object again, and then it will destroy itself

                    if (res > 0)
                        RETURN(res);
                }
#endif
            }
            result = false;
            break;
        case WM_GETTEXT:
            if (!widget->isWindow()) {
                int ret = 0;
                QAccessibleInterface *acc = QAccessible::queryAccessibleInterface(widget);
                if (acc) {
                    QString text = acc->text(QAccessible::Name, 0);
                    if (text.isEmpty())
                        text = widget->objectName();
                    ret = qMin<int>(wParam - 1, text.size());
                    text.resize(ret);
                    memcpy((void *)lParam, text.utf16(), (text.size() + 1) * sizeof(ushort));
                    delete acc;
                }
                if (!ret) {
                    result = false;
                    break;
                }
                RETURN(ret);
            }
            result = false;
            break;
#endif
        case WT_PACKET:
            if (ptrWTPacketsGet) {
                if ((nPackets = ptrWTPacketsGet(qt_tablet_context, QT_TABLET_NPACKETQSIZE, &localPacketBuf))) {
                    result = widget->translateTabletEvent(msg, localPacketBuf, nPackets);
                }
            }
            break;
        case WT_PROXIMITY:

            #ifndef QT_NO_TABLETEVENT
            if (ptrWTPacketsGet && ptrWTInfo) {
                const bool enteredProximity = LOWORD(lParam) != 0;
                PACKET proximityBuffer[1]; // we are only interested in the first packet in this case
                const int totalPacks = ptrWTPacketsGet(qt_tablet_context, 1, proximityBuffer);
                if (totalPacks > 0) {
                    const UINT currentCursor = proximityBuffer[0].pkCursor;

                    UINT csr_physid;
                    ptrWTInfo(WTI_CURSORS + currentCursor, CSR_PHYSID, &csr_physid);
                    UINT csr_type;
                    ptrWTInfo(WTI_CURSORS + currentCursor, CSR_TYPE, &csr_type);
                    const UINT deviceIdMask = 0xFF6; // device type mask && device color mask
                    quint64 uniqueId = (csr_type & deviceIdMask);
                    uniqueId = (uniqueId << 32) | csr_physid;

                    // initialising and updating the cursor should be done in response to
                    // WT_CSRCHANGE. We do it in WT_PROXIMITY because some wintab never send
                    // the event WT_CSRCHANGE even if asked with CXO_CSRMESSAGES
                    const QTabletCursorInfo *const globalCursorInfo = tCursorInfo();
                    if (!globalCursorInfo->contains(uniqueId))
                        tabletInit(uniqueId, csr_type, qt_tablet_context);

                    currentTabletPointer = globalCursorInfo->value(uniqueId);
                    tabletUpdateCursor(currentTabletPointer, currentCursor);
                }
                qt_tabletChokeMouse = false;

                QTabletEvent tabletProximity(enteredProximity ? QEvent::TabletEnterProximity
                                                              : QEvent::TabletLeaveProximity,
                                             QPoint(), QPoint(), QPointF(), currentTabletPointer.currentDevice, currentTabletPointer.currentPointerType, 0, 0,
                                             0, 0, 0, 0, 0, currentTabletPointer.llId);
                QApplication::sendEvent(qApp, &tabletProximity);
            }
            #endif // QT_NO_TABLETEVENT

            break;
#ifdef Q_WS_WINCE_WM
        case WM_SETFOCUS: {
            HIMC hC;
            hC = ImmGetContext(hwnd);
            ImmSetOpenStatus(hC, TRUE);
            ImmEscape(NULL, hC, IME_ESC_SET_MODE, (LPVOID)IM_SPELL);
            result = false;
        }
        break;
#endif
        case WM_KILLFOCUS:
            if (!QWidget::find((HWND)wParam)) { // we don't get focus, so unset it now
                if (!widget->hasFocus()) // work around Windows bug after minimizing/restoring
                    widget = (QETWidget*)QApplication::focusWidget();
                HWND focus = ::GetFocus();
                //if there is a current widget and the new widget belongs to the same toplevel window
                //or if the current widget was embedded into non-qt window (i.e. we won't get WM_ACTIVATEAPP)
                //then we clear the focus on the widget
                //in case the new widget belongs to a different widget hierarchy, clearing the focus
                //will be handled because the active window will change
                const bool embedded = widget && ((QETWidget*)widget->window())->topData()->embedded;
                if (widget && (embedded || ::IsChild(widget->window()->internalWinId(), focus))) {
                    widget->clearFocus();
                    result = true;
                } else {
                    result = false;
                }
            } else {
                result = false;
            }
            break;
        case WM_THEMECHANGED:
            if ((widget->windowType() == Qt::Desktop) || !qApp || QApplication::closingDown()
                                                         || QApplication::type() == QApplication::Tty)
                break;

            if (widget->testAttribute(Qt::WA_WState_Polished))
                QApplication::style()->unpolish(widget);

            if (widget->testAttribute(Qt::WA_WState_Polished))
                QApplication::style()->polish(widget);
            widget->repolishStyle(*QApplication::style());
            if (widget->isVisible())
                widget->update();
            break;

#ifndef Q_WS_WINCE
        case WM_INPUTLANGCHANGE: {
            wchar_t info[7];
            if (!GetLocaleInfo(MAKELCID(lParam, SORT_DEFAULT), LOCALE_IDEFAULTANSICODEPAGE, info, 6)) {
                inputcharset = CP_ACP;
            } else {
                inputcharset = QString::fromWCharArray(info).toInt();
            }
            QKeyMapper::changeKeyboard();
            break;
        }
#else
        case WM_COMMAND: {
            bool OkCommand = (LOWORD(wParam) == 0x1);
            bool CancelCommand = (LOWORD(wParam) == 0x2);
            if (OkCommand)
                QApplication::postEvent(widget, new QEvent(QEvent::OkRequest));
            if (CancelCommand)
                widget->showMinimized();
            else
#ifndef QT_NO_MENUBAR
                QMenuBar::wceCommands(LOWORD(wParam));
#endif
            result = true;
        }
            break;
        case WM_HELP:
            QApplication::postEvent(widget, new QEvent(QEvent::HelpRequest));
            result = true;
            break;
#endif

        case WM_MOUSELEAVE:
            // We receive a mouse leave for curWin, meaning
            // the mouse was moved outside our widgets
            if (widget->internalWinId() == curWin) {
                bool dispatch = !widget->underMouse();
                // hasMouse is updated when dispatching enter/leave,
                // so test if it is actually up-to-date
                if (!dispatch) {
                    QRect geom = widget->geometry();
                    if (widget->parentWidget() && !widget->isWindow()) {
                        QPoint gp = widget->parentWidget()->mapToGlobal(widget->pos());
                        geom.setX(gp.x());
                        geom.setY(gp.y());
                    }
                    QPoint cpos = QCursor::pos();
                    dispatch = !geom.contains(cpos);
                    if ( !dispatch && !QWidget::mouseGrabber()) {
                        QWidget *hittest = QApplication::widgetAt(cpos);
                        dispatch = !hittest || hittest->internalWinId() != curWin;
                    }
                    if (!dispatch) {
                        HRGN hrgn = qt_tryCreateRegion(QRegion::Rectangle, 0,0,0,0);
                        if (GetWindowRgn(curWin, hrgn) != ERROR) {
                            QPoint lcpos = widget->mapFromGlobal(cpos);
                            dispatch = !PtInRegion(hrgn, lcpos.x(), lcpos.y());
                        }
                        DeleteObject(hrgn);
                    }
                }
                if (dispatch) {
                    if (qt_last_mouse_receiver && !qt_last_mouse_receiver->internalWinId())
                        QApplicationPrivate::dispatchEnterLeave(0, qt_last_mouse_receiver);
                    else
                        QApplicationPrivate::dispatchEnterLeave(0, QWidget::find((WId)curWin));
                    curWin = 0;
                    qt_last_mouse_receiver = 0;
                }
            }
            break;

        case WM_CANCELMODE:
            {
                // this goes through QMenuBar's event filter
                QEvent e(QEvent::ActivationChange);
                QApplication::sendEvent(qApp, &e);
            }
            break;

        case WM_IME_NOTIFY:
            // special handling for ime, only for widgets in a popup
            if (wParam  == IMN_OPENCANDIDATE) {
                imeParentWnd = hwnd;
                if (QApplication::activePopupWidget()) {
                    // temporarily disable the mouse grab to allow mouse input in
                    // the ime candidate window. The actual handle is untouched
                    if (autoCaptureWnd)
                        ReleaseCapture();
                }
            } else if (wParam  == IMN_CLOSECANDIDATE) {
                imeParentWnd = 0;
                if (QApplication::activePopupWidget()) {
                    // undo the action above, when candidate window is closed
                    if (autoCaptureWnd)
                        SetCapture(autoCaptureWnd);
                }
            }
            result = false;
            break;
#ifndef QT_NO_GESTURES
#if !defined(Q_WS_WINCE) || defined(QT_WINCE_GESTURES)
        case WM_GESTURE: {
            GESTUREINFO gi;
            memset(&gi, 0, sizeof(GESTUREINFO));
            gi.cbSize = sizeof(GESTUREINFO);

            QApplicationPrivate *qAppPriv = QApplicationPrivate::instance();
            BOOL bResult = false;
            if (qAppPriv->GetGestureInfo)
                bResult = qAppPriv->GetGestureInfo((HANDLE)msg.lParam, &gi);
            if (bResult) {
                if (gi.dwID == GID_BEGIN) {
                    // find the alien widget for the gesture position.
                    // This might not be accurate as the position is the center
                    // point of two fingers for multi-finger gestures.
                    QPoint pt(gi.ptsLocation.x, gi.ptsLocation.y);
                    QWidget *w = widget->childAt(widget->mapFromGlobal(pt));
                    qAppPriv->gestureWidget = w ? w : widget;
                }
                if (qAppPriv->gestureWidget)
                    static_cast<QETWidget*>(qAppPriv->gestureWidget)->translateGestureEvent(msg, gi);
                if (qAppPriv->CloseGestureInfoHandle)
                    qAppPriv->CloseGestureInfoHandle((HANDLE)msg.lParam);
                if (gi.dwID == GID_END)
                    qAppPriv->gestureWidget = 0;
            } else {
                DWORD dwErr = GetLastError();
                if (dwErr > 0)
                    qWarning() << "translateGestureEvent: error = " << dwErr;
            }
            result = true;
            break;
        }
#endif // !defined(Q_WS_WINCE) || defined(QT_WINCE_GESTURES)
#endif // QT_NO_GESTURES
#ifndef QT_NO_CURSOR
        case WM_SETCURSOR: {
            QCursor *ovr = QApplication::overrideCursor();
            if (ovr) {
                SetCursor(ovr->handle());
                RETURN(TRUE);
            }
            result = false;
            break;
        }
#endif
        default:
            result = false;                        // event was not processed
            break;
        }
    }

    if (evt_type != QEvent::None) {                // simple event
        QEvent e(evt_type);
        result = qt_sendSpontaneousEvent(widget, &e);
    }

    if (result)
        RETURN(false);

do_default:
#ifndef QT_NO_IM
    RETURN(QWinInputContext::DefWindowProc(hwnd,message,wParam,lParam))
#else
    RETURN(TRUE);
#endif
}
```
<details>
####最后全部事件通过QApplication::SendEvent（）派发出去，
```
inline bool QCoreApplication::sendEvent(QObject *receiver, QEvent *event)
{  if (event) event->spont = false; return self ? self->notifyInternal(receiver, event) : false; }
```
####可以看到sendEvent调用了notifyInternal()，这个函数
```
bool QCoreApplication::notifyInternal(QObject *receiver, QEvent *event)
{
    // Make it possible for Qt Jambi and QSA to hook into events even
    // though QApplication is subclassed...
    bool result = false;
    void *cbdata[] = { receiver, event, &result };
    if (QInternal::activateCallbacks(QInternal::EventNotifyCallback, cbdata)) {
        return result;
    }

    // Qt enforces the rule that events can only be sent to objects in
    // the current thread, so receiver->d_func()->threadData is
    // equivalent to QThreadData::current(), just without the function
    // call overhead.
    QObjectPrivate *d = receiver->d_func();
    QThreadData *threadData = d->threadData;
    ++threadData->loopLevel;

#ifdef QT_JAMBI_BUILD
    int deleteWatch = 0;
    int *oldDeleteWatch = QObjectPrivate::setDeleteWatch(d, &deleteWatch);

    bool inEvent = d->inEventHandler;
    d->inEventHandler = true;
#endif

    bool returnValue;
    QT_TRY {
        returnValue = notify(receiver, event);
    } QT_CATCH (...) {
        --threadData->loopLevel;
        QT_RETHROW;
    }

#ifdef QT_JAMBI_BUILD
    // Restore the previous state if the object was not deleted..
    if (!deleteWatch) {
        d->inEventHandler = inEvent;
    }
    QObjectPrivate::resetDeleteWatch(d, oldDeleteWatch, deleteWatch);
#endif
    --threadData->loopLevel;
    return returnValue;
}
```
#### 上面函数中notify(receiver, event)起到真正派发的作用，下面我们分析notify（）
```
bool QCoreApplication::notify(QObject *receiver, QEvent *event)
{
    Q_D(QCoreApplication);
    // no events are delivered after ~QCoreApplication() has started
    if (QCoreApplicationPrivate::is_app_closing)
        return true;

    if (receiver == 0) {                        // serious error
        qWarning("QCoreApplication::notify: Unexpected null receiver");
        return true;
    }

#ifndef QT_NO_DEBUG
    d->checkReceiverThread(receiver);
#endif

    return receiver->isWidgetType() ? false : d->notify_helper(receiver, event);
}

```
####上面是QCoreApplication的notify()函数，可以看到如果是widget的话直接返回false。下面是带GUI的notify()
```
bool QApplication::notify(QObject *receiver, QEvent *e)
{
    Q_D(QApplication);
    // no events are delivered after ~QCoreApplication() has started
    if (QApplicationPrivate::is_app_closing)
        return true;

    if (receiver == 0) {                        // serious error
        qWarning("QApplication::notify: Unexpected null receiver");
        return true;
    }

#ifndef QT_NO_DEBUG
    d->checkReceiverThread(receiver);
#endif

    // capture the current mouse/keyboard state
    if(e->spontaneous()) {
        if (e->type() == QEvent::KeyPress
            || e->type() == QEvent::KeyRelease) {
            QKeyEvent *ke = static_cast<QKeyEvent*>(e);
            QApplicationPrivate::modifier_buttons = ke->modifiers();
        } else if(e->type() == QEvent::MouseButtonPress
            || e->type() == QEvent::MouseButtonRelease) {
                QMouseEvent *me = static_cast<QMouseEvent*>(e);
                QApplicationPrivate::modifier_buttons = me->modifiers();
                if(me->type() == QEvent::MouseButtonPress)
                    QApplicationPrivate::mouse_buttons |= me->button();
                else
                    QApplicationPrivate::mouse_buttons &= ~me->button();
            }
#if !defined(QT_NO_WHEELEVENT) || !defined(QT_NO_TABLETEVENT)
            else if (false
#  ifndef QT_NO_WHEELEVENT
                     || e->type() == QEvent::Wheel
#  endif
#  ifndef QT_NO_TABLETEVENT
                     || e->type() == QEvent::TabletMove
                     || e->type() == QEvent::TabletPress
                     || e->type() == QEvent::TabletRelease
#  endif
                     ) {
            QInputEvent *ie = static_cast<QInputEvent*>(e);
            QApplicationPrivate::modifier_buttons = ie->modifiers();
        }
#endif // !QT_NO_WHEELEVENT || !QT_NO_TABLETEVENT
    }

#ifndef QT_NO_GESTURES
    // walk through parents and check for gestures
    if (d->gestureManager) {
        switch (e->type()) {
        case QEvent::Paint:
        case QEvent::GraphicsSceneDrop:
        case QEvent::DynamicPropertyChange:
        case QEvent::NetworkReplyUpdated:
            break;
        default:
            if (d->gestureManager->thread() == QThread::currentThread()) {
                if (receiver->isWidgetType()) {
                    if (d->gestureManager->filterEvent(static_cast<QWidget *>(receiver), e))
                        return true;
                } else {
                    // a special case for events that go to QGesture objects.
                    // We pass the object to the gesture manager and it'll figure
                    // out if it's QGesture or not.
                    if (d->gestureManager->filterEvent(receiver, e))
                        return true;
                }
            }
            break;
        }
    }
#endif // QT_NO_GESTURES

    // User input and window activation makes tooltips sleep
    switch (e->type()) {
    case QEvent::Wheel:
    case QEvent::ActivationChange:
    case QEvent::MouseButtonDblClick:
        d->toolTipFallAsleep.stop();
        // fall-through
    case QEvent::Leave:
        d->toolTipWakeUp.stop();
    default:
        break;
    }

    bool res = false;
    if (!receiver->isWidgetType()) {
        res = d->notify_helper(receiver, e);
    } else switch (e->type()) {
#if defined QT3_SUPPORT && !defined(QT_NO_SHORTCUT)
    case QEvent::Accel:
        {
            if (d->use_compat()) {
                QKeyEvent* key = static_cast<QKeyEvent*>(e);
                res = d->notify_helper(receiver, e);

                if (!res && !key->isAccepted())
                    res = d->qt_dispatchAccelEvent(static_cast<QWidget *>(receiver), key);

                // next lines are for compatibility with Qt <= 3.0.x: old
                // QAccel was listening on toplevel widgets
                if (!res && !key->isAccepted() && !static_cast<QWidget *>(receiver)->isWindow())
                    res = d->notify_helper(static_cast<QWidget *>(receiver)->window(), e);
            }
            break;
```
上面是带GUI的QApplication::notify()的部分代码，可以看到事件处理函数最终到了notify_helper()
```
bool QCoreApplicationPrivate::notify_helper(QObject *receiver, QEvent * event)
{
    // 首先发送到所有Application的EventFilters（事件过滤器）
    if (sendThroughApplicationEventFilters(receiver, event))
        return true;
     //再发送到所有Object的EventFilters（事件过滤器）
    if (sendThroughObjectEventFilters(receiver, event))
        return true;
    // 最后调用对象的event函数
    return receiver->event(event);
}
```
object在Event()里面对不同的世界进行分类，再调用不同的事件处理函数，最后调用掉了我们常用的事件函数，不同的对象event()函数处理也有区别，这里以Widget为例

```
bool QWidget::event(QEvent *event)
{
    Q_D(QWidget);

    // ignore mouse events when disabled
    if (!isEnabled()) {
        switch(event->type()) {
        case QEvent::ContextMenu:
#ifndef QT_NO_WHEELEVENT
        case QEvent::Wheel:
#endif
            return false;
        default:
            break;
        }
    }
    switch (event->type()) {
    case QEvent::MouseMove:
        mouseMoveEvent((QMouseEvent*)event);
        break;

    case QEvent::MouseButtonPress:
        // Don't reset input context here. Whether reset or not is
        // a responsibility of input method. reset() will be
        // called by mouseHandler() of input method if necessary
        // via mousePressEvent() of text widgets.
#if 0
        resetInputContext();
#endif
        mousePressEvent((QMouseEvent*)event);
        break;

    case QEvent::MouseButtonRelease:
        mouseReleaseEvent((QMouseEvent*)event);
        break;

    case QEvent::MouseButtonDblClick:
        mouseDoubleClickEvent((QMouseEvent*)event);
        break;

     case QEvent::NonClientAreaMouseButtonPress: {
        QWidget* w;
        while ((w = QApplication::activePopupWidget()) && w != this) {
            w->close();
            if (QApplication::activePopupWidget() == w) // widget does not want to disappear
                w->hide(); // hide at least
            }
        break;
        }
........

```
####至此关于PostEvent的事件的派发到结束的flow以及走完。