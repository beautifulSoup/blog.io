---
layout: post
title: Android SharedPreferences 源码分析
date: 2016-5-3
categories: blog
tags: [Development,Android]
description: 本篇博客主要分析了Android SharedPreferences的关键源码。
---

## 概览
SharedPreferences（以下使用SP简称）在Android中作为一种使用简单的数据存储形式被广泛用来存储一些不需要做数据库操作的数据，比如用户配置项等。本文将从源码入手分析其实现，并据此提出一些使用中需要注意的事项。

## 分析

### 源码入口

SP是一个interface，首先我们得找到它的具体实现。在SP的使用中，我们通过getSharedPreferences()获取SP实例如下：

```
SharedPreferences sp =getSharedPreferences(TAG, MODE);    
```

而getSharedPreferences 是Context接口中的方法。稍微对Android SDK源码有所了解的朋友都知道，Context的主要实现类——ContextImpl。在该类中，我们可以找到getSharedPreferences方法的具体实现

```
    @Override
    public SharedPreferences getSharedPreferences(String name, int mode) {
        SharedPreferencesImpl sp;
        synchronized (ContextImpl.class) {
            if (sSharedPrefs == null) {
                sSharedPrefs = new ArrayMap<String, ArrayMap<String, SharedPreferencesImpl>>();
            }

            final String packageName = getPackageName();
            ArrayMap<String, SharedPreferencesImpl> packagePrefs = sSharedPrefs.get(packageName);
            if (packagePrefs == null) {
                packagePrefs = new ArrayMap<String, SharedPreferencesImpl>();
                sSharedPrefs.put(packageName, packagePrefs);
            }

            // At least one application in the world actually passes in a null
            // name.  This happened to work because when we generated the file name
            // we would stringify it to "null.xml".  Nice.
            if (mPackageInfo.getApplicationInfo().targetSdkVersion <
                    Build.VERSION_CODES.KITKAT) {
                if (name == null) {
                    name = "null";
                }
            }

            sp = packagePrefs.get(name);
            if (sp == null) {
                File prefsFile = getSharedPrefsFile(name);
                sp = new SharedPreferencesImpl(prefsFile, mode);
                packagePrefs.put(name, sp);
                return sp;
            }
        }
        if ((mode & Context.MODE_MULTI_PROCESS) != 0 ||
            getApplicationInfo().targetSdkVersion < android.os.Build.VERSION_CODES.HONEYCOMB) {
            // If somebody else (some other process) changed the prefs
            // file behind our back, we reload it.  This has been the
            // historical (if undocumented) behavior.
            sp.startReloadIfChangedUnexpectedly();
        }
        return sp;
    }
```

从以上代码我们可以获取到以下几点信息：

1. SP的实现类为SharedPreferencesImpl。
2. 在ContextImpl中维护了一个从包名到SP字典的字典。
3. 传null作为SP的name是被允许的，它会生成一个以null.xml为名的文件。
4. 当sdk version 在3.0以下时支持“MODE\_MULTI\_PROCESS”模式，在该模式下，每次获取SP实例时，判断文件是否被改动，在被改动的情况下重新从文件加载数据，以实现多进程数据同步。但是后来发现即使如此在某些情况下还是不能保证多进程数据一致性，因此就被deprecated了，在3.0以上即使设置了MODE\_MULTI\_PROCESS也没有任何作用。

### SharedPreferencesImpl

接下来是让我们移步SharedPreferencesImple，看一下SP的具体实现。

#### 成员变量

