---
# Feel free to add content and custom Front Matter to this file.

layout: default
paginate:
  collection: posts
  per_page: 5
---

<% paginator.resources.each do |post| %>
<article>
    <h3><%= link_to post.data.title, post.relative_url %></h3>
    <div><small><%= post.date.strftime('%B %d, %Y') %></small></div>

    <section>
        <%= post.summary %>
    </section>
</article>
<% end %>

<% if paginator.total_pages > 1 -%>
  <ul class="pagination">
    <% if paginator.previous_page %>
    <li>
      <%= link_to "Previous Page", paginator.previous_page_path %>
    </li>
    <% end %>
    <% if paginator.next_page %>
    <li>
      <%= link_to "Next Page", paginator.next_page_path %>
    </li>
    <% end %>
  </ul>
<% end %>
