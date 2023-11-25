# Debug Rails UJS - Invalid authenticity token error

Some of legacy project I’ve worked on few years ago received new requirement. We should be able to `Destroy` some of the entities from the index page. It sounds like a no-brainer CRUD requirements. But after adding the one liner well known for all Rails devs:
```erb
  = link_to 'Destroy', account, method: :delete, data: { confirm: 'Are you sure?' }
```

I've noticed something strange - **request method isn't right and the data confirmation modals isn't showing**. I've quickly realized that I am missing the Rails UJS initialization in my `application.js`:
```javascript
import Rails from '@rails/ujs';
Rails.start();
```

After adding that request method and the confirmation windows were displayed. Feature like this shouldn't take more than 5 minutes so aren't we already done? Unfortunately, not this time.

For my surprise the controller `destroy` method started throwing `Invalid authenticity token` error afterwards.

What was the cause? The error message told me only one thing - I am missing the `CSRF token` in request payload. But why was the CSRF token even required? It's a simple `link_to`!

To find the answer I had to realize one thing: the `method: :delete` isn't a HTML supported tag. You can't change link method using only HTML. So when, how it's done? It must be related to the Rails UJS library right?

Bingo! Long story short - when you click a link with defined `method` parameter Rails UJS will catch your request. Create a virtual form setting it's method to the one specified in `link_to`. And finally submit this form. That's why you might get the `Invalid authenticity token` error.

I strongly recommend reading [this](https://www.ombulabs.com/blog/learning/javascript/behind-the-scenes-rails-ujs.html) article about the details of Rails UJS. I've used this library a lot but in fact never realized what's the magic behind it.

The point were above, but if you still want some solution for the `Invalid authenticity token` error, just not forget as me about the
```erb
<%= csrf_meta_tags %>
```
in your application layout (`application.html.erb`).

The `csrf_meta_tags` are used by Rails UJS to set the `csrf` input in the virtual form which is submitted when `method` is used on the link.
