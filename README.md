# Basic Redis Chat App Demo C# (.Net Core 5)
Showcases how to impliment chat app in .Net Core, SignalR and Redis. This example uses **pub/sub** feature combined with **server-side events** for implementing the message communication between client and server.

<a href="https://raw.githubusercontent.com/redis-developer/basic-redis-chat-app-demo-dotnet/main/docs/screenshot000.png?raw=true"><img src="https://raw.githubusercontent.com/redis-developer/basic-redis-chat-app-demo-dotnet/main/docs/screenshot000.png?raw=true" width="100%" width="50%"></a>
<a href="https://raw.githubusercontent.com/redis-developer/basic-redis-chat-app-demo-dotnet/main/docs/screenshot001.png?raw=true"><img src="https://raw.githubusercontent.com/redis-developer/basic-redis-chat-app-demo-dotnet/main/docs/screenshot001.png?raw=true" width="100%" width="50%"></a>


## Technical Stacks
* Frontend - *React*, *Socket* (@microsoft/signalr)
* Backend - *.Net Core 5.0*, *Redis* (Microsoft.Extensions.Caching.StackExchangeRedis)

## Database Schema
### User
```C#
public class User : BaseEntity
{
  public int Id { get; set; }
  public string Username { get; set; }
  public bool Online { get; set; } = false;
}
```
### ChatRoom
```C#
public class ChatRoom : BaseEntity
{
  public string Id { get; set; }
  public IEnumerable<string> Names { get; set; }
}
```
### ChatRoomMessage
```C#
public class ChatRoomMessage : BaseEntity
{
  public string From { get; set; }
  public int Date { get; set; }
  public string Message { get; set; }
  public string RoomId { get; set; }
}
```

## How document and each data type is stored in Redis.

### How do you store a document?
We want to store a document(like a user) in redis.
Basically, *indexable* and *sortable* fields are stored in hash while we store rest of the fields in RedisJSON. We can apply RediSearch queries once we store in hash.

Redis is used mainly as a database to keep the user/messages data and for sending messages between connected servers.

The real-time functionality is handled by **Server Sent Events** for server->client messaging. Additionally each server instance subscribes to the `MESSAGES` channel of pub/sub and dispatches messages once they arrive.

- The chat data is stored in various keys and various data types.
  - User data is stored in a hash set where each user entry contains the next values:
    - `username`: unique user name;
    - `password`: hashed password
  - Additionally a set of rooms is associated with user
  - **Rooms** are sorted sets which contains messages where score is the timestamp for each message
    - Each room has a name associated with it
  - **Online** set is global for all users is used for keeping track on which user is online.

**User** hash set is accessed by key `user:{userId}`. The data for it stored with `HSET key field data`. User id is calculated by incrementing the `total_users` key (`INCR total_users`)

**Username** is stored as a separate key (`username:{username}`) which returns the userId for quicker access and stored with `SET username:{username} {userId}`.

**Rooms** which user belongs too are stored at `user:{userId}:rooms` as a set of room ids. A room is added by `SADD user:{userId}:rooms {roomId}` command.

**Messages** are stored at `room:{roomId}` key in a sorted set (as mentioned above). They are added with `ZADD room:{roomId} {timestamp} {message}` command. Message is serialized to an app-specific JSON string.

### How the data is accessed:

**Get User** `HGETALL user:{id}`. Example: `HGETALL user:2`, where we get data for the user with id: 2.

**Online users:** `SMEMBERS online_users`. This will return ids of users which are online

**Get room ids of a user:** `SMEMBERS user:{id}:rooms`. Example: `SMEMBERS user:2:rooms`. This will return IDs of rooms for user with ID: 2

**Get list of messages** `ZREVRANGE room:{roomId} {offset_start} {offset_end}`. 
Example: `ZREVRANGE room:1:2 0 50` will return 50 messages with 0 offsets for the private room between users with IDs 1 and 2.


