# 3 ActiveRecord Mistakes That Slow Down Rails Apps
#public/github/rails

[3 ActiveRecord Mistakes That Slow Down Rails Apps: Count, Where and Present](https://www.speedshop.co/2019/01/10/three-activerecord-mistakes.html)

* Ideally, a Rails controller action should execute one SQL query per action
	* NewRelic is a great way to monitor the number of queries!
* Usually, when a query is executed halfway through a controller action (somewhere deep in a partial, for example) it means that you haven’t `preloaded`the data that you needed.
* 


## `count`
* `count` on an ActiveRecord  relation executes a `COUNT` **every time**
	* only use `count` when you want to execute a count _right now_
* The most common cause of unnecessary count queries is when you count an association you will use later in the view (or have already used):
```ruby
# _messages.html.erb
# Assume @messages = user.messages.unread, or something like that

<h2>Unread Messages: <%= @messages.count %></h2>

<% @messages.each do |message| %>
blah blah blah
<% end %>
```
	* this executes two queries, a `COUNT` and a `SELECT`
		* `messages.count` does the count
		* `messages.each` does the select
	* changing `.count` to `.size` eliminates the `COUNT` query
		* `size` checks if the relation has been loaded
```ruby
# File activerecord/lib/active_record/relation.rb, line 210
def size
  loaded? ? @records.length : count(:all)
end
```
	* `count`, on the other hand, calls `calculate` which doesn’t memoize anything
``` ruby
def count(column_name = nil)
  if block_given?
    # ...
    return super()
  end

  calculate(:count, column_name)
end
```
* in the `_messages.html.erb` example, changing `count` to `size` would still make two queries because the messages weren’t loaded yet. You could either:
	* move the `size` line to after the `each`, where the records are loaded, or
	* use `load` - `@messages.load.count` will cause the records to load immediately, instead of lazily
* devs should probably always use `size` _unless_ the only thing you want is the count, and you never load the relation besides that

## `where`
* `where` means filtering is done by the database
* `where` will _always_ try to execute a query
``` ruby
# app/views/posts/_post.html.erb
<% @posts.each do |post| %>
  <%= post.content %>
  <%= render partial: :comment, collection: post.active_comments %>
<% end %>
```
``` ruby
class Post < ActiveRecord::Base
  def active_comments
    comments.where(soft_deleted: false)
  end
end
```
	* this will cause a SQL query on every rendering of the post partial
	* using the `where` in a scope on a association (like `comments`) has the same effect
* Author has two rules
	* Don’t call scopes on associations when you’re rendering collections
	* don’t put query methods, like `where`, in instance methods of an ActiveRecord::Base class
### In scopes
* calling scopes on associations means we can’t preload
	* we can preload the comments, but we can’t preload the _active_ comments
	* this isn’t bad when you do it only once (on show routes, for instance), but it is bad when you do it on a collection (N+1)
* creating a new association can fix this
``` ruby
class Post
  has_many :comments
  has_many :active_comments, -> { active }, class_name: "Comment"
end

class Comment
  belongs_to :post
  scope :active, -> { where(soft_deleted: false) }
end

class PostsController
  def index
    @posts = Post.includes(:active_comments)
  end
end
```
	* with the view unchanged, this will only have two queries! One on the posts table, and one on comments
### In base classes