```
//对应的数据文件
private final File mFile;
//在写操作时备份的文件，如果写操作失败，会从备份文件中恢复数据
private final File mBackupFile;
//用户设置的SP模式
private final int mMode;
//内存中缓存的SP数据
private Map<String, Object> mMap;     // guarded by 'this'
//正在做些磁盘操作的进程数
private int mDiskWritesInFlight = 0;  // guarded by 'this'
//是否已经加载完成
private boolean mLoaded = false;      // guarded by 'this'
文件上一次修改的时间戳和mStatSize 一起被用来判断文件是否被修改
private long mStatTimestamp;          // guarded by 'this'
private long mStatSize;               // guarded by 'this'

//写磁盘锁
private final Object mWritingToDiskLock = new Object();
//被用来作为mListeners字典中所有键的值，意义不明
private static final Object mContent = new Object();
//通过registerOnSharedPreferenceChangeListener注册进来的listeners，所有的listeners作为键缓存在mListeners这个map中，然而所有值都是mContent.
private final WeakHashMap<OnSharedPreferenceChangeListener, Object> mListeners =
        new WeakHashMap<OnSharedPreferenceChangeListener, Object>();
```

#### 初始化过程

以下是SPImpl的构造方法。它初始化了成员变量，并调用startLoadFromDisk()用来初始化数据。

```
SharedPreferencesImpl(File file, int mode) {
        mFile = file;
        mBackupFile = makeBackupFile(file);
        mMode = mode;
        mLoaded = false;
        mMap = null;
        startLoadFromDisk();
    }
```

在startLoadFromDisk中启用了一个线程调用了真正的读文件加载数据方法 —— loadFromDiskLocked。该方法读取文件并使用XmlUtils类对读取的数据进行解析。

```
private void startLoadFromDisk() {
        synchronized (this) {
            mLoaded = false;
        }
        new Thread("SharedPreferencesImpl-load") {
            public void run() {
                synchronized (SharedPreferencesImpl.this) {
                    loadFromDiskLocked();
                }
            }
        }.start();
    }
    
private void loadFromDiskLocked() {
        if (mLoaded) {
            return;
        }
        //在写文件出错情况下，mFile 文件会被直接delete，这时候就需要从备份文件中恢复数据。
        if (mBackupFile.exists()) {
            mFile.delete();
            mBackupFile.renameTo(mFile);
        }

        // Debugging
        if (mFile.exists() && !mFile.canRead()) {
            Log.w(TAG, "Attempt to read preferences file " + mFile + " without permission");
        }

        Map map = null;
        StructStat stat = null;
        try {
            stat = Os.stat(mFile.getPath());
            if (mFile.canRead()) {
                BufferedInputStream str = null;
                try {
                    str = new BufferedInputStream(
                            new FileInputStream(mFile), 16*1024);
                    map = XmlUtils.readMapXml(str);
                } catch (XmlPullParserException e) {
                    Log.w(TAG, "getSharedPreferences", e);
                } catch (FileNotFoundException e) {
                    Log.w(TAG, "getSharedPreferences", e);
                } catch (IOException e) {
                    Log.w(TAG, "getSharedPreferences", e);
                } finally {
                    IoUtils.closeQuietly(str);
                }
            }
        } catch (ErrnoException e) {
        }
        mLoaded = true;
        //修改成员变量
        if (map != null) {
            mMap = map;
            mStatTimestamp = stat.st_mtime;
            mStatSize = stat.st_size;
        } else {
            mMap = new HashMap<String, Object>();
        }
        notifyAll();
    }
```

#### 读取数据

对于SP的使用，我们用的最多的可能就是取数据，以取String类型数据为例，getString方法首先取得SharedPreferencesImpl.this对象锁，然后同步等待从文件加载数据完成，最后再返回数据。因此这里会有一定的延迟。如果是在UI线程执行SP的取数据操作，可能会对UI流畅度方面造成一定的影响，不过在实践中我们认为这可以忽略不计。

```
public String getString(String key, @Nullable String defValue) {
        synchronized (this) {
            awaitLoadedLocked();
            String v = (String)mMap.get(key);
            return v != null ? v : defValue;
        }
    }

private void awaitLoadedLocked() {
        if (!mLoaded) {
            // Raise an explicit StrictMode onReadFromDisk for this
            // thread, since the real read will be in a different
            // thread and otherwise ignored by StrictMode.
            BlockGuard.getThreadPolicy().onReadFromDisk();
        }
        while (!mLoaded) {
            try {
                wait();
            } catch (InterruptedException unused) {
            }
        }
    }
    
```

