# Understanding redis
Date: 2025-12-29 <br>
*Written by: Tanmoy Saha*

#### What is redis?
Redis is an in-memory database. It uses your RAM (Random Access Memory) to store the data and for this the read and write operations really fast in Redis. Normal databases like Postgres or MongoDB uses our normal storage (If we consider SSD) to store the data. And for this reason to read data from normal database it took 100-200 microseconds where as Redis need only 20 nanoseconds to read same data from the RAM. And that is why Redis uses RAM as primary storage. Redis does have an additional feature called disk persistence for durability, it stores the snapshot of cached data inside hard disk in background as a backup so, instance startup won't wipe-out the data. We store data in **key: value** format in Redis.
Datatype Redis support - 
- String
- Hash
- List
- Set
- Bitmap
- Geospatial
Using extension modules we can store probabilistic datatypes like hyper loglog, top-k, cuckoo filter, t-digest and bloom filter (We will explore this part in another blog to know more).

#### Let's install Redis using docker (Because it's the easiest method available)
Follow these steps - 
- open this link - https://redis.io/docs/latest/get-started/
- Click "Run Redis on Docker".
- Copy the docker command (You can copy it from here alos :) - 
	- `docker run -d --name redis -p 6379:6379 redis:latest`
- Run `docker ps` command to check the current containers available.
- Run this command to execute the Redis on docker - 
	- `docker exec -it "image_id" redis-cli`

#### Now let's practice some docker commands   

| **Explanation**                                                                                       | **Command**                                      | **Expected Output**                                                               |
| ----------------------------------------------------------------------------------------------------- | ------------------------------------------------ | --------------------------------------------------------------------------------- |
| Check Redis server is running status                                                                  | PING                                             | PONG                                                                              |
| To check Redis server information                                                                     | INFO                                             | Information about the Redis server                                                |
| To store a key and value. Here key is name and value is tanmoy                                        | SET name tanmoy                                  | OK                                                                                |
| To get the stored value.                                                                              | GET name                                         | "tanmoy"                                                                          |
| We can store a value with expiration duration. We need to give the duration in seconds after the key. | SET name 10 tanmoy                               | "tanmoy" after 10 secs (nil)                                                      |
| We can store data in a list. Using LPUSH in this command.                                             | LPUSH names "tanmoy"                             | (integer) 1                                                                       |
| Pushing another value inside names key from right side using RPUSH command.                           | RPUSH names <br>"sandip"                         | (integer) 2                                                                       |
| Let's get the values we pushed inside the list. LRANGE + key + start_index + end_index (-1 means all) | LRANGE names 0 -1                                | 1) tanmoy <br>2) sandip                                                           |
| To check the length of the list use LLEN                                                              | LLEN names                                       | (integer) 2                                                                       |
| To remove one value from the left side we can use LPOP                                                | LPOP names                                       | "tanmoy"                                                                          |
| To remove one value from the right side we can use RPOP                                               | RPOP names                                       | "sandip"                                                                          |
| To store a Hash table datatype we need to use HSET followed by keys and values                        | HSET user:1 name tanmoy surname saha gender male | (integer) 3                                                                       |
| To get one value by key HGET                                                                          | HGET user:1 name                                 | "tanmoy"                                                                          |
| To get all the values use HGETALL                                                                     | HGETALL user:1                                   | 1) "name"<br>2) "tanmoy"<br>3) "surname"<br>4) "saha"<br>5) "gender"<br>6) "male" |

So, these are some basic knowledge of my about Redis. In the next article we will dive deep with some real world implementation. Till then enjoy your life.

*Happy engineering!*
