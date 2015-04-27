#Android: AIDL Service
#Android AIDL 服务
`Android` `AIDL` `Service`

##引子
应用中心需要对其他应用提供一个服务，以根据包名查询某一个应用的当前最新版本号。<br/>
第一次开发ADIL服务，也碰到了一些问题，写下来复习一下。<br/>
<br/>

##AidlServerDemo
###定义aidl接口
定义一个.aidl为后缀的文件，IDE会自动生成同名的Java实现类放到gen目录下。<br/>
aidl文件需要提供给客户端，所以最好放到单独的包下。<br/>
一些注意事项如下，具体细节网上各种文档都有说明，这里不详细列举。有些细节在示例代码的注释中也有说明。<br/>
1. package后面的包路径要与aidl文件所在的包路径一致。<br/>
2. 复杂类型或者其他aidl类型需要import。<br/>
3. 不允许使用访问权限修饰符。<br/>
4. 只允许定义方法。<br/>
5. 自定义的类型需要实现android.os.Parcelable接口，并且定义aidl文件。<br/>
6. Parcelable对象的属性读写顺序必须保持一致，详见createFromParcel方法和writeToParcel方法。<br/>
下面是com.example.aidlserver.aidl包下的全部代码。<br/><br/>
com.example.aidlserver.aidl.IAppVersionService.aidl文件<br/>

```
	
	package com.example.aidlserver.aidl;
	
	import com.example.aidlserver.aidl.AppVersion;
	
	interface IAppVersionService{
		/**
		*获取应用中心最新版本号
		*/
		AppVersion getLatestVersion(String appPackage);
	}
```
<br/>
com.example.aidlserver.aidl.AppVersion.aidl文件<br/><br/>

```
	
	package com.example.aidlserver.aidl;
	
	//Use parcelable NOT Parcelable
	parcelable AppVersion;
```
<br/>
com.example.aidlserver.aidl.AppVersion.java文件<br/><br/>

