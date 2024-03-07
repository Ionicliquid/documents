``` cpp
void ProcessState::startThreadPool()  
{  
AutoMutex _l(mLock);  
if (!mThreadPoolStarted) {  
if (mMaxThreads == 0) {  
ALOGW("Extra binder thread started, but 0 threads requested. Do not use "  
"*startThreadPool when zero threads are requested.");  
}  
  
mThreadPoolStarted = true;  
spawnPooledThread(true);  
}  
}

void ProcessState::spawnPooledThread(bool isMain)  
{  
if (mThreadPoolStarted) {  
String8 name = makeBinderThreadName();  
ALOGV("Spawning new pooled thread, name=%s\n", name.string());  
sp<Thread> t = sp<PoolThread>::make(isMain);  
t->run(name.string());  
}  
}
```