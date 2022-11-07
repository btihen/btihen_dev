---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Durable_rails_config"
subtitle: ""
summary: ""
authors: ["btihen"]
tags: ['Ruby']
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2021-05-10T20:46:07+02:00
lastmod: 2021-05-10T20:46:07+02:00
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
# Rails Setup

- [Arkensi Event Store](https://railseventstore.org/docs/v2/install/)
- [Rails DDD / Event Store Training](https://products.arkency.com/ddd-training/)
- [Publish-Subscribe Tutorial](https://www.toptal.com/ruby-on-rails/the-publish-subscribe-pattern-on-rails)
- [SimpleEventBus](https://github.com/mikhailvs/simple-event-bus)
- [event_bus](https://github.com/kevinrutherford/event_bus)
- [EventMachine-Channels](https://github.com/eventmachine/eventmachine)
- [EventBus - Simple](https://stackoverflow.com/questions/10247961/is-there-a-ruby-message-bus-gem)
```
require 'rubygems'
require 'eventmachine'

module Messagebus
  class Server
    attr_accessor :connections
    attr_accessor :channel

    def initialize
      @connections = []
      @channel = EventMachine::Channel.new
    end

    def start
      @signature = EventMachine.start_unix_domain_server '/tmp/messagebus.sock', Connection do |conn|
        conn.server = self
      end
    end

    def stop
      EventMachine.stop_server(@signature)

      unless wait_for_connections_and_stop
        EventMachine.add_periodic_timer(1) { wait_for_connections_and_stop }
      end
    end

    def wait_for_connections_and_stop
      if @connections.empty?
        EventMachine.stop
        true
      else
        puts "Waiting for #{@connections.size} connection(s) to finish ..."
        false
      end
    end
  end

  class Connection < EventMachine::Connection
    attr_accessor :server

    def post_init
      log "Connection opened"
    end

    def server=(server)
      @server = server

      @subscription = server.channel.subscribe do |data|
        self.log "Sending #{data}"
        self.send_data data
      end
    end

    def receive_data(data)
      log "Received #{data}"
      server.channel.push data
    end

    def unbind
      server.channel.unsubscribe @subscription
      server.connections.delete(self)
      log "Connection closed"
    end

    def log(msg)
      puts "[#{self.object_id}] #{msg}"
    end
  end
end

EventMachine::run {
  s = Messagebus::Server.new
  s.start
  puts "New server listening"
}
```