#### 存数据

看完读取数据，接下来再介绍下写数据。对于SP的写操作主要是通过Editor接口来完成的，SPImpl中的内部类EditorImpl是Editor的具体实现。

所有的put打头的方法只是将需要修改的键值保存到mModified这个字典中。

```
public Editor putString(String key, @Nullable String value) {
            synchronized (this) {
                mModified.put(key, value);
                return this;
            }
        }
```

所以重头戏是我们经常使用的commit和apply方法。这两个方法都是首先修改内存中缓存的mMap的值，然后将数据写到磁盘中。它们的主要区别是commit会等待写入磁盘后再返回，而apply则在调用写磁盘操作后就直接返回了，但是这时候可能磁盘中数据还没有被修改。

```
        public void apply() {
            final MemoryCommitResult mcr = commitToMemory();
            final Runnable awaitCommit = new Runnable() {
                    public void run() {
                        try {
                            mcr.writtenToDiskLatch.await();
                        } catch (InterruptedException ignored) {
                        }
                    }
                };

            QueuedWork.add(awaitCommit);

            Runnable postWriteRunnable = new Runnable() {
                    public void run() {
                        awaitCommit.run();
                        QueuedWork.remove(awaitCommit);
                    }
                };

            SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);

            // Okay to notify the listeners before it's hit disk
            // because the listeners should always get the same
            // SharedPreferences instance back, which has the
            // changes reflected in memory.
            notifyListeners(mcr);
        }

        
        public boolean commit() {
            MemoryCommitResult mcr = commitToMemory();
            SharedPreferencesImpl.this.enqueueDiskWrite(
                mcr, null /* sync write on this thread okay */);
            try {
                mcr.writtenToDiskLatch.await();
            } catch (InterruptedException e) {
                return false;
            }
            notifyListeners(mcr);
            return mcr.writeToDiskResult;
        }

```

以上两个方法都调用了commitToMemory。该方法主要根据mModified和是否被clear修改mMap的值，然后返回写磁盘需要的一些相关值。

```

 private MemoryCommitResult commitToMemory() {
            MemoryCommitResult mcr = new MemoryCommitResult();
            synchronized (SharedPreferencesImpl.this) {
                // We optimistically don't make a deep copy until
                // a memory commit comes in when we're already
                // writing to disk.
                if (mDiskWritesInFlight > 0) {
                    // We can't modify our mMap as a currently
                    // in-flight write owns it.  Clone it before
                    // modifying it.
                    // noinspection unchecked
                    mMap = new HashMap<String, Object>(mMap);
                }
                mcr.mapToWriteToDisk = mMap;
                mDiskWritesInFlight++;

                boolean hasListeners = mListeners.size() > 0;
                if (hasListeners) {
                    mcr.keysModified = new ArrayList<String>();
                    mcr.listeners =
                            new HashSet<OnSharedPreferenceChangeListener>(mListeners.keySet());
                }

                synchronized (this) {
                    if (mClear) {
                        if (!mMap.isEmpty()) {
                            mcr.changesMade = true;
                            mMap.clear();
                        }
                        mClear = false;
                    }

                    for (Map.Entry<String, Object> e : mModified.entrySet()) {
                        String k = e.getKey();
                        Object v = e.getValue();
                        // "this" is the magic value for a removal mutation. In addition,
                        // setting a value to "null" for a given key is specified to be
                        // equivalent to calling remove on that key.
                        if (v == this || v == null) {
                            if (!mMap.containsKey(k)) {
                                continue;
                            }
                            mMap.remove(k);
                        } else {
                            if (mMap.containsKey(k)) {
                                Object existingValue = mMap.get(k);
                                if (existingValue != null && existingValue.equals(v)) {
                                    continue;
                                }
                            }
                            mMap.put(k, v);
                        }

                        mcr.changesMade = true;
                        if (hasListeners) {
                            mcr.keysModified.add(k);
                        }
                    }

                    mModified.clear();
                }
            }
            return mcr;
        }	
```

