# appframework-java

Appframework-java is a java Software Development Kit which provide java style api for lark bot development. 
It's also a lightweight framework for lark open platform callback event processing, 
developers need to only register event handlers to deal with callback event without caring about the data format or safety verification.

# 1. Quick start

### 1.1 Dependency

```xml
<dependency>
    <artifactId>appframework-sdk</artifactId>
    <groupId>com.larksuite.appframework</groupId>
    <version>${version}</version>
</dependency>
```


### 1.2 Configuration
Every lark app has a appId and appSecret. AppId is a unique identity from a app, but it's a random string generated by lark open platform which not friendly to remember or human identification.
Developer could name the apps and interact with frame work api using the short name.

Firstly, wo prepare app configuration in advance to build a AppConfiguration object.

```java

AppConfiguration ac = new AppConfiguration();
ac.setAppShortName("myAppName");    // app name, will be used to identify a app, should be unique
ac.setAppId(appId);
ac.setAppSecret(appSecret);
ac.setEncryptKey(encryptKey);
ac.setVerificationToken(verificationToken);
ac.setIsIsv(Boolean.parseBoolean(isIsv));
```

We provide a easier way to build AppConfiguration object from properties using AppConfiguration.loadFromProperties. 

```java
AppConfiguration conf = AppConfiguration.loadFromProperties(String appShortName, Properties properties);
```

Server configuration properties example, 
```properties

# larksuite.appframework.${appShortName}.appId
larksuite.appframework.test-app1-isv.appId=cli_xxxxxxxxxx
larksuite.appframework.test-app1-isv.appSecret=xxxxxxxxxxxxxxxx
larksuite.appframework.test-app1-isv.encryptKey=xxxxxxxxxxxx
larksuite.appframework.test-app1-isv.verificationToken=xxxxxxxxxxxx
larksuite.appframework.test-app1-isv.isIsv=true
```

### 1.3 Create LarkAppInstance

LarkAppInstance is a app instance, all the operations of a app are provided by a LarkAppInstance. We can think of it as the bootstrap of the framework.
Usually, we create a LarkAppInstance with a LarkAppInstanceFactory as follow.

```java
AppConfiguration appConfiguration = AppConfiguration.loadFromProperties("test-app1-isv", properties); //test-app1-isv is a app short name

LarkAppInstanceFactory.AppEventListener myTestAppListener1 = LarkAppInstanceFactory
        .createAppEventCallbackListener()
        .onMessageEvent(new App1MessageEventHandler())
        .onBotInvitedEvent(new App1AddRobotEventHandler());

LarkAppInstance ins = LarkAppInstanceFactory
       .builder(appConfiguration)
       .appTicketStorage(new RedisAppTicketStorage())
       .registerAppEventCallbackListener(myTestAppListener1)
       .create();
```

AppTicketStorage is a storage interface should be implemented by developer which used to persist app ticket. 
App ticket is a necessary token for ISV app when fetching app access token. 
If you are building a ISV app, you should implement the storage with Redis or Mysql.
For internal app, just ignoring the configuration item is ok.


### 1.4 Register your event handlers to AppEventListener

BotAppsServerFactory.AppEventListener is a registry for event handlers for a specific app which provide almost all lark event
handling registration, and all those event types that you didn't listen on will be ignored silently.


### 1.5 Receive events

The following is a example servlet for lark open platform event callback. App framework don't provide the ability to serve as a web server,
you can run the code on any kind of web server such as tomcat / jetty / resin. 
Just receive the post request data, and invoke receiveLarkNotify method of LarkAppInstance with passing event data of string as parameters,
the invocation will return your response data back to lark open platform. Then send back the response data through your http server synchronously.

```java
public class App1EventServlet extends HttpServlet {

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {

        String requestString = com.google.common.io.CharStreams.toString(new InputStreamReader(req.getInputStream(), StandardCharsets.UTF_8));

        System.out.println("LarkNotifyApp1Servlet requestString: " + requestString);

        String respData = Global.botApp1.receiveLarkNotify(requestString);

        resp.setStatus(HttpStatus.OK_200);
        resp.getWriter().println(respData);
    }
}

```

### 1.6 Use lark client to send request

LarkClient encapsulated all the functions that request lark server proactively. 
We can get a LarkClient instance by calling the getLarkClient method of a LarkAppInstance, 
or a InstanceContext which always be a parameter in event handlers.
The following is simple example of sending text message to a user who has the user id "123456".

```java
Message msg = new TextMessage("Hello");
larkAppInstance.getLarkClient().sendChatMessage(MessageDestinations.UserId("123456"), msg);
```

