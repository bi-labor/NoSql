# Bevezetés

## Célkitűzés

Az előadással egybekötött gyakorlat célja, hogy bemutassuk a `NoSQL`
adatbázisok telepítését, használatát és különféle alkalmazásokba való
integrációs lehetőségeit. A gyakorlat elvégzésével a hallgató képes lesz
egy `NoSQL` alapú webes és asztali alkalmazás összeállítására, mely
adatfogadásra és adat kiszolgálásra is alkalmas lehet adattárház és BI
rendszerek irányába.

## Előfeltételek
-------------

A gyakorlat elvégzéséhez szükséges eszközök Windows platformot és Ubuntu
virtuális gépet (pl. VirtualBox) feltételezve:

-   `REDIS` telepítése és a 6379-es port megnyitása, hogy az
    alkalmazások tudjanak kapcsolódni:

    -   [*http://redis.io/topics/quickstart*](http://redis.io/topics/quickstart)

    -   VirtualBox-&gt;Machine-&gt;Settings-&gt;Network-&gt;Port forward
        ([*http://www.howtogeek.com/122641/how-to-forward-ports-to-a-virtual-machine-and-use-it-as-a-server/*](http://www.howtogeek.com/122641/how-to-forward-ports-to-a-virtual-machine-and-use-it-as-a-server/))

![](media/image1.png){width="4.28125in" height="2.28125in"}

### REDIS

A REDIS egy rendkívül gyors egyszerű kulcs-érték pár amely a memóriát
kihasználva nagyon gyors válaszidővel is képes rendelkezni.

A REDIS cluster üzemmódot is támogat az új verzióban, így a
skálázhatóság is biztosított.

#### Telepítés

A telepítés menete a REDIS weboldalán megtalálható:
([*http://redis.io/topics/quickstart*](http://redis.io/topics/quickstart)
), a főbb parancsok tömören:

```shell
sudo apt-get update
sudo apt-get install build-essential
sudo apt-get install tcl8.5
wget http://download.redis.io/releases/redis-stable.tar.gz
tar xzf redis-stable tar.gz
cd redis-stable/
make
make test
sudo make install
ls
cd utils/
ls
sudo ./install\_server.sh
```

Sikeres telepítést követően a szolgáltatás automatikusan el is indul,
melyhez próbáljuk ki a konzolos csatlakozást:

```shell
redis-cli
```

Szolgáltatásindítás és leállítás:

```shell
sudo service redis\_6379 start
sudo service redis\_6379 stop
```

#### REDIS konzol kezelése

Próbáljuk ki a következő alap REDIS parancsokat néhány példa adaton.

REDIS-ben több adatbázisunk is lehet, melyek egy számmal vannak
reprezentálva, az alapértelmezett a 0-s adatbázis.

Válasszuk ki az alapértelmezett adatbázist (nem kötelező):

```
SELECT 0
```
Illesszünk be egy értéket:

```
SET department AUT
```
Kérdezzük le a beillesztett értéket:

```
GET department
```
REDIS-ben az adatok elsőként a memóriába kerülnek tárolásra, de a REDIS
gondoskodik a perzisztálásról/lemezre mentésről is (alapesetben
másodpercenként), így a gyors és hosszú távú adattárolás is megoldott.

Lehetőség van azonban megadott ideig tárolni csak egy adatot. A
következő paranccsal létrehozunk egy értéket, és beállítjuk, hogy csak
10 másodpercig legyen érvényes. A TTL parancs a hátralevő érvényességi
idő kiírására szolgál:

```
SET department AUT
EXPIRE department 10
TTL department
```
Amennyiben mégis szeretnénk perzisztálni, akkor a PERSIST parancsot kell
használni:
```
PERSIST department
```
Ebben az esetben a TTL már -1 értéket ad vissza.

```
SCUBSCRIBE mychannel
PSUBSCRIBE my\*
PUBLUSH mychannel “test message”
```

##### List példa:

```
LPUSH mylist a // (integer) 1
LPUSH mylist b // (integer) 2
LPUSH mylist c // (integer) 3
LRANGE mylist 0 -1 // ”a”, “b”, “c”
LLEN mylist // (integer) 3
DEL mylist // (integer) 1
```
# Asztali REDIS Chat Alkalmazás Megvalósítása


## Project létrehozása


![](media/image2.png){width="5.445859580052494in"
height="4.017531714785652in"}

Következő feladatként készítsünk egy JavaFX alapú chat alkalmazást. Az
alkalmazás használatához a felhasználóknak meg kell adni a nevüket, majd
ezt követően különböző szobákat tud létrehozni, melyekbe üzeneteket
írhatnak. Az üzenetek, az új szobák létrehozása, valamint az online
felhasználók listája az alkalmazásban automatikusan megjelennek és
frissülnek.

Első lépésként hozzunk létre WebStorm-ban egy Java projektet.

Ezután töltsük le a Redis eléréséhez a `Jedis` nevezetű Java
osztálykönyvtárat az alábbi URL-ről:

[*http://search.maven.org/remotecontent?filepath=redis/clients/jedis/2.8.1/jedis-2.8.1.jar*](http://search.maven.org/remotecontent?filepath=redis/clients/jedis/2.8.1/jedis-2.8.1.jar)

A letöltött jar fájlt másoljuk be a projektünk alá, majd importáljuk be.
Ezt a File/Project Structure/Libraries menüpont alatt tehetjük meg.

## Adatbázis kapcsolat létrehozása


Hozzunk létre egy `chat` nevű package-t, abban pedig egy `ChatDBManager`
Java osztályt, amely a Redis műveletekért lesz felelős. Az osztályt a
Singleton tervezési minta alapján készítsük el és vegyük fel a
csatlakozási paramétereket tartalmazó statikus property-ket:

```java
public class ChatDBManager {

    private static String REDIS_HOST = "string.hu";
    private static Integer REDIS_DB = 1;


    public static ChatDBManager instance = null;

    protected ChatDBManager() {

    }

    public static ChatDBManager getInstance() {
        if (instance == null) {
            instance = new ChatDBManager();
        }
        return instance;
    }
}
```
Egészítsük ki az osztályt a Redis csatlakozást kezelő függvényekkel:

**private** Jedis jedis = null;

**public** void connect() {

jedis = **new** Jedis(REDIS\_HOST);

jedis.connect();

jedis.select(REDIS\_DB);

}

**public** void disconnect() {

**if** (jedis != **null**) {

jedis.disconnect();

}

}

Hozzunk létre egy új metódust, ami a bejelentkezés után regisztrálja,
kilépés után pedig törli a felhasználót a Redis adatbázisban. A
jelenlévő felhasználók regisztrációja az online:*FELHASZNÁLÓ* kulcs
létrehozásával történik.

**private** String currentUser;

**public** void setCurrentUser(String currentUser) {

**this**.currentUser = currentUser;

}

**public** String getCurrentUser() {

**return** currentUser;

}

**public** void registerUser(String username) {

jedis.set("online:" + username, "true");

setCurrentUser(username);

}

**public** void unregisterCurrentUser() {

jedis.del("online:" + getCurrentUser());

}

Hozzunk létre egy metódust a jelenlévő felhasználók lekéréséhez:

**public** List&lt;String&gt; getOnlineUsers() {

ScanParams params = **new** ScanParams();

params.match("online:\*");

ScanResult&lt;**String**&gt; scanResult = jedis.scan("0", params);

**String** nextCursor = scanResult.getStringCursor();

boolean scanEnd = **false**;

ArrayList&lt;**String**&gt; users = **new** ArrayList&lt;&gt;();

**while** (!scanEnd) {

**for** (**String** key : scanResult.getResult()) {

users.add(key.replaceFirst("online:", ""));

}

scanResult = jedis.scan(nextCursor, params);

nextCursor = scanResult.getStringCursor();

**if** (nextCursor.equals("0")) {

scanEnd = **true**;

}

}

**return** users;

}

A metódus az *online:\** minta alapján kigyűjti az adatbázisból a
felhasználókat, majd visszatér azok listájával (levágva a kulcsokról az
*online:* előtagot).

Ezek után készítsük el az üzenetek adatbázisba mentéséhez és lekéréséhez
szükséges metódusokat:

**public** void sendMessage(String message) {

**String** msg = (**new** **Date**()).getTime() + ":" + getCurrentUser()
+ ":" + message;

jedis.rpush(getCurrentRoom() + ":messages", msg);

}

**public** List&lt;String&gt; getMessages() {

**return** jedis.lrange(getCurrentRoom() + ":messages", 0, -1);

}

Az üzenetek Redis listákban tárolódnak, melyekbe rpush (right push)
segítségével ”jobbról” szúrjuk be az új üzeneteket. A listák kulcsa
tartalmazza a szoba nevét, amely így épül fel: *SZOBA*:messages. A
szobákhoz tartozó listákban az üzenetek az alábbi módon tárolódnak:

*IDŐBÉLYEG*:*FELHASZNÁLÓ*:*ÜZENET*

Az üzeneteket lrange-el tudjuk kiolvasni a listákból, amit 0, -1
paraméterrel hívunk meg. Ez azt jelenti, hogy az összes listaelemet
lekérdezzük.

Következő lépésben hozzuk létre a szobákat kezelő metódusokat:

**public** void addRoom(String roomName) {

jedis.rpush("rooms", roomName);

}

**public** List&lt;String&gt; getRooms() {

**return** jedis.lrange("rooms", 0, -1);

}

**private** String currentRoom = "main";

**public** void setCurrentRoom(String currentRoom) {

**this**.currentRoom = currentRoom;

}

**public** String getCurrentRoom() {

**return** currentRoom;

}

**public** Map&lt;String,Long&gt; getRoomsLength(){

//TODO: implement

**return** null;

}

Az üzenetekhez hasonlóan a szobák neveit is listában (*rooms*) tároljuk.
Az aktuálisan megnyitott szoba nyilvántartását a currentRoom
property-ben tároljuk.

Felület létrehozása
-------------------

Készítsük el JavaFX alkalmazásunk belépési pontjaként szolgáló *Main*
osztályt a *main* package-en belül:

**public** **class** Main **extends** Application {

@Override

**public** void start(Stage primaryStage) **throws** **Exception**{

Parent root = FXMLLoader.load(getClass().getResource("login.fxml"));

primaryStage.setTitle("NoSQL labor");

primaryStage.setScene(**new** Scene(root, 900, 600));

primaryStage.show();

}

@Override

**public** void stop(){

ChatDBManager.getInstance().unregisterCurrentUser();

ChatDBManager.getInstance().disconnect();

}

**public** **static** void main(**String**\[\] args) {

launch(args);

}

}

Ez az osztály két dologért felelős. Létrehozza a Scene -ünket a
*login.fxml* alapján, valamint az ablak bezárása esetén a kilépteti a
felhasználót és lezárja a kapcsolatot a Redis-el.

Készítsük el FXML-ben a bejelentkező képernyő felületét. Ehhez a *main*
package-en belül hozzuk létre a *login.fxml* nevű fájlt:

&lt;?**xml** version="1.0" encoding="UTF-8"?&gt;

&lt;?import javafx.scene.control.Button?&gt;

&lt;?import javafx.scene.control.Label?&gt;

&lt;?import javafx.scene.control.TextField?&gt;

&lt;?import javafx.scene.layout.GridPane?&gt;

&lt;GridPane alignment="center" hgap="10" vgap="10"
xmlns:fx="http:**//**javafx.com/fxml/1"

xmlns="http:**//**javafx.com/javafx/8"
fx:controller="main.MainController"&gt;

&lt;children&gt;

&lt;Label text="Felhasználói név:"/&gt;

&lt;TextField fx:id="username" GridPane.columnIndex="1"/&gt;

&lt;Button fx:id="loginBtn" text="Belépés" onAction="\#btnSubmit"
GridPane.columnIndex="2"/&gt;

&lt;/children&gt;

&lt;/GridPane&gt;

Hozzuk létre az FXML-ben hivatkozott *MainController* Java osztályt. Ez
az osztály felelős a felhasználói bejelentkezésért.

**public** **class** MainController **implements** Initializable {

@FXML

**private** **TextField** username;

@Override

**public** void initialize(**URL** location, **ResourceBundle**
resources) {

ChatDBManager.getInstance().connect();

}

**public** void btnSubmit(**ActionEvent** actionEvent) **throws**
**IOException** {

**if** (!"".equals(username.getText())) {

ChatDBManager.getInstance().registerUser(username.getText());

Parent parent =
FXMLLoader.load(getClass().getResource("/chat/chat.fxml"));

Scene scene = **new** Scene(parent);

Stage stage = (Stage) ((Node)
actionEvent.getSource()).getScene().getWindow();

stage.setScene(scene);

stage.show();

}

}

}

A bejelentkezés gombra kattintáskor az FXML fájlban megadott btnSubmit
metódus fog lefutni, melyben beregisztráljuk a felhasználót Redis-ben, a
fentebb létrehozott registerUser metódus meghívásával. Ezután átváltunk
egy új *Scene* -re, melynek kinézetét a *chat.fxml* -ben definiáljuk.

Hozzuk létre a *chat.fxml* fájlt a *chat* package-en belül:

&lt;?**xml** version="1.0" encoding="UTF-8"?&gt;

&lt;?import javafx.scene.control.\*?&gt;

&lt;?import javafx.scene.layout.AnchorPane?&gt;

&lt;?import javafx.scene.layout.VBox?&gt;

&lt;?import javafx.scene.text.Font?&gt;

&lt;AnchorPane maxHeight="-Infinity" maxWidth="-Infinity"
minHeight="-Infinity" minWidth="-Infinity" prefHeight="665.0"

prefWidth="900.0" xmlns="http:**//**javafx.com/javafx/8"
xmlns:fx="http:**//**javafx.com/fxml/1"

fx:controller="chat.ChatController"&gt;

&lt;children&gt;

&lt;Label layoutX="715.0" layoutY="25.0" text="Online Felhasználók"&gt;

&lt;font&gt;

&lt;Font size="17.0"/&gt;

&lt;/font&gt;

&lt;/Label&gt;

&lt;VBox fx:id="online" layoutX="719.0" layoutY="49.0"
prefHeight="481.0" prefWidth="163.0"/&gt;

&lt;ScrollPane fx:id="messageScroll" layoutX="9.0" layoutY="43.0"
prefHeight="470.0" prefWidth="693.0"&gt;

&lt;content&gt;

&lt;VBox fx:id="messages" prefHeight="468.0" prefWidth="668.0"/&gt;

&lt;/content&gt;

&lt;/ScrollPane&gt;

&lt;TabPane fx:id="roomsTabPane" layoutX="9.0" layoutY="10.0"
prefHeight="481.0" prefWidth="673.0"&gt;

&lt;tabs&gt;

&lt;Tab id="main" closable="false" text="Fő szoba"/&gt;

&lt;Tab id="\_newroom" closable="false" style="-fx-background-color:
\#85C899;" text="+ Új szoba"&gt;

&lt;content&gt;

&lt;AnchorPane minHeight="0.0" minWidth="0.0" prefHeight="503.0"
prefWidth="485.0"&gt;

&lt;children&gt;

&lt;TextField fx:id="roomName" layoutX="261.0" layoutY="242.0"/&gt;

&lt;Label layoutX="170.0" layoutY="247.0" text="Szoba neve:"/&gt;

&lt;Button layoutX="449.0" layoutY="242.0"

onAction="\#addRoomAction" text="Létrehoz"/&gt;

&lt;/children&gt;

&lt;/AnchorPane&gt;

&lt;/content&gt;

&lt;/Tab&gt;

&lt;/tabs&gt;

&lt;/TabPane&gt;

&lt;TextArea fx:id="message" layoutX="9.0" layoutY="534.0"
prefHeight="66.0" prefWidth="578.0"/&gt;

&lt;Button fx:id="sendBtn" layoutX="596.0" layoutY="534.0"
onAction="\#sendAction"

prefHeight="66.0" prefWidth="105.0" text="Küldés"/&gt;

&lt;/children&gt;

&lt;/AnchorPane&gt;

Hozzuk létre az fxml fájlhoz tartozó kontroller osztályt
*ChatController* néven és @FXML annotációval hozzunk létre hivatkozást
az egyes nézet elemekre.

**public** **class** ChatController **implements** Initializable {

@FXML

**private** VBox online;

@FXML

**private** **TextArea** message;

@FXML

**private** **ScrollPane** messageScroll;

@FXML

**private** **TextField** roomName;

@FXML

**private** TabPane roomsTabPane;

@FXML

**private** VBox messages;

@Override

**public** void initialize(**URL** location, **ResourceBundle**
resources) {

loadOnlineUsers();

loadRooms();

loadMessages();

roomsTabPane.getSelectionModel().selectedItemProperty().addListener(**new**
ChangeListener&lt;Tab&gt;() {

@Override

**public** void changed(ObservableValue&lt;? **extends** Tab&gt;
observableValue, Tab tab, Tab <span id="__DdeLink__3474_1624206728"
class="anchor"></span>changedTab) {

ChatDBManager.getInstance().setCurrentRoom(changedTab.getId());

loadMessages();

}

});

}

}

A nézet betöltődése után az *initialize* metódus fog automatikusan
meghívódni. Ebben inicializáljuk az egyes felületi elemek tartalmát
(jelenlévő felhasználók listája, szobák listája, üzenetek). A szobák
listáját Tab-ok formájában jelenítjük meg. Ha a felhasználó rákattint az
egyik tab-ra, akkor átállítjuk az aktuális szobát, majd újratöltjük az
üzenetek listáját, hogy az aktuális szobához tartozóak jelenjenek meg.
Ehhez beregisztrálunk egy *listener* függvényt, ahol ezeket a
műveleteket lekezeljük.

Készítsük el az initialize metódusban hivatkozott további inicializáló
függvényeket:

**private** void loadOnlineUsers() {

online.getChildren().clear();

**List**&lt;**String**&gt; onlineUsers =
ChatDBManager.getInstance().getOnlineUsers();

**for** (**String** onlineUser : onlineUsers) {

**Label** user = **new** **Label**(onlineUser);

**if** (onlineUser.equals(ChatDBManager.getInstance().getCurrentUser()))
{

user.setFont(**Font**.font("Verdana", FontWeight.BOLD, 12));

}

online.getChildren().add(user);

}

}

**private** void addMessage(String msg) {

**String**\[\] msgParts = msg.split(":", 3);

**Label** date = **new** **Label**(**new**
**Timestamp**(**Long**.parseLong(msgParts\[0\])).toString());

date.setId("date");

date.setPrefWidth(200);

**Label** user = **new** **Label**("@" + msgParts\[1\]);

user.setPrefWidth(100);

HBox hBox = **new** HBox();

hBox.getChildren().add(date);

hBox.getChildren().add(user);

hBox.getChildren().add(**new** Text(msgParts\[2\]));

hBox.setPrefHeight(20);

hBox.setSpacing(10);

hBox.setPadding(**new** **Insets**(5, 5, 5, 5));

hBox.setStyle("-fx-background-color: \#DFDFDF; -fx-border-color:
transparent transparent gray transparent");

messages.getChildren().add(hBox);

messageScroll.setVvalue(1.0);

}

**private** void addRoom(String room) {

Tab tab = **new** Tab(room);

tab.setId(room);

VBox messagesVBox = **new** VBox();

messagesVBox.setId("messages");

tab.setContent(messagesVBox);

roomsTabPane.getTabs().add(tab);

}

**private** void loadRooms() {

**List**&lt;**String**&gt; rooms =
ChatDBManager.getInstance().getRooms();

**for** (**String** room : rooms) {

addRoom(room);

}

}

**private** void loadMessages() {

messages.getChildren().clear();

**List**&lt;**String**&gt; messages =
ChatDBManager.getInstance().getMessages();

**for** (**String** message : messages) {

addMessage(message);

}

}

Következő lépésben hozzuk létre az üzenet küldés és a szoba létrehozás
gombhoz tartozó akció metódusokat:

**public** void sendAction(ActionEvent actionEvent) {

**if** (!"".equals(message.getText())) {

ChatDBManager.getInstance().sendMessage(message.getText());

message.setText("");

}

}

**public** void addRoomAction(ActionEvent actionEvent) {

**if** (!"".equals(roomName.getText())) {

ChatDBManager.getInstance().addRoom(roomName.getText());

ChatDBManager.getInstance().setCurrentRoom(roomName.getText());

roomName.setText("");

}

}

Publish-Subscribe megvalósítása
-------------------------------

Alkalmazásunk ezek után már működőképes, csak nem szinkronizálja más
felhasználók tevékenységeit. Nem jelennek meg automatikusan az
alkalmazás indítása óta küldött új üzenetek, bejelentkezett új
felhasználók és a létrehozott új szobák (csak újraindítás után).
Következő lépésekben ezt fogjuk implementálni a Redis Publish/Subscribe
módszer alkalmazásával. Ennek lényege, hogy különböző csatornákra
üzenetet küldünk, melyeket minden feliratkozott kliens megkap.

Először is egészítsük ki *ChatDBManager* osztályunkat, hogy kezelni
tudja a csatornákra való feliratkozást és az azokon érkező üzenetek
fogadását:

**public** void subscribe(String channel, JedisPubSub listener) {

**Thread** thread = **new** **Thread**(**new** JedisTask(listener,
channel));

thread.setDaemon(**true**);

thread.start();

}

**private** **static** **class** JedisTask **implements** Runnable {

Jedis jedis = **null**;

JedisPubSub channelListener;

**String** channel;

JedisTask(JedisPubSub listener, **String** channel) {

**this**.channelListener = listener;

**this**.channel = channel;

**this**.jedis = **new** Jedis(REDIS\_HOST);

jedis.connect();

jedis.select(REDIS\_DB);

}

@Override

**public** void run() {

**try** {

jedis.psubscribe(channelListener, channel);

**while** (**true**) {

**Thread**.sleep(200);

}

} **catch**(**InterruptedException** e) {

e.printStackTrace();

}

}

}

Csatornákra feliratkozni későbbiekben a létrehozott *subscribe*
metóduson keresztül tudunk, ami egy külön szálon felcsatlakozik a
Redis-re a JedisTask subclass segítségével, majd a csatornákon érkező
üzeneteket egy átadott listener segítségével tudjuk lekezelni.

Ezek után egészítsük ki az említett műveleteket, hogy egy csatornán
jelezzék a feliratozott klienseknek.

Először a felhasználók regisztrációiról küldjünk a *users*:*reload*
csatornán egy üzenetet, annak érdekében, hogy jelezzük, hogy a
kliensalkalmazásokban újra kell tölteni az online felhasználók listáját.
Ehhez a *registerUser* és az *uregisterCurrentUser* metódosunkba vegyünk
fel egy publish metódushívást (a változtatás vastagon van jelölve):

**public** void registerUser(String username) {

jedis.set("online:" + username, "true");

setCurrentUser(username);

**jedis.publish("users:reload", "");**

}

**public** void unregisterCurrentUser() {

jedis.del("online:" + getCurrentUser());

**jedis.publish("users:reload", "");**

}

A csatornán küldött üzenet tartalma üres string, mivel azt jelen esetben
nem kezeljük le (bármi lehet).

Ezután egészítsük ki publish hívással a *sendMessage* és *addRoom*
metódusokat is.

**public** void sendMessage(String message) {

**String** msg = (**new** **Date**()).getTime() + ":" + getCurrentUser()
+ ":" + message;

jedis.rpush(getCurrentRoom() + ":messages", msg);

**jedis.publish("room:" + getCurrentRoom() + ":message", msg);**

}

**public** void addRoom(String roomName) {

jedis.rpush("rooms", roomName);

**jedis.publish("rooms:new", roomName);**

}

Ebben a két esetben a publish üzenet tartalma már nem üres string, hanem
a küldött chat üzenet, valamint a szoba neve.

Végül nem maradt más hátra, mint ezekre a csatornákra való feliratkozás.
Térjünk vissza a *ChatController* osztályunkhoz, majd egészítsük ki az
initialize() függvényünket egy subscribeToRedisChannels() hívással:

@Override

**public** void initialize(URL location, ResourceBundle resources) {

loadOnlineUsers();

loadRooms();

loadMessages();

roomsTabPane.getSelectionModel().selectedItemProperty().addListener(**new**
ChangeListener&lt;Tab&gt;() {

@Override

**public** void changed(ObservableValue&lt;? **extends** Tab&gt;
observableValue, Tab tab, Tab t1) {

ChatDBManager.getInstance().setCurrentRoom(t1.getId());

loadMessages();

}

});

**subscribeToRedisChannels();**

}

A *subscribeToRedisChannels* tartalma pedig a következő lesz:

**private** void subscribeToRedisChannels() {

ChatDBManager.getInstance().subscribe("users:reload", **new**
JedisPubSub() {

@Override

**public** void onPMessage(**String** pattern, **String** channel,
**String** room) {

Platform.runLater(**new** Runnable() {

@Override

**public** void run() {

loadOnlineUsers();

}

});

}

});

ChatDBManager.getInstance().subscribe("room:\*:message", **new**
JedisPubSub() {

@Override

**public** void onPMessage(**String** pattern, **String** channel,
**String** message) {

Platform.runLater(**new** Runnable() {

@Override

**public** void run() {

**String** roomName = channel.split(":")\[1\];

**if** (roomName.equals(ChatDBManager.getInstance().getCurrentRoom())) {

addMessage(message);

messageScroll.setVvalue(1.0);

}

}

});

}

});

ChatDBManager.getInstance().subscribe("rooms:new", **new** JedisPubSub()
{

@Override

**public** void onPMessage(**String** pattern, **String** channel,
**String** room) {

Platform.runLater(**new** Runnable() {

@Override

**public** void run() {

addRoom(room);

**if** (room.equals(ChatDBManager.getInstance().getCurrentRoom())) {

roomsTabPane.getSelectionModel().selectLast();

}

}

});

}

});

}

Első esetben a users:reload csatornára iratkozunk fel, ahol ha üzenet
érkezik akkor a loadOnlineUsers metódusunk meghívásával újratöltjük az
online felhasználók listáját. A Platform.runLater hívásra azért van
szükség, mert a csatornákon az üzenetek külön szálon érkeznek és ennek
segítségével tudunk a JavaFX szálunkon módosítani a megjelenített
elemeken.

A következő feliratkozás a room:\*:message mintára illeszkedő
csatornákra történik. Itt a \* segítségével illesztünk a különböző szoba
nevekre, amit később visszanyerünk a csatorna nevéből a ”:” elválasztó
mentén felbontva azt. Ezután ellenőrizzük, hogy az aktuálisan érkezett
chat üzenet ugyanabban a szobában jött-e, mint ahol jelenleg vagyunk. Ha
igen, akkor hozzáadjuk azt az üzenetet listához (a teljes újratöltés
nélkül).

Végül feliratkozunk a *rooms*:*new* csatornára is, ahol lekezeljük az
újonnan keletkezett szobák felvételét is.

Önálló feladatok
================

Szoba nevek kiemelése
---------------------

Valósítsa meg, hogy ha egy olyan szobába jön üzenet, ami jelenleg nem
aktív, akkor a szoba neve vastagon legyen jelölve.

**Figyelem:** a fő szobára is működjön megfelelően.

**Tipp**: ne szövegre, hanem id-ra vizsgáld a szobanevet.

Egészítse ki az előző feladatot úgy, hogy ha rákattintunk egy vastagon
jelölt szoba nevére, akkor az váltson vissza nem vastagra

**Figyelem:** az új szoba tab háttere maradjon zöld!

Grafikon kirajzolás szobák statisztikájáról:
--------------------------------------------

![](media/image3.png){width="6.222929790026247in"
height="4.808412073490814in"}

Valósítson meg egy új képernyőt, amely egy kördiagrammon ábrázolja az
egyes szobákba érkezett üzeneteket számát. Képernyőn jelenjen meg egy
jelmagyarázat és egy vissza gomb is, mellyel az előző nézetre lehet
visszanavigálni. Valamint a szobanevek után zárójelben jelenjen meg az
üzenetek száma.

Lépések:

1.  Új gomb felvétel a képernyőn

![](media/image4.png){width="1.5753204286964129in"
height="1.1328149606299212in"}

1.  ChatDBManager *getRoomsLength()* függvény implementálása

2.  Grafikon kirajzoló felület (fxml) és kontroller létrehozása

    a.  PieChart példa:
        [*http://docs.oracle.com/javafx/2/charts/pie-chart.htm\#CIHFDADD*](http://docs.oracle.com/javafx/2/charts/pie-chart.htm#CIHFDADD)

3.  Vissza gomb létrehozása

Bónusz feladat:
---------------

Grafikon folyamatosan frissüljön új üzenet érkezésekor.

Visszajelzés:
-------------

[*http://goo.gl/forms/Uqetd4oLgp*](http://goo.gl/forms/Uqetd4oLgp)
