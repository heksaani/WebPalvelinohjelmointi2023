**Note: This part of the course is still in beta testing. You can already try the material and exercises, but they cannot be submitted yet and the exercises can change before they are released.**

You will continue to develop your application from the point you arrived at the end of week 7. The material that follows comes with the assumption that you have done all the exercises of the previous week. In case you have not done all of them, you can take the sample answer to the previous week from the submission system.

## Hotwire

Ruby on Rails version 7.x introduces a new functionality called [Hotwire](https://hotwired.dev/), aimed at simplifying the creation of dynamic views with minimal reliance on JavaScript. Hotwire empowers Rails developers to incorporate partial reloading of user interface elements in a similar fashion to popular JavaScript libraries like [React](https://react.dev/), all while leveraging the familiar syntax of the Ruby language.

### Why Hotwire?

Throughout its history, the Rails framework has been renowned for its ability to enable the rapid development of **fully-featured applications**. However, over the past 15 years, the concept of a _"fully-featured application"_ has evolved significantly. Today's users have come to expect dynamic, mobile-friendly experiences that encompass faster, partial, and interactive page loads.

To meet these expectations, developers have had to rely on additional software tools, such as the React library, to build the necessary functionality. Unfortunately, this approach adds complexity to the applications and often diminishes the role of the View component within the Rails [MVC](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller) architecture, reducing it to merely serving as a [REST](https://en.wikipedia.org/wiki/Representational_state_transfer) or [GraphQL](https://en.wikipedia.org/wiki/GraphQL) API.

With the introduction of Hotwire, Rails aims to tackle the challenges posed by the rapidly evolving JavaScript landscape. This is in contrast to the more steadily-paced Ruby ecosystem, which tends to favor incremental and conservative evolution. Hotwire offers a more streamlined and cohesive approach to fulfilling the requirements of full-featured, full-stack applications. It achieves this by eliminating the reliance on disparate tools and aligning with the ethos of Rails as a comprehensive platform for web application development. Hotwire provides the tools to construct dynamic and interactive user experiences while maintaining consistency with the familiar Rails paradigms.

## Introduction to Hotwire Components

Hotwire encompasses three core components, each serving a specific purpose: Turbo, Stimulus, and Strada (not covered in this course).

1. **Turbo**

**Turbo Frames** and **Turbo Streams** enhance page loading speed by dividing the page into components and facilitating dynamic updates.

- **Turbo Streams over HTTP** - Delivering updates directly through HTTP responses, reducing page reloads.
- **Turbo Streams over Action Cable** - Real-time updates using Action Cable's WebSocket framework.
- **WebSockets with Turbo** - Bidirectional communication for real-time updates and interactivity.

2. **Stimulus**

Stimulus is a lightweight JavaScript framework that enhances interactivity and user interactions in server-rendered HTML views. By attaching JavaScript behavior to HTML elements, it improves the user experience without complex frameworks or extensive coding.

3. **Strada**

Strada is an extension of Hotwire that allows developers to build iOS and Android applications using Rails and Turbo. Currently, Strada is being developed as separate repositories: [turbo-ios](https://github.com/hotwired/turbo-ios) for iOS and [turbo-android](https://github.com/hotwired/turbo-android) for Android, respectively.

### Turbo frames, the first steps

Before we start, let us simplify our app a bit. Start by removing the mini profiler by deleting the following line from <i>Gemfile</i>

```
gem 'rack-mini-profiler'
```

and by running _bundle install_. 

Let us also remove all the code that is implementing the [server side caching](https://github.com/mluukkai/WebPalvelinohjelmointi2023/blob/main/english/week7.md#server-caching-functionality). So from the view templates, we get rid of all the the _cache_ elements that wrap the real page content:

```html
<h1>Beers</h1>

<% cache "beerlist-#{@order}", skip_digest: true do %>
  <div id="beers">
    <table class="table table-striped table-hover">
       ...
    </table>
  </div>
<% end %>
```

and from the corresponding controllers, the guards that prevent full page render should also be removed. Eg. in the controller/beers.rb the change is the following: 

```ruby
  def index
    @order = params[:order] || 'name'

    # remove this line:
    return if request.format.html? && fragment_exist?("beerlist-#{@order}")

    @beers = Beer.all

    @beers = case @order
             when "name" then @beers.sort_by(&:name)
             when "brewery" then @beers.sort_by { |b| b.brewery.name }
             when "style" then @beers.sort_by { |b| b.style.name }
             when "rating" then @beers.sort_by(&:average_rating).reverse
             end
  end
```

Now we are ready to begin!

Turbo Frames provide a convenient way to update specific parts of a page upon request, allowing us to focus on updating only the necessary content while keeping the rest of the page intact.

Let us add a turbo frame to the styles page:

```html
<h1>Styles</h1>

<div id="styles">
  <% @styles.each do |style| %>
    <p>
      <%= link_to style.name, style %>
    </p>
  <% end %>
</div>

<%= link_to "New style", new_style_path %>

<br><br>

<%= turbo_frame_tag "about_style" do %>
  <%= link_to "about", styles_path %>
<% end %>
```

The turbo frame is created with a helper function <i>turbo_frame_tag</i> that has a identifier as a parameter. 

The generated HTML looks like the following:

![image](../images/8-1.png)

So the frame has just a link element that points back to the page itself. 

Our intention is to show some infromation about beer styles within the turboframe when the user clicks the link.


Let us now create a partial */views/styles/_about.html.erb* that also has the same turbo frame id:

```html
<%= turbo_frame_tag "about_style" do %>
  <div class="card">
    <p>Beer styles differentiate and categorise beers by colour, flavour, strength, ingredients, production method, recipe, history, or origin.</i>
    <p>The modern concept of beer styles is largely based on the work of writer Michael Jackson in his 1977 book The World Guide To Beer. In 1989, Fred Eckhardt furthered Jackson's work publishing The Essentials of Beer Style. Although the systematic study of beer styles is a modern phenomenon, the practice of distinguishing between different varieties of beer is ancient, dating to at least 2000 BC.</p>
    <p>What constitutes a beer style may involve provenance, local tradition, ingredients, aroma, appearance, flavour and mouthfeel. The flavour may include the degree of bitterness of a beer due to bittering agents such as hops, roasted barley, or herbs; and the sweetness from the sugar present in the beer.</p>
    <p>Source <a href="https://en.wikipedia.org/wiki/Beer_style">Wikipedia</a></p>
  </div>
  <% end %>
```

Now when user clicks the link, that creates a GET request to the same url and the request is handled by the controller function _index_. We can use the helper function *turbo_frame_request?* to detect the turbo request and handle it accordingly:

```rb
class StylesController < ApplicationController

  def index
    if turbo_frame_request?
      # this was a request from the turbo frame
      render partial: 'about'
    else
      # this was a normal requesst
      @styles = Style.all
    end
  end

  // ...
end
```

So in case of a turbo request (that is link "about" is clicked), there is now full page reload but the partial <i>about</i> is rendered. 

As we see from the developer cosole, the response of the turbo request does not contain the full HTML of the page, only the HTML fragment that will be inserted to the turboframe:

![image](../images/8-2.png)

From the console we can also see, that the GET request caused by the link clicking within the frame has a special header that tells Rails controller to treat the request as a turbo request and not cause a full page reload:

![image](../images/8-3.png)

We are now using the index controller both for rendering the whole styles page and for the partial that renders the about information. Let us split rendering the partial to on own controller. The *routes.rb* extends as follows

```rb
Rails.application.routes.draw do
  resources :styles do
    get 'about', on: :collection
  end
  # ...
end
```

The controller cleans up a fit:

```rb
class StylesController < ApplicationController
  before_action :set_style, only: %i[show edit update destroy]

  def index
    @styles = Style.all
  end

  # own controller function for the partial
  def about 
    render partial: 'about'
  end
  # ...
end
```

The link is changed accordingly:

```html
<%= turbo_frame_tag "about_style" do %>
  <%= link_to "about", about_styles_path %>
<% end %>
```

Instead of having an individual view for each style, let us show the style details when clicking the style name on the list. We begin with wrapping the list in a turbo frame:

```html
<h1>Styles</h1>

<div id="styles">
  <%= turbo_frame_tag "styles" do %>
    <% @styles.each do |style| %>
      <%= link_to style.name, style %>
    <% end %>
  <% end %>

  <%= turbo_frame_tag "style_details" do %>
  <% end %>  
</div>
```

We add the following to partial *_details.html.erb*:

```html
<%= turbo_frame_tag "styles" do %>
  <h3><%= style.name %></h3>
  <p>
    <strong>Description:</strong>
    <%= style.description %>
  </p>

  <h4>beers</h4>

  <ul>
    <% @style.beers.each do |beer| %>
      <li>
        <%= link_to beer.name, beer %>
      </li>
    <% end %>
  </ul>  
<% end %>
```

Clicking a style name now causes a turbo frame request for a single style, the controller is altered to render the above partial in these cases:

```rb
class StylesController < ApplicationController
  # ...

  def show
    if turbo_frame_request?
      render partial: 'details', locals: { style: @style } 
    end
    # the default is a full page reload
  end

  # ...
end
```

Notice now that the partial is given the _@style_ as a variable!

Now when a style name is clicked, the list of styles is replaced with the details of a particular style.

### Targetting a different frame

This is perhaps not quite what we want. Instead, let the style list remain visible all the time, and add a new turbo frame (with id "style_details") where the details of the clicked style are shown:

```html
<div id="styles">
  <%= turbo_frame_tag "styles" do %>
    <% @styles.each do |style| %>
      <%= link_to style.name, style, data: { turbo_frame: "style_details" } %>
    <% end %>
  <% end %>

  <%= turbo_frame_tag "style_details" do %>
  <% end %>  
</div>

<%= link_to "New style", new_style_path %>
</div>
```

Since we now  want to target a different turbo frame instaad of the one where links (or othe component) reside, we must define the targetted frame as an attribute. As seen from the above snippet it is done as follows:

```html
link_to style.name, style, data: { turbo_frame: "style_details" }
```

The turbo frame tag in the partial _details.html.erb needs to be changed accordingly:

```html
<%= turbo_frame_tag "style_details" do %>
  <h3><%= style.name %></h3>
  <p>
    <strong>Description:</strong>
    <%= style.description %>
  </p>

  # ...
<% end %>
```

The result is finally as we expectedit to be:

![image](../images/8-4.png)

<blockquote>

## Exercise 1

Extend the user page so that when clicking a rating, the basic info of the rated beer are shown. Your solution could look like this

![image](../images/8-5.png)

</blockquote>

### Pagination

Before continuing with the Hotwire further, let's take a slight detour. After last week's [increased amount of beers](https://github.com/mluukkai/WebPalvelinohjelmointi2023/blob/main/english/week7.md#server-caching-functionality) you start to wonder that it would be kinda nice to have a pagination for our beers page. Let's add it first without utilizing Hotwire features.

First we start by adding links for previous and next pages to the end of our beers table:

**app/views/beers/index.html.erb**

```html
<table class="table table-striped table-hover">
  <thead>
  # ...
  </thead>
  <tbody>
    <% beers.each do |beer| %>
      # ...
    <% end %>
    <tr>
      <td colspan="2" class="text-center">
        <%= link_to "<<< Previous page", beers_path %>
      </td>
      <td colspan="2" class="text-center">
        <%= link_to "Next page >>>", beers_path %>
      </td>
    </tr>
  </tbody>
</table>
```

Our links don't do much yet so let's add some logic to the controller as well. Last week we defined ordering of the beers in our controller like so:

**app/controllers/beers_controller.rb**

```ruby
def index
  @order = params[:order] || 'name'

  @beers = Beer.all

  @beers = case @order
            when "name"    then @beers.sort_by(&:name)
            when "brewery" then @beers.sort_by { |b| b.brewery.name }
            when "style"   then @beers.sort_by { |b| b.style.name }
            when "rating"  then @beers.sort_by(&:average_rating).reverse
            end
end
```

Which contains a bit of a problem. Method `sort_by` will load all the beers to main memory as an array and only then sort the order. But now we would want to fetch only limited amount of records from the database at a time, only what is needed for the current page. There's no sense fetching all the beers. That's why we'll opt for using ActiveRecord SQL queries for the ordering.

Let us at the first forget about the different orderings and get the pagination to work for beers ordered by name. The controller changes as follows:

```ruby
class BeersController < ApplicationController
  PAGE_SIZE = 20

  def index
    @order = params[:order] || 'name'
    @page = params[:page]&.to_i || 1
    @last_page = (Beer.count / PAGE_SIZE).ceil
    offset = (@page - 1) * PAGE_SIZE

    @beers = Beer.order(:name).limit(PAGE_SIZE).offset(offset)
  end

  # ...
end
```

We are using a combination of ActiveRecord [order](https://edgeguides.rubyonrails.org/active_record_querying.html#ordering), [limit and offset](https://edgeguides.rubyonrails.org/active_record_querying.html#limit-and-offset) to control what page of the ordered beers is queried from the database. 

Pay attention to the `@last_page` instance variable as we are going to need it in our view. Now we are going to add the new `@page` instance variable to all of our links in the index and redefine our previous and next page links:

**app/views/beers/index.html.erb**

```html
<table class="table table-striped table-hover">
  <thead>
      <th><%= link_to "Name", beers_path(page: @page, order: "name")%></th>
      <th><%= link_to "Style", beers_path(page: @page, order: "style")%></th>
      <th><%= link_to "Brewery", beers_path(page: @page, order: "brewery")%></th>
      <th><%= link_to "Rating", beers_path(page: @page, order: "rating")%></th>
  </thead>
  <tbody>
    <% @beers.each do |beer| %>
      <tr>
        <td><%= link_to beer.name %></td>
        <td><%= link_to beer.style.name, beer.style %></td>
        <td><%= link_to beer.brewery.name, beer.brewery %></td>
        <td><%= round(beer.average_rating) %></td>
      </tr>
    <% end %>
    <tr>
      <td colspan="2" class="text-center">
        <% unless @page == 1 %>
          <%= link_to "<<< Previous page", beers_path(page: @page - 1, order: @order) %>
        <% end %>
      </td>
      <td colspan="2" class="text-center">
        <% unless @page == @last_page %>
          <%= link_to "Next page >>>", beers_path(page: @page + 1, order: @order) %>
        <% end %>
      </td>
    </tr>
  </tbody>
</table>
```

Pagination works now nicely with the default ordering! We need a bit more advanced use of ActiveRecord to get also the other orders to work.

When ordering based on brewery or style name, we can not just use the data in the beer object, we must do a SQL [join](https://edgeguides.rubyonrails.org/active_record_querying.html#joining-tables) to get the assosiated rows also from the database to do the ordering based on the fields of those. The contreller extends as follows:


```ruby
class BeersController < ApplicationController
  PAGE_SIZE = 20

  def index
    @order = params[:order] || 'name'
    @page = params[:page]&.to_i || 1
    @last_page = (Beer.count / PAGE_SIZE).ceil
    offset = (@page - 1) * PAGE_SIZE

    @beers = case @order
      when "name"    then Beer.order(:name)
        .limit(PAGE_SIZE).offset(offset)
      when "brewery" then Beer.joins(:brewery)
        .order("breweries.name").limit(PAGE_SIZE).offset(offset)
      when "style"   then Beer.joins(:style)
        .order("styles.name").limit(PAGE_SIZE).offset(offset)
    end

  end

  # ...
end
```

So now depending on the order user wants, a different kind of query is executed to get the beers.

The last one, ordering by ratings is the most tricky case. One way to achieve the functionality is shown below. The required SQL mastery is beyond the objectives of this course, so you may just copy paste the code and believe that it works.

```ruby
class BeersController < ApplicationController
  PAGE_SIZE = 20

  def index
    @order = params[:order] || 'name'
    @page = params[:page]&.to_i || 1
    @last_page = (Beer.count / PAGE_SIZE).ceil
    offset = (@page - 1) * PAGE_SIZE

    @beers = case @order
      when "name"    then Beer.order(:name)
        .limit(PAGE_SIZE).offset(offset)
      when "brewery" then Beer.joins(:brewery)
        .order("breweries.name").limit(PAGE_SIZE).offset(offset)
      when "style"   then Beer.joins(:style)
        .order("styles.name").limit(PAGE_SIZE).offset(offset)
      when "rating"  then Beer.left_joins(:ratings)
        .select("beers.*, avg(ratings.score)")
        .group("beers.id")
        .order("avg(ratings.score) DESC").limit(PAGE_SIZE).offset(offset)
    end

  end

  # ...
end
```


And voilà! We have working pagination for our beers. But one thing that is kinda annoying is that when we navigate between the pages, the whole pages gets reloaded with menus and all even though the contents of the table are the only thing changing. Here is where we come to where Turbo Frames can help us...

<blockquote>

## Exercise 2

Change the ratings page to show all ratings in a paginated form. The default order is to show the most recent rating first, add a button that allows reversing the order. Your solution could look like the following:


![image](../images/8-6.png)

</blockquote>

## Turbo Frames

Turbo Frames provide a convenient way to update specific parts of a page upon request, allowing us to focus on updating only the necessary content while keeping the rest of the page intact.

To begin, let's create a new partial called `_beers_page.html.erb` to the folder `app/views/beers` and extract the table containing the beers from our `beers/index.html.erb` file. This way, our `index.html.erb` will appear as follows:

**app/views/beers/index.html.erb**

```html
<h1>Beers</h1>

<% cache "beerlist-#{@page}-#{@order}", skip_digest: true do %>
  <div id="beers">
    <%= render "beers_page", beers: @beers, page: @page, order: @order, last_page: @last_page  %>
  </div>
<% end %>

<%= link_to('New Beer', new_beer_path) if current_user %>
```

Once the above is functioning correctly, we can enclose our table within the `_beers_page.html.erb` partial using a turbo frame:

**app/views/beers/\_beers_page.html.erb**

```html
<%= turbo_frame_tag "beers_page" do %>
  <table class="table table-striped table-hover">
    <!-- ... -->
  </table>
<% end %>
```

By using a Turbo Frame, all links and buttons within it will be controlled by Turbo, allowing the rendering of new pages triggered by the links within the Turbo Frame. However, to ensure proper functionality, we need to make a slight modification to our `index` method in `beers_controller.rb` in file path `app/controllers/beers_controller.rb`:

```ruby
def index
  # ...
  if turbo_frame_request?
    render partial: "beers_page",
      locals: { beers: @beers, page: @page, order: @order, last_page: @last_page }
  else
    render :index
  end
end
```

The `turbo_frame_request?` condition ensures that when the request is made within a Turbo Frame, only the partial containing our beer table is returned. We can now observe the behavior within the network tab of our browser's developer tools.

![image](../images/ratebeer-w8-2.png)

We can see that the headers include the ID of the Turbo Frame we are targeting, allowing Turbo to identify which part of the page should be replaced.

![image](../images/ratebeer-w8-3.png)

Indeed, the response contains only the partial and excludes the application layout that accompanies the HTML document. Turbo automatically handles this aspect.

The only remaining issue is that the links to beers, breweries, and styles are no longer functional. Turbo attempts to load the links and replace our table with their content but fails to find a suitable turbo tag for replacement. We can easily resolve this by adding the target attribute to our links:

**app/views/beers/\_beers_page.html.erb**

```html
<% beers.each do |beer| %>
  <tr>
    <td><%= link_to beer.name, beer, data: { turbo_frame: "_top"} %></td>
    <td><%= link_to beer.style.name, beer.style, data: { turbo_frame: "_top"} %></td>
    <td><%= link_to beer.brewery.name, beer.brewery, data: { turbo_frame: "_top"} %></td>
    <td><%= round(beer.average_rating) %></td>
  </tr>
<% end %>
```

The `target="_top"` signifies that Turbo should break out of the frame and replace the entire page with the opened link. Alternatively, the `target` could be set to `_self`, targeting the current frame, or the ID of another Turbo Frame, in which case Turbo would attempt to replace that specific frame.

We also notice that the URL remains unchanged when navigating between pages, and using the browser's back button may lead to unexpected results. We can easily address this by promoting our Turbo Actions into visits:

**app/views/beers/\_beers_page.html.erb**

```html
<%= turbo_frame_tag "beers_page", data: { turbo_action: "advance" } do %>
```

Under the hood, Turbo utilizes JavaScript to manipulate the [HTML DOM](https://www.w3schools.com/js/js_htmldom.asp) of the page, eliminating the need for us to write any JavaScript code ourselves!

<blockquote>

## Exercise 1

`turbo_frame_tag` has an attribute `src` available that will lazy load the contents of the source address into the turbo frame.

1. Refactor breweries page so that there is new partial `_breweries_list.html.erb` which is used separately to list breweries under active breweries and retired breweries.
2. Create new endpoints behind `breweries/active` and `breweries/retired` routes that return the partials for the active and retired breweries respectively.
3. Use `turbo_frame_tag` with `src` attribute to lazy load active and retired breweries into their respective turbo frames.
4. Fix the links to breweries so that they work inside the turbo frames.
</blockquote>

## Turbo Streams

The purpose of [Turbo Streams](https://turbo.hotwired.dev/handbook/streams) is to enable page updates in fragments. For example, when a page displays a list of breweries, instead of performing a complete page reload, a single brewery can be appended or removed from the list in response to a change.

In modern web applications, achieving this behavior often involves having a separate server-side REST API or GraphQL API, commonly referred to as the back-end, to provide the necessary information in JSON format. The front-end queries this back-end, receives the JSON data, and renders the required HTML while updating the DOM accordingly.

Turbo simplifies this process by streaming pre-rendered HTML, compiled on the back-end, directly to the browser and handling the necessary actions internally.

Key concepts in Turbo Streams include **actions**, **targets**, and **templates**. These concepts determine which action should be applied to specific target elements using specific template data.

### Turbo Streams Actions

In Turbo Streams, **actions** are a fundamental concept used to specify the changes or updates that should be performed on the client-side HTML DOM in response to a server-side event. An action represents a specific operation that can be applied to one or more target elements within a Turbo Stream response.

**Actions** are defined using HTML-like syntax and consist of a combination of elements and attributes. Each action includes a target element, which represents the HTML element on the client-side that needs to be updated, and one or more operations that define how the target element should be modified.

The operations that can be applied to a target element include:

| Action    | Description                                                      |
| --------- | ---------------------------------------------------------------- |
| `append`  | Appends new content after the target element.                    |
| `prepend` | Prepends new content before the target element.                  |
| `replace` | Replaces the content of the target element with new content.     |
| `update`  | Updates specific attributes or properties of the target element. |
| `remove`  | Removes the target element from the DOM.                         |

### Turbo Streams Targets

In order for **actions** to function properly, Turbo requires the identification of target elements within the DOM. This can be achieved by assigning unique HTML `id` parameters to individual elements or by utilizing `class` parameters to target multiple elements.

For identifying a single element, one can explicitly create an ID value in the view or leverage the convenient Rails [dom_id](https://api.rubyonrails.org/classes/ActionView/RecordIdentifier.html) helper, which automatically generates the ID tag. For example:

```html
<div id="<%= dom_id(brewery) %>">
  <%= brewery.name %>
</div>
```

This would result in a unique ID like:

```html
<div id="brewery_55">Sinebrychoff</div>
```

Alternatively, when targeting **multiple elements** based on specific [CSS class selectors](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Selectors), such as:

```html
<div class="active" id="brewery_55">Sinebrychoff</div>
<div class="active" id="brewery_62">Laitilan Wirvoitusjuomatehdas</div>
<div class="retired" id="brewery_71">Pyynikin käsityöläispanimo</div>
```

you could use the remove action to remove all retired breweries from the list by targeting the class ".retired".

### Turbo Streams Templates

To leverage the capabilities of Turbo Streams, view templates should be designed as [partials](https://guides.rubyonrails.org/layouts_and_rendering.html#using-partials) that can be rendered individually. This enables targeted streaming of changes to specific components. For example, when streaming updates for breweries and appending new breweries to a list, the brewery row should be implemented as a partial.

To prepare the Breweries index page for streaming, extract the row rendering logic (created in Exercise 1) from `app/views/breweries/_breweries_list.html.erb`:

**app/views/breweries/\_breweries_list.html.erb**

```html
<tbody>
  <% breweries.each do |brewery| %>
    <tr %>">
      <td><%= link_to brewery.name, brewery, data: { turbo_frame: "_top"} %></td>
      <td><%= brewery.year %></td>
      <td><%= brewery.beers.count %></td>
      <td><%= round(brewery.average_rating) %></td>
    </tr>
  <% end %>
</tbody>
```

To new partial file `app/views/breweries/_brewery_row.html.erb`:

**app/views/breweries/\_brewery_row.html.erb**

```html
<tr %>">
  <td><%= link_to brewery.name, brewery, data: { turbo_frame: "_top"} %></td>
  <td><%= brewery.year %></td>
  <td><%= brewery.beers.count %></td>
  <td><%= round(brewery.average_rating) %></td>
</tr>
```

And change the original code to use this partial :

```html
<tbody id="<%= status %>_brewery_rows">
  <% breweries.each do |brewery| %>
    <%= render "brewery_row", brewery: brewery %>
  <% end %>
</tbody>
```

Pay attention to new ID that we give to the tbody element. We need `active_brewery_rows` or `retired_brewery_rows` ID to **target** the **action** of appending new breweries as children of the correct table. If in Exercise 1 you did not define local `status` or something similar containing `active`/`retired` information for the different brewery listings, you should do that now as it will help us later.

Next, let's enable the addition of new breweries directly from the index page. Replace the following code in `app/views/breweries/index.html.erb`:

**app/views/breweries/index.html.erb**

```html
<%= link_to "New brewery", new_brewery_path if current_user %>
```

With the following Turbo Frame tag:

**app/views/breweries/index.html.erb**

```html
<%= turbo_frame_tag "new_brewery", src: new_brewery_path if current_user %>
```

This Turbo Frame will include a part of our existing code from the `new_brewery` path.

In `app/views/breweries/_new.html.erb` specify which part of the view you want to show in the Turbo Frame:

**app/views/breweries/\_new.html.erb**

```html
<h1>New brewery</h1>

<%= turbo_frame_tag "new_brewery" do %>
  <%= render "form", brewery: @brewery %>
<% end %>
# ...
```

![image](../images/ratebeer-w8-4.png)

In order to append the created brewery to the list without doing a full page update, we need to modify the response in the create action of the `app/controllers/breweries_controller.rb` file.

By adding the `format.turbo_stream` block, we specify that the response should be rendered as a Turbo Stream template with the action of appending the new brewery row to the target element with the ID `active_brewery_rows` or `retired_brewery_rows`. These steps enable the addition of breweries directly from the index page while only appending the created brewery to the list without refreshing the entire page.

**app/controllers/breweries_controller.rb**

```ruby
def create
  @brewery = Brewery.new(brewery_params)

  respond_to do |format|
    if @brewery.save
      format.turbo_stream {
        status = @brewery.active? ? "active" : "retired"
        render turbo_stream: turbo_stream.append("#{status}_brewery_rows", partial: "brewery_row", locals: { brewery: @brewery })
      }
      format.html { redirect_to brewery_url(@brewery), notice: "Brewery was successfully created." }
      format.json { render :show, status: :created, location: @brewery }
    else
      format.html { render :new, status: :unprocessable_entity }
      format.json { render json: @brewery.errors, status: :unprocessable_entity }
    end
  end
end
```

This change involves one line of code, but under the hood, several components enable this to work:

1. When the browser initiates a new request, the Turbo framework includes the `text/vnd.turbo-stream.html` in the request's `Accept` headers. This informs the server that it expects a Turbo Stream template instead of a full page update.

![image](../images/ratebeer-w8-turbo-streams-header.png)

2. The controller, based on the `Accept` header, recognizes the request's Turbo Stream format and responds by rendering a Turbo Stream template instead of a complete page. This ensures that only the necessary HTML fragments are sent back to the browser.

3. Using the `turbo_stream.append` method, a response HTML fragment is generated with the action set to `append`. This fragment targets the element with the identifier `brewery_rows` and utilizes the `_brewery_row.html.erb` partial to generate the content. Here is an example of the resulting fragment:

```html
<turbo-stream action="append" target="active_brewery_rows"
  ><template
    ><tr id="brewery_65">
      <td>
        <a data-turbo-frame="_top" href="/breweries/65"
          >Laitilan Wirvoitusjuomatehdas</a
        >
      </td>
      <td>1995</td>
      <td>0</td>
      <td>0.0</td>
    </tr></template
  ></turbo-stream
>
```

4. With the table body previously assigned an ID, such as `<tbody id="active_brewery_rows">`, Turbo knows to append the generated template as the last child of the table body element. It intelligently places the new content in the appropriate location. You can test this behavior by removing or altering the ID and observing the resulting outcome.

### Dynamic Updates with ActionCable

[ActionCable](https://edgeguides.rubyonrails.org/action_cable_overview.html) enables dynamic updates by utilizing [WebSockets](https://en.wikipedia.org/wiki/WebSocket), providing real-time streaming of content to multiple browsers. Unlike traditional request-response logic, which requires the browser to send a request and await a response, ActionCable allows seamless updates without the need to refresh the page. This functionality has been available since Rails version 5 and is based on WebSockets technology. For a very brief tutorial on WebSockets, see [WebSockets in 100 seconds](https://www.youtube.com/watch?v=1BfCnjr_Vjg).

To establish a connection between the browser and the server for listening to changes, we define a stream or channel. In our view file, `app/views/breweries/index.html.erb`, we "subscribe" to updates using the following line of code:

**app/views/breweries/index.html.erb**

```html
<%= turbo_stream_from "breweries_index" %>
```

This code establishes a WebSocket connection between the browser and the server, enabling the browser to receive real-time updates.

To publish updates, we utilize the Brewery model `app/models/brewery.rb`. Whenever a new brewery is created, the following code is triggered:

**app/models/brewery.rb**

```ruby

after_create_commit -> { broadcast_append_to "breweries_index", partial: "breweries/brewery_row", target: "active_brewery_rows" }, if: :active?
after_create_commit -> { broadcast_append_to "breweries_index", partial: "breweries/brewery_row", target: "retired_brewery_rows" }, if: :retired?
```

This code broadcasts an `append` action to the `breweries_index` channel, targeting the element with the ID `active_brewery_rows` or `retired_brewery_rows`. It uses the `_brewery_row.html.erb` partial to create the template for the new brewery. Essentially, it replicates the same functionality we implemented earlier by responding to client requests with fragments. However, the difference lies in the fact that the HTML fragment is now broadcasted to all browsers subscribed to the `breweries_index` channel, thanks to the power of WebSockets.

You can test the functionality by opening two browser windows side by side and creating a new brewery. You'll observe that the updates are instantly reflected in both windows, demonstrating the real-time nature of ActionCable and WebSockets.

![image](../images/ratebeer-w8-two-browsers.png)

During the process, you may have noticed a problem: when a user adds new breweries, duplicate entries appear in the list. Let's take a moment to understand why this happens.

The reason is that the current user actually receives the fragments twice. Firstly, as an HTTP response to the form submission triggered by the create action in the controller. Secondly, as a WebSocket update triggered by the `after_create_commit` hook in the model.

To address this issue, there are several possible solutions:

1. Comment out the stream template in the HTTP response from the controller. However, this approach has a downside: if there are any issues with WebSockets, the user won't see the effect of submitting a new brewer.

2. Conditionally trigger the `after_create_commit` hook in the model based on the logged-in user. This approach ensures that the user only receives the WebSocket update once.

3. Opt for a simpler solution by giving each row a unique identifier. Let's proceed with this approach here.

In `app/views/breweries/_brewery_row.html`, add the following line of code:

**app/views/breweries/\_brewery_row.html**

```html
<tr id="<%= dom_id(brewery) %>">
  <td>...
```

By assigning a unique identifier to each row, we prevent duplication. This unique identifier becomes essential if we want to trigger actions, such as removing a specific brewery.

To observe the WebSocket connection details, you can use the browser's developer tools.

![image](../images/ratebeer-w8-console-websockets.png)

It's worth noting that in our example, we used a simple string, `breweries_index`, as the identifier for the channel since there is only one `breweries_index`. However, in certain scenarios, you may want to use an object to identify the stream. For instance, if you implement the ability to add new beers to a specific brewery from the Brewery page and stream the added data only to that page, you would want to use something like `@brewery` instead of `"breweries_index"`. This way, different brewery pages can be targeted with the streaming updates.

<blockquote>

## Exercise 2

Enhance the breweries list functionality by adding a button or text "X" for removing a brewery from the database (see [Rails views documentation](https://guides.rubyonrails.org/layouts_and_rendering.html#rendering-by-default-convention-over-configuration-in-action)). The implementation should follow these steps:

1. **Initial Removal (No Turbo, Full Page Reload)**
   Initially, make the removal work without using Turbo, requiring a full page reload after the delete action.

2. **Dynamic Removal with Turbo Streams**
   Improve the functionality by dynamically removing the deleted brewery from the list using Turbo Streams. Ensure the removal is reflected in the UI without requiring a full page reload.

3. **WebSocket Integration for Real-Time Updates**
   Leverage WebSockets to stream the removal action to all connected browsers in real time.

4. **Confirmation Pop-up**
   Enhance user experience by introducing a confirmation pop-up. When a user clicks the remove button, a confirmation dialog should appear with the text "Are you sure you want to remove brewery X and all beers associated with it?". The pop-up should provide options for "Cancel" and "Remove" actions.

</blockquote>

<blockquote>

## Exercise 3

Notice that _Number of Active Breweries_ and _Number of Retired Breweries_ require a full page reload to reflect the actual numbers. Make these numbers dynamic so that any addition or retirement of a brewery by any user triggers real-time updates. The changes should be streamed to reflect the updated counts instantly.

Hint: you can render multiple turbo stream messages from a controller response by placing them in an array.
</blockquote>

## Stimulus

[Stimulus](https://stimulus.hotwired.dev/) is a JavaScript framework designed to enhance interactivity in HTML, eliminating the need for extensive custom JavaScript development. By utilizing a set of JavaScript modules that can be seamlessly attached to HTML elements using specific attributes, Stimulus enables developers to create simpler and more maintainable code.

One of the notable advantages of Stimulus is its seamless integration with Rails, making it an ideal choice for web applications built on the Rails framework. Leveraging the latest browser technologies, Stimulus delivers a fast and seamless user experience, ensuring optimal performance.

Stimulus utilizes key concepts such as **controllers**, **actions**, **targets**, and **values** to determine the appropriate actions to apply to specific target elements based on specific data.

**Benefits**

- Reduces the amount of custom JavaScript required to implement common functionality, streamlining development efforts.
- Provides a straightforward and intuitive API for dispatching and listening to events on webpages, simplifying event handling.
- Enhances performance by avoiding the overhead associated with full-fledged JavaScript frameworks.

**Considerations**

- Stimulus may not be the ideal choice for complex applications that demand extensive client-side functionality.
- Careful organization and structuring of Stimulus controllers is necessary to prevent bloated and unmanageable code.
- Developers who are unfamiliar with the Stimulus framework may experience a learning curve and added complexity during development.

### Deleting ratings

Let's try our hand at Stimulus with implementing a feature that allows users to delete multiple beer ratings at once without need for a full page reload.

We can start by creating a new partial file named `_ratings.html.erb` within the `/app/views/users` folder.

Then we extract the ratings code section (shown below) from the `/app/views/users/show.html.erb` file and place it into the ratings partial file.

**/app/views/users/\_ratings.html.erb**

```html
<ul>
  <% @user.ratings.each do |rating| %>
    <li>
      <%= "#{rating.score} #{rating.beer.name}" %>
      <% if @user == current_user %>
        <%= button_to 'delete', rating, method: :delete, form: { style:'display:inline-block;', data: { 'turbo-confirm': 'Are you sure?' } } %>
      <% end %>
    </li>
  <% end %>
</ul>
```

Then we delete the ratings code from the `/app/views/users/show.html.erb` file and insert the ratings partial using the `<%= render partial: 'ratings' %>` statement under the ratings header.

**/app/views/users/show.html.erb**

```html
<h4>Ratings</h4>
<%= render partial: 'ratings' %>
```

We can then modify the partial by removing the list elements and delete button from the `/app/views/users/_ratings.html.erb` file, like so:

**/app/views/users/\_ratings.html.erb**

```html
<div class="ratings mb-4">
  <% @user.ratings.each do |rating| %>
    <div class="rating">
      <% if @user == current_user %>
        <input type="checkbox" name="ratings[]" value="<%= rating.id %>" />
      <% end %>
      <span><%= "#{rating.score} #{rating.beer.name}" %></span>
    </div>
  <% end %>
  <% if @user == current_user %>
    <button>Delete selected</button>
  <% end %>
</div>
```

With the modified templates ready, let's update the `routes.rb` file (`/app/config/routes.rb`) to handle the ratings destroy action. Remove the `destroy` action from the ratings resources and add a separate delete method to handle the removal of rating IDs.

**/app/config/routes.rb**

```ruby
resources :ratings, only: [:index, :new, :create]
delete 'ratings', to: 'ratings#destroy'
```

Modify the `destroy` method within the `ratings_controller.rb` file (`app/controllers/ratings_controller.rb`) to handle the deletion of multiple rating IDs. We can do it like this:

**app/controllers/ratings_controller.rb**

```ruby
def destroy
  destroy_ids = request.body.string.split(',')
  # Loop through multiple rating IDs and delete them if they exist and belong to the current user
  destroy_ids.each do |id|
    rating = Rating.find_by(id: id)
    rating.destroy if rating && current_user == rating.user
  # Rescue in case one of the rating IDs is invalid so we can continue deleting the rest 
  rescue StandardError => e
    puts "Rating record has an error: #{e.message}"
  end
  @user = current_user
  respond_to do |format|
    format.html { render partial: '/users/ratings', status: :ok, user: @user }
  end
end
```
![image](../images/ratebeer-w8-8.png)
Right now the delete button does not really do anything as it's not connected to our route or controller any way. This is where we start using Stimulus.

### Stimulus Controllers

When working with Stimulus, it is essential to follow a specific naming convention for **controller** files. Each controller file should be named in the format `[identifier]_controller.js`, where the identifier corresponds to the data-controller attribute associated with the respective controller in your HTML markup.
By adhering to this naming convention, Stimulus can seamlessly link the controllers in your HTML with their corresponding JavaScript files.

Let's start by creating a `ratings_controller.js` and put it to file path `/app/JavaScript/controllers/ratings_controller.js`:

**/app/JavaScript/controllers/ratings_controller.js**

```JavaScript
import { Controller } from "@hotwired/stimulus";

export default class extends Controller {
   // We'll talk about the connect in a second... 
   connect() {
      console.log("Hello, Stimulus!");
   }
}
```

In our `_ratings.html.erb` we can edit `<div>` element with the `data-controller` attribute, with value `ratings`.

```html
<div data-controller="ratings" class="ratings mb-4">
  # ...
</div>
```
This binds it to the Stimulus controller named `ratings_controller.js`. It's important to note that the scope of a Stimulus controller includes the element it is connected to and its children, but not the surrounding elements.

We can then reload the page and see from the console log that we have indeed connected to the Stimulus controller.

![image](../images/ratebeer-w8-9.png)

`connect()` is a callback method which Stimulus supports out of the box which is executed when the controller is connected to the DOM.

### Lifecycle Methods

Lifecycle methods in Stimulus provide a capability for executing code at specific stages in the lifecycle of a controller. These methods, defined within the controller class, offer hooks for initialization, connection to the DOM, and disconnection from the DOM.

Here is an example showcasing the available lifecycle methods in a Stimulus controller:

```JavaScript
import { Controller } from 'stimulus';

export default class extends Controller {
  initialize() {
    // Executed once when the controller is first instantiated
  }

  [name]TargetConnected(target) {
    // Executed anytime a target is connected to the DOM
  }

  [name]TargetDisconnected(target) {
    // Executed anytime a target is disconnected from the DOM
  }

  connect() {
    // Executed when the controller is connected to the DOM
  }

  disconnect() {
    // Executed when the controller is disconnected from the DOM
  }
}
```

These lifecycle methods provide developers with the flexibility to perform specific tasks at appropriate points in the controller's lifecycle. The `initialize` method can be used for initial setup or one-time actions. The `[name]TargetConnected` and `[name]TargetDisconnected` methods enable dynamic handling of targets as they are connected or disconnected from the DOM. The `connect` and `disconnect` methods offer a convenient way to set up or clean up resources associated with the controller when it is connected or disconnected from the DOM.

By leveraging these lifecycle methods, developers can ensure proper initialization, respond to changes in DOM state, and maintain organized and maintainable code in their Stimulus controllers.

### Stimulus Actions

**Actions** in Stimulus are methods defined within a controller that respond to user events or changes in the application state. These actions are identified using the data-action attribute and can be triggered by various events, such as clicks, form submissions, or custom events.

Let's add an action to our `<button>` element, data-action attributes are used with the format `event->controller#method`.

**/app/views/users/\_ratings.html.erb**

```html
<div data-controller="ratings" class="ratings mb-4">
  # ...
  <% if @user == current_user %>
    <button data-action="click->ratings#destroy">Delete selected</button>
  <% end %>
</div>
```

Here a click event listener is attached to the `<button>` element with the controller identifier `ratings`, and it calls the `destroy` method when clicked.

Stimulus provides default events for certain elements, which means that the click event can be omitted from the data-action attribute for buttons. For example, `<button data-action="ratings#destroy">Delete selected</button>` would achieve the same result.

Here is a complete list of elements and their default events:

| Element             | Default Event |
| ------------------- | ------------- |
| `a`                 | `click`       |
| `button`            | `click`       |
| `details`           | `toggle`      |
| `form`              | `submit`      |
| `input`             | `input`       |
| `input type=submit` | `click`       |
| `select`            | `change`      |
| `textarea`          | `input`       |

We can now write the `destroy` method in our `ratings_controller.js`:

**/app/JavaScript/controllers/ratings_controller.js**

```JavaScript
import { Controller } from "@hotwired/stimulus";

export default class extends Controller {
  destroy() {
    // Confirmation dialog for the user
    const confirmDelete = confirm(
      "Are you sure you want to delete these selected ratings?"
    );
    if (!confirmDelete) {
      return;
    }
    
    // Retrieve the selected rating IDs from the checkboxes
    const selectedRatingsIDs = Array.from(
      document.querySelectorAll('input[name="ratings[]"]:checked'),
      (checkbox) => checkbox.value
    );
    
    // Include the CSRF token in the request headers so that Rails recognizes us as the logged in user
    const csrfToken = document.querySelector('meta[name="csrf-token"]').content;
    const headers = { "X-CSRF-Token": csrfToken };

    // Send a DELETE request to the ratings controller with the selected rating IDs
    fetch("/ratings", {
      method: "DELETE",
      headers: headers,
      body: selectedRatingsIDs.join(","),
    })
      .then((response) => {
        if (response.ok) {
          response.text().then((html) => {
            document.querySelector("div.ratings").innerHTML = html;
          });
        } else {
          throw new Error("Something went wrong");
        }
      })
      .catch((error) => {
        console.log(error);
      });
  }
}
```
And now we finally have a working delete button for deleting multiple ratings!

### Beer tax calculator

To showcase a bit more Stimulus features we'll build a handy beer tax calculator for our application so that you know how much you are supporting Helsinki University and other public services with each beer bought. To start let's add new path to our `routes.rb`.

**/config/routes.rb**

```ruby
Rails.application.routes.draw do
   #...
   get 'calculator', to: 'misc#calculator'
   # ...
end
```

And create a new controller file for out path:

**/app/controllers/misc_controller.rb**

```ruby
class MiscController < ApplicationController
   def calculator
   end
end
```

And then add link for the calculator to our navbar:

**/app/views/layouts/application.html.erb**

```html
<!--(...)-->
<li class="nav-item">
  <%= link_to 'styles', styles_path, { class: "nav-link" } %>
</li>
<li class="nav-item">
  <%= link_to 'tax calculator', calculator_path, { class: "nav-link" } %>
</li>
<!--(...)-->
```

Lastly we can create a Stimulus controller file for our calculator to `app/JavaScript/controllers`:

**/app/JavaScript/controllers/calculator_controller.js**

```JavaScript
import { Controller } from "@hotwired/stimulus";

    export default class extends Controller {}
}
```

#### Stimulus Targets

**Targets** in Stimulus are special attributes that allow a controller to reference and manipulate specific elements within its scope. To define targets, you need to add the `data-[controller name]-target` attribute to the HTML elements. Stimulus scans your controller class and identifies target names in the static `targets` array. It automatically adds three properties for each target name: `sourceTarget`, which evaluates to the first matching target element, `sourceTargets`, which evaluates to an array of all matching target elements, and `hasSourceTarget`, which returns boolean value `true` or `false` depending on the presence of a matching target.

Let's create a form for our calculator containing some targets to collect.
```html
<h2>Beer tax calculator</h2>

<div data-controller="calculator" class="container">
   <form data-action="calculator#calculate">
      <div>
         <label>Amount</label>
         <input type="number" min="0" step="0.001" value="0.000" required="true" data-calculator-target="amount" />
         <label>liters</label>
      </div>
      <div>
         <label>Alcohol by volume (ABV)</label>
         <input type="number" min="0" step="0.01" value="0.00" required="true" data-calculator-target="abv" />
         <label>%</label>
      </div>
      <div>
         <label>Price </label>
         <input type="number" min="0" step="0.01" value="0.00" required="true" data-calculator-target="price" />
         <label>€</label>
      </div>
      </br>
      <button>Calculate</button>
   </form>
</div>
```

To add target names to the controller's list of target definitions, you need to update the `calculator_controller.js` file accordingly. This will automatically create properties with names `nameTarget` for each which return the first matching target element. You can then use this property to read the value of the element and for testing print each value to console.

**/app/JavaScript/controllers/calculator_controller.js**

```JavaScript
import { Controller } from "@hotwired/stimulus";

export default class extends Controller {
   static targets = ["amount", "abv", "price"];

   calculate(event) {
      // Prevent the default form submission from reloading the page.
      event.preventDefault();
      const amount = parseFloat(this.amountTarget.value);
      const abv = parseFloat(this.abvTarget.value);
      const price = parseFloat(this.priceTarget.value);
      console.log(amount, abv, price);
   }
}
```

When testing the submit button we can see that we are getting the values printed to JavaScript console.

![image](../images/ratebeer-w8-10.png)

#### Stimulus Values

**Values** in Stimulus are a way to store and access data within a controller using the `value` method. These values can be declared in various ways, such as static values defined in the controller, attributes on HTML elements, or dynamic values.

For our calculator app we can create attribute `data-calculator-vat-value` for having value saved for value added tax. In this case, the value gets set to `0.24`.

Let's also add div for us to input the result of our calculations. Notice that the div needs to be inside the `<div data-controller="calculator">` element so that our calculator controller can see and access it.

```html
<h2>Beer tax calculator</h2>

<% vat = 0.24 %>
<div data-controller="calculator" data-calculator-vat-value="<%= vat %>" class="container">
   <form data-action="calculator#calculate">
      // (...)
      <div>
         <p>Value added tax <%= vat * 100 %>%</p>
      </div>
      </br>
      <button>Calculate</button>
   </form>
   </br>
   <div id="result">
   </div>
</div>
```

In the controller file `calculator_controller.js`, a static values array is created, including the attribute name `vat` with a type of `Number`. By adding the attribute `data-calculator-vat-value="0.24"` to the `<div>` element, the value `0.24` is assigned to the `vatValue` variable in the controller and converted into number with JavaScript's `Number()` -function. This value can then be accessed and used within the controller's code.

Now we can finish the code for our calculator.

**/app/JavaScript/controllers/hello_controller.js**

```JavaScript
import { Controller } from "@hotwired/stimulus";

export default class extends Controller {
   static targets = ["amount", "abv", "price"];
   static values = { vat: Number };
   calculate(event) {
      event.preventDefault();
      const amount = parseFloat(this.amountTarget.value);
      const abv = parseFloat(this.abvTarget.value);
      const price = parseFloat(this.priceTarget.value);
      // Amounts of alcohol tax per liter of pure alcohol for beers.
      let alcoholTax = 0;
      switch (true) {
         case (abv < 0.5):
            alcoholTax = 0;
         case (abv <= 3.5):
            alcoholTax = 0.2835;
         case (abv > 3.5):
            alcoholTax = 0.3805;
      }
      const beerTax = (amount * abv * alcoholTax);
      const vatAmount = (price * this.vatValue);
      const taxPercentage = ((beerTax + vatAmount) / price * 100);
      const result = document.getElementById("result")
      result.innerHTML = "<p>Beer has " + beerTax.toFixed(2) + "€ of alcohol tax and " + vatAmount.toFixed(2) + "€ of value added tax.</p>" +
              "<p>" + taxPercentage.toFixed(1) + "% of the price is taxes.</p>"
   }
}
```

And now we have a beautifully working beer tax calculator!

![image](../images/ratebeer-w8-11.png)

<blockquote>

## Exercise 4

Improve beer tax calculator by changing the amount field to be dropdown selection containing most common beer can and bottle sizes, for example these: 0.33, 0.375, 0.5, 0.66, 0.75, 1, 1.3 and 1.5 liters.
</blockquote>

<blockquote>

## Exercise 5

Continuing from the exercise 4, add option `Custom` to the dropdown. When custom option is selected, there is user fillable custom amount field added to the form. If user switches back to pre-defined amount in the dropdown, custom amount field gets removed from the form.

![image](../images/ratebeer-w8-12.png)

Hint: Remember that you have `this.has[name]Target` checker available to check if named target has been defined.
</blockquote>


<blockquote>

## Exercise 6

Add _select all_ checkbox input to users ratings partial and event that selects/deselects all users ratings when that checkbox is selected/deselected.
</blockquote>

<blockquote>

## Exercise 7

When we add new breweries in the brewery page, our form does not get emptied out after adding the brewery. Use Stimulus to clear all form inputs (also the checkbox) after the form is submitted.

Hint: Turbo offers `turbo:submit-end` event that is fired after form is submitted which you can user to trigger an action. More turbo events can be found here: https://turbo.hotwired.dev/reference/events
</blockquote>

<blockquote>

## Exercise 8

For the form for creating a new brewery, add a select field that gets its data (breweries) from the PRH API

- Add select input-field and get its data (breweries) from the PRH API.
- After user selects brewery from select input, get selected brewery's data from the PRH API and fill brewery name and registration year to new form inputs (name, year).


- API url to get all breweries: https://avoindata.prh.fi/bis/v1?totalResults=true&maxResults=500&businessLine=Oluen%20valmistus
- API url to get the single brewery data: https://avoindata.prh.fi/bis/v1?businessId=${breweryId}

You can assume year of the registration date as the year of the brewery's establishment, unless it is before 1980's as those records don't seem to match to the actual establishment year. Leave the year field empty for those.
</blockquote>

## ActionCable, Redis, and Heroku / Fly.io Integration

To ensure seamless operation of **ActionCable**, the foundational component for Turbo streams, in a production environment, it is necessary to have [Redis](https://redis.io/) installed. Please follow the steps outlined below to ensure a proper configuration.

### Heroku
1. **Verify Redis Installation**

Check if you already have a Redis add-on by running the following command in your terminal:

`$ heroku addons | grep heroku-redis`

If no Redis add-on is listed, proceed to create a Redis instance using the command:

`$ heroku addons:create heroku-redis:mini -a your-app-name`

2. **Gemfile Configuration**

By default,  Heroku configures your `Gemfile` with a Redis gem version higher than 5. However, if you are running a Rails version lower than 7.0.4, you will need an older version of the gem. Update your `Gemfile` with the following Redis configuration:

```ruby
gem "redis", ">= 3", "< 5"
```

### Fly.io
1. **Verify Redis Installation**

Check if you already have a Redis instance by running the following command in your terminal:

`$ fly redis list`

If no Redis instance is listed or if you encountered issues during the installation process (e.g., failure during `fly launch`), proceed to create a Redis instance using the command:

`$ fly redis create`

2. **Fetch Redis Address and Configure the Server**

Retrieve the Redis address by executing the following command in your terminal:

`$ fly redis status <your-redis-instance-name>`

This command will provide detailed information, including a line displaying the private URL in the format: `Private URL = redis://default:somethingsomething...something.upstash.io`. Configure the server by setting the Redis URL as a secret. Execute the command:

`$ fly secrets set REDIS_URL=redis://default:somethingsomething...something.upstash.io`

3. **Gemfile Configuration**

By default, Fly.io configures your `Gemfile` with a Redis gem version higher than 5. However, if you are running a Rails version lower than 7.0.4, you will need an older version of the gem. Update your `Gemfile` with the following Redis configuration:

```ruby
gem "redis", ">= 3", "< 5"
```

## Submitting the exercises

Commit all your changes and push the code to Github. Deploy to the newest version of Heroku or Fly.io, too. Remember to check with Rubocop that your code still adheres to style rules.

If you have problems with Heroku, remember to use <code>heroku logs</code> to view the logs. The same can be done for Fly.io with <code>fly logs</code>.

This part of the course is still in beta testing. You can already try the material and exercises, but they cannot be submitted yet and the exercises can change before they are released.