# 2. Function details
### 2.1 Supported Events

As so far, developers can register handler to deal with event type as follow.

| Event type | Event class |
| :---: | :---: |
| app_open | AppEnabledEvent |
| approval | ApprovalEvent |
| app_status_change | AppStatusChangeEvent |
| app_ticket | AppTicketEvent |
| add_bot | BotInvitedEvent|
| remove_bot| BotRemovedEvent |
| user_add | ContactsUpdatesEvent|
| leave_approval| LeaveApprovalEvent|
| message|MessageEvent |
| order_paid| OrderPaidEvent|
| work_approval| OvertimeApprovalEvent|
| p2p_chat_create| P2pChatCreateEvent|
| remedy_approval|RemedyApprovalEvent |
| shift_approval | ShiftApprovalEvent|
| trip_approval | TripApprovalEvent|


### 2.2 Lark client functions

| Function name | Remark |
| :---: | :---: |
| sendChatMessageIsv | for ISV app  |
| sendChatMessage | for internal app |
| batchSendChatMessageIsv | for ISV app |
| batchSendChatMessage | for internal app |
| uploadImageIsv | for ISV app |
| uploadImage | for internal app |
| fetchGroupListIsv | for ISV app |
| fetchGroupList | for internal app |
| fetchGroupInfoIsv | for ISV app |
| fetchGroupInfo | for internal app |

### 2.3 Message types

#### 2.3.1 Text message

TextMessage

#### 2.3.2 Image message

ImageMessage

#### 2.3.3 Share group message

ShareGroupMessage

#### 2.3.4 Post message

PostMessage

Usage example
```java
PostBuilder.Post post = PostBuilder.newPost();
PostBuilder.Language zhCn = post.createZhCnLanguage("I'm the title");

zhCn.createLine()
    .createTextTag("Line one", false) // unEscape: false
    .creatATag("go to github", "https://github.com");

zhCn.createLine()
    .createTextTag("Line two", false)
    .creatAtTag("123456")   // @Someone useId: 123456
    .createImgTag("xxxxxx", 300, 400); // append a image of key: xxxxxx, width:300, height: 400

Message msg = new PostMessage(post.toContent());
```

#### 2.3.5 Card message

CardMessage

A card message could be complicated, the code segment below is a simple example for building one.

```java
Card card = new Card(new Config(true), new Header(new Text(Text.Mode.PLAIN_TEXT, "I am the title"))); // config and header are necessary

Button button = new Button("TestCard.btn", new Text(Text.Mode.PLAIN_TEXT, "submit btn"))
                        .setValue(ImmutableMap.of("k1", "v1"));

Action action = new Action(Lists.newArrayList(button));

card.setModules(Lists.newArrayList(
                    div,
                    new Hr(),
                    action
));

Message msg = new CardMessage(card.toObjectForJson());
```

# 3. SpringBoot support

Spring boot is a very common used framework for Java application development. 
For more easier integrated into project Constructed with SpringBoot framework,
we provide a springboot start tool which may reduce duplicated work when building a lark app.

### 3.1 Maven Dependency

```xml
<dependency>
    <artifactId>appframework-spring-boot-starter</artifactId>
    <groupId>com.larksuite.appframework</groupId>
    <version>${version}</version>
</dependency>
```

### 3.2 Configuration

For supporting multiple app in one project, the starter can only read yaml configuration, the whole profile would be like this

```yaml
spring:
  application:
    name: example

server:
  port: 7070

larksuite:
  appframework:
    feishu: true
    notify:
      basePath: /notify
    apps[0]:
      appShortName: app1Name
      appId: cli_xxx
      appSecret: xxxxxxxxxxxx
      encryptKey: xxxxxxxxxxxx
      verificationToken: xxxxxxxxxxxx
      isIsv: false
    apps[1]:
      appShortName: app2Name
      appId: cli_xxx
      appSecret: xxxxxxxxxxxx
      encryptKey:
      verificationToken:
      isIsv: false
```

#### 3.2.1 larksuite.appframework.feishu
If we set larksuite.appframework.feishu to true, all requests to lark open platform will be under the domain "open.feishu.cn", otherwise "open.larksuite.com" as default.

#### 3.2.2 larksuite.appframework.notify.basePath
If larksuite.appframework.notify.basePath configured, starter will create a http servlet mapped to the path. 
The app instances will receive notify from open platform, through the configured base path.
As "/notify" is set as the base path in the above example, 2 paths will be listening for each app instance: 