#### 写磁盘

最后一块比较重要的就是SP的写磁盘操作。之前介绍的apply和commit都调用了enqueueDiskWrite(）方法。

以下为其具体实现。writeToDiskRunnable中调用writeToFile写文件。如果参数中的postWriteRunable为null，则该Runnable会被同步执行，而如果不为null，则会将该Runnable放入线程池中异步执行。在这里也验证了之前提到的commit和apply的区别。

 

```
private void enqueueDiskWrite(final MemoryCommitResult mcr,
                                  final Runnable postWriteRunnable) {
        final Runnable writeToDiskRunnable = new Runnable() {
                public void run() {
                    synchronized (mWritingToDiskLock) {
                        writeToFile(mcr);
                    }
                    synchronized (SharedPreferencesImpl.this) {
                        mDiskWritesInFlight--;
                    }
                    if (postWriteRunnable != null) {
                        postWriteRunnable.run();
                    }
                }
            };

        final boolean isFromSyncCommit = (postWriteRunnable == null);

        // Typical #commit() path with fewer allocations, doing a write on
        // the current thread.
        if (isFromSyncCommit) {
            boolean wasEmpty = false;
            synchronized (SharedPreferencesImpl.this) {
                wasEmpty = mDiskWritesInFlight == 1;
            }
            if (wasEmpty) {
                writeToDiskRunnable.run();
                return;
            }
        }

        QueuedWork.singleThreadExecutor().execute(writeToDiskRunnable);
    }
```

#### Mode相关

下面我们分析一下获取SP实例时的第二个参数 mode，上文我们已经提到了一个被废弃的mode——MODE\_MULTI\_PROCESS。在SPImpl中在writeToFile方法中用到了该参数。

```
ContextImpl.setFilePermissionsFromMode(mFile.getPath(), mMode, 0);
```

再看setFilePermissionsFromMode方法，根据MODE参数设置文件的读写权限，而从中我们可以看出能够造成影响的两个MODE分别为MODE\_WORLD\_READABLE和MODE\_WORLD\_WRITEABLE。而这两个值在API17之后被Deprecated，所以不建议使用，如果需要多App共享数据，建议使用ContentProvider。

```
    @SuppressWarnings("deprecation")
    static void setFilePermissionsFromMode(String name, int mode,
            int extraPermissions) {
        int perms = FileUtils.S_IRUSR|FileUtils.S_IWUSR
            |FileUtils.S_IRGRP|FileUtils.S_IWGRP
            |extraPermissions;
        if ((mode&MODE_WORLD_READABLE) != 0) {
            perms |= FileUtils.S_IROTH;
        }
        if ((mode&MODE_WORLD_WRITEABLE) != 0) {
            perms |= FileUtils.S_IWOTH;
        }
        if (DEBUG) {
            Log.i(TAG, "File " + name + ": mode=0x" + Integer.toHexString(mode)
                  + ", perms=0x" + Integer.toHexString(perms));
        }
        FileUtils.setPermissions(name, perms, -1, -1);
    }
```

## 总结

以上为对SP代码的分析，根据以上分析，提出以下几个在使用SP中需要注意的点。

1. MODE\_MULTI\_PROCESS 无法保证多进程数据一致性，在3.0以上已经没有任何作用。
2. SP从初始化到读取到数据存在一定延迟，因为需要到文件中读取数据，因此可能会对UI线程流畅度造成一定影响。
3. commit在将数据写入磁盘后才会返回，而apply则直接返回但无法保证写磁盘已经完成，只能保证内存中数据的正确性。
4. MODE\_WORLD\_READABLE和MODE\_WORLD\_WRITEABLE已经被废弃，不建议使用，如果需要多App共享数据，建议使用ContentProvider，Github上有一个对ContentProvider的封装Tray，用起来和SP差不多还是蛮不错的。