# scaling-api-rate
```ruby
# How many requests per second do you want a user to be allowed to do?
REPLENISH_RATE = 100

# How much bursting do you want to allow?
CAPACITY = 5 * REPLENISH_RATE

SCRIPT = File.read('request_rate_limiter.lua')

def check_request_rate_limiter(user)
  # Make a unique key per user.
  prefix = 'request_rate_limiter.' + user

  # You need two Redis keys for Token Bucket.
  keys = [prefix + '.tokens', prefix + '.timestamp']

  # The arguments to the LUA script. time() returns unixtime in seconds.
  args = [REPLENISH_RATE, CAPACITY, Time.new.to_i, 1]

  begin
    allowed, tokens_left = redis.eval(SCRIPT, keys, args)
  rescue RedisError => e
    # Fail open. We don't want a hard dependency on Redis to allow traffic.
    # Make sure to set an alert so you know if this is happening too much.
    # Our observed failure rate is 0.01%.
    puts 'Redis failed: ' + e
    return
  end

  if !allowed
    raise RateLimitError.new(status_code = 429)
  end
end
```

Here is the corresponding `request_rate_limiter.lua` script:

```lua
local tokens_key = KEYS[1]
local timestamp_key = KEYS[2]

local rate = tonumber(ARGV[1])
local capacity = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

local fill_time = capacity/rate
local ttl = math.floor(fill_time*2)

local last_tokens = tonumber(redis.call("get", tokens_key))
if last_tokens == nil then
  last_tokens = capacity
end

local last_refreshed = tonumber(redis.call("get", timestamp_key))
if last_refreshed == nil then
  last_refreshed = 0
end

local delta = math.max(0, now-last_refreshed)
local filled_tokens = math.min(capacity, last_tokens+(delta*rate))
local allowed = filled_tokens >= requested
local new_tokens = filled_tokens
if allowed then
  new_tokens = filled_tokens - requested
end

redis.call("setex", tokens_key, ttl, new_tokens)
redis.call("setex", timestamp_key, ttl, now)

return { allowed, new_tokens }
```

## Concurrent requests limiter

Because Redis is so fast, doing the naive thing works. Just add a random token to a set at the start of a request and remove it from the set when you're done. If the set is too large, reject the request.

