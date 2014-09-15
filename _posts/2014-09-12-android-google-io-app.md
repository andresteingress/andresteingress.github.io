---
layout: post
title: Google I/O App Insights
categories:
- android
- java
tags: []
status: publish
type: post
published: true
---
A while ago I came across the Google I/O App in one of the latest [Android Developers blog posts](http://android-developers.blogspot.co.at/2014/07/google-io-2014-app-source-code-now.html). I thought it would be interesting to have a look at some of the internals, to also gain some insights at how Android applications are developed at Google and what third party libraries are actually used there.

The source code for the Google I/O app is available [on GitHub](https://github.com/google/iosched).

### `build.gradle`

My journey through the source code began with the `settings.gradle` file. It contains the information about the projects Gradle modules. This app consists of two modules: one for the wearable version and the other one for the Android version.

This article will not talk about the implementation of the wearable version, I will create a separate blog post for that.

I have to say that the Android version was of particular interest for me, so I decided to go on with the Android modules `build.gradle` dependencies section that holds all the external dependencies that are needed by the implementation:

```groovy
dependencies {
    wearApp project(':Wearable')

    compile 'com.google.android.gms:play-services:5+' 
    compile 'com.android.support:support-v13:20.+'
    compile 'com.android.support:support-v4:20.+'
    compile 'com.google.android.apps.dashclock:dashclock-api:+'
    compile 'com.google.code.gson:gson:2.+'
    compile('com.google.api-client:google-api-client:1.+') {
        exclude group: 'xpp3', module: 'shared'
        exclude group: 'org.apache.httpcomponents', module: 'httpclient'
        exclude group: 'junit', module: 'junit'
        exclude group: 'com.google.android', module: 'android'
    }
    compile 'com.google.api-client:google-api-client-android:1.17.+'
    compile 'com.google.apis:google-api-services-plus:+'
    compile 'com.github.japgolly.android:svg-android:2.0.6'
    compile fileTree(dir: 'libs', include: '*.jar')
    compile files('../third_party/glide/library/libs/glide-3.2.0a.jar')
    compile files('../third_party/basic-http-client/libs/basic-http-client-android-0.88.jar')

    compile('com.google.maps.android:android-maps-utils:0.3+') {
        exclude group: "com.google.android.gms"
    }

    compile 'com.google.http-client:google-http-client-gson:+'
    compile 'com.google.apis:google-api-services-drive:+'
}
```

As you can see in the code snippet above, several external dependencies have been included. 

### Dashclock API

Let's start with the first dependency that gained my attention:

```groovy
compile 'com.google.android.apps.dashclock:dashclock-api:+'
```

The [Android dash-clock project](https://code.google.com/p/dashclock/wiki/API) comes with an alternative lock screen clock widget implementation that can be used to show additional status items. Showing additional information on the lock screen is done by implementing so-called `DashClockExtension` extension descendant classes, as described in the [`DashClockExtension` documentation](http://api.dashclock.googlecode.com/git/reference/com/google/android/apps/dashclock/api/DashClockExtension.html). Although this API looked pretty interesting, I couldn't find any use for it in the Google I/O application and also removing it from the dependencies did work, so I guess it might have been planned to use it, but actually it was never implemented.

### GSON

Next up is Google's JSON library: [Gson](https://code.google.com/p/google-gson/):

```groovy
compile 'com.google.code.gson:gson:2.+'
```

The Google I/O app's main purpose is the give an overview of all the scheduled talks at Google I/O and also allow some interaction for the user to give feedback about visited sessions. Gson is used to parse JSON that comes from Google's web services and contains the entire conference data.

One particular piece of code that shows some Gson usage is the `ConferenceDataHandler`. This handler basically is responsible for parsing most of the JSON data that holds information about the scheduled conference sessions, speakers, etc. Instead of parsing the JSON content directly to an object tree, it registers "handlers" for every JSON property in a map:

```java
mHandlerForKey.put(DATA_KEY_ROOMS, mRoomsHandler = new RoomsHandler(mContext));
mHandlerForKey.put(DATA_KEY_BLOCKS, mBlocksHandler = new BlocksHandler(mContext));
mHandlerForKey.put(DATA_KEY_TAGS, mTagsHandler = new TagsHandler(mContext));
mHandlerForKey.put(DATA_KEY_SPEAKERS, mSpeakersHandler = new SpeakersHandler(mContext));
mHandlerForKey.put(DATA_KEY_SESSIONS, mSessionsHandler = new SessionsHandler(mContext));
mHandlerForKey.put(DATA_KEY_SEARCH_SUGGESTIONS, mSearchSuggestHandler = new SearchSuggestHandler(mContext));
mHandlerForKey.put(DATA_KEY_MAP, mMapPropertyHandler = new MapPropertyHandler(mContext));
mHandlerForKey.put(DATA_KEY_EXPERTS, mExpertsHandler = new ExpertsHandler(mContext));
mHandlerForKey.put(DATA_KEY_HASHTAGS, mHashtagsHandler = new HashtagsHandler(mContext));
mHandlerForKey.put(DATA_KEY_VIDEOS, mVideosHandler = new VideosHandler(mContext));
mHandlerForKey.put(DATA_KEY_PARTNERS, mPartnersHandler = new PartnersHandler(mContext));
```

With the registered handlers set up, it parses the JSON response body property by property in `processDataBody`:

```java
private void processDataBody(String dataBody) throws IOException {
    JsonReader reader = new JsonReader(new StringReader(dataBody));
    JsonParser parser = new JsonParser();
    try {
        reader.setLenient(true); // To err is human

        // the whole file is a single JSON object
        reader.beginObject();

        while (reader.hasNext()) {
            // the key is "rooms", "speakers", "tracks", etc.
            String key = reader.nextName();
            if (mHandlerForKey.containsKey(key)) {
                // pass the value to the corresponding handler
                mHandlerForKey.get(key).process(parser.parse(reader));
            } else {
                LOGW(TAG, "Skipping unknown key in conference data json: " + key);
                reader.skipValue();
            }
        }
        reader.endObject();
    } finally {
        reader.close();
    }
}
```

When we have a look at one of the handler classes, let's say at `SessionsHandler`, we will see that it not only encapsulates the code for parsing the session JSON objects, but also code for building so-called "content provider operations". The [`ContentProviderOperation`](http://developer.android.com/reference/android/content/ContentProviderOperation.html) class is a class from the Android SDK that is used to build content provider actions such as inserting, updating or deleting entities stored by a content provider. The handler classes provide methods to directly create content provider operations based on the current state of an entity. E.g. if a session is new, needs to be updated or deleted, its `makeContentProviderOperations` method from the handler class will create the appropriate operation. Let's have a look now how actually parsing JSON is done for the `SessionsHandler`:

```java
@Override
public void process(JsonElement element) {
    for (Session session : new Gson().fromJson(element, Session[].class)) {
        mSessions.put(session.id, session);
    }
}
```

The code is quite slick. It uses an array of `Session` model classes as GSON target type and GSON will create the instances and populate the available properties from the JSON values:

```java
public class Session {
    public String id;
    public String url;
    public String description;
    public String title;
    public String[] tags;
    public String startTimestamp;
    public String youtubeUrl;
    public String[] speakers;
    public String endTimestamp;
    public String hashtag;
    public String subtype;
    public String room;
    public String captionsUrl;
    public String photoUrl;
    public boolean isLivestream;
    public String mainTag;
    public String color;
    public String relatedContent;
    public int groupingOrder;

    public String getImportHashCode() {
        StringBuilder sb = new StringBuilder();
        sb.append("id").append(id == null ? "" : id)
                .append("description").append(description == null ? "" : description)
                .append("title").append(title == null ? "" : title)
                .append("url").append(url == null ? "" : url)
                .append("startTimestamp").append(startTimestamp == null ? "" : startTimestamp)
                .append("endTimestamp").append(endTimestamp == null ? "" : endTimestamp)
                .append("youtubeUrl").append(youtubeUrl == null ? "" : youtubeUrl)
                .append("subtype").append(subtype == null ? "" : subtype)
                .append("room").append(room == null ? "" : room)
                .append("hashtag").append(hashtag == null ? "" : hashtag)
                .append("isLivestream").append(isLivestream ? "true" : "false")
                .append("mainTag").append(mainTag)
                .append("captionsUrl").append(captionsUrl)
                .append("photoUrl").append(photoUrl)
                .append("relatedContent").append(relatedContent)
                .append("color").append(color)
                .append("groupingOrder").append(groupingOrder);
        for (String tag : tags) {
            sb.append("tag").append(tag);
        }
        for (String speaker : speakers) {
            sb.append("speaker").append(speaker);
        }
        return HashUtils.computeWeakHash(sb.toString());
    }

    public String makeTagsList() {
        int i;
        if (tags.length == 0) return "";
        StringBuilder sb = new StringBuilder();
        sb.append(tags[0]);
        for (i = 1; i < tags.length; i++) {
            sb.append(",").append(tags[i]);
        }
        return sb.toString();
    }

    public boolean hasTag(String tag) {
        for (String myTag : tags) {
            if (myTag.equals(tag)) {
                return true;
            }
        }
        return false;
    }
}
```

What's interesting about this class (and the other model classes) is the `getImportHashCode` method. This method is needed to find out about changes that might have been done on already processed entities and is actually a main method to be used by the data sync logic implemented by the `SyncAdapter`.

### google-api-client

Next up in our list of dependencies is the [Google APIs client library](https://code.google.com/p/google-api-java-client/) and its Android extension. Both libraries are used in conjunction with the Google Plus API from the next dependency

```groovy
compile 'com.google.apis:google-api-services-plus:+'
```

to fetch the latest announcements via the `AnnouncementsFetcher` class. Once the announcements are fetched from the Google+ profile, they are stored by the content provider `ScheduleProvider`:

```java
Plus plus = new Plus.Builder(httpTransport, jsonFactory, null)
        .setApplicationName(NetUtils.getUserAgent(mContext))
        .setGoogleClientRequestInitializer(
                new CommonGoogleClientRequestInitializer(Config.API_KEY))
        .build();

ActivityFeed activities;
try {
    activities = plus.activities().list(Config.ANNOUNCEMENTS_PLUS_ID, "public")
            .setMaxResults(100l)
            .execute();
    if (activities == null || activities.getItems() == null) {
        throw new IOException("Activities list was null.");
    }

} catch (IOException e) {
    LOGE(TAG, "Error fetching announcements", e);
    return batch;
}

// ...

StringBuilder sb = new StringBuilder();
for (Activity activity : activities.getItems()) {
    // ...

    // Insert announcement info
    batch.add(ContentProviderOperation
            .newInsert(ScheduleContract
                    .addCallerIsSyncAdapterParameter(Announcements.CONTENT_URI))
            .withValue(SyncColumns.UPDATED, System.currentTimeMillis())
            .withValue(Announcements.ANNOUNCEMENT_ID, activity.getId())
            .withValue(Announcements.ANNOUNCEMENT_DATE, activity.getUpdated().getValue())
            .withValue(Announcements.ANNOUNCEMENT_TITLE, activity.getTitle())
            .withValue(Announcements.ANNOUNCEMENT_ACTIVITY_JSON, activity.toPrettyString())
            .withValue(Announcements.ANNOUNCEMENT_URL, activity.getUrl())
            .build());
}
```

Again, the `ContentProviderOperation` builder methods are used to create the appropriate operations and return them to the class client.

### Android SVG

Next up is a very interesting dependency: the [Android SVG library](https://github.com/japgolly/svg-android):

```groovy
compile 'com.github.japgolly.android:svg-android:2.0.6'
```

The SVG Android project adds support for showing scalable vector graphic files in an Android application. In the Google I/O application it is used to show the location of different floors in the Google I/O venue.

One place to have a look at SVG processing is the `ConferenceDataHandler` implementation, again, a handler class:


```java
private void processMapOverlayFiles(Collection<Tile> collection, boolean downloadAllowed) throws IOException, SVGParseException {
    boolean shouldClearCache = false;
    ArrayList<String> usedTiles = Lists.newArrayList();
    
    for (Tile tile : collection) {
        final String filename = tile.filename;
        final String url = tile.url;

        usedTiles.add(filename);

        if (!MapUtils.hasTile(mContext, filename)) {
            shouldClearCache = true;
            
            if (MapUtils.hasTileAsset(mContext, filename)) {
                
                MapUtils.copyTileAsset(mContext, filename);

            } else if (downloadAllowed && !TextUtils.isEmpty(url)) {
                try {
                    // download the file only if downloads are allowed and url is not empty
                    File tileFile = MapUtils.getTileFile(mContext, filename);
                    BasicHttpClient httpClient = new BasicHttpClient();
                    httpClient.setRequestLogger(mQuietLogger);
                    HttpResponse httpResponse = httpClient.get(url, null);
                    FileUtils.writeFile(httpResponse.getBody(), tileFile);

                    // ensure the file is valid SVG
                    InputStream is = new FileInputStream(tileFile);
                    SVG svg = new SVGBuilder().readFromInputStream(is).build();
                    is.close();
                } catch (IOException ex) {
                    LOGE(TAG, "FAILED downloading map overlay tile "+url+
                            ": " + ex.getMessage(), ex);
                } catch (SVGParseException ex) {
                    LOGE(TAG, "FAILED parsing map overlay tile "+url+
                            ": " + ex.getMessage(), ex);
                }
            } else {
                LOGD(TAG, "Skipping download of map overlay tile" +
                        " (since downloadsAllowed=false)");
            }
        }
    }

    if (shouldClearCache) {
        MapUtils.clearDiskCache(mContext);
    }

    MapUtils.removeUnusedTiles(mContext, usedTiles);
}
```

The code looks if the SVG graphic is available in the APK's asset directory. If so, it copies the file to a custom directory. If not, it downloads the SVG and uses the svg-android library to validate if it is a valid SVG graphic.

The main place where the SVG graphics are later used is in the `MapFragment` implementation. It uses a [`TileOverlay`](http://developer.android.com/reference/com/google/android/gms/maps/model/TileOverlay.html) and registers multiple `TileProvider` implementations of type `SVGTileProvider` class. The `SVGTileProvider` uses the previously shown `SVGBuilder` in order to draw the currently shown floor onto the map.

```java
public SVGTileProvider(File file, float dpi) throws IOException {
    // ...

    SVG svg = new SVGBuilder().readFromInputStream(new FileInputStream(file)).build();
    mSvgPicture = svg.getPicture();
    
    // ...
}

// later on when drawing:

public byte[] getTileImageData(int x, int y, int zoom) {
    mStream.reset();

    Matrix matrix = new Matrix(mBaseMatrix);
    float scale = (float) (Math.pow(2, zoom) * mScale);
    matrix.postScale(scale, scale);
    matrix.postTranslate(-x * mDimension, -y * mDimension);

    mBitmap.eraseColor(Color.TRANSPARENT);
    Canvas c = new Canvas(mBitmap);
    c.setMatrix(matrix);

    // NOTE: Picture is not thread-safe.
    synchronized (mSvgPicture) {
        mSvgPicture.draw(c);
    }

    BufferedOutputStream stream = new BufferedOutputStream(mStream);
    mBitmap.compress(Bitmap.CompressFormat.PNG, 0, stream);
    try {
        stream.close();
    } catch (IOException e) {
        Log.e(TAG, "Error while closing tile byte stream.");
        e.printStackTrace();
    }
    return mStream.toByteArray();
}
```

As can be seen in the code above, the method `getTileImageData` applies some scaling and translating, but in the end it draws the `mSvgPicture` onto a newly created `Canvas` and writes it to the resulting `ByteArrayOutputStream`. In order to enhance performance on creating the tile graphics, there is the `CachedTileProvider` implementation that uses a disk LRU cache to cache results on disk.

I found it very refreshing to see an application of the svg-android library in action. Its definetly an implementation option to carry in mind for future Android apps.

### Glide

Another third party library in use is [Glide](https://github.com/bumptech/glide):

```groovy
compile files('../third_party/glide/library/libs/glide-3.2.0a.jar')
```

Glide is an image loading and caching library that comes with extensions to other commonly used libraries such as `OkHttp` and `Volley`. In the Google I/O application the `Glide` API is encapsulated in the `ImageLoader` class. 

One interesting detail in this class is the `VariableWidthImageLoader` implementation:

```java
// ...
private static final Pattern PATTERN = Pattern.compile("__w-((?:-?\\d+)+)__");
// ...

@Override
protected String getUrl(String model, int width, int height) {
    Matcher m = PATTERN.matcher(model);
    int bestBucket = 0;
    if (m.find()) {
        String[] found = m.group(1).split("-");
        for (String bucketStr : found) {
            bestBucket = Integer.parseInt(bucketStr);
            if (bestBucket >= width) {
                // the best bucket is the first immediately bigger than the requested width
                break;
            }
        }
        if (bestBucket > 0) {
            model = m.replaceFirst("w"+bestBucket);
            LOGD(TAG, "width="+width+", URL successfully replaced by "+model);
        }
    }
    return model;
}
```

The `VariableWidthImageLoader` is used by Glide in order to return a customized URL that should be used for a given width and height. The implementation above looks for an image indicator in the current URL (think of `model` as being an URL to an image) that might look like `__w-200-400-800__`. If this indicator is available it replaces it with `w<desiredWith>` to actually fetch an image with a width that is actually larger than the requested width.

We used a similar pattern in our applications for image URLs (though with a `width` request parameter), but I wasn't aware of Glide providing such a nice API to inject this behaviour.

### Basic HTTP Client

Of course, the Android [basic http client implementation] (https://code.google.com/p/basic-http-client/) must also not be missed. It is needed to execute the actual HTTP requests for example in the `RemoteConferenceDataFetcher` that fetches the JSON content from Google servers. In fact, it first fetches only a so-called manifest file and checks whether data has changed based on that manifest. A detailed explanation on the actual synchronisation of the conference data can be found at the [Android developers blog](http://android-developers.blogspot.co.at/2014/09/conference-data-sync-gcm-google-io.html).

### Conclusion

This article had a look at some places in the Google I/O Android application and showed some third party libraries in use. The application has been [open-sourced on GitHub](https://github.com/google/iosched) and is available under the Apache license. 
