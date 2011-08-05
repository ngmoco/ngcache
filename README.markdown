## What is NGCache ##

NGCache is a write-through and read-through caching library for ActiveRecord.

Read-Through: Queries like `User.find(:all, :conditions => ...)` will first look in Memcached and then look in the database for the results of that query. If there is a cache miss, it will populate the cache.

Write-Through: As objects are created, updated, and deleted, all of the caches are *automatically* kept up-to-date and coherent.

For all those interested this GEM is a simplification of cache-money for use in Rails 3.x. By simplification I mean that not all the caching cases covered by cache-money are supported and will never be. For instance: NGCache will NOT support random indexes, that was a bad idea. It caused huge cache thrash. However, NGCache will support unique indexes, which makes sense.

What we are aiming for is to solve the 80% problem, find by ID or find by unique index and provide class methods to cache and clear whatever instance method you want.

## Howto ##
### What kinds of queries are supported? ###

Many styles of ActiveRecord usage are supported:

 * `User.find`
 * `User.find_by_id`
 * `User.find(:conditions => {:id => ...})`
 * `User.find(:conditions => ['id = ?', ...])`
 * `User.find(:conditions => 'id = ...')`
 * `User.find(:conditions => 'users.id = ...')`

As you can see, the `find_by_`, `find_all_by`, hash, array, and string forms are all supported.

Queries with joins/includes are unsupported at this time. In general, any query involving just equality (=) and conjunction (AND) is supported by `NGCache`. Disjunction (OR) and inequality (!=, <=, etc.) are not typically materialized in a hash table style index and are unsupported at this time.

Queries with limits and offsets are supported. In general, however, if you are running queries with limits and offsets you are dealing with large datasets. It's more performant to place a limit on the size of the `NGCache` index like so:

    DirectMessage.index :user_id, :limit => 1000
    
In this example, only queries whose limit and offset are less than 1000 will use the cache.

### Multiple unique indices are supported ###

    class User < ActiveRecord::Base
      index :screen_name
      index :email
    end

#### `with_scope` support ####

`with_scope` and the like (`named_scope`, `has_many`, `belongs_to`, etc.) are fully supported. For example, `user.devices.find(1)` will first look in the cache if there is an index like this:

    class Device < ActiveRecord::Base
     index [:user_id, :id]
    end

### Transactions ###

Because of the parallel requests writing to the same indices, race conditions are possible. We have created a pessimistic "transactional" memcache client to handle the locking issues.

The memcache client library has been enhanced to simulate transactions.

    $cache.transaction do
      $cache.set(key1, value1)
      $cache.set(key2, value2)
    end

The writes to the cache are buffered until the transaction is committed. Reads within the transaction read from the buffer. The writes are performed as if atomically, by acquiring locks, performing writes, and finally releasing locks. Special attention has been paid to ensure that deadlocks cannot occur and that the critical region (the duration of lock ownership) is as small as possible.

Writes are not truly atomic as reads do not pay attention to locks. Therefore, it is possible to peak inside a partially committed transaction. This is a performance compromise, since acquiring a lock for a read was deemed too expensive. Again, the critical region is as small as possible, reducing the frequency of such "peeks".

#### Rollbacks ####

    $cache.transaction do
      $cache.set(k, v)
      raise
    end

Because transactions buffer writes, an exception in a transaction ensures that the writes are cleanly rolled-back (i.e., never committed to memcache). Database transactions are wrapped in memcache transactions, ensuring a database rollback also rolls back cache transactions.

Nested transactions are fully supported, with partial rollback and (apparent) partial commitment (this is simulated with nested buffers).

### Locks ###

In most cases locks are unnecessary; the transactional Memcached client will take care locks for you automatically and guarantees that no deadlocks can occur. But for very complex distributed transactions, shared locks are necessary.

    $lock.synchronize('lock_name') do
      $memcache.set("key", "value")
    end
    
## Installation ##

#### Step 1: Get the GEM ####

    % sudo gem install ngcache
    
    Add the gem you your Gemfile:
    gem 'ngcache'
    
#### Step 2: Configure cache client

In your environment, create a cache client instance configured for your cache servers.
  
    $memcached = Memcached.new( ...servers..., ...options...)

Currently supported cache clients are: memcached, memcache-client

#### Step 3: Configure Caching

Add the following to an initializer:

    NGCache.configure :repository => $memcached, :adapter => :memcached

Supported adapters are :memcache_client, :memcached. :memcached is assumed and is only compatible with Memcached clients.
Local or transactional semantics may be disabled by setting :local => false or :transactional => false.

Caching can be disabled on a per-environment basis in the environment's initializer:
    
    NGCache.enabled = false
    
#### Step 4: Add indices to your ActiveRecord models ####

Queries like `User.find(1)` will use the cache automatically. For more complex queries you must add indices on the attributes that you will query on. For example, a query like `User.find(:all, :conditions => {:name => 'bob'})` will require an index like:

    class User < ActiveRecord::Base
      index :name
    end
    
For queries on multiple attributes, combination indexes are necessary. For example, `User.find(:all, :conditions => {:name => 'bob', :age => 26})`

    class User < ActiveRecord::Base
      index [:name, :age]
    end

#### Optional: Selectively cache specific models

There may be times where you only want to cache some of your models instead of everything.

In that case, you can omit the following from your `config/initializers/cache_money.rb`

	class ActiveRecord::Base
	  is_cached
	end
		
After that is removed, you can simple put this at the top of your models you wish to cache:

	is_cached

Just make sure that you put that line before any of your index directives. Note that all subclasses of a cached model are also cached.

## Acknowledgments ##

Thanks to

