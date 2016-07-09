---
title: 【Android】访问远程数据的步骤(Messenger、AIDL、ContentProvider
date: 2016-07-09 22:11:09
categories: Lan's tech
tags:
  - Android
---
**阅读书籍：《Android开发艺术探索》**
**作者：任玉刚**
**本文为阅读其中IPC相关章节所做的简单总结，相关示例代码来自于书中。**
# IPC相关
## Messenger
1.远程service 创建Messenger对象: mMessenger，并通过onBind方法提供IBinder对象; 
 
```java
private static class MessengerHandler extends Handler {
    @Override
    public void handleMessage(Message msg) {
        super.handleMessage(msg);
    }
}
private final Messenger mMessenger = new Messenger(new MessengerHandler());
@Nullable
@Override
public IBinder onBind(Intent intent) {
    return mMessenger.getBinder();
}
```
2.client 绑定远程 service，通过ServiceConnection对象获得IBinder对象， 再通过该IBinder对象获得Messenger对象1:cm1;

```java
private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            mService = new Messenger(service);
            Message msg = Message.obtain();
            msg.what = 99;
            Bundle data = new Bundle();
            data.putString("msg", "hello, this is client");
            msg.setData(data);
            msg.replyTo = mGetReplyMessenger;
            try {
                mService.send(msg);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
    };
```

3.同时要用Handler创建一个新的Messenger对象，cm2，用来处理远程service返回的消息。用msg.replyTo = cm2 进行设置。之后通过cm1.send(msg)向远程service发送消息。远程service中的sm1接受消息并且处理。
3.sm1中Handler对象处理消息时， 可以通过msg.replyTo获取要回复的Messenger对象。使用msg.replyTo.send(msg), 即可在客户端Messenger对象cm2 的handler中进行消息的处理。

## AIDL
1. 创建 Bean， 实现Parcelable接口；
2. 创建 Manager AIDL文件， 如果用到上述 Bean， 需要建立 Bean.AIDL， 并申明那个类为parcelable

```aidl
package cn.lankton.aidl;
parcelable Book;
```
Manager 的 aidl文件如下:

```aidl
// IBookManager.aidl
package cn.lankton.aidl;

import cn.lankton.aidl.Book;

// Declare any non-default types here with import statements

interface IBookManager {
    List<Book> getBookList();
    void addBook(in Book book);
}

```
**提供了供client调用的方法**  
之后编译运行一次， IDE自动生成Manager对应的同名java文件(根据此例为 IBookManager.java)。Eclipse位于gen目录下，Android Studio位于build/generated目录下
3. 创建远程service。内部创建一个继承IBookManager.Stub的Binder对象， 实现其中的方法， 并通过onBind方法， 返回该Binder对象。

```java
public class BookManagerService extends Service {

    private CopyOnWriteArrayList<Book> mBookList = new CopyOnWriteArrayList<>();

    private Binder mBinder = new IBookManager.Stub() {

        @Override
        public List<Book> getBookList() throws RemoteException {
            return mBookList;
        }

        @Override
        public void addBook(Book book) throws RemoteException {
            mBookList.add(book);
        }
    };

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }

    @Override
    public void onCreate() {
        super.onCreate();
        mBookList.add(new Book(1, "Android"));
        mBookList.add(new Book(2, "iOS"));
    }
}
```
4. client 创建ServiceConnection， 并进行bindService，通过onServiceConnected获得IBinder对象，并通过asInterface方法获得远程service中的IBookManager对象，即可对远程service的数据进行操作。

```java
public class MainActivity extends AppCompatActivity {

    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            IBookManager bookManager = IBookManager.Stub.asInterface(service);
            try {
                List<Book> list = bookManager.getBookList();
                Log.i("AIDLDemo", "query book list, list type: " + list.getClass().getCanonicalName());
                Log.i("AIDLDemo", "query book list: " + list.toString());
                Book book = new Book(3, "swift");
                bookManager.addBook(book);
                List<Book> newList = bookManager.getBookList();
                Log.i("AIDLDemo", "after add book, list: " + newList.toString());
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Intent intent = new Intent(this, BookManagerService.class);
        bindService(intent, mConnection, BIND_AUTO_CREATE);
    }
}
```
## ContentProvider
一个提供跨进程访问数据的解决方案。  
1. 建立远端ContentProvider， 并在manifest中申明authority和name;  
```xml
<provider
            android:authorities="cn.lankton.contentproviderdemo.book.provider"
            android:name=".BookProvider"
            android:process=":provider"/>
```
其中， authorities最为重要， 其是本地访问远端ContentProvider的唯一表识。  
2. 实现ContentProvider中的各类方法：oncreate、 insert、 delete、 update、 query、 getType等， 暴露对数据增删改查的能力;  
3. 在ContentProvider中， 可以通过UriMatcher， 将Uri子路径和int型绑定, 方便处理不同子路径的Uri。  
 
``` java
private static final UriMatcher sUriMatcher = new UriMatcher(UriMatcher.NO_MATCH);
static {
    sUriMatcher.addURI(AUTHORITY, "book", BOOK_URI_CODE);
    sUriMatcher.addURI(AUTHORITY, "user", USER_URI_CODE);
}
private String getTableName(Uri uri) {
    String tableName = null;
    switch (sUriMatcher.match(uri)) {
        case BOOK_URI_CODE:
            tableName = DBOpenHelper.BOOK_TABLE_NAME;
            break;
        case USER_URI_CODE:
            tableName = DBOpenHelper.USER_TABLE_NAME;
            break;
        default:
            break;
    }
    return tableName;
}
```

接下来， 在本地访问远程数据  
1. 通过远程ContentProvider的authorities拼接Uri;  
2. 通过getContentResolver获得ContentResolver对象
3. 通过该ContentResolover对象和上面的Uri对象，对远程的数据增删改查。（实际可以理解为调用远程ContentProvider的各种相应方法  


```java
Uri bookUri = Uri.parse("content://cn.lankton.contentproviderdemo.book.provider/book");
ContentValues values = new ContentValues();
values.put("_id", 6);
values.put("name", "程序的现代艺术");
getContentResolver().insert(bookUri, values);
Cursor bookCursor = getContentResolver().query(bookUri, new String[]{"_id", "name"}, null, null, null);
while (bookCursor.moveToNext()) {
    Book book  = new Book();
    book.bookId = bookCursor.getInt(bookCursor.getColumnIndex("_id"));
    book.bookName = bookCursor.getString(bookCursor.getColumnIndex("name"));
    Log.d(TAG, "query book:" + book.bookId + "," + book.bookName);
}
bookCursor.close();
```

需要注意：
ContentProvider的query, update, insert, delete四大方法存在多线程并发访问的。用同一个SQLiteDataBase对象可以实现同步， 但如果远程数据是List等情况，必须自己想办法保证同步。