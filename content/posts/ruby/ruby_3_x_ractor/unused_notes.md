---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Ruby Racto"
subtitle: "String inputs that strip away leading, trailing and double spaces using typed virtual attributes"
summary: "String inputs that strip away leading, trailing and double spaces using typed virtual attributes"
authors: ["btihen"]
tags: ['Ruby', "Virtual Attributes"]
categories: ["Code", "Ruby Language"]
date: 2023-01-08T01:11:22+02:00
lastmod: 2023-01-08T01:11:22+02:00
featured: false
draft: true

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: ""
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---

## Thread Pool

https://rossta.net/blog/a-ruby-antihero-thread-pool.html

```
# fast
def fib_inject(n)
  (0..n).inject([1,0]) { |(a,b), _| [b, a+b] }[0]
end

# slow
def fib_recursive( n )
  return  n  if ( 0..1 ).include? n
  ( fib_recursive( n - 1 ) + fib_recursive( n - 2 ) )
end

# Ruby 3.2

#

# Thread Pool
#############
def fibonacci( n )
  return  n  if ( 0..1 ).include? n
  ( fibonacci( n - 1 ) + fibonacci( n - 2 ) )
end

class ThreadPool
  def initialize(size:)
  end

  def schedule(*args, &block)
    block.call(args)
  end

  def shutdown
  end
end

pool = ThreadPool.new(size: 5)
rounds = 40

t1 = Time.now
rounds.times do |i|
  p i
  pool.schedule {
    p fibonacci( i )
  }
end
p Time.now - t1

# 46 seconds - 40 - slow recusion


# fast
def fast_fib(n) = (0..n).inject([1,0]) { |(a,b), _| [b, a+b] }[0]


pool = ThreadPool.new(size: 5)
rounds = 10_000

t1 = Time.now
rounds.times do |i|
  p i
  pool.schedule {
    p fast_fib( i )
  }
end
p Time.now - t1

# 15 seconds - 10_000 - fast (inject)
pool.shutdown


# Ractors
#########
# slow
def fibonacci( n )
  return  n  if ( 0..1 ).include? n
  ( fibonacci( n - 1 ) + fibonacci( n - 2 ) )
end
pool = Ractor.new {
  loop {
    Ractor.yield(Ractor.receive)
  }
}
workers = (1..5).map do |i|
  Ractor.new(pool, name: "r#{i}") do |p|
    loop {
      n = p.take
      p n
      Ractor.yield(fibonacci(n))
    }
  end
end

rounds = 40
t1 = Time.now
rounds.times { |i| pool.send(i)}   # send 3 requests
rounds.times { p Ractor.select(*workers)} # take 3 responses
p Time.now - t1

# 20 seconds - 40 - slow (recursion) - at the end only a few threads run



def fast_fib(n) = (0..n).inject([1,0]) { |(a,b), _| [b, a+b] }[0]
pool = Ractor.new {
  loop {
    Ractor.yield(Ractor.receive)
  }
}
workers = (1..5).map do |i|
  Ractor.new(pool, name: "r#{i}") do |p|
    loop {
      Ractor.yield(fast_fib(p.take))
    }
  end
end

rounds = 10_000
t1 = Time.now
rounds.times { |i| pool.send(i)}   # send 3 requests
rounds.times { p Ractor.select(*workers)} # take 3 responses
p Time.now - t1

# 12 seconds - 10_000 - fast (enum)
```

## Video - author

https://www.youtube.com/watch?v=0kM7yFM6Dao

watch with htop

single thread

(1..4).map { Thread.new { loop {} } }

Multi Thread

(1..4).map { Ractor.new { loop {} } }

r = Ractor.new( 2**61 - 1) { |i| i. prime? }
p r.take

Sequential:

(t1 = Time.now
  n1 = 2**61 - 1
  n2 = 2**61 - 15
  [n1.prime?, n2.prime?]
)
p Time.now - t1

Parellel

(t1 = Time.now
  n1 = 2**61 - 1
  n2 = 2**61 - 15
  r1 = Ractor.new(n1) { |i| i. prime? }
  r1 = Ractor.new(n2) { |i| i. prime? }
  [n1.prime?, n2.prime?]
)
p Time.now - t1


Commmunication Types (control flow)

Push: Ractor#send & Ractor.receive (threaded message passing)
Pull: Ractor#yield & Ractor.take


## Push example (actor style - creates an incoming queue (infinite))

Ractor.main (send) --> Ractor 1 (receive)

```
# copies object (id differ)
r1 = Ractor.new do
  r_obj = receive
  p r_obj.object_id
end
obj = 'Hello'
p obj.object_id
r1.send(obj)
r1.take # waits for execution
```

r1 = Ractor.new do
  r_obj = Ractor.receive
  p r_obj[0].object_id
end
obj = [[1], 2]
p obj[0].object_id
r1.send(obj)
r1.take # waits for execution


## Pull Communication (rendevous communication)

Ractor.main (yield) <-- Ractor 1 (take)

r1 = Ractor.new do
  3.times { |i| Ractor.yield i }
  :fin # result of block yielded
end
4.times { p r1.take }

### Example - pipeline

  Ractor 1 ------>  Ractor 2
  ^                        |
  |                        |
  ------  Ractor.main <-----
main: r1.send     main: r2.take

def task(n) = n + 1
r1 = Ractor.new { loop { Ractor.yield(task(Ractor.receive)) } }
r2 = Ractor.new(r1) { |r1| loop { Ractor.yield(task(r1.take)) } }
r1.send(1)
p r2.take
task(1)

### load balancing
                           ----> Ractor 1 (r1: main.take)
                          /
Ractor.main --------------
main: Ractor.yield(obj)   \
                           ----> Ractor 2 (r2: main.take)
