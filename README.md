# IfinitySDK for Android

## Dependencies

Android version 4.4 or higher


## Installation

The easiest way to use our SDK is to import module to your IDE and add it to your dependencies


# Quick Start

## Authentication

Authentication is a must have for using the IfinitySDK. Without it, SDK won’t work or it will keep throwing exceptions. Every next step in this tutorial requires authorization. Every launch of your app will need to begin with the process showed below.

```java
// obtain those parameters at the geos.zone website
private static final String sClientId = "XXXX";
private static final String sClientSecret = "YYYY";

private void authenticate() {
	// get the instance of DataManager
	DataManager dataManager = DataManager.getInstance();
	// pass your client ID and client secret, obtained at the geos.zone 
	dataManager.authenticateWithClientID(sClientId, sClientSecret,
			new OnAuthenticateUserFinished() {
 
		@Override
		public void onOAuthSuccess() { 
			// handle success authentication
		}

		@Override
		public void onOAuthFailure(String message) {
			// handle failed authentication
			Log.e("onOAuthFailure", message);
		}
	}); 
}
```


## Loading venue data

After authenticating with our API, you can load venues from geos.zone. You can do this by using single instance of `DataManager`, passing your current or needed location and creating a callback to determine whether your data is ready to fetch from database.

```java
public void loadData() {
	// pass Context, LatLng and OnQuerySuccess objects
	DataManager.getInstance().loadDataForLocationWithSuccess(getBaseContext(),
			mCurrentLocation, new OnQuerySuccess() {

		@Override
		public void onQuerySuccess() {
			// handle your succesful data load
		}

		@Override
		public void onQueryError(String message, int code) {
			// handle your failed loading
		}

	});
}
```


## Accessing local database

After determining that your data was successfully loaded, you can easily access it from your local database storage. To achieve that you need to call the data access methods of the single `DataManager` instance.

```java
// fetch areas for specific floor
List<IFArea> areaFromFloorList = DataManager.getInstance().fetchAreasFromCacheForFloorId(getBaseContext(), floorId);

// fetch areas for specific venue
List<IFArea> areaFromVenueList = DataManager.getInstance().fetchAreasFromCacheForVenueId(getBaseContext(), venueId);

// fetch all venues from database
List<IFVenue> venueList = DataManager.getInstance().fetchVenuesFromCache(getBaseContext());

// fetch beacons for specific floor
List<IFBeacon> beaconList = DataManager.getInstance().fetchBeaconsForFloor(getBaseContext(), floorId);

// get content for specific area
IFContent areaContent = DataManager.getInstance().getContentForAreaId(areaId, getBaseContext());

// get content for specific time period
IFTimePush timeContent = DataManager.getInstance().getContentForTime(now, venueId, getBaseContext());

// get content for specific venue
IFContent venueContent = DataManager.getInstance().getContentForVenue(venueId, getBaseContext());
```


## Entering the venue

For detection of venues in physical world you need to access the single instance of the `IfinityBluetoothManager` class. This class will provide you with the information about nearby beacons state, venues you’re in, and the floor you’re on.

## Detecting many-beacons venue floor

```java
private int mVenueId = 0;

private void registerForVenueEnter(IfinityBluetoothManager bluetoothManager) {
	bluetoothManager.startManager(getBaseContext());
	bluetoothManager.addOnFloorChangedListener(mOnFloorChangedListener);
}

private OnFloorChangedListener mOnFloorChangedListener = new OnFloorChangedListener() {

	@Override
	public void floorChanged(IFVenue venue, IFFloorPlan floorplan, String tileUrl) {

		if(mVenueId == 0){
			// handle your venue entering here
			mVenueId =  venue.getVenueId();
		}

		if(mVenueId != venue.getVenueId()){
			// handle your venue change here
		}

		// handle your floor change here 
		setFloorplan(floorplan);
	}
};
```


## Detecting one-beacon venue

If you want to detect one-beacon venues too, you also need to register listener for it. But for doing so, you need to get the list of known venues from the local database and then pass it to the `addOnVenueContentListener` along with your listener.

```java
private void registerForOneBeaconVenue() {
	List<IFVenue> venueList = DataManager.getInstance().fetchVenuesFromCache(getBaseContext());

	IfinityBluetoothManager.getInstance().addOnVenueContentListener(
			new OnVenueContentListener() {

		@Override
		public void venueContent(IFVenue venue, IFContent content, IFTimePush tp) {
			// handle your venue content here, also your timed notifications
		}

	}, venueList);
}
```


