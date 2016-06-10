---
layout:	post
title:	"A queue peek adapter for JMS"
date:	2016-06-10 08:10:00 +0200
categories:	Development
tags:	[Java, JMS]
---

## Introduction

I've been working with some Java technologies of late, including tools like JMS.

Coming from a .Net world, JMS is very similar in concept to a technology like WCF. It provides an abstract way of working with message queues.

One thing which I found a little frustrating for development testing purposes was the lack of a `QueueReceiver` which acts in a peek function as opposed to pulling the actual message.

This is useful because sometimes you can't control adding new messages to the queue, and so developing against a peek message means you can recyle your messages with very little additional code.

## The Code

So without further babble, I wrote an adapter which adapts the JMS `QueueBrowser` to a `QueueReciever`.

Consuming messages in this may is then as simple as:

{% highlight java %}
MessageConsumer consumer = liveMessageMode
    ? s.createConsumer(destination)
    : new QueuePeekAdapter(s.createBrowser(destination));
{% endhighlight %}

where `destination` is obvisouly the queue that you are monitoring. Note that the destination must be a Queue descendant, and not a topic.

My Adapter class implmentation is pretty much a wrapper that peeks the first message every time. It wouldn't be too difficult to make it peek all messages though.

{% highlight java %}
import sun.reflect.generics.reflectiveObjects.NotImplementedException;

import javax.jms.*;
import java.util.Enumeration;

class QueuePeekAdapter implements QueueReceiver {
    private QueueBrowser browser;

    public QueuePeekAdapter(QueueBrowser browser) {
        this.browser = browser;
    }

    public Queue getQueue() throws JMSException {
        return browser.getQueue();
    }

    public String getMessageSelector() throws JMSException {
        return browser.getMessageSelector();
    }

    public MessageListener getMessageListener() throws JMSException {
        throw new NotImplementedException();
    }

    public void setMessageListener(MessageListener messageListener) throws JMSException {
        throw new NotImplementedException();
    }

    public Message receive() throws JMSException {
        Enumeration<Message> e = browser.getEnumeration();
        if (e.hasMoreElements()) {
            return e.nextElement();
        }

        return null;
    }

    public Message receive(long l) throws JMSException {
        return receive();
    }

    public Message receiveNoWait() throws JMSException {
        return receive();
    }

    public void close() throws JMSException {
        browser.close();
    }
}
{% endhighlight %}

## Conclusion

So, above is an adaption of the `QueueBrowser` to a `QueueReceiver` which makes recycling messages from JMS almsot trivial.