main = Ractor.current
r1 = Ractor.new(main) { |m| p r1: m.take }
r2 = Ractor.new(main) { |m| p r2: m.take }
Ractor.yield(:message)
Ractor.yield(:message)
Ractor.yield(:message)


### Cluster - taking from multiple Ractors (cluster)

Ractor 1 ---
            \
Ractor 2  --------> Ractor.main
            /       Ractor.select(r1, r2, r3)
Ractor 3  --

### Worker pool
                                          A sent 'obj' will be handled
                                          by an idle Ractor (r1, r2, r3)

                                          --> Ractor worker 1 --|
                                         /                      |
Ractor.main ------> Ractor.bridge ----------> Ractor worker 2 --|
bridge.send(obj)    loop {               \                      |
    ^                 Ractor.yield(       --> Ractor worker 3 --|
    |                   Ractor.receive) }                       |
    |                                                           |
    -------------------------------------------------------------

require 'prime'
def task(n) = [n, n.prime?]
bridge = Ractor.new { loop { Ractor.yield(Ractor.receive) } }
workers = (1..3).map do |i|
  Ractor.new(bridge, name: "r#{1}") do |b|
    loop { Ractor.yield(task(b.take)) }
  end
end
3.times { |i| bridge.send(11 + i)}   # send 3 requests
3.times { p Ractor.select(*workers)} # take 3 responses

### Complex Pool

                                          --> Ractor worker 1 --> Ractor 3 -|
                                         /                                  |
Ractor.main ------> Ractor.bridge ----------> Ractor worker 2 --> Ractor 4 -|
bridge.send(obj)    loop {               \                                  |
    ^                 Ractor.yield(       --> Ractor worker 3 --> Ractor 5 -|
    |                   Ractor.receive) }                                   |
    |                                                                       |
    -------------------------------------------------------------------------


### exceptions

```
def task(char) = char + '2'
r1 = Ractor.new(name: 'r1') { task(receive) }

r1.send('1')
p r1.take

# without loop after first usage it closes
begin
  r1.send('2')
rescue Ractor::ClosedError => e
  p e
end


r1 = Ractor.new(name: 'r1') { loop { task(receive) } }

# if wrong type it closes with an error
begin
  r1.send(1)
  p r1.take
rescue Ractor::RemoteError => e
  p e
  p e.cause
  p e.ractor
end
```

## Exception chaining

when r1 dies it kills r2 too!

Ractor.main  ------>  Ractor 1    ------>  Ractor 2 ----
  ^   main: r1.send   r1: main.receive     r2: r1.take |
  |                                                    |
  |<----------------------------------------------------
main: r2.take

```
def task1(n) = n + '1'
def task2(n) = n + 2
r1 = Ractor.new(name: 'r1') { loop { Ractor.yield(task1(Ractor.receive)) } }
r2 = Ractor.new(r1, name: 'r2') { |r1| loop { Ractor.yield(task2(r1.take)) } }
r1.send(1)
begin
  r2.take
rescue Ractor::RemoteError => e
  p e
  p e.cause
  p e.ractor
end
# after first error r1 closes
begin
  r2.take
rescue Ractor::ClosedError => e
  p e
end
```

### Supervising Ractors

```ruby
def task1(n) = n + 1
def task2(n) = n + '2'
r1 = Ractor.new(name: 'r1') { loop { Ractor.yield( task1(receive) ) } }
r2 = Ractor.new(r1, name: 'r2') { |r1| loop { Ractor.yield( task2(r1.take) ) } }

# r1 & r2 dies
begin
  r1.send('1')
  r2.take
  p 'success'
rescue Ractor::RemoteError => e
  p 'recovery'
  p e
  p e.cause
  p e.ractor
  p 'restarting r1 & r2'
  # when r1 dies it kills all downstream ractors
  r1 = Ractor.new(name: 'r1') { loop { Ractor.yield( task1(receive) ) } }
  r2 = Ractor.new(r1, name: 'r2') { |r1| loop { Ractor.yield( task2(r1.take) ) } }
end

# just r2 dies
begin
  r1.send(1)
  r2.take
  p 'success'
rescue Ractor::RemoteError => e
  p 'recovery'
  p e
  p e.cause
  p e.ractor
  case e.ractor.name
  when 'r1'
    p 'restarting r1 & r2'
    # when r1 dies it kills all downstream ractors
    r1 = Ractor.new(name: 'r1') { loop { Ractor.yield( task1(receive) ) } }
    r2 = Ractor.new(r1, name: 'r2') { |r1| loop { Ractor.yield( task2(r1.take) ) } }
  when 'r2'
    p 'restarting r2'
    # only kills r2 - unlikely to die
    r2 = Ractor.new(r1, name: 'r2') { |r1| loop { Ractor.yield( task2(r1.take) ) } }
  end
end




def task(char) = char + '2'
r1 = Ractor.new(name: 'r1') { task(receive) }

r1.send('1')
p r1.take

# without loop after first usage it closes
begin
  r1.send('2')
rescue Ractor::ClosedError => e
  p e
  r1 = Ractor.new(name: 'r1') { loop { task(receive) } }
end

# if wrong type it closes with an error
begin
  r1.send(1)
  p r1.take
rescue Ractor::RemoteError => e
  p e
  p e.cause
  p e.ractor
  p 'restating' if e.ractor.name == 'r1'
  r1 = Ractor.new(name: 'r1') { loop { task(receive) } }
end

# after error r1 closes too
begin
  r1.take
rescue Ractor::ClosedError => e
  p e
  r1 = Ractor.new(name: 'r1') { loop { task(receive) } }
end
```


## Raft / Ractor example!

https://www.youtube.com/watch?v=_O3NBm_C3rM
