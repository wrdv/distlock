# Scalalock [![Maven Central](https://img.shields.io/maven-central/v/com.weirddev/scalalock-api_2.12.svg?label=Maven%20Central)](https://search.maven.org/search?q=g:com.weirddev%20AND%20a:scalalock-api_2.12)

Distributed lock library in scala. based on mongodb (using [mongo-scala-driver](https://github.com/mongodb/mongo-scala-driver) )

consist of 2 modules:
- _scalalock-api_ - contains the api and the locking logic
- _scalalock-mongo_ - an api implementation using mongodb as the lock persistence store. implementation is dependant on [mongo-scala-driver](https://github.com/mongodb/mongo-scala-driver).


### Usage

1. Add dependencies to `build.sbt`:
  - Option A - if you're using [mongo-scala-driver](https://github.com/mongodb/mongo-scala-driver) 
    ```scala
    libraryDependencies ++= Seq(
      "com.weirddev" %% "scalalock-api" % "1.0.5",
      "com.weirddev" %% "scalalock-mongo" % "1.0.5"
    )
    ```
  - Option B - if you're using [reactivemongo](https://github.com/ReactiveMongo/ReactiveMongo) 
    ```scala
    libraryDependencies ++= Seq(
      "com.weirddev" %% "scalalock-api" % "1.0.5",
      "com.weirddev" %% "scalalock-reactivemongo" % "1.0.5"
    )
    ```

2. wrap the block that should be synchronized across all nodes in cluster with ```Lock#acquire()``` method call

    - For `scalalock-mongo` (based on `mongo-scala-driver`)
    
    ```scala
    import com.mongodb.ConnectionString
    import com.weirddev.scalalock.mongo.MongoDistributedLock
    import org.mongodb.scala.{MongoClient, MongoClientSettings, MongoDatabase}
    import scala.concurrent.Future
    import scala.concurrent.duration._

    protected val db: MongoDatabase = MongoClient(MongoClientSettings.builder()
        .applyConnectionString(new ConnectionString("mongodb://localhost:27017"))
        .build()).getDatabase("test")

    val distLock:Lock = new MongoDistributedLock(db)

    distLock.acquire("some_task_id", 10 minutes){
      Future{
        println("this block needs to execute in isolation across all application nodes in cluster")
        Thread.sleep(5000)
        "Task Completed"
      }
    }
    ```
    - Alternatively, using `scalalock-reactivemongo` (based on `org.reactivemongo`)
    
    ```scala
    import com.weirddev.scalalock.reactivemongo.ReactiveMongoDistributedLock
    import reactivemongo.api.{DefaultDB, MongoConnection, MongoDriver}
    import scala.concurrent.ExecutionContext.Implicits.global
    import scala.concurrent.Future
    import scala.concurrent.duration._
    
    private val mongoUri: String = "mongodb://localhost:27017/test"
      val driver = new MongoDriver
      val database: Future[DefaultDB] = for {
        uri <- Future.fromTry(MongoConnection.parseURI(mongoUri))
        con = driver.connection(uri)
        dn <- Future(uri.db.get)
        db <- con.database(dn)
      } yield db
    
      val distLock = new ReactiveMongoDistributedLock(database)
    
    distLock.acquire("some_task_id", 10 minutes){
      Future{
        println("this block needs to execute in isolation across all application nodes in cluster")
        Thread.sleep(5000)
        "Task Completed"
      }
    }
    ```
    - A simpler alternative using `scalalock-reactivemongo` in Play! Framework (add a dependency on reactivemongo play, i.e. `"org.reactivemongo" %% "play2-reactivemongo" % "0.16.5-play26"` )
    
    ```scala
    import com.weirddev.scalalock.reactivemongo.ReactiveMongoDistributedLock
    import scala.concurrent.{ExecutionContext, Future}
    import scala.concurrent.duration._
    import javax.inject.Inject
    import play.modules.reactivemongo.ReactiveMongoApi
    
    class MyRepository @Inject()(override val reactiveMongoApi: ReactiveMongoApi)(implicit val ec: ExecutionContext){
      val distLock = new ReactiveMongoDistributedLock(reactiveMongoApi.database)
    
      distLock.acquire("some_task_id", 10 minutes){
        Future{
          println("this block needs to execute in isolation across all application nodes in cluster")
          Thread.sleep(5000)
          "Task Completed"
        }
      }
    }
    ```

3. For other usage scenarios, review the [integrative test code](https://github.com/wrdv/scalalock/blob/master/scalalock-mongo/src/it/scala/com/weirddev/scalalock/MongoDistributedLockTest.scala)

### Contributing/Developing
Welcomed :) - Please refer to [`CONTRIBUTING.md`](./CONTRIBUTING.md) file.

### License
Copyright (c) 2016 - 2019  [WeirdDev](http://weirddev.com).
Licensed for free usage under the terms and conditions of Apache V2 - [Apache V2 License](https://www.apache.org/licenses/LICENSE-2.0).
