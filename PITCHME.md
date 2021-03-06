---?image=img/title.jpg&size=cover
<style>
.reveal .slides {
    text-align: left;
}
.reveal .slides h1 {
  font-size: 3.75rem;
}
.reveal .slides section>* {
    margin-left: 0;
    margin-right: 0;
}
</style>

# The Importance of Ports

## Murphy Randle
- Work: [Day One](https://dayoneapp.com)
- Twitter: [@splodingsocks](https://twitter.com/splodingsocks)
- Elm Town Podcast!

Note:
- I'm Murphy Randle
- I'm the host of Elm Town
- and I work at Day One where we make a journaling app.
- Today I'm going to show you why ports are so important to us at Day One.
- But first:

---?image=img/story-time.jpg&size=cover

# Story Time
## or, how this talk was born

<span style="font-size: .75rem; color: rgba(255,255,255,0.5); position: absolute; bottom: 1rem; left: 1rem">
Photo by Dariusz Sankowski on Unsplash
</span>

+++

## The Situation
- Client-side database |
- Hand-written native code to wrap DB |

## The Sadness
- Runtime errors 😭 |
- No compiler help |

+++

## The Conversation
- Mentioned in Slack bad experience with native code |
- Mentioned that using ports would be even worse |

## The Change of Perspective
- Evan replied |
- I had been using an incorrect mental model of ports |
- I revised my brain and re-architected |

+++

# The Result
- No more native code |
- 💸💰🤑💸💰🤑💸💰🤑 |
- Elm & JS remain separate worlds with separate concerns |
- Scaling JS-Elm interaction is simple (a simple pattern is simple to follow) |

---

# Let's Build Something

[pocketjournal.splode.co](https://pocketjournal.splode.co)
- Users can create, edit, and delete entries |
- Data stored locally in IndexedDB |

Note:
- No IndexedDB package available for Elm
- We're going to need ports for this
- They're what allow Elm to talk back and forth with JS

+++

> [Ports are] like JavaScript-as-a-Service

from guide.elm-lang.org

Note:
- Elm guide says javascript as a service

+++

# Outgoing port
```elm
port doSomethingInJsWithAString : String -> Cmd msg
-- That's it, no body definition
```

# Incoming Port
```elm
port stringsComingFromJs : (List String -> msg) -> Sub msg
-- That's it, no body definition
```

Note:
- Let's just go ahead and take a stab at that approach
- Warning, this is not the correct approach.

---

# First [wrong] Attempt

+++?code=example/src/BadPortExample.elm&lang=elm

@[4-11](Make a port for each request and its accompanying response)
@[14-18](Plus a new message for each one of the "responses" above)
@[21-29](Plus a case for each of those "response" messages in the update function)
@[32-36](Plus a new subscription fo reach of those "reponse" messages)
@[39-50](Plus JS code to subscribe to and trigger each of these ports)

+++

# What makes this hard and unpleasant?

+++

- Adding a JS operation requires two new ports
- Adding a JS operation potentially requires changes across 3+ files |
- Request + response ports don't feel natural |
  - What if a response comes when there's no request? |
  - What if a response comes back multiple times? |
  - When a response comes through, how do I connect it with a particular request? |

+++

# JavaScript as a service

- Ports make JavaScript == Service
  - BUT 🛄
    - Service `/=` server
    - Ports `/=` HTTP + Promises

Note:
- Coming from JS and hearing something "as-a-service" may make me think we're talking about promises and HTTP requests
- That's incorrect here
- Service is not a server
- Ports are not promises
- They follow a different model entirely:

---

# The Actor Model

Note:
- The actor model

+++

# The Actor Model

![](img/actor-model.jpg)

Note:
- The actor model?

+++

# The Actor Model

The interaction between Elm & JS is modeled on this mature design pattern made for concurrent systems.

+++

This is why the request / response port pairs felt awkward.

+++

~~request / response~~
<br>
Information in / Information out

+++

Sort of like chatting in Slack!

+++

For one message out, we can get many in:
![](img/port-msg-example-one-out-many-in.png)

+++

Or none in:
![](img/port-msg-example-multiple-out-no-in.png)

+++

Or many out, and just one in:
![](img/port-msg-example-2-out-1-in.png)

+++

Messages in may be out of order:
![](img/port-msg-example-out-of-order.png)

+++

Or they may (practically) never come:
![](img/port-msg-example-pending-forever.png)

+++

And we might never send any out, but still get some in:
![](img/port-msg-example-many-in.png)

+++

This is a good thing. 👍
And we're not the first to the party.

+++

# Languages / Frameworks / APIs that use the Actor model

- Erlang / Elixir |
- Web Workers (in the HTML Standard) |
- Scala (Akka) |
- Python (using Pulsar) |
- and more! |
- Listen to Elm Town 13 for more background |

---

# Second Attempt

+++

Let's simplify. Use just one port.
- Don't "wrap" some particular JS library API |
- Design an API that makes sense for talking with JS |

+++

Let's simplify more and let JS own all the data.
- Notify the DB of changes to enties |
- When changes happen, the DB sends all entries back to Elm |

+++

# Designing a pattern for an API
- And let's call it the "Outside Info" pattern|

Note:
- Your API will be specific to each application you write
- This is just a pattern for designing that API

+++?code=example/src/OutsideInfo.elm

@[57-60](Just two ports now, for the whole app)
@[53-54](One generic data structure for sending information between worlds)
@[42-46](Any and all messages to the outside appear in this type)
@[49-50](Any and all messages that can come from the outside appear in this type)
@[8](A single function for communicating with JS that produces a command)
@[11-12](Handles all different message types and converts them to OutsideInfo)
@[24](A single subscription for getting info from JS)
@[29-32](Decodes data into a message based on the tag in OutsideInfo)
@[34-35](If a message comes along we don't understand, we can log it, or show an error in the UI)

+++?code=example/src/index.js

@[6](Subscribe to the single Elm -> JS port)
@[9](Make a new if / else branch for each type in InfoForOutside )
@[50](Send in a message to Elm for any type in InfoForElm)

+++

# The "Outside Info" Pattern In Summary

- One port |
- One data structure for moving information across language borders |
  - Serialize data on exit |
  - Deserialize data on entrance |
- Send info, receive info, no requests/responses |

---
# Why the Actor Model? A Case Study
+++?image=graphics/Before@2x.png&size=contain
Note:
- We had a problem
- We were downloading so much data while syncing journals that it totally locked up the UI
- The solution was to use workers and move that into a thread
- Dreaded the big refactor and expected lots of changes
- But:

+++?image=graphics/After@2x.png&size=contain

Note:
- Web workers use the same actor-model style API and message passing
- All we had to do was move the code to a worker, and the main JS thread just became a message broker between Elm and the worker process
- And here's how it looks in real life:
+++

# DEMO TIME

---

# That's It!
