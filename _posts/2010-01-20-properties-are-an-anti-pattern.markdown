---
layout: post
title: Properties are an anti-pattern
published: true
---

A while back I attended a course with [Greg Young](http://codebetter.com/blogs/gregyoung/) focusing on [Domain Driven Design](http://domaindrivendesign.org/) and command and query separation (CQRS). Here Greg was talking about how properties on your entities can be an [anti-pattern](http://en.wikipedia.org/wiki/Anti-pattern). This got me thinking, and I must say that I agree, not only in a DDD context, but as good object oriented design principle.

If you are using the same entities for both querying and updating (commands), you can’t get rid of the properties all together, but you can limit their use to querying by making them read-only.

Let me explain by giving an example.

## Model behavior not state

Take a prescription object with one property, <code>CeasedAt</code>. (Ceasing a prescription means aborting the treatment.) To cease a prescription we simply set <code>DateTime.Now</code> on this property. Fine, that works.

{% highlight csharp linenos %}
public class Prescription
{
    public DateTime? CeasedAt { get; set; }
}
{% endhighlight %}

Then of course the requirements changes. We now have to specify a reason why the prescription is ceased. We add a new property to the prescription object, <code>CeasedBecauseOf</code>.

{% highlight csharp linenos %}
public class Prescription
{
    public DateTime? CeasedAt { get; set; }
    public string CeasedBecauseOf { get; set; }
}
{% endhighlight %}

Fine, but how can we ensure that this is set every time we set the <code>CeasedAt</code> property? Well, the problem is that we can’t. All the developers must know this when using this class, and let’s face it, there is a very small chance of that happening, at least over a period of time. Of course we can add a comment describing the requirement, but I don’t like comments. Not everybody reads them, and they tend to get outdated over time as the code changes and the comments doesn’t. I want the code to express it self.

So let’s try to express this requirement using nothing but code. We keep the <code>CeasedAt</code> property but make the setter private. Then we add a method, <code>Cease</code>.

{% highlight csharp linenos %}
public class Prescription
{
    public DateTime? CeasedAt { get; private set; }

    public void Cease()
    {
        if (CeasedAt != null)
            throw new InvalidOperationException("Prescription is already ceased.");

        CeasedAt = DateTime.Now;
    }
}
{% endhighlight %}

Much better! Not only is the code now both easier to understand as well as use, but it’s also easier to change since we have encapsulated how we cease a prescription. We are also protected against somebody somewhere accidentally setting <code>CeasedAt</code> on a already ceased prescription.

When requirement changes we simply add reason as a parameter to the <code>Cease</code> method.

{% highlight csharp linenos %}
public class Prescription
{
    public DateTime? CeasedAt { get; private set; }
    public string CeasedBecauseOf { get; private set; }

    public void Cease(string reason)
    {
        if (CeasedAt != null)
            throw new InvalidOperationException("Prescription is already ceased.");

        if (string.IsNullOrEmpty(reason))
            throw new ArgumentNullException("reason");

        CeasedAt = DateTime.Now;
        CeasedBecauseOf = reason;
    }
}
{% endhighlight %}

This will ensure at compile time that the requirement is fulfilled. The code also expresses the requirements more clearly. When we cease a prescription we have to specify a reason. In the previous example we could not know that from just reading the code, and therefore we were running a risk of introducing a bug if not all developers on our team was familiar with this requirement.

## Don’t Repeat Yourself

One other problem with the first example where we weren’t encapsulating how to cease a prescription is that we violate the [DRY (Don’t Repeat Yourself) principle](http://en.wikipedia.org/wiki/Don't_repeat_yourself). Let’s say that we can cease a prescription ten different places in our program, we then have ten places where we actually specify how to cease a prescription: by setting a property. I can almost guaranty you that this will lead to a future bug report, or at the very least, fragile and hard to change code.