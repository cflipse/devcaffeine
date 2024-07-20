---
# Feel free to add content and custom Front Matter to this file.

layout: default
paginate:
  collection: posts
  per_page: 10
---

<% paginator.resources.each do |post| %>
<article>
    <h3><%= post.data.title %></h3>
    <section>
        <%= post.summary %>
    </section>
</article>
<% end %>
