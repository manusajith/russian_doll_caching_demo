##Rails 4 Released##

As promised by [@tenderlove](http://twitter.com/tenderlove "@tenderlove") during [#rubyconfindia](https://twitter.com/search?q=%23rubyconfindia&src=typd&mode=realtime "rubyconfindia") held in Pune, the rails core team has finally [released Rails 4](http://weblog.rubyonrails.org/2013/6/25/Rails-4-0-final/ "Rails 4 Released"). Huge credits to [@dhh](http://twitter.com/dhh "@dhh"), [@tenderlove](http://twitter.com/tenderlove "@tenderlove") and all other contributors for giving us this wonderful framework and consistantly improvising its features. One thing that make me love rails the most is the community support. As [@dhh](http://twitter.com/dhh "@dhh") mentions in the weblog there were more than [10000 commits](https://github.com/rails/rails/compare/3-2-stable...4-0-0 "10000 commits") by over more than [500 developers](http://contributors.rubyonrails.org/contributors/in-time-window/this-year "500 developers"). Really amazing how the community supports this framework.

There are some noticeable chagnes in the the new version of rails.
Some really cool features like Russian Doll Caching, Turbolinks, Strong Parameters etc has managed to find place in the core while some of the lesser used were extracted as gems. One of the key focus in the  latest verison of rails has been to make the framework simpler and faster. The inclusion of turbolinks and russian doll caching will have a good role to play in this context.

##Upgrade to Rails 4##

Upgrading your rails version from rails 3.2 to rails 4.0 is very easy. First you need to install rails 4.0 in your machine using the following command

        gem install rails --version 4.0.0 --no-ri --no-rdoc

##Porting to Rails 4##

If you are planning to port an existing application to rails 4 then you need to take care of some existing configurations that might have been modified or even removed in rails 4. You can find more details and the guide [here](http://edgeguides.rubyonrails.org/upgrading_ruby_on_rails.html#upgrading-from-rails-3-2-to-rails-4-0 "Rails 4 Upgrade guide"). Also some of the gems that are being used now may have some issues with Rails 4. So it is upto you to decide whether to upgrade or not.

##Russian Doll Caching##

Russian doll caching is an advanced caching mechanism that is being used in rails 4. In Russian doll caching the fragment caches are nested so that it can be reused.

For demonstrating this lets consider a simple blog application where we have many posts and associated comments. Assuming we have a Post model for all the articles posted and a Comment model which belongs to the posts. For simplicity I havn't included the validations in the model for the example given below.


        class Post < ActiveRecord::Base
          has_many :comments
        end
        
        class Comment < ActiveRecord::Base
          belongs_to :post
        end


Fragment cache can be implemented on this app by using the following code.

        #post view
        <!-- app/views/posts/show.html.erb -->
        <% cache @post do%>
            <h1><%= @post.title %></h1>
            <div>
                <%= @post.content %>
            </div>
            <%= render @post.comments %>
        <% end %>

        #partial for rendering the comments
        <!-- app/views/comments/_comment.html.erb -->
        #cache comment is used because it helps to identfiy the written file as comments
        <% cache comment do %>
            <div>
                <p><%= comment.name %></p>
                <p><%= comment.content %></p>
            </div>
        <% end %>



Here we cache and render the post in the show.html.erb file, also I used a partial for rendering the comments for that particular post. Here the comments are nested inside the post view.

In this case the cache would be written as:


views/posts/1-20130628004417

views/comments/2-20130628004417

views/comments/2-20130628004417

Here the individual comments need not be expired everytime. We can re-use these individual comments if we are using nested fragment cache.

Here if we obeserve the filename, we could find that the first part is the view path, second part is the model name and the last part is the timestamp at which the cache is generated.

ie in general: view_path/model/id-timestamp

##Problem##

One of the disadvantages of using fragment caching is that, everytime we change the template we need to invalidate the cache as well. So it was always hard to maintain the fragment caching.

##Work around##

One of the workarounds for this problem was adding a version no: to your template files. So that each time you modify the template, you need to bump the verison so that there is no conflict.

Eg: In the initial step the version of the template would be "v1"
        
        <!-- app/views/posts/show.html.erb -->
        <% cache ["v1", @post] do%>
            <h1><%= @post.title %></h1>
            <div>
                <%= @post.content %>
            </div>
            <%= render @post.comments %>
        <% end %>
        
        #partial for rendering the comments
        <!-- app/views/comments/_comment.html.erb -->
        <% cache ["v1",comment] do %>
            <div>
                <p><%= comment.name %></p>
                <p><%= comment.content %></p>
            </div>
        <% end %>


Now the cached files would be as follows:

views/v1/posts/1-20130628011230

views/v1/comments/1-20130628011230

views/v1/comments/2-20130628011230

ie in general view_path/version/model/timestamp

So in due course if we need to modify the template the we need to bump the version of the template to "v2" so that the cache is invalidated.
Eg If we changed the heading of post from `<h1>` to `<h2>` and changed the styling in the comments name to `<b>` we need to upgrade the version of the template also.

So the above code should be modified to as follows:

        <!-- app/views/posts/show.html.erb -->
        <% cache ["v2", @post] do%>
            <h2><%= @post.title %></h2>
            <div>
                <%= @post.content %>
            </div>
            <%= render @post.comments %>
        <% end %>
        
        #partial for rendering the comments
        <!-- app/views/comments/_comment.html.erb -->
        <% cache ["v2",comment] do %>
            <div>
                <p><b><%= comment.name %></b></p>
                <p><%= comment.content %></p>
            </div>
        <% end %>

Now the new cached files generated would be as follows:

views/v2/posts/1-20130628012420

views/v2/comments/1-20130628012420

views/v2/comments/2-20130628012420

So if we forgot to bump the version of our template files then the cached version would be served. This was one of the major drawbacks of using fragment caching.

##Solution##
Rails 4 has a easy and neat way of resolving this. Thanks for [@dhh](http://twitter.com/dhh "@dhh") for open sourcing the [cache_digests](cache_digests "cache_digests") gem, which effectively solves the above mentioned issue. Every time the pages are cached, a MD5 digest of the template file will be generated and will be appended to the file name so that the digest remains unique. Hence at no point of time a conflict will occur.

So in our case the genrated cache fiels would be as follows:

views/posts/1-20130628012420/f13bb1bed03db9d68a7d9a48aafeec78

views/comments/1-20130628012420/cb59608fced567a14b13a6e5c5c8a1d2

views/comments/2-20130628012420/295bdf147d837711c36b3e3bf9ec9957

At no point of time the MD5 of the template will be the same. Hence at no point of time there would be a conflict between the generated caches.