## How it works?

### Sign in
![How it works](docs/screenshot000.png)

### Chats
![How it works](docs/screenshot001.png)

The chat server works as a basic *REST* API which involves keeping the session and handling the user state in the chat rooms (besides the WebSocket/real-time part).

When the server starts, the initialization step occurs. At first, a new Redis connection is established and it's checked whether it's needed to load the demo data. 

## Using in .Net Core
### Startup
```C#
//In Startup -> ConfigureServices
var redisEndpointUrl = (Environment.GetEnvironmentVariable("REDIS_ENDPOINT_URL") ?? "127.0.0.1:6379").Split(':');
var redisHost = redisEndpointUrl[0];
var redisPort = redisEndpointUrl[1];

var redisPassword = Environment.GetEnvironmentVariable("REDIS_PASSWORD");
if (redisPassword != null)
{
    redisConnectionUrl = $"{redisHost}:{redisPort},password={redisPassword}";
}
else
{
    redisConnectionUrl = $"{redisHost}:{redisPort}";
}
var redis = ConnectionMultiplexer.Connect(redisConnectionUrl);
services.AddSingleton<IConnectionMultiplexer>(redis);

//In some service
var redis = serviceScope.ServiceProvider.GetService<IConnectionMultiplexer>();
var redisDatabase = redis.GetDatabase();
```




### Initialization
For simplicity, a key with **total_users** value is checked: if it does not exist, we fill the Redis database with initial data.
```EXISTS total_users``` (checks if the key exists)


The demo data initialization is handled in multiple steps:

**Creating of demo users:**
We create a new user id: `INCR total_users`. Then we set a user ID lookup key by user name: ***e.g.*** `SET username:nick user:1`. And finally, the rest of the data is written to the hash set: ***e.g.*** `HSET user:1 username "nick" password "bcrypt_hashed_password"`.

Additionally, each user is added to the default "General" room. For handling rooms for each user, we have a set that holds the room ids. Here's an example command of how to add the room: ***e.g.*** `SADD user:1:rooms "0"`.

**Populate private messages between users.**
At first, private rooms are created: if a private room needs to be established, for each user a room id: `room:1:2` is generated, where numbers correspond to the user ids in ascending order. 

***E.g.*** Create a private room between 2 users: `SADD user:1:rooms 1:2` and `SADD user:2:rooms 1:2`.

Then we add messages to this room by writing to a sorted set: 
***E.g.*** `ZADD room:1:2 1615480369 "{'from': 1, 'date': 1615480369, 'message': 'Hello', 'roomId': '1:2'}"`. 

We use a stringified *JSON* for keeping the message structure and simplify the implementation details for this demo-app.

**Populate the "General" room with messages.** Messages are added to the sorted set with id of the "General" room: `room:0`

### Example: Prepare User Data in Redis HashSet
```C#
var usernameKey = $"username:{username}";
// Yeah, bcrypt generally ins't used in .NET, this one is mainly added to be compatible with Node and Python demo servers.
var hashedPassword = BCrypt.Net.BCrypt.HashPassword(password);
var nextId = await redisDatabase.StringIncrementAsync("total_users");
var userKey = $"user:{nextId}";
await redisDatabase.StringSetAsync(usernameKey, userKey);
await redisDatabase.HashSetAsync(userKey, new HashEntry[] {
    new HashEntry("username", username),
    new HashEntry("password", hashedPassword)
});
```

### Example: Prepare Room Data in Redis SortedSet
```C#
var roomKey = $"room:{roomId}";
var message = new ChatRoomMessage()
{
    From = fromId,
    Date = timeStamp,
    Message = content,
    RoomId = roomId
};
await redisDatabase.SortedSetAddAsync(roomKey, JsonSerializer.Serialize(message), message.Date);
```

