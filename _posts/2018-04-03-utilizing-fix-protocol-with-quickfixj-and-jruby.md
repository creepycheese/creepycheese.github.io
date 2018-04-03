---
layout: post
title: Utilizing FIX Protocol with Ruby and QuickfixJ.
tags: jruby
categories: ruby jruby fix
---

# Utilizing FIX Protocol with Ruby and QuickfixJ.
## Fix protocol overview.
FIX Protocol stands for Financial Information eXchange Protocol. It is commonly used for currency and stock trading. To perform Fix protocol messages exchange you need to create session: establish persistent tcp connection between client and broker. There can be many TCP connections(sessions). Each for the predefined type of messages/information(For example, My broker can define one session and end-point for fetching prices and another for buying/selling). Or maybe you want to connect to many serves/markets with single app.

Message format is dead simple and looks something like(%- stands for interpolation):
`%TAG=%VALUE%DELIMETER`
Message with many tags will look like:
`%TAG=%VALUE%DELIMETER%=%TAG=%VALUE%DELIMETER%TAG=%VALUE%DELIMETER`
Example LOGON message:
`8=FIX4.49=8335=A34=149=SENDER52=20180202-15:21:59.60256=RECEIVER98=0108=20141=Y10=084`
Here are some tags parsed and explained by great online tool [FIX Parser](https://fixparser.targetcompid.com/) (Pic.1):
![Message Explained]({{"/assets/quickfix_message.png" | absolute_url }}){:height="180px" width="740px"}
	**Pic.1.** *FIX Protocol message format explained.*

You can also find thorough description of each standard tag and standard message here: [FIX  Dictionary](https://www.onixs.biz/fix-dictionary/4.2). But there are always some user-defined fields(which defined by your broker or vendor)

## Motivation.
I work for financial organization Rocketbank LLC. Recently I had task to implement real-time currency purchase for our clients. We utilize micro service architecture approach. To accomplish this task was decided implement micro-service(lets call it ***FIX Service***), which will purchase it directly from currency broker and will be something like proxy between our system and currency broker, which also decodes outgoing messages from our system, to broker and vice versa. Also we need to be notified about the current currency rates and broadcast it to all services interested in it via **rabbitmq**. Oversimplified architecture looks like the following(Pic.2): 
1. We connect to 3-rd party broker via FIX protocol, who will sell/buy currency and who notifies us about currency rates. Permanently ask it for current market currency rates(In terms of FIX protocol  sends [MarketDataRequest](https://www.onixs.biz/fix-dictionary/4.2/msgType_V_86.html)).
2. On сurrency rate change we send it to RabbitMq Fanout exchange.
3. All services which queues bound to exchange got notified.

![Oversimplified Architecture]({{ "/assets/oversimplified_arch.png" | absolute_url }})
	**Pic.2.** *Oversimplified architecture. Currency rate broadcasting.*

Also we need to purchase currency when multi-currency operation performed buy our client. It looks like the following (Pic.3)
1. Service requests purchase of specified currency for another currency(USD for RUR in example below)
2. ***FIX Service*** (which connected to broker) translates our message(it can be in format of JSON, XML, Protocol Buffer, [paste your own inter-service communication format]) to FIX format(*New Order Single*) and sends it to broker.
3. ***FIX Service*** Receives *Execution Report Response* from broker

![Purchase]({{ "/assets/purchase_arch.png" | absolute_url }})
  **Pic.3.** *Oversimplified architecture. Purchase.*

## Tools
There is a limited number of Fix protocol implementation for each language and platform.  The most popular open source implementation is [QuickFIX](http://www.quickfixengine.org/), which was initially implemented in C++, but there are «ports» for *Ruby*, *Python*, *Java* and *C#*.  I will focus on using this in *Ruby* .  
Also Quickfix handles many nitty-gritty details like:
- Constructing messages
- Adding headers
- Tracking Sequence numbers
- Handling many session
- Handlings disconnects and reconnects

## Ruby Quickfix.
I found it overcomplicated(and lack of documentation) to use with SSL and documentation was very poor for Ruby, only some codesamples. 
## QuickfixJ
Documentation and amount of topics on forums about this implementation looked better. So I decided to use wrapper for JRuby.[quickfix-jruby](https://github.com/connamara/quickfix-jruby)

## Setup and Code.
## Short overview of Quickfix.
As for me I defined 3 core «parts». They are:
- **Config File**. Here you store your connection attributes for each session like host, connection type, sender and receiver and so on. For full number of available configuration fields refer to [Configuring QuickFIX/J](https://www.quickfixj.org/usermanual/2.0.0//usage/configuration.html) 
- **FIX «Application» callbacks implementation** [Creating Your Application](https://www.quickfixj.org/usermanual/2.0.0//usage/application.html). It Ruby it looks like:

```ruby
# /app/application.rb
module Fix
  class Application
	  # JRuby 'implements' interface
    include quickfix.Application

    def initialize(*args)
		# Some initialization

      super
    end

    def on_create(session_id)
      # Callback executed when session created
    end

    def on_logon(session_id)
      # callback executed when logon succeed
		# here we start requesting market data for example
      Fix.request_market_data(sell_currency: 'RUR', buy_currency: 'USD')
    end

    def on_logout(session_id)
    end

    def on_message(message, session_id)
    end

    def to_admin(message, session_id)
    end

    def to_app(message, session_id)
    end

    def from_app(msg, session_id)
#callback executed when message is received from server(broker)
      case msg
      when quickfix.fix44.MarketDataSnapshotFullRefresh
        #when market data received
      when quickfix.fix44.ExecutionReport
      else
        puts "#{msg.class} Not handled"
      end
    end

    def from_admin(msg, session_id)
    end
  end
```

- **Dictionary File.**  Large XML file where messages and corresponding tags are defined. You can see heartbeat message example below:

```
<message name='Heartbeat' msgtype='0' msgcat='admin'>
<field name='TestReqID' required='N' />
</message>
```

## Configuring SSL.

The most frustrating part for me was to connect via SSL. My vendor sent me `.pem`-formatted certificate and I had to generate `JKS`(Java Key Store) Certificate for my Quickfix-client to successfully connect. I’ve spent much time trying to convert certificate from `.pem` to `.jks`(I’m not  very experienced Java-programmer).
For me the following commands did the trick:

```
$ openssl pkcs12 -export -in my_vendor_certificate.pem -out cert_key.p12    

$ keytool -importkeystore -srckeystore cert_key.p12 -srcstoretype pkcs12 -destkeystore jks_certificate.jks -deststoretype jks
```

### Configuration.
My configuration File looks like the following
`app/config/settings.fix`: 
```
[DEFAULT]
FileStorePath=incoming
FileLogPath=outgoing
SocketUseSSL=Y
SocketKeyStore=/app/config/jks_certificate.jks
SocketKeyStorePassword=password
ResetOnLogon=Y

[SESSION]
BeginString=FIX.4.4
SenderCompID=SENDER
TargetCompID=RECEIVER
ConnectionType=initiator
StartTime=21:00:25
EndTime=21:00:05
HeartBtInt=20
SocketConnectHost=host.com
SocketConnectPort=32383
SequensReset=Y
UseDataDictionary=Y
DataDictionary=/app/config/FIX44.xml

[SESSION]
BeginString=FIX.4.4
SenderCompID=SENDER
TargetCompID=RECEIVER
ConnectionType=initiator
StartTime=21:00:25
EndTime=21:00:05
HeartBtInt=20
SocketConnectHost=host.com
SocketConnectPort=32384
SequensReset=N
UseDataDictionary=Y
DataDictionary=/app/config/FIX44.xml
```

Here I define *common* settings (use SSL connection, path to my certificate file) for each FIX Protocol session(market data and trade session).  And specify connection parameters(host and port, session start and end time) for each session individually.  More thorough explanation of each variable you can fine here: [Configuring QuickFIX/J](https://www.quickfixj.org/usermanual/2.0.0//usage/configuration.html) 

### Application Initialization.
Since I can have only one connection for each session I decided to use top-level module `Fix` as singleton for application start, communication and sending messages.

```ruby
# app/fix.rb
require 'quickfix'
require_relative 'application'

module Fix
  extend self
  FIX_SETTINGS = '/app/config/settings.fix'

 # helper to retreive current sessions
  def initiator
    @initiator
  end

  def start
	  #initialize fix application with callbacks
    application = Application.new
    session_settings = quickfix.SessionSettings.new(java.io.FileInputStream.new(FIX_SETTINGS))
    store_factory = quickfix.FileStoreFactory.new(session_settings)
    log_factory = quickfix.FileLogFactory.new(session_settings)
    message_factory = quickfix.DefaultMessageFactory.new
    @initiator = quickfix.SocketInitiator.new(application, store_factory, session_settings, log_factory, message_factory)
	   #start our sessions
    initiator.start
  end
end
``` 

Then we can just: `Fix.start` using REPL.
To do this I made `shebang`-jruby file to start my PRY-console session.

*# app/bin/console*:
```bash
#!/usr/bin/env jruby
require_relative '../config/setup'
require_relative '../lib/fix'

STDOUT.puts "Starting console"
require "pry"

Pry.start
```

### Constructing message
First thing we need to do is to create an instance of desired message. IFor example «Market Data Request» 
```ruby
market_data = quickfix.fix44.MarketDataRequest.new
```

Next thing we need to do is to specify fields. We need to create an instance of field with given value put into constructor and  assign it to message: 
```ruby
# create an instance of market data request id

market_data_request_id = quickfix.field.MDReqID.new("#{SecureRandom.uuid}")

# assign it to message
market_data.set_field(market_data_request_id)
```

And we can use `to_s` on message to preview result:
```ruby
market_data.to_s => 
"8=FIX4.4\u00019=46\u000135=V\u0001262=8d33a894-4ead-400d-8d16-36d77381804d\u000110=194\u0001"
```

Also, there is concept called *groups* . It’s like an array. Next example shows how to add MdEntryTypes. I want to receive bid and offer prices from my market-data session:

```ruby
# create group
md_entry_group = quickfix.fix44.MarketDataRequest::NoMDEntryTypes.new

# add element. Specify we want to receive bid stock prices 
md_entry_group.set(quickfix.field.MDEntryType.new(quickfix.field.MDEntryType::BID))

# another one. Specify we want to receive offer stock prices
md_entry_group.set(quickfix.field.MDEntryType.new(quickfix.field.MDEntryType::OFFER))

# add to message
market_data.add_group(md_entry_group)
```


### Sending messages
So. We «constructed our message» with desired parameters and now we want to send it and receive some response. For this purpose I defined `.send_to_market_data_session` method on my `Fix` module. Roughly it looks like: 

```ruby
# here RECEIVER is target comp id from settings file
MARKET_DATA_TARGET = 'RECEIVER'

# identify market data session. We do not want to send messages to wrong session.
# We can also can have many market data sessions for different "Servers"

def market_data_session
  @market_data_session ||= initiator.sessions.find { |s| s.target_comp_id == MARKET_DATA_TARGET }
end

# send it
def send_to_market_data_session(message)
  quickfix.Session.send_to_target(message, market_data_session)
end

# inside app

Fix.send_to_market_data_session(market_data)
```

### Handling response
To handle response we simply define our callbacks provided by QuickfixJ framework in our `Application` class. Callbacks depend on type of message you want to handle. You can find more detailed example with full application class above. Most of messages are handled in `from_app` callback. We just sent `Market data Request` so we want to handle market data snapshot. For this I do: 
```ruby
# app/application.rb
# inside class Application
def from_app(msg, session_id)
  case msg
  when quickfix.fix44.MarketDataSnapshotFullRefresh
    puts msg.(quickfix.field.Symbol.new).value
  else
    puts "#{msg.class} Not handled"
  end
end
```

I am just printing `Symbol` . But you can do anything you want. Symbol is a pair of currencies which looks like `currency_i_want_to_sell/currency_i_want_to_receive`

### Defining custom messages and properties
Some vendors define custom messages or fields for messages which are not part of FIX protocol specification. All used messages, properties and fields are described in dictionary file, I mentioned it above already.  Fields are described by `name`, `number` and `type`.  On example below I define custom field `Market Data Type`.  Number value is provided by my vendor. Name is arbitrary value, but usually also described by vendor.
  
```xml
 <field number='11010' name='MarketDataType' type='INT'>
  <value enum='1' description='FULL' />
  <value enum='2' description='A RANGE' />
 </field>
```

And add it to `MarketDataSnapshotFullRefresh` response: 
```xml
  <message name='MarketDataSnapshotFullRefresh' msgtype='W' msgcat='app'>
   <field name='MDReqID' required='N' />
   <component name='Instrument' required='Y' />
   <component name='UndInstrmtGrp' required='N' />
   <component name='InstrmtLegGrp' required='N' />
   <field name='FinancialStatus' required='N' />
   <field name='CorporateAction' required='N' />
   <field name='NetChgPrevDay' required='N' />
   <component name='MDFullGrp' required='Y' />
   <field name='ApplQueueDepth' required='N' /> 
   <field name='ApplQueueResolution' required='N' />
   <field name='MarketDataType' required='N' /> <<<---------------//HERE
  </message>
```

### Summary
I covered only the basics of FIX-protocol usage and QuickfixJ framework.  You always can refer to [QuickfixJ Documentation](https://www.quickfixj.org/usermanual/2.0.0//usage/configuration.html) . 
And [Example Application for Java on Github](https://github.com/quickfix-j/quickfixj/blob/master/quickfixj-examples/executor/src/main/java/quickfix/examples/executor/Application.java).  Also your vendor should have provided documentation for all the fields and possible protocol interactions.
Happy Coding!
