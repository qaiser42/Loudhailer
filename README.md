### LoudHailer – Reaching your audience!
A mnmlst voice recognition app based on [Akka Streams](http://akka.io), [wit.ai](https://wit.ai/) (aka NLP as service) and [Firebase](firebase.google.com) for messaging. 

![Vision](http://i.imgur.com/CvWgRl0l.png) 

### What's this?
Building the next fancy [Goldberg machine](https://en.wikipedia.org/wiki/Rube_Goldberg_machine) _just_ for learning purposes convinced myself fairly quickly.
With this idea in mind, I started this mini blog series to document my learn achievements during this side project.

### The idea
What's LoudHailer about? Functionally it's a voice recognition app to detect emergency cases.

![Vision](http://i.imgur.com/CvWgRl0l.png) 

Technically spoken it's a stream processing application based on [Akka Streams](http://akka.io) and consists of the actual voice recording and a processing backend.
The backend automatically detects alarming words and notifies clients about an incident. For this I created a tiny Android app using [Kotlin](http://kotlinlang.org).
Also to make it super duper easy to use, I dockerized LoudHailer to run it directly on a [Raspberry](http://blog.hypriot.com/getting-started-with-docker-on-your-arm-device/). Cool, huh?

### Akka Streams 101
 
If you have no clue about [Akka (Streams)](http://akka.io) or are unfamiliar with the term `stream processing`, then have a look at this superior writeup by Kevin Webber:
[Diving into Akka Streams](https://medium.com/@kvnwbbr/diving-into-akka-streams-2770b3aeabb0), though knowing all the nitty gritty details is not needed at this point. 

### No fluff, just stuff™

LoudHailer's main processing skeleton looks like this:

{% highlight scala %}
Source.tick(0 second, 5 seconds, ())
        .map(sample)
        .map(analyse)
        .runForeach(act)
{% endhighlight %}

Every five seconds we're sampling, analysing the recording and acting on the spoken words. Let's examine the steps `sample`, `analyse` and `act` in more detail:

{% highlight scala %}
def sample: Unit => Sample = _ => SoundRecorder.sample
{% endhighlight %}

This function obviously returns a `Sample` value, which itself is a type alias for `Xor[Error, Array[Byte]]`. The semantic behind this type is:
either there was an error during the sampling or we're successfully referencing the recording as a raw byte array. `SoundRecorder.sample` again is rather boring: it's utilizing
the `javax.sound` API to take the actual record.

Next we'll examine a more interesting part of the stream: analysing the recording via [wit.ai](http://wit.ai) (NLP-as-a-Service)

{% highlight scala %}
def analyse: Sample => Future[Hypothesis] = {
    case Xor.Left(e) => Future.failed(e)
    case Xor.Right(data) =>
      for {
        response <- request(data)
        hypothesis <- Unmarshal(response.entity).to[Hypothesis]
      } yield hypothesis
  }
  
def request: Array[Byte] => Future[HttpResponse] = data =>
  Http().singleRequest(HttpRequest(
    method = HttpMethods.POST,
    uri = witUrl,
    headers = List(headers.RawHeader("Authorization", s"Bearer $witToken")),
    entity = HttpEntity(contentType = `audio/wav`, data)))

{% endhighlight %}

For a given Sample object, `analyse` returns a `Future[Hypothesis]`. What you're seeing in the `Xor.Right` case is a [for-comprehension](http://docs.scala-lang.org/tutorials/FAQ/yield.html),
which is just syntactic sugar for chaining `flatMap`, `map`, `filter`, etc. functions in a more concise way on the underlying type. Here it's a `Future` object. I assume that you're familiar with
the idea of `Futures` and `Promises` and will therefore omit any further introduction. Otherwise have a look [here](http://docs.scala-lang.org/overviews/core/futures.html). 

Let's dig deeper into it: depending whether the sampling was successful (`Xor.Right`), [wit.ai](https://wit.ai)  will be requested to analyse the raw audio stream. The eventual response will be transformed into a
`Hypothesis` object, which, as the name suggests, simply holds the hypothetical[^1] spoken word or sentence as a `String` value. 

[^1]: Why voice recognition and speech in general is actually a probabilistic phenomenon is comprehensibly explained here: [cmusphinx: Basic Concepts of Speech](http://cmusphinx.sourceforge.net/wiki/tutorialconcepts)

Finally `act` verifies if the present hypothesis is a blacklisted term, and if so, then we're broadcasting it via [Firebase](https://firebase.google.com/) it to the clients.

{% highlight scala %}
def act: Future[Hypothesis] => Unit = f => {
    f.onComplete {
      case Success(h) => if (blackList.contains(response.hypothesis)) broadcastEvent()
      case Failure(e) => e.printStackTrace()
    }
  }
  
def broadcastEvent() = {
    val body = Map(
      "to" -> "/topics/alert".asJson,
      "data" -> Map(
        "message" -> "An incident occurred.".asJson
      ).asJson
    ).asJson

    Http().singleRequest(
      HttpRequest(
        method = HttpMethods.POST,
        uri = fireBaseUrl,
        headers = List(headers.RawHeader("Authorization", s"key=$fireBaseToken")),
        entity = HttpEntity(contentType = `application/json`, body.noSpaces)))
  }
{% endhighlight %}

### Summing up

Frankly, the overall stream is rather naïvely realized… For example, it lacks to throttle the stream, if the external web services adhere to api call restrictions. Also there's no information reflux
from the clients perspective.
Certainly the person in need would be greatly eased, if the clients were responding with an acknowledgment. This would also enable the clients to intercommunicate. Maybe I'll tackle this in another weekend hack?

### How to run the stream?
1. Create an account @ [wit.ai](https://wit.ai/) and copy the access token to [wit.token](https://github.com/qabbasi/Loudhailer/blob/master/voice-recognition-stream/src/main/resources/application.conf) 
2. Create an account @ [Firebase](http://firebase.google.com) and copy the access token to [firebase.token](https://github.com/qabbasi/Loudhailer/blob/master/voice-recognition-stream/src/main/resources/application.conf) 

### How to run the client app?
1. Create a new client project at [Firebase](http://firebase.google.com) (by now I assume you already have an account)
2. Just follow the instructions there and finally copy the `google-services.json` into [the root of the client directory.](https://github.com/qabbasi/Loudhailer/tree/master/LoudHailerClient) 