### Example: Get all My Rooms
```C#
//fetch all my rooms
var roomIds = await _database.SetMembersAsync($"user:{userId}:rooms");
var rooms = new List<ChatRoom>();
foreach (var roomIdRedisValue in roomIds)
{
    var roomId = roomIdRedisValue.ToString();
    //fetch all users in the rooms
    var name = await _database.StringGetAsync($"room:{roomId}:name");
    if (name.IsNullOrEmpty)
    {
        // It's a room without a name, likey the one with private messages 
        var roomExists = await _database.KeyExistsAsync($"room:{roomId}");
        if (!roomExists)
        {
            continue;
        }

        var userIds = roomId.Split(':');
        if (userIds.Length != 2)
        {
            throw new Exception("You don't have access to this room");
        }

        rooms.Add(new ChatRoom()
        {
            Id = roomId,
            Names = new List<string>() {
                (await _database.HashGetAsync($"user:{userIds[0]}", "username")).ToString(),
                (await _database.HashGetAsync($"user:{userIds[1]}", "username")).ToString(),
            }
        });
    }
    else
    {
        rooms.Add(new ChatRoom()
        {
            Id = roomId,
            Names = new List<string>() {
                name.ToString()
            }
        });
    }
}
return rooms;
```

### Example: Send Message
```C#
public async Task SendMessage(UserDto user, ChatRoomMessage message)
{
  await _database.SetAddAsync("online_users", message.From);
  var roomKey = $"room:{message.RoomId}";
  await _database.SortedSetAddAsync(roomKey, JsonConvert.SerializeObject(message), (double)message.Date);
  await PublishMessage("message", message);
}
```

### Pub/sub
After initialization, a pub/sub subscription is created: `SUBSCRIBE MESSAGES`. At the same time, each server instance will run a listener on a message on this channel to receive real-time updates. 

Again, for simplicity, each message is serialized to ***JSON***, which we parse and then handle in the same manner, as WebSocket messages.

Pub/sub allows connecting multiple servers written in different platforms without taking into consideration the implementation detail of each server.

### Real-time chat and session handling

When a WebSocket/real-time server is instantiated, which listens for the next events:

**Connection**. A new user is connected. At this point, a user ID is captured and saved to the session (which is cached in Redis). Note, that session caching is language/library-specific and it's used here purely for persistence and maintaining the state between server reloads. 

A global set with `online_users` key is used for keeping the online state for each user. So on a new connection, a user ID is written to that set: 

**E.g.** `SADD online_users 1` (We add user with id 1 to the set **online_users**).

After that, a message is broadcasted to the clients to notify them that a new user is joined the chat.

**Disconnect**. It works similarly to the connection event, except we need to remove the user for **online_users** set and notify the clients: `SREM online_users 1` (makes user with id 1 offline).

**Message**. A user sends a message, and it needs to be broadcasted to the other clients. The pub/sub allows us also to broadcast this message to all server instances which are connected to this Redis:

`PUBLISH message "{'serverId': 4132, 'type':'message', 'data': {'from': 1, 'date': 1615480369, 'message': 'Hello', 'roomId': '1:2'}}"`

Note we send additional data related to the type of the message and the server id. Server id is used to discard the messages by the server instance which sends them since it is connected to the same `MESSAGES` channel. 

`type` field of the serialized JSON corresponds to the real-time method we use for real-time communication (connect/disconnect/message). 

`data` is method-specific information. In the example above it's related to the new message.



## How to run it locally?

#### Write in environment variable or Dockerfile actual connection to Redis:
```
   REDIS_ENDPOINT_URL = "Redis server URI"
   REDIS_PASSWORD = "Password to the server"
```

#### Run frontend

```sh
cd client
yarn install
yarn start
```

#### Run backend

```sh
dotnet restore
dotnet build
dotnet run
```

## Try it out

#### Deploy to Heroku

<p>
    <a href="https://heroku.com/deploy" target="_blank">
        <img src="https://www.herokucdn.com/deploy/button.svg" alt="Deploy to Heorku" />
    </a>
</p>