## Detecting indoor location

IfinitySDK can provide you with a specific data about the user location inside a venue: his/her position and area he/she is inside. To obtain this information, you need to get a single instance of the `IndoorLocationManager` and register the listeners for the events you want to handle.
To use the `IndoorLocationManager`, the `IfinityBluetoothManager` must be started first.

```java
private void initIndoor() {
	IfinityBluetoothManager bluetoothManager = IfinityBluetoothManager.getInstance();
	bluetoothManager.startManager(getBaseContext());

	IndoorLocationManager indoorLocationManager = IndoorLocationManager.getInstance();
	indoorLocationManager.startManager(getBaseContext());

	indoorLocationManager.addOnEnterAreaListener(mAreaListener);
	indoorLocationManager.addOnUserPositionChanged(mOnUserPositionChangedListener);
}

private OnEnterAreaListener mAreaListener = new OnEnterAreaListener() {

	@Override
	public void areaLeft(IFArea area, IFContent content) {
		// handle leaving area
	}

	@Override
	public void areaEntered(IFArea area, IFContent content) {
		// handle entering area
	}

};

private OnUserPositionChangedListener mOnUserPositionChangedListener =
		new OnUserPositionChangedListener() {

	@Override
	public void positionChanged(final LatLng position, double estimatedError) {
		// handle user position update
	}

};
```


## Push Manager

IfinitySDK allows usage of background push notification in your app easily. To do that, you need to extend `PushManager` class from our SDK, which is an Android Service class.

```java
public class BackgroundContentService extends PushManager {

	@Override
	public void onCreate() {
		super.onCreate();
	}

	@Override
	public void onDestroy() {
		super.onDestroy();
	}

	public void startListnening(Context context){
		super.startScanning(context);
	}

	public void stopListening(){
		super.stopScanning();
	}

	@Override
	protected void notificationFound(IFContent c) {
		// keep the super method invokation to prevent old contents from appearing
		super.notificationFound(c);
		buildNotification(c);
	}

	private void buildNotification(IFContent c) {
		Intent activityIntent = new Intent(getBaseContext(), MainActivity.class).putExtra("content_id", c.getContentId());
		PendingIntent intent = PendingIntent.getActivity(getBaseContext(), c.getContentId(), activityIntent, 0);

		NotificationCompat.Builder builder =
				new NotificationCompat.Builder(this)
				.setSmallIcon(R.drawable.ic_launcher)
				.setContentTitle(getResources().getString(R.string.app_name))
				.setContentIntent(intent)
				.setAutoCancel(true)
				.setContentText(c.getName());
		Notification n = builder.build();
		NotificationManager notificationManager = (NotificationManager) getSystemService(NOTIFICATION_SERVICE);

		notificationManager.notify(c.getContentId(), n);
	}
}
```

To use this class, you need to declare it inside your `AndroidManifest.xml` file within `<application></application>` tags.

```xml
<service android:name="your.package.name.BackgroundContentService" />
```

Then, just start it from your Activity.

```java
@Override
public onPause() {
	super.onPause();
	startService(new Intent(YourActivity.this, BackgroundContentService.class));
}

@Override
public onResume() {
	// stop it when app is resumed
	super.onResume();
	stopService(new Intent(YourActivity.this, BackgroundContentService.class));
}
```

This class can also run in a foreground for handling all sources of content (area, venue, timed pushes, remote pushes). To achieve that just start your service in the `onCreate()` method and stop it within the `onDestroy()` call.


## Displaying assigned content 

Displaying the content from the `IFContent` class is easy, you just need to build an url address and pass it to your WebView component.

```java
private void loadWebContent(IFContent content, WebView webView){
	String url = IfinitySDK.getDomain()
			+ "api/content/html?access_token="
			+ DataManager.getInstance().getAccessToken() + "&id="
			+ content.getContentId();
			mWvContent.loadUrl(data, map);
	webView.loadUrl(url);
}
```

IfinitySDK has many types of supported notifications, but as a developer, you should be only worried about the IFContent class. All other types are decorated to be provided to you as IFContent objects.

## Sample project

Project described above will be available shortly on our github account
