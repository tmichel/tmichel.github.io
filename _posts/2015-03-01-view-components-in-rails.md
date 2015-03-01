---
layout: post
title: View components in Rails
---

I started poking around with Rails just after the release of 3.1. Since then not much changed in how we approach the view layer. We use the same abstraction or to be more precise we are stuck with the lack of abstractions and doing everything on the lowest possible level of string manipulation.

The interesting thing about this is how far we can go without noticing the lack of available tools. With templates, partials and myriads of built-in helpers we can get by for a long time. But there always comes a moment when our partials become too complicated and stuffed with logic or coupled to the context so much that  it makes it really hard to reuse them.

When this moment came I started wishing for view components. The Rails-world is so big, there must have been somebody else who ran into this and came up with an elegant solution. To my surprise I couldn't find one that fits all my needs.

## What are view components anyway?

Before I go any further I'd like to take a moment and explain what view components are (at least for me). My first exposure to web application development was a framework called [Wicket][1] and I tend to think this is where my definition of view component comes from. Wicket tries really hard to bridge the gap between desktop and web development, but at the end it fails miserably. It is a pain to use but the component concept it promotes is ingenious.

A view component should be ...

* completely self-contained, which means it can be rendered anywhere on its own
* easily extendable (think OO inheritance)
* reusable across your application (or even between applications)

This means I can have a generic table component to render `<table>` elements and I can use that directly in my views. But nothing holds me back from pulling that out into its own component and create an `ArticleTable` that can easily render itself anywhere and the only thing it needs is a list of articles. The `ArticleTable` could use the template from its parent or provide its own if absolute control is needed over how it should be rendered.

## The Rails way

What I described above is simply not possible with Rails. The best you can do is create helpers or [helper classes][2] which spit out some string. This way you lose the flexibility and clarity of templates because you have to shove pieces of html into your ruby class or use the clunky `content_tag` that Rails provides.

Another option could be a partial with a backing (view)model (or presenter if you like). This is a bit better but there is a huge disconnect between the template and the view model/logic. You have to remember to pass in the right view model every time you want to use your partial + view model combo. Not to mention that the syntax is pretty ugly:

~~~ruby
render partial: 'articles/table', locals: { articles: ArticleTableViewModel.new(@articles) }
~~~

## Arbre

[Arbre][3] basically fell out of [ActiveAdmin][4] for the same reasons this article is being written. Arbre has a few warts and bugs but it is really usable and provides most of the abstractions you need, and also provides a [straightforward way to create components][5].

Unfortunately this is an all-in solution. It comes with its own `Arbre::Context` which makes it really hard to integrate with existing template engines. You basically have to drop erb or haml or whatever you've been using. This makes it virtually impossible to integrate it into an existing application.

The closest I could get with embedding it into an erb template was to instantiate a new `Arbre::Context` within Rails' view context.

~~~ruby
# in helpers/application_helper.rb
def arbre(&block)
  Arbre::Context.new(assigns, self, &block)
end
~~~

And then use this in a view:

~~~erb
<div id="container">
  <%= arbre { article_table(articles) } %>
</div>
~~~

But this is where integration ends. The following is not working:

~~~erb
<div id="container">
  <%= arbre do %>
    <% panel 'Article panel' %>
  <% end %>
</div>
~~~

Arbre is full-on ruby - your templates become ruby scripts that only vaguely resemble html. It takes some time to get familiar with how it works and how to work around its quirks (eg. using Rails' built-in helpers).

~~~ruby
# in app/views/articles/show.html.arb
div class: 'container' do
  div class: 'row' do
    h3 article.title
    text_node link_to('Edit', edit_article_path(article))
  end
end
~~~

And it gets a little worse when you write your own components and you have to put the template code into a component's `build` method.

From the Arbre docs:

~~~ruby
class Panel < Arbre::Component
  builder_method :panel

  def build(title, attributes = {})
    super(attributes)

    h3(title, class: "panel-title")
  end
end
~~~

Arbre is a good choice when you use it inside ActiveAdmin but for general purposes it presents you with more challenges than solutions.

There is a very similar project called [erector][6]. I haven't looked into it properly but it seems to have the same problem of mixing "template" code into ruby classes. I think a clear distinction between the template and logic is really useful.

## Cells

[Cells][7] describes itself as _view components for Rails_ but it has a slightly different definition of what a component is. It naturally plays well with existing solutions and really easy to add it to an application. When you have a `CommentCell` you can render that in a template with the `cell` helper method.

~~~erb
<div class="container">
  <%= cell(:comment, @comment) %>
</div>
~~~

It does a superb job of separating the presentation from the view logic: every cell has its own templates and a cell class which is basically the view model.

Cells is basically a way to replace Rails' partials with maintainable and self-contained components. It falls short when you try to create generic components such as a table or a panel. The `cell` helper does not accept a block and you cannot `yield` from a cell's template. Nick Sutterer, the author of Cells, has [lots of great articles about how Cells works][8]. I especially liked the one which showcases Cells with [a sidebar component][9].

Cells gets so many things right but the ability to easily create generic components is sorely missing. Also there are a few conceptual things that I disagree with. A cell is basically a small MVC stack (in Rails kinda way) so it can have different actions or states (in Cells terminology) and these all have corresponding templates. I think that a view component should have only one representation.

In cells you could do the following:

~~~ruby
class CommentCell < Cells::ViewModel
  def show
    # renders comment/show.erb
    render
  end

  def flagged
    # renders comment/flagged.erb
    render
  end
end

# in a view
<%= cell(:comment, @comment).call(:flagged) %>
~~~

I would much rather see two separate classes since I already have to manually decide which state I want to render.

## The ideal solution?

There is no silver bullet. I couldn't find an existing gem that would solve all my problems. It seems that at the end I have to get dirty and write my own view components library. In the meantime Cells is the best option I have.

This post became quite lengthy but I still have some thoughts on the subject. It is very likely I will post a follow up with a wish list for a view components library and some code experiments.


[1]: https://wicket.apache.org/
[2]: http://teotti.com/building-intention-revealing-ruby-on-rails-helpers/
[3]: https://github.com/activeadmin/arbre
[4]: https://github.com/activeadmin/activeadmin
[5]: https://github.com/activeadmin/arbre#components
[6]: https://github.com/erector/erector
[7]: https://github.com/apotonick/cells
[8]: http://nicksda.apotomo.de/?s=cells
[9]: http://nicksda.apotomo.de/2010/11/lets-write-a-reusable-sidebar-component-in-rails-3/
