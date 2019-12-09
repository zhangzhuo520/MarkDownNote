#QT的延迟处理DefferredMethod
-------------
看了klayout源码的一个关于延迟处理的通用模板的方法，记录下了
 
![](https://i.imgur.com/hyB2VMY.png)



#源码如下
###defferred_excution.h
```
class DefferredMethodScheduler;
class DefferredMethodBase
{
public:
    explicit DefferredMethodBase(bool compressed) :
        m_compressed(compressed),
        m_scheduled(false)
    {

    }

    virtual ~DefferredMethodBase();
    virtual void execute() = 0;
    friend class DefferredMethodScheduler;
private:
    bool m_compressed;
    bool m_scheduled;
};

class DefferredMethodScheduler: public QObject
{
public:
    explicit DefferredMethodScheduler();

    static DefferredMethodScheduler *instance();

    void schedule(DefferredMethodBase *);

    void unqueue(DefferredMethodBase *);

    bool is_disabled();

    void do_enabel(bool);

    static void enable(bool);

    static void execute();

protected:
    virtual void queue_event() = 0;

    virtual void do_execute();

private:
    int m_disable_count;
    bool m_schedule;
    QMutex m_mutex;
    static DefferredMethodScheduler *s_instance;
    std::list <DefferredMethodBase *> m_method_list;
};

template <class T>
class DefferredMethod : public DefferredMethodBase
{
public:
    DefferredMethod(T *t, void (T::*method)(), bool compressed = true):
        DefferredMethodBase(compressed),
        m_pt(t),
        m_method(method)
    {

    }

    ~DefferredMethod()
    {
        if(DefferredMethodScheduler::instance())
        {
            DefferredMethodScheduler::instance()->unqueue(this);
        }
    }

    void cancel()
    {
        if(DefferredMethodScheduler::instance())
        {
            DefferredMethodScheduler::instance()->unqueue(this);
        }
    }

    virtual void execute()
    {
        cancel();
        (m_pt->*m_method)();
    }

    void operator () ()
    {
        if(DefferredMethodScheduler::instance())
        {
            DefferredMethodScheduler::instance()->schedule(this);
        }
        else
        {
            execute();
        }
    }


private:
    T *m_pt;
    void (T::*m_method)();
};
```
###defferred_excution.cc
```
DefferredMethodScheduler * DefferredMethodScheduler::s_instance = nullptr;
DefferredMethodScheduler::DefferredMethodScheduler():
    m_disable_count(0),
    m_schedule(false)
{
}

DefferredMethodScheduler *DefferredMethodScheduler::instance()
{
    if(!s_instance)
    {
        return new DefferredMethodScheduleQt();
    }
    return s_instance;
}

void DefferredMethodScheduler::schedule(DefferredMethodBase * method)
{
    MutexLoker loker(&m_mutex);
    if(method->m_compressed || method->m_scheduled)
    {
        m_method_list.push_back(method);
        if(!m_schedule)
        {
            queue_event();
            m_schedule = true;
        }
        method->m_scheduled = true;
    }
}

void DefferredMethodScheduler::unqueue(DefferredMethodBase * method)
{
    for(std::list <DefferredMethodBase *>::iterator itor = m_method_list.begin(); itor != m_method_list.end();)
    {
        std::list <DefferredMethodBase *>::iterator mm = itor;
        ++ mm;
        if(*itor == method)
        {
            (*itor)->m_scheduled = false;
            m_method_list.erase(itor);
        }
        itor = mm;
    }
}

bool DefferredMethodScheduler::is_disabled()
{
    return m_disable_count;
}

void DefferredMethodScheduler::do_enabel(bool enable)
{
   MutexLoker loker(&m_mutex);
    if(enable)
    {
        if(m_disable_count == 0) return;
        m_disable_count --;
    }
    else
    {
        m_disable_count ++;
    }
}

void DefferredMethodScheduler::do_execute()
{
    std::list <DefferredMethodBase *> methods;
    m_mutex.lock();
    methods.swap(m_method_list);
    m_schedule = false;
    m_mutex.unlock();

    for(std::list <DefferredMethodBase *>::iterator itor = methods.begin(); itor != methods.end(); itor ++)
    {
        (*itor)->m_scheduled = false;
        (*itor)->execute();
    }
}

void DefferredMethodScheduler::enable(bool enbale)
{
    if(instance())
    {
        instance()->do_enabel(enbale);
    }
}

void DefferredMethodScheduler::execute()
{
    if(instance())
    {
        instance()->do_execute();
    }
}

DefferredMethodBase::~DefferredMethodBase()
{
}

```
###DefferredMethodScheduleQt.h
```
class DefferredMethodScheduleQt : public DefferredMethodScheduler
{
    Q_OBJECT
public:
    DefferredMethodScheduleQt();
    ~DefferredMethodScheduleQt();

    virtual bool event(QEvent *);

public slots:
    void slotTimer();

protected:
    virtual void queue_event();

private:
    QTimer *m_timer;
};
```
###DefferredMethodScheduleQt.cc
```
DefferredMethodScheduleQt::DefferredMethodScheduleQt()
{
    m_timer = new QTimer;
    m_timer->setInterval(0);
    m_timer->setSingleShot(true);
    connect(m_timer, SIGNAL(timeout()), this, SLOT(slotTimer()));
}

DefferredMethodScheduleQt::~DefferredMethodScheduleQt()
{
    //not yet...
}

bool DefferredMethodScheduleQt::event(QEvent *event)
{
    if(event->type() == QEvent::User)
    {
        slotTimer();
        return true;
    }
    else
    {
        return QObject::event(event);
    }

}

void DefferredMethodScheduleQt::slotTimer()
{
    if(is_disabled())
    {
        m_timer->start();
    }
    else
    {
        try{
            do_execute();
        }
        catch(...)
        {}
    }
}

void DefferredMethodScheduleQt::queue_event()
{
    qApp->postEvent(this, new QEvent(QEvent::User));
}
```
###实际使用
DefferredMethod<Navitor> m_test_method;
m_test_method(&Navitor, &Navitor::method)