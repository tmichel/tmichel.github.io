---
layout: post
title: "Object-oriented controversies: Tell Don't Ask vs the Web"
---

Lately I have been thinking a lot about object-oriented principles and ways to better apply them in my day-to-day work. The more I think the more questions I have. Principles and best practices contradicts each other more often than not. This post is not a rant but rather me trying to understand some portions of object-oriented programming.

The mainstream view of OOP seems to focus on classes and inheritance while the original interpretation is more about message passing and the [Tell Don't Ask][tda] principle.

> Procedural code gets information then makes decisions. Object-oriented code tells objects to do things.
>
> -- Alec Sharp

Basically the Tell Don't Ask principle says that you should tell objects to do things and not query their internal state, make a decision and _then_ tell them what to do.

Not so good:

~~~ruby
def sound_alarm(alarm)
  if alarm.enabled? && alarm.working?
    alarm.start_siren
  end
end
~~~

Better:

~~~ruby
class Alarm
  def sound
    if enabled? && working?
      sound_siren
    end
  end

  # ...
end

alarm.sound
~~~

This seems quite natural as one of the core principles of OO is that data and behavior should be co-located: a behavior is "responsible" for the data that it needs to operate on.

On its own Tell Don't Ask seems to be very close to the One True Way. Locality, encapsulation and data hiding are very pleasant side effects. Code written this way is very readable and the [refactoring from the "old" way][tda_refactor] is pretty straightforward.

Things start to go south when you think about how to apply it in the context of web applications or more specifically in the context of Rails. You should put behavior where the data is and in a web application the most important data resides in models. It does make sense to couple data and behavior. For example if I query the user's name the returning `String` object does not know how to make sense of itself. It is trivial that it should not know. The `user` object should know what its `name` actually means. The Tell Don't Ask principle says that instead of asking the user for its name I should tell the user things to do with its name.

Following this logic basically every behavior would end up in the model classes. This is something that we know (by experience) is a bad idea. Hundreds or thousands of lines of code crammed into a single class is a recipe for disaster. [Service Objects][service_object] grew out of this frustration.

Service objects are great. It takes some time to shift your mind that classes can actually be verbs and instead of representing a piece of the real world they represent actions in the real world. At first I thought service objects (with names like `CreateOrder`) were abominations. You get a strange feeling about using verbs for classes, after all real world object and real world actions are different beasts. But with time I grew to appreciate service objects for their simplicity. They nicely obey the [Single Responsibility Principle][srp], they are basically glorified functions with their own internal state.

Let's look at a trivial example:

~~~ruby
class MentionUser
  attr_reader :post

  def initialize(post)
    @post = post
  end

  def call(user)
    return unless user.has_access_to?(post)
    return unless post.published?

    if post.content.include?("@#{user.nick}")
      post.watchers << user
      UserNotificationMailer.mentioned_in_post(user, post).deliver_later
    end
  end
end
~~~

This is nice and easy. All the behavior is in one place, we extracted this from the `User` model. It is testable and it has a clear responsibility. From the point of Tell Don't Ask this is really bad. It queries objects for their internal state and then tells them what to do based on the queried data.

Nowadays the general wisdom is to keep your models thin (to act as a gateway layer to your database) and if possible only contain associations and very trivial derived properties. All the business related logic should go into service objects and kept as a separate layer. Martin Fowler [wrote about this phenomenon][anemic_model] about 12 years ago and called it an anti-pattern. As always, he is quite right. Service objects are a lot closer to procedural programming than object-oriented programming although they work surprisingly well.

Now we have two concepts that contradict each other. Tell Don't Ask is really sensible but service objects are based on hard-earned experience. Some people tried to [close the gap between them][service_object_event] but it only addresses one level: the controller uses the Tell Don't Ask principle on service objects but service objects are still querying models for their internal state and make decisions based on them.

At this point I was really stuck and it seemed that this conflict is impossible to resolve. After quite some thinking I realized that I treated the model as a fixed piece of the puzzle as if it cannot be moved or modified. ActiveRecord models are a very crude view of the data that makes up the domain of the application. It is not just ActiveRecord though, we could also mention JPA and its entity beans but .NET EntityFramework is also guilty as charged.

A user model is a very general view of our domain's user concept. When we interact with a user we don't always need every information that is associated with our user concept. For example for authenticating a user we really only need its username and password. What if we had a model to capture this. A small and very focused model that could hold the data and the behavior at the same time but adhere to SRP. It is certainly not a revolutionary idea but realizing that models should not be tied to database tables gives you a new perspective.

Suddenly ActiveRecord seems more like a constraint and not a convenient way to access the database. A more flexible mediation layer might be a better option where I can easily map database tables (even from different sources) to my domain models. [ROM][rom] would be a great candidate here. Splitting up our domain concepts by context is taking us awfully close to [DCI][dci] which is a huge topic that I'm not going to touch on.

Conclusion? As always there is no silver bullet. Service objects (with their procedural style) could work as well as the Tell Don't Ask approach. A touch of pragmatism is needed to evaluate the different approaches and use a mixture that fits our needs. Although my gut feeling tells me that mixing too many approaches would lead to the same spaghetti code that we strive to avoid.

Thanks [Balazs Varga][balo_blog] and Zsofia Langi for reviewing.

[tda]: https://pragprog.com/articles/tell-dont-ask
[tda_refactor]: https://robots.thoughtbot.com/tell-dont-ask
[service_object]: http://brewhouse.io/blog/2014/04/30/gourmet-service-objects.html
[srp]: https://en.wikipedia.org/wiki/Single_responsibility_principle
[anemic_model]: http://www.martinfowler.com/bliki/AnemicDomainModel.html
[service_object_event]: http://blog.jasonharrelson.com/blog/2014/03/04/ruby-oop-events-and-the-tell-dont-ask-principle/
[rom]: http://rom-rb.org/
[dci]: https://en.wikipedia.org/wiki/Data,_context_and_interaction
[balo_blog]: http://blog.vbalazs.me/