1. "/notify/event/${appName}" for callback event
2. "/notify/card/${appName}" for card event

We can find the paths in our project starting logs, and configure them to lark developer web console.

### 3.3 Listening for callback events

```java
@LarkEventHandlers(appName="someAppName")
public class EventHandlers {

    @Handler
    public Object onRobotAdd(AddBotEvent event, LarkClient larkClient) {

        try {
            larkClient.sendChatMessage(
                    MessageDestinations.ChatId(event.getOpenChatId()),
                    new TextMessage("Hello, I'm Echo Robot, try say to me."));
        } catch (LarkClientException e) {
            e.printStackTrace();
        }

        return null;
    }

    @Handler
    public Object onMessageEvent(MessageEvent event, InstanceContext ic) {
        return null;
    }
}
```

Adding an annotation LarkEventHandlers to a class means we define some event handlers in the class.
As shown above, "appName" is a property of "LarkEventHandlers" which meanings all methods defined in the class are for the app who has the specific name.
To make things simpler, "appName" can be ignored in single app project.
A Handler on a method indicates that this method is a handler for one kind of event, but only those methods which have exactly one parameter of event type take effective.
In most situation, we need to use LarkClient or InstanceContext when handling the event, just add a parameter of those type to the method parameter list will meet your needs.


How does the framework map a event to a handler? Just declare the event type as the first parameter of the method annotated as Handler.
If you want to use the LarkClient or InstanceContext client, just add the declaring of the type in the method parameter list.


### 3.3 Listening for card events

```java
@LarkEventHandlers(appName="someAppName")
public class CardEventHandlers {
    @Handler
    public Card onCardEvent(CardEvent event) {
        
        return null;
    }

    @CardAction(methodName = "testCard.btn1")
    public Card onActionMethod1(CardEvent cardEvent, LarkClient larkClient) {
        return null;
    }

    @CardAction(methodName = "testCard.btn2")
    public Card onActionMethod2(CardEvent cardEvent, LarkClient larkClient) {
        return null;
    }
}
```

The example above show two ways to react on card events. 
A "@CardAction" annotated method will handle with the card action with statemented method name,
and "@Handler" annotated method with CardEvent parameter will handle all the other card actions.
Obviously, the handler for card events can be omitted If you have declared methods for all card actions. 

As a necessary parameter "methodName" of "@CardAction", it's the key for routing card actions to method handlers.
We must define a method name for a action element when building a card message, it will be used to map a use action to a handler.


```java
Card testCard = new Card(new Config(true), new Header(new Text(Text.Mode.PLAIN_TEXT, "I am the title"))); // config and header are necessary

Action action = new Action(Lists.newArrayList(
    new Button("testCard.btn1", new Text(Text.Mode.PLAIN_TEXT, "button-1")),
    new Button("testCard.btn2", new Text(Text.Mode.PLAIN_TEXT, "button-2")),
    new Button("testCard.btn3", new Text(Text.Mode.PLAIN_TEXT, "button-3"))
));

card.setModules(Lists.newArrayList(action));

Message msg = new CardMessage(card.toObjectForJson());
```
As the example above, we have a card message in which there exists three buttons, we want to program for each click action of the three.
As the class "CardEventHandlers" defined above, method "onActionMethod1" will react for user action of button "testCard.btn1", 
method "onActionMethod2" will react for user action of button "testCard.btn2", 
and the method "onCardEvent" will deal with the left "testCard.btn3" button action.



### 3.4 Getting a lark client instance

In some situations, we want to use lark client instance proactively. 
So a more free way to get a lark client or app instance is needed. 
In spring context, autowiring is the common used way, as follow.

```java
@Component
public class MyBusinessService {
    
    @Autowired
    private LarkClient larkClient;

    @Autowired
    private LarkAppInstance larkAppInstance;
    
    // ...
}
```

In multiple app project, the code above would cause exception on project starting up. 
Obviously, the reason is there exists more than one candidate for autowiring.
To resolving the problem, we should declare a concrete bean name for the spring autowiring mechanism.
For example, we have two apps named "app1" and "app2". The following is the optimized code.

```java
@Component
public class MyBusinessService {
    
    @Autowired
    private LarkClient app1LarkClient;  // variable name must be: "${appName}LarkClient"

    @Resource(name="app2LarkAppInstance")  // "${appName}LarkAppInstance"
    private LarkAppInstance app2LarkAppInstance; // arbitrary variable name

    // ...
}
``` 





# 4. Examples

appframework-jetty-example

appframework-springboot-example