```Java
	
	package com.example.aidlserver.aidl;
	
	import android.os.Parcel;
	import android.os.Parcelable;
	
	
	public class AppVersion implements Parcelable {
	
		private String appPackage;
		private String versionCode;
		private String versionName;
		
		public AppVersion() {
		    super();
		}
		
		public AppVersion(String appPackage, String versionCode, String versionName) {
		    super();
		    this.appPackage = appPackage;
		    this.versionCode = versionCode;
		    this.versionName = versionName;
		}
		
		// Must implements Parcelable.Creator<T>
		// Must named as CREATOR
		public static final Parcelable.Creator<AppVersion> CREATOR = new Creator<AppVersion>() {
		
		                                                               @Override
		                                                               public AppVersion createFromParcel(Parcel src) {
		                                                                   // read orderly like writeToParcel
		                                                                   AppVersion v = new AppVersion();
		                                                                   v.setAppPackage(src.readString());
		                                                                   v.setVersionCode(src.readString());
		                                                                   v.setVersionName(src.readString());
		                                                                   return v;
		                                                               }
		
		                                                               @Override
		                                                               public AppVersion[] newArray(int size) {
		                                                                   return new AppVersion[size];
		                                                               }
		
		                                                           };
		
		@Override
		public int describeContents() {
		    return 0;
		}
		
		@Override
		public void writeToParcel(Parcel dest, int flags) {
		    // write orderly like createFromParcel
		    dest.writeString(appPackage);
		    dest.writeString(versionCode);
		    dest.writeString(versionName);
		}
		
		public String getAppPackage() {
		    return appPackage;
		}
		
		public void setAppPackage(String appPackage) {
		    this.appPackage = appPackage;
		}
		
		public String getVersionCode() {
		    return versionCode;
		}
		
		public void setVersionCode(String versionCode) {
		    this.versionCode = versionCode;
		}
		
		public String getVersionName() {
		    return versionName;
		}
		
		public void setVersionName(String versionName) {
		    this.versionName = versionName;
		}
		
	}
```
<br/>
###实现服务
定义一个服务类继承android.app.Service类，并覆盖onBind接口，该接口应该返回aidl实现类的抽象内部类Stub的子类。<br/>
由于是服务类，不需要提供给客户端，放到aidl之外包里。<br/>
```Java
	
	package com.example.aidlserver.service;
	
	import android.app.Service;
	import android.content.Intent;
	import android.os.IBinder;
	import android.os.RemoteException;
	import android.util.Log;
	
	import com.example.aidlserver.aidl.AppVersion;
	import com.example.aidlserver.aidl.IAppVersionService;
	
	public class AppVersionService extends Service {
	
	    private static final String TAG = "AppVersionService";
	
	    public class AppVersionServiceImpl extends IAppVersionService.Stub {
	
	        @Override
	        public AppVersion getLatestVersion(String appPackage) throws RemoteException {
	            Log.d(TAG, "getLatestVersion");
	
	            // Call server to get data, mock here
	            AppVersion latestVersion = new AppVersion();
	            latestVersion.setAppPackage("com.test");
	            latestVersion.setVersionCode("21");
	            latestVersion.setVersionName("2.1.0");
	
	            String text = "At server AppVersionService: LastestVersion: " + latestVersion.getAppPackage() + ";"
	                    + latestVersion.getVersionCode() + ";" + latestVersion.getVersionName();
	            Log.d(TAG, text);
	
	            return latestVersion;
	        }
	    }
	
	    @Override
	    public IBinder onBind(Intent intent) {
	        Log.d(TAG, "onBind");
	        IBinder binder = new AppVersionServiceImpl();
	        return binder;
	    }
	
	}
```
<br/>
###暴露服务
修改AndroidManifest.xml文件，添加service。<br/>
service的android:name指向服务类，内容为服务类全路径去掉应用包名的内容。例如服务类为“com.example.aidlserver.service.AppVersionService”，应用包名为“com.example.aidlserver”，那么android:name为“.service.AppVersionService”。<br/>
action的android:name是客户端绑定服务时需要的，定义一个唯一的名称即可，这里就是aidl的全名。<br/>
```XML
	
	<service android:name=".service.AppVersionService" >
	    <intent-filter>
	        <!-- action name, shared with client -->
	        <action android:name="com.example.aidlserver.aidl.IAppVersionService" />
	    </intent-filter>
	</service>
```
##AidlClientDemo
###拷贝aidl
将server提供的aidl文件拷贝过来，不能修改包路径。<br/>
本例中拷贝IAppVersionService.aidl、AppVersion.aidl、AppVersion.java。
###服务调用
1. 定义ServiceConnection，实现onServiceConnected（会被bindService动作触发）和onServiceDisconnected调用（会被unbindService动作触发）。<br/>
2. 使用bindService绑定服务。<br/>
3. 使用aidl声明的接口调用服务。<br/>
4. 使用unbindService解除绑定。<br/>
下面为代码示例，一个包含3个按钮的界面，分别提供绑定，调用和解绑定功能。<br/><br/>
```Java
	
	package com.example.aidlclient;
	
	import android.app.Activity;
	import android.content.ComponentName;
	import android.content.Context;
	import android.content.Intent;
	import android.content.ServiceConnection;
	import android.os.Bundle;
	import android.os.IBinder;
	import android.os.RemoteException;
	import android.util.Log;
	import android.view.Menu;
	import android.view.MenuItem;
	import android.view.View;
	import android.view.View.OnClickListener;
	import android.widget.Button;
	import android.widget.Toast;
	
	import com.example.aidlserver.aidl.AppVersion;
	import com.example.aidlserver.aidl.IAppVersionService;
	
	public class MainActivity extends Activity implements OnClickListener {
	
	    // action name difined in server's AndroidManifest.xml
	    private static final String ACTION_NAME       = "com.example.aidlserver.aidl.IAppVersionService";
	
	    private static final String TAG               = "MainActivity";
	
	    // Service instance
	    private IAppVersionService  appVersionService = null;
	
	    private Button              bindBtn;
	    private Button              callBtn;
	    private Button              unbindBtn;
	
	    private ServiceConnection   connection        = new ServiceConnection() {
	
	                                                      @Override
	                                                      public void onServiceDisconnected(ComponentName arg0) {
	                                                          Log.d(TAG, "onServiceDisconnected");
	                                                          appVersionService = null;
	                                                      }
	
	                                                      @Override
	                                                      public void onServiceConnected(ComponentName componentName,
	                                                              IBinder binder) {
	                                                          Log.d(TAG, "onServiceConnected");
	                                                          appVersionService = IAppVersionService.Stub
	                                                                  .asInterface(binder);
	
	                                                          callBtn.setEnabled(true);
	                                                          unbindBtn.setEnabled(true);
	                                                      }
	                                                  };
	
	    @Override
	    protected void onCreate(Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	        setContentView(R.layout.activity_main);
	
	        bindBtn = (Button) findViewById(R.id.bindBtn);
	        bindBtn.setOnClickListener(this);
	        callBtn = (Button) findViewById(R.id.callBtn);
	        callBtn.setOnClickListener(this);
	        unbindBtn = (Button) findViewById(R.id.unbindBtn);
	        unbindBtn.setOnClickListener(this);
	
	        bindBtn.setEnabled(true);
	        callBtn.setEnabled(false);
	        unbindBtn.setEnabled(false);
	    }
	
	    @Override
	    public void onClick(View v) {
	        switch (v.getId()) {
	        case R.id.bindBtn: {
	            Log.d(TAG, "bind service");
	            Intent intent = new Intent(ACTION_NAME);
	            // bindService will trigger onServiceConnected
	            boolean bindResult = bindService(intent, connection, Context.BIND_AUTO_CREATE);
	            Log.d(TAG, "bindResult: " + bindResult);
	            bindBtn.setEnabled(false);
	            callBtn.setEnabled(true);
	            unbindBtn.setEnabled(true);
	            break;
	        }
	        case R.id.callBtn: {
	            Log.d(TAG, "call service");
	            AppVersion latestVersion = null;
	            try {
	                // call service
	                latestVersion = appVersionService.getLatestVersion("com.test");
	            }
	            catch (RemoteException e) {
	                Log.e(TAG, "call service", e);
	                e.printStackTrace();
	            }
	            if (latestVersion != null) {
	                String text = "At client AppVersionService called: LastestVersion: " + latestVersion.getAppPackage()
	                        + ";" + latestVersion.getVersionCode() + ";" + latestVersion.getVersionName();
	                Toast.makeText(getApplicationContext(), text, Toast.LENGTH_LONG).show();
	                Log.d(TAG, text);
	            }
	            break;
	        }
	        case R.id.unbindBtn: {
	            Log.d(TAG, "unbind service");
	            unbindService(connection);
	            bindBtn.setEnabled(true);
	            callBtn.setEnabled(false);
	            unbindBtn.setEnabled(false);
	            break;
	        }
	        }
	    }
	
	    @Override
	    public boolean onCreateOptionsMenu(Menu menu) {
	        // Inflate the menu; this adds items to the action bar if it is present.
	        getMenuInflater().inflate(R.menu.main, menu);
	        return true;
	    }
	
	    @Override
	    public boolean onOptionsItemSelected(MenuItem item) {
	        // Handle action bar item clicks here. The action bar will
	        // automatically handle clicks on the Home/Up button, so long
	        // as you specify a parent activity in AndroidManifest.xml.
	        int id = item.getItemId();
	        if (id == R.id.action_settings) {
	            return true;
	        }
	        return super.onOptionsItemSelected(item);
	    }
	
	}
```
<br/>
##示例工程下载
Adt编译运行通过。<br/>
[AidlServerDemo.zip](https://github.com/duanjfeng/trickle/blob/master/attaches/AidlServerDemo.zip)<br/>
[AidlClientDemo.zip](https://github.com/duanjfeng/trickle/blob/master/attaches/AidlClientDemo.zip)<br/>

##相关链接<br/>
[Android AIDL应用间交互](http://www.cnblogs.com/trinea/archive/2012/11/08/2701390.html)<br/>
[android开发AIDL实例](http://blog.csdn.net/wangkuifeng0118/article/details/7277680)<br/>
[Android aidl文档翻译](http://elsila.blog.163.com/blog/static/1731971582012285264906/)<br/>
[android项目中bindService失败的原因](http://blog.csdn.net/hbzh2008/article/details/7641627)<br/>

<br/>
2015-4-27