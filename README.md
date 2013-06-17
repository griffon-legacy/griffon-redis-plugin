
Redis support
-------------

Plugin page: [http://artifacts.griffon-framework.org/plugin/redis](http://artifacts.griffon-framework.org/plugin/redis)


The Redis plugin enables lightweight access to [Redis][1] datastores using [Jedis][2].
This plugin does NOT provide domain classes nor dynamic finders like GORM does.

Usage
-----
Upon installation the plugin will generate the following artifacts in `$appdir/griffon-app/conf`:

 * RedisConfig.groovy - contains the database definitions.
 * BootstrapRedis.groovy - defines init/destroy hooks for data to be manipulated during app startup/shutdown.

A new dynamic method named `withRedis` will be injected into all controllers,
giving you access to a `redis.clients.jedis.Jedis` object, with which you'll be able
to make calls to the database. Remember to make all database calls off the EDT
otherwise your application may appear unresponsive when doing long computations
inside the EDT.

This method is aware of multiple databases. If no databaseName is specified when calling
it then the default database will be selected. Here are two example usages, the first
queries against the default database while the second queries a database whose name has
been configured as 'internal'

    package sample
    class SampleController {
        def queryAllDatabases = {
            withRedis { databaseName, jedis -> ... }
            withRedis('internal') { databaseName, jedis -> ... }
        }
    }

This method is also accessible to any component through the singleton `griffon.plugins.redis.RedisConnector`.
You can inject these methods to non-artifacts via metaclasses. Simply grab hold of a particular metaclass and call
`RedisEnhancer.enhance(metaClassInstance, redisProviderInstance)`.

Configuration
-------------
### Dynamic method injection

The `withRedis()` dynamic method will be added to controllers by default. You can
change this setting by adding a configuration flag in `griffon-app/conf/Config.groovy`

    griffon.redis.injectInto = ['controller', 'service']

### Events

The following events will be triggered by this addon

 * RedisConnectStart[config, databaseName] - triggered before connecting to the database
 * RedisConnectEnd[databaseName, jedisPool] - triggered after connecting to the database
 * RedisDisconnectStart[config, databaseName, jedisPool] - triggered before disconnecting from the database
 * RedisDisconnectEnd[config, databaseName] - triggered after disconnecting from the database

### Multiple Stores

The config file `RedisConfig.groovy` defines a default datasource block. As the name
implies this is the datasource used by default, however you can configure named datasources
by adding a new config block. For example connecting to a datasource whose name is 'internal'
can be done in this way

    datasources {
        internal {
            password = null
            timeout = 2000i
            port = 6379i
            database = 1i
            pool {
                testWhileIdle = true
                minEvictableIdleTimeMillis = 60000
                timeBetweenEvictionRunsMillis = 30000
                numTestsPerEvictionRun = -1       
            }
        }
    }

This block can be used inside the `environments()` block in the same way as the
default datasource block is used.

### Example

A trivial sample application can be found at [https://github.com/aalmiray/griffon_sample_apps/tree/master/persistence/redis][3]

Testing
-------
The `withRedis()` dynamic method will not be automatically injected during unit testing, because addons are simply not initialized
for this kind of tests. However you can use `RedisEnhancer.enhance(metaClassInstance, redisProviderInstance)` where 
`redisProviderInstance` is of type `griffon.plugins.redis.RedisProvider`. The contract for this interface looks like this

    public interface RedisProvider {
        Object withRedis(Closure closure);
        Object withRedis(String serverName, Closure closure);
        <T> T withRedis(CallableWithArgs<T> callable);
        <T> T withRedis(String serverName, CallableWithArgs<T> callable);
    }

It's up to you define how these methods need to be implemented for your tests. For example, here's an implementation that never
fails regardless of the arguments it receives

    class MyRedisProvider implements RedisProvider {
        Object withRedis(String serverName = 'default', Closure closure) { null }
        public <T> T withRedis(String serverName = 'default', CallableWithArgs<T> callable) { null }
    }

This implementation may be used in the following way

    class MyServiceTests extends GriffonUnitTestCase {
        void testSmokeAndMirrors() {
            MyService service = new MyService()
            RedisEnhancer.enhance(service.metaClass, new MyRedisProvider())
            // exercise service methods
        }
    }


[1]: http://redis.io
[2]: https://github.com/xetorthio/jedis
[3]: https://github.com/aalmiray/griffon_sample_apps/tree/master/persistence/redis

