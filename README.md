# BroadcastReceiver
브로드캐스트 리시버란 해당 유저에게 SNS메세지 알림이나 뱃더리 알림 등을 알려 주는 컴포넌트입니다.

크게 두가지로 나뉘는데 하나는 코드레벨에서 등록되는 동적 리시버이고 또다른 하나는 매니패스트레 등록하는 리시버가 있습니다.

동적리시버의 특징으로는 원하는 시간과 상황에 따라 등록하고 헤제할 수 있으며 앱이 실행되니 않으면 리시버를 등록할 수 없습니다.

정적리시버는 앱이 실행되지 않아도 모든 이벤트를 받을 수 있습니다.

***
### :wrench: 프로젝트 사용 사례
<img src = "https://user-images.githubusercontent.com/48902047/132104240-092107dd-a019-4e3d-bdff-6793f8d59db5.jpg" width="20%" height="20%"> <img src = "https://user-images.githubusercontent.com/48902047/132104255-0b7492dc-5fe1-4618-a8bc-68744b01de76.jpg" width="20%" height="20%">

+ [조치원 수호대](https://github.com/tnvnfdla1214/homemade_guardian) 
  + 게시물 댓글이 달리면 알림
  + 채팅이 올시 알림
  + 나에대한 리뷰가 작성시 알림
***
### :lollipop: 설명

저희 앱은 notification의 형태인 포그라운드 서비스를 이용하여 앱이 실행되는 상황에서만 알림이 올 수 있게 동적 리시버러 설계되었습니다.

해당 메세지 알림은 파이어베이스 기능중 FCM이란 기능이 있는데 플랫폼 환경별(GCM)으로 개발 해야 하는데 FCM은 교차 플랫폼 메시지 솔루션이기떄문에 플랫폼에 종속되지 않는 장점 이있습니다.

또 본연의 서버의 기능을 수행하면서 또하나의 복잡한 알림기능까지 포함한다면 서버의 처리량이 많아 느려질 것입니다. 따라서 알림의 기능은 다른 서버가 해주고 본서버는 알림기능을 제공하는 서버에 알림이 있는지만 요청을 하는 구조입니다.

<img src = "https://user-images.githubusercontent.com/48902047/132104539-45ca4bc9-9103-4fed-8a50-a00f4459ff69.png">

1. 디바이스에 앱이 설치된후 최초 실행되면서 고유 식별자인 디바이스 토큰이 발급된다. 이 토큰을 앱 서버에 등록한다.

2. 앱 서버에서 FCM 연결 서버로 푸시 알림을 요청한다. 이때 준비물은 디바이스 토큰과 API 서버 키이다.

3. FCM 연결 서버는 토큰을 대상으로 알림 메시지를 푸시한다.


#### 라이브러리 추가

```Java
/*build.gradle*/

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])

    implementation 'androidx.appcompat:appcompat:1.1.0' //
    implementation 'androidx.constraintlayout:constraintlayout:1.1.3' //
    implementation 'com.google.firebase:firebase-core:17.0.0'  //
    implementation 'com.google.firebase:firebase-analytics:17.4.4'

    implementation 'com.google.code.gson:gson:2.8.5' //
    implementation 'com.squareup.okhttp3:okhttp:3.10.0' //

    implementation 'com.google.firebase:firebase-messaging'
    implementation 'com.google.firebase:firebase-analytics'


}

```

#### 메니패스트 추가

사용할 서비스를 등록해준다.

```Java
/*AndroidMenifest.xml*/

        <service android:name="bias.zochiwon_suhodae.homemade_guardian_beta.Main.common.MyFirebaseMessagingService"
            android:enabled="true"
            android:exported="true"
            android:stopWithTask="false">
            <intent-filter>
                <action android:name="com.google.firebase.MESSAGING_EVENT" />
                <action android:name="com.google.firebase.INSTANCE_ID_EVENT" />
            </intent-filter>

        </service>

```

#### MyFirebaseMessagingService 추가

onMessageReceived 에서 메세지를 수신하여 sendNotification 메소드에 보내줌으로써 노티피케이션을 발생시킨다.

```Java
/*MyFirebaseMessagingService.java*/

class MyFirebaseMessagingService : FirebaseMessagingService() {
    private val TAG = "FirebaseTest"

    // 메세지가 수신되면 호출
    override fun onMessageReceived(remoteMessage: RemoteMessage) {
        if(remoteMessage.data.isNotEmpty()){
            sendNotification(remoteMessage.notification?.title,
                remoteMessage.notification?.body!!)
        }
        else{

        }
    }

    // Firebase Cloud Messaging Server 가 대기중인 메세지를 삭제 시 호출
    override fun onDeletedMessages() {
        super.onDeletedMessages()
    }

    // 메세지가 서버로 전송 성공 했을때 호출
    override fun onMessageSent(p0: String) {
        super.onMessageSent(p0)
    }

    // 메세지가 서버로 전송 실패 했을때 호출
    override fun onSendError(p0: String, p1: Exception) {
        super.onSendError(p0, p1)
    }

    // 새로운 토큰이 생성 될 때 호출
    override fun onNewToken(token: String) {
        super.onNewToken(token)
        sendRegistrationToServer(token)
    }

    private fun sendNotification(title: String?, body: String){
        val intent = Intent(this,MainActivity::class.java)
        intent.addFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP) // 액티비티 중복 생성 방지
        val pendingIntent = PendingIntent.getActivity(this, 0 , intent,
            PendingIntent.FLAG_ONE_SHOT) // 일회성

        val channelId = "channel" // 채널 아이디
        val defaultSoundUri = RingtoneManager.getDefaultUri(RingtoneManager.TYPE_NOTIFICATION) // 소리
        val notificationBuilder = NotificationCompat.Builder(this, channelId)
            .setContentTitle(title) // 제목
            .setContentText(body) // 내용
            .setAutoCancel(true)
            .setSound(defaultSoundUri)
            .setContentIntent(pendingIntent)

        val notificationManager = getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager

        // 오레오 버전 예외처리
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val channel = NotificationChannel(channelId,
                "Channel human readable title",
                NotificationManager.IMPORTANCE_DEFAULT)
            notificationManager.createNotificationChannel(channel)
        }

        notificationManager.notify(0 , notificationBuilder.build()) // 알림 생성
    }

    // 받은 토큰을 서버로 전송
    private fun sendRegistrationToServer(token: String){

    }
}

```

#### SendNotification 추가

```Java
/*SendNotification.java*/

public class SendNotification {
    public static final MediaType JSON = MediaType.parse("application/json; charset=utf-8");
    public static void sendNotification(final String regToken, final String title, final String messsage){
        new AsyncTask<Void, Void, Void>() {
            @Override
            protected Void doInBackground(Void... parms) {
                try {
                    OkHttpClient client = new OkHttpClient();
                    JSONObject json = new JSONObject();
                    JSONObject dataJson = new JSONObject();
                    dataJson.put("body", messsage);
                    dataJson.put("title", title);
                    json.put("notification", dataJson);
                    json.put("to", regToken);
                    RequestBody body = RequestBody.create(JSON, json.toString());
                    Request request = new Request.Builder()
                            .header("Authorization", "key=" + "AAAAwOrSSfQ:APA91bG5atF_mkGFr_DdlIOTvA4BjXvZJ4cyuQOlNK_AOQNzaJyfBDeTYkhP6pVKXYrHsllc2QZNJKfx5pq46I290MMQd6wHzx5pVWJzfKFuv2FYia4sW_BQicqqDBRzQ7QcpjEyK9qT")
                            .url("https://fcm.googleapis.com/fcm/send")
                            .post(body)
                            .build();
                    Response response = client.newCall(request).execute();
                    String finalResponse = response.body().string();
                }catch (Exception e){
                    Log.d("error", e+"");
                }
                return  null;
            }
        }.execute();
    }
    public static void sendCommentNotification(final String regToken, final String title, final String messsage, final String Marketmodel_Uid){
        new AsyncTask<Void, Void, Void>() {
            @Override
            protected Void doInBackground(Void... parms) {
                try {
                    OkHttpClient client = new OkHttpClient();
                    JSONObject json = new JSONObject();
                    JSONObject dataJson = new JSONObject();
                    dataJson.put("body", messsage);
                    dataJson.put("title", title);
                    dataJson.put("tag", Marketmodel_Uid);
                    json.put("notification", dataJson);
                    json.put("to", regToken);
                    json.put("tag", Marketmodel_Uid);
                    RequestBody body = RequestBody.create(JSON, json.toString());
                    Request request = new Request.Builder()
                            .header("Authorization", "key=" + "AAAAwOrSSfQ:APA91bG5atF_mkGFr_DdlIOTvA4BjXvZJ4cyuQOlNK_AOQNzaJyfBDeTYkhP6pVKXYrHsllc2QZNJKfx5pq46I290MMQd6wHzx5pVWJzfKFuv2FYia4sW_BQicqqDBRzQ7QcpjEyK9qT")
                            .url("https://fcm.googleapis.com/fcm/send")
                            .post(body)
                            .build();
                    Response response = client.newCall(request).execute();
                    String finalResponse = response.body().string();
                }catch (Exception e){
                    Log.d("error", e+"");
                }
                return  null;
            }
        }.execute();
    }
}

```
#### 클래스에 기능 추가

해당 기능에서 현재 메세지를 받을 유저의 token을 DB에서 찾아  SendNotification를 통해 받을유저의 token, 보내는 사람의 닉네임, body부분을 보내줍니다.

```Java
/*ChatFragment.java*/
    .
    .
    private void sendGson(final String MessageModel_Message) {
        final DocumentReference documentReference = FirebaseFirestore.getInstance().collection("USERS").document(currentUser_Uid);
        documentReference.get().addOnCompleteListener(new OnCompleteListener<DocumentSnapshot>() {
            @Override
            public void onComplete(@NonNull Task<DocumentSnapshot> task) {
                if (task.isSuccessful()) {
                    DocumentSnapshot document = task.getResult();
                    if (document != null) {
                        if (document.exists()) {  //데이터의 존재여부
                            final UserModel userModel = document.toObject(UserModel.class);
                            final DocumentReference documentReference = FirebaseFirestore.getInstance().collection("USERS").document(To_User_Uid);
                            documentReference.get().addOnCompleteListener(new OnCompleteListener<DocumentSnapshot>() {
                                @Override
                                public void onComplete(@NonNull Task<DocumentSnapshot> task) {
                                    if (task.isSuccessful()) {
                                        DocumentSnapshot document = task.getResult();
                                        if (document != null) {
                                            if (document.exists()) {  //데이터의 존재여부
                                                UserModel TouserModel = document.toObject(UserModel.class);
                                                //SendNotification.sendNotification(TouserModel.getUserModel_Token(), userModel.getUserModel_NickName(), MessageModel_Message);
                                                SendNotification.sendCommentNotification(TouserModel.getUserModel_Token(), userModel.getUserModel_NickName(), "메세지가 도착했습니다!",ChatRoomListModel_RoomUid);
                                            }
                                        }
                                    }
                                }
                            });
                        }
                    }
                }
            }
        });

    }
    .
    .

```