Again the [full code is available](#file-2-check_concurrent_requests_limiter-rb) and a sketch follows:

```ruby
# The maximum length a request can take
TTL = 60

# How many concurrent requests a user can have going at a time
CAPACITY = 100

SCRIPT = File.read('concurrent_requests_limiter.lua')

class ConcurrentRequestLimiter
  def check(user)
    @timestamp = Time.new.to_i

    # A string of some random characters. Make it long enough to make sure two machines don't have the same string in the same TTL.
    id = Random.new.bytes(4)
    key = 'concurrent_requests_limiter.' + user
    begin
      # Clear out old requests that probably got lost
      redis.zremrangebyscore(key, '-inf', @timestamp - TTL)
      keys = [key]
      args = [CAPACITY, @timestamp, id]
      allowed, count = redis.eval(SCRIPT, keys, args)
    rescue RedisError => e
      # Similarly to above, remember to fail open so Redis outages don't take down your site
      log.info('Redis failed: ' + e)
      return
    end

    if allowed
      # Save it for later so we can remove it when the request is done
      @id_in_redis = id
    else
      raise RateLimitError.new(status_code: 429)
    end
  end

  # Call this method after a request finishes
  def post_request_bookkeeping(user)
    if not @id_in_redis
      return
    end
    key = 'concurrent_requests_limiter.' + user
    removed = redis.zrem(key, @id_in_redis)
  end

  def do_request(user)
    check(user)

    # Do the actual work here

    post_request_bookkeeping(user)
  end
end
```

The content of `concurrent_requests_limiter.lua` is simple and is meant to guarantee the atomicity of the ZCARD and ZADD.

```lua
local key = KEYS[1]

local capacity = tonumber(ARGV[1])
local timestamp = tonumber(ARGV[2])
local id = ARGV[3]

local count = redis.call("zcard", key)
local allowed = count < capacity

if allowed then
  redis.call("zadd", key, timestamp, id)
end

return { allowed, count }
```

## Fleet usage load shedder

We can now move from preventing abuse to adding stability to your site with load shedders. If you can categorize traffic into buckets where no fewer than X% of your workers should be available to process high-priority traffic, then you're in luck: this type of algorithm can help. We call it a _load shedder_ instead of a rate limiter because it isn't trying to reduce the rate of a specific user's requests. Instead, it adds backpressure so internal systems can recover. 

When this load shedder kicks in it will start dropping non-critical traffic. There should be alarm bells ringing and people should be working to get the traffic back, but at least your core traffic will work. For Stripe, high-priority traffic has to do with creating charges and moving money around, and low-priority traffic has to do with analytics and reporting.

The great thing about this load shedder is that its implementation is identical to the Concurrent Requests Limiter, except you don't use a user-specific key, you just use a global key.

```ruby
limiter = ConcurrentRequestLimiter.new
def check_fleet_usage_load_shedder
  if is_high_priority_request
    return
  end

  begin
    return limiter.do_request('fleet_usage_load_shedder')
  rescue RateLimitError
    raise RateLimitError.new(status_code: 503)
  end
end
```

## Worker utilization load shedder

This load shedder is the last resort, and only kicks in when a machine is under heavy pressure and needs to offload. The code for determining how many workers are in use is dependent on your infrastructure. The general outline is to figure out some measure of "Is our infrastructure currently failing?" If that function returns something non-zero, start throwing out your least important requests (after waiting a short period to allow imprecise measurements) with higher and higher probability. After a period of time doing that, move on to more requests until you are throwing out everything except for the most critical traffic.

The most important behavior for this load shedder is to slowly take action. Don't start throwing out traffic until your infrastructure has been sad for quite a while (30 seconds), and don't instantaneously add traffic back. Sharp changes in shedding amounts will cause wild swings and lead to failure modes that are hard to diagnose.

As before, the [full script with a small test suite is available](#file-3-check_worker_utilization_load_shedder-rb), and here is a sketch:

```ruby
END_OF_GOOD_UTILIZATION = 0.7
START_OF_BAD_UTILIZATION = 0.8

# Assuming a sample rate of 8 seconds, so 28 == 2.5 * 8 == guaranteed 3 samples
NUMBER_OF_SECONDS_BEFORE_SHEDDING_STARTS = 28
NUMBER_OF_SECONDS_TO_SHED_ALL_TRAFFIC = 120

RESTING_SHED_AMOUNT = -NUMBER_OF_SECONDS_BEFORE_SHEDDING_STARTS / NUMBER_OF_SECONDS_TO_SHED_ALL_TRAFFIC

@shedding_amount_last_changed = 0
@shedding_amount = 0

def check_worker_utilization_load_shedder
  chance = drop_chance(current_worker_utilization)
  if chance == 0
    dropped = false
  else
    dropped = Random.rand() < chance
  end
  if dropped
    raise RateLimitError.new(status_code: 503)
  end
end

def drop_chance(utilization)
  update_shedding_amount_derivative(utilization)
  how_much_traffic_to_shed
end

def update_shedding_amount_derivative(utilization)
  # A number from -1 to 1
  amount = 0

  # Linearly reduce shedding
  if utilization < END_OF_GOOD_UTILIZATION
    amount = utilization / END_OF_GOOD_UTILIZATION - 1
  # A dead zone
  elsif utilization < START_OF_BAD_UTILIZATION
    amount = 0
  # Shed traffic
  else
    amount = (utilization - START_OF_BAD_UTILIZATION) / (1 - START_OF_BAD_UTILIZATION)
  end

  # scale the derivative so we take time to shed all the traffic
  @shedding_amount_derivative = clamp(amount, -1, 1) / NUMBER_OF_SECONDS_TO_SHED_ALL_TRAFFIC
end

def how_much_traffic_to_shed
  now = Time.now().to_f
  seconds_since_last_math = clamp(now - @shedding_amount_last_changed, 0, NUMBER_OF_SECONDS_BEFORE_SHEDDING_STARTS)
  @shedding_amount_last_changed = now
  @shedding_amount += seconds_since_last_math * @shedding_amount_derivative
  @shedding_amount = clamp(@shedding_amount, RESTING_SHED_AMOUNT, 1)
end

def current_worker_utilization
  # Returns a double from 0 to 1.
  # 1 means every process is busy, .5 means 1/2 the processes are working, and 0 means the machine is servicing 0 requests
  # This is infra dependent on how to read this value
end

def clamp(val, min, max)
  if val < min
    return min
  elsif val > max
    return max
  else
    return val
  end
end
```
