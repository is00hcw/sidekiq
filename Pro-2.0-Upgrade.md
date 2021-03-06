# Upgrading to Sidekiq Pro 2.0

Sidekiq Pro 2.0 allows nested batches for more complex job workflows.
It also removes deprecated APIs, changes the batch data format and
how features are activated.  Read carefully to ensure your upgrade goes
smoothly.

## Nested Batches

Batches can now be nested within the `jobs` method.
This feature enables Sidekiq Pro to handle workflow processing of any size
and complexity!

```ruby
a = Sidekiq::Batch.new
a.on(:success, SomeCallback)
a.jobs do
  SomeWork.perform_async

  b = Sidekiq::Batch.new
  b.on(:success, MyCallback)
  b.jobs do
    OtherWork.perform_async
  end
end
```

Parent batch callbacks are not processed until any child batch callbacks have
run successfully.  In the example above, `MyCallback` will always fire
before `SomeCallback` because `b` is considered a child of `a`.

Of course you can dynamically add child batches while a batch job is executing.

```ruby
def perform(*args)
  do_something(args)

  if more_work?
    # Sidekiq::Worker#batch returns the Batch this job is part of.
    batch.jobs do
      b = Sidekiq::Batch.new
      b.on(:success, MyCallback)
      b.jobs do
        OtherWork.perform_async
      end
    end
  end
end
```

## Batch Data

The batch data model was overhauled.  Batch data should take
significantly less space in Redis now.  A simple benchmark shows 25%
savings but real world savings should be even greater.

* Batch 2.x BIDs are 14 character URL-safe Base64-encoded strings, e.g.
  "vTF1-9QvLPnREQ".  Batch 1.x BIDs were 16 character hex-encoded
  strings, e.g. "4a3fc67d30370edf".
* In 1.x, batch data was not removed until it naturally expired in Redis.
  In 2.x, all data for a batch is removed from Redis once the batch has
  run any success callbacks.
* Because of the former point, batch expiry is no longer a concern.
  Batch expiry is hardcoded to 30 days and is no longer user-tunable.
* Failed batch jobs no longer automatically store any associated
  backtrace in Redis.

**There's no data migration required.  Sidekiq Pro 2.0 transparently handles
both old and new format.**

**Note that you CANNOT go back to Pro 1.x once you've created batches
with 2.x.  The new batches will not process correctly with 1.x.**

## Reliability

You no longer need to require anything to use Reliability features.

* Activate reliable fetch:
```ruby
Sidekiq.configure_server do |config|
  config.reliable_fetch!
end
```
* Activate reliable push:
```ruby
Sidekiq::Client.reliable_push!
```

## Other Changes

* You must require `sidekiq/notifications` if you want to use the
  existing notification schemes.  I don't recommend using them as the
  newer-style `Sidekiq::Batch#on` method is simpler and more flexible.
* Several classes have been renamed.  Generally these classes are ones
  you should not need to require/use in your own code, e.g. the Batch
  middleware.
* The Web UI now shows the Sidekiq Pro version in the footer.
