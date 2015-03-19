
Welcome to Part 2 of our 3-part series on content container components!  If you are just joining in, you first might want to visit [Part 1](https://www.airpair.com/posts/review/55032890c878110c00117af1) for an introduction to `ot-site` and UI components.

This tutorial assumes some knowledge of writing Angular directives.  If you need more background on directives, the [official directive guide](https://docs.angularjs.org/guide/directive) is a good place to start.  

## Let's Get Started

In this tutorial, we will be using an Angular 1.3 directive to re-create the `ot-site` container component from Part 1 -- which will require us to explore some advanced strategies for transcluding in multiple points of a directive. Exciting!

As a reminder, the empty container looks something like this:

![Empty ot-site container](https://content-na.drive.amazonaws.com/cdproxy/templink/IC1Wtxw34nmKQ_xm9_738AvsfDB0OJp9gSvFpn-6SN8LAYspN)

We can break it down into 5 parts:


* 3 sections into which arbitrary content can be added (header, menu, body)

* 2 static items that come with every usage (logo, footer)

![Empty ot-site container, labeled](https://content-na.drive.amazonaws.com/cdproxy/templink/iaubtyeI-HFVCoeI5jBZ0Ap4FNwC6Jr49H4VCXNTXyMLAYspN)

Before we dive into transclusion, let's first build the empty scaffold.  

All we have to do is add a `template` property to the normal directive definition, then drop in the appropriate HTML. It will look something like this:

```javascript
angular.module("ot-components")
.directive("otSite", function() {
  return {
    template: `
      <div id="site">													
        <header> 
          <svg id="logo"></svg>
        </header>
        <nav></nav> 
        <main></main>
        <footer>
          © 2015 OpenTable, Inc.
        </footer>
      </div>`
  };
});
```

As you can see, it's pretty straightforward to create the empty component. The tricky part is ahead—if it's going to be a true "container", we'll need add the ability for component users to pass in their content.  

## Intro to Transclusion 

We have several options for passing data into a directive in Angular. The most ubiquitous method is using HTML attributes—it's relatively straightforward to leverage attributes to set toggles in your directive or pass in small strings of text.

For example, to set a title, a directive might expose a `title` attribute like this:

```markup
<ot-site title="My App">
</ot-site>
```

However, for an application container, attributes won't be flexible enough. Can you imagine trying to pass paragraphs of text into an attribute or having sufficient toggles to cover every combination of structures/styles that could potentially be in the body tag?  With truly arbitrary content, what users really need is the ability to define their own custom HTML templates.  A strategy called "**transclusion**" makes this possible.

Transclusion allows component users to add their own custom template by inserting it between the directive's opening and closing tags.  That user template will then be appended to some previously-agreed-upon part of the directive’s internal template—while maintaining a connection to the original scope from which it came. This last part is essential because it allows any Angular bindings in the user's template to come with it and work as expected. 

If `ot-site` were a transcluding directive—and the user wanted to pass in a greeting—their markup would look something like this:

**component user's markup**

```markup
<ot-site>
  <div> Hello world! </div>
</ot-site>
```

So how do we make `ot-site` a transcluding directive?

The first thing we have to do is indicate to Angular that we will be transcluding by setting the `transclude` property to true in the DDO (Directive Definition Object).

```javascript
angular.module("ot-components")
.directive("otSite", function() {
  return {
    transclude: true,
    template: `
      <div id="site">													
        <header>					
          <svg id="logo"></svg>
        </header>					
        <nav></nav>				
        <main></main>												
        <footer>					
          © 2015 OpenTable, Inc.
        </footer>				
      </div>`														
  };
});
```

Why is setting this property necessary?  

Well, during the compilation phase of the directive lifecycle, the default behavior is for Angular to replace anything between the directive’s tags with the directive's internal template. This normally makes sense—you set a template so it will be added to the DOM. But in this case, the template would **overwrite** the user’s content (which means it can't be appended properly later).

If you set `transclude` to `true`, Angular will save a **clone**, or copy, of the user's content before adding in the directive template. That way, it won't be overwritten and will be available to add to the DOM during the linking phase.

Now how do we actually take the clone that we saved and add it to the right place in the DOM?

### Adding the User's Content

For most transclusion situations, you'll simply want to use the core Angular directive, [`ng-transclude`](https://docs.angularjs.org/api/ng/directive/ngTransclude).  If you add `ng-transclude` to any element in your directive's template, a clone of the user's content will simply be appended to that element.

However, `ng-transclude` doesn't perform any extra processing to separate out different parts of the user template.  It will always append the **entire** block of content wherever it appears.  

Why is this a problem for us?  Our container component should support passing content into each of our three sections separately.

Let's say that our component user wanted to pass in something more complicated than simply a "Hello World"—let's say they want to pass in three strings—one to render in the head, one to render in the menu, and one to render in the body.

**component user markup**

```markup
<ot-site>
  <div>
    I render in head.
  </div>
  <div>
    I render in menu.
  </div>
  <div>
    I render in body.
  </div>
</ot-site>
```

If we simply added an `ng-transclude` directive to each section that supports custom content (as below)...

```javascript
angular.module("ot-components")
.directive("otSite", function() {
  return {
    transclude: true,
    template: `
      <div id="site">													
        <header ng-transclude>	
          <svg id="logo"></svg>
        </header>					
        <nav ng-transclude></nav> 								
        <main ng-transclude></main>								
        <footer>					
          © 2015 OpenTable, Inc.
        </footer>				
      </div>`														
  };
});
```

...we would get something like this:

![Content repeating in ot-site](https://content-na.drive.amazonaws.com/cdproxy/templink/dasXWt-e5efGIZu9jvusOKZhMUMZ3QH6d13NeKZHLbMLAYspN?viewBox=1440)

Clearly, that's not ideal.  We wanted each string to render just once in its appropriate section - rather than everything rendering everywhere.

What we really want is for each string to be dealt with individually, so it can be sent to the correct destination in our container.  To do so, we need to abandon `ng-transclude` and write our own custom transclusion functionality.

## Custom Transclusion

To support multiple entry points, we will need to tweak how we ask our component user to pass in content.  The user needs some way to indicate to the directive where each element they add should go.  

How about requiring a `transclude-to` attribute on each element that acts as a sort of "shipping label"?  

**component user markup**

```markup
<ot-site>
  <div transclude-to="head">
    I render in head.
  </div>
  <div transclude-to="menu">
    I render in menu.
  </div>
  <div transclude-to="body">
    I render in body.
  </div>
</ot-site>
```

With the above markup, the directive can now detect that the first div should be sent to the "head", the second to the "menu", and so on.

But we're not done.  We still need to add the corresponding labels on the directive template side—`transclude-id` attributes.  This way, when a component user defines _"head"_ as the destination, we can find an exact match.

```javascript
angular.module("ot-components")

.directive("otSite", function() {
  return {
    transclude: true,
    template: `
      <div id="site">													
        <header transclude-id="head">	
          <svg id="logo"></svg>
        </header>					
        <nav transclude-id="menu"></nav> 								
        <main transclude-id="body"></main>								
        <footer>					
          © 2015 OpenTable, Inc.
        </footer>				
      </div>`														
  };
});
```

### Matching

We have labels on each side, so now we can write custom DOM manipulation logic to match each element in the user's template to a section of the directive.

Let's step through what that might look like, taking for granted for now that we have access to a clone of the user's content (we'll revisit that later).

First thing, we need to loop through every element in the `clone`...

```javascript
angular.forEach(clone, function(cloneEl) {});
```

For each `cloneEl`...

1) We need to grab the `value` of that element’s `transclude-to` attribute. That will give us the ID of our destination element.

```javascript
var destinationId = cloneEl.attributes["transclude-to"].value;
```

2) Then, we need to find the element in the directive’s template whose `transclude-id` property matches that ID.

```javascript
var destination = elem.find('[transclude-id="'+destinationId+'"]');
```

3) Lastly, we need to append the `cloneEL` to its target destination.

```javascript
destination.append(cloneEl);
```

If we put that all together, we get:

```javascript
angular.forEach(clone, function(cloneEl) {
  var destinationId = cloneEl.attributes["transclude-to"].value;
  var destination = elem.find('[transclude-id="'+destinationId+'"]');
  destination.append(cloneEl);
});

```

### Error Handling 


This is all the DOM manipulation we have to do ... provided that everything goes as planned.  What if the component user forgets to pass in a transclude-to attribute on an element? Or it's invalid?

In that case, the directive will never find a valid `destination` (as it will be looking for a nonexistent `destinationId`) -- which means the clone element in question will never be properly appended to the DOM. 

Because of that, it won’t be properly destroyed when the scope is destroyed.  This causes a memory leak. 

We can't let that happen, so let’s check to see that we actually are able to find a target destination - this will check not only for a transclude-id but that it is a valid one - and if not, we can clean up the offending element.  

```javascript
angular.forEach(clone, function(cloneEl) {
  var destinationId = cloneEl.attributes["transclude-to"].value;
  var destination = elem.find('[transclude-id="'+destinationId+'"]');
  if (destination.length) {
    destination.append(cloneEl);
  } else {
    cloneEl.remove();
  }
});

```

That should do it.


### The transclude function

While writing our custom matching process, we’ve been assuming access to a `clone` variable that represents the user’s content—how do we actually get access to that?

The answer is something called the `transclude` function.  

```javascript
transclude(function(clone) {
  # DOM manipulation
});
```

The `transclude` function is a special linking function that becomes available after you set transclude to `true`.  It takes a callback function that has access to a copy of the user’s content, so you can perform any necessary processing or matching in that context.

The `transclude` function becomes available in three different locations in the directive definition: as the third parameter of the `compile` function, as an injectable of the `controller` function, and as the fifth parameter of the `link` function.   So which `transclude` function should you use?

```javascript
compile: function(tElem, tAttrs, transclude) {...}
```
```javascript
controller: function($scope, $element, $transclude) {...}
```
```javascript
link: function(scope, iElem, iAttrs, ctrl, transclude) {...}
```

The first thing to know is that not all `transclude` functions are the same.  The `transclude` function passed to you in the `compile` function is slightly different than the one passed to you in the `controller` and the `link` function. Unlike the latter two, it doesn’t auto-generate the correct transclusion scope for you.   Which means that, at best, you can only get it to work with broken bindings.  For this reason, it's actually been deprecated.

Which leaves us with the `transclude` function in the `controller` and the `link` function—what's the difference?

Mostly, it's execution order.

![Directive life cycle, controller](https://content-na.drive.amazonaws.com/cdproxy/templink/aaKEXw9uELKBNTz0NnKCwqkJIZkgn0kgAbvNt8j1rcELAYspN?viewBox=1440)

*Design credit: [Rachael L Moore](http://morewry.com)*

The controller of a parent directive will be instantiated before its child directives are linked.  This is why controllers are ideal for directive communication—the parent can define any shared methods/scope properties before the children will need to use them.

But this order isn’t great for DOM manipulation.  If you try to manipulate the DOM before child directives are linked, the compiler’s linking function might have trouble finding the children to link them properly.

![Directive life cycle, post-link](https://content-na.drive.amazonaws.com/cdproxy/templink/h5iEt_VZs1DpiC-r9Pk5SNTBAfYMtnvbnGVefZeeCvQLAYspN?viewBox=1440)

*Design credit: [Rachael L Moore](http://morewry.com)*

Generally speaking, the safest time for DOM manipulation is the `link` (aka `post-link`) function because a parent directive’s `link` function will run **after** the child directives have already been found and linked.

So the `link` function is really the one we want to use. All we have to do is add a `link` function to our directive definition, grab the fifth parameter (the `transclude` function) and call it with the DOM manipulation we already wrote:

```javascript
angular.module("ot-components")
.directive("otSite", function() {
  return {
    transclude: true,
    template: ...,
    link: function(scope, elem, attr, ctrl, transclude) {
      transclude(function(clone) {
        angular.forEach(clone, function(cloneEl) {
          var destinationId = cloneEl.attributes["transclude-to"].value;
          var destination = elem.find('[transclude-id="'+ destinationId +'"]');
          if (destination.length) {
            destination.append(cloneEl);
          } else { 
            cloneEl.remove();
          }
        });
      });
    }
  };
});
```

There is only one last thing we have to do to make this a viable component...

## Isolate Scope

...scope.  Hooray!  The obvious choice for a reusable component is to use an isolate scope—which you can set by adding an object to the `scope` property.

But how will this affect our user’s template or bindings?  Can we transclude and still have an isolate scope?

Absolutely.  Let’s take a look at the scope hierarchy you’d get with an isolate scope on a transcluding directive.  You can see the DOM structure on the left, color-coded by scope, and the matching scopes organized by encapsulation on the right:

![Scope diagram](https://content-na.drive.amazonaws.com/cdproxy/templink/5r83oavt9YkTvpI6El9f7ogockdk_8SrLMEdwd3mlKULAYspN?viewBox=1440)

*Design credit: Simon Attley*

You can see that there are actually two `ot-site` scopes that are treated differently—the scope of the directive’s internal template (represented by the `ot-site` tags) and the scope of the user’s template (the `divs` in the middle here).   

The directive’s template has to have an isolate scope, so it can be completely insulated from its environment.  If you look at the inheritance diagram on the right, you can see that it is completely isolated in its own box and won’t inherit anything.  

However, given that our container is wrapping arbitrary content, we want the user to be able to pass in a template that may have bindings from the controller.  For this reason, the user’s template scope will not share that isolate scope, and instead will be a child of its outer scope—the controller scope. That way, it can inherit any properties from the controller that the user wants to pass in.

Given this standard setup, we should just go ahead and add our isolate scope:

```javascript
angular.module("ot-components")

.directive("otSite", function() {
  return {
    scope: {},
    transclude: true,
    template: ...,
    link: function(scope, elem, attr, ctrl, transclude) {
      transclude(function(clone) {
        angular.forEach(clone, function(cloneEl) {
          var destinationId = cloneEl.attributes["transclude-to"].value;
          var destination = elem.find('[transclude-id="'+ destinationId +'"]');
          if (destination.length) {
            destination.append(cloneEl);
          } else { 
            cloneEl.remove();
          }
        });
      });
    }
  };
});
```

That should work! 

If our component user uses the markup from the earlier example:

**component user markup**

```markup
<ot-site>
  <div transclude-to="head">
    I render in head.
  </div>
  <div transclude-to="menu">
    I render in menu.
  </div>
  <div transclude-to="body">
    I render in body.
  </div>
</ot-site>

```

...we get the intended output!

![Final ot-site output](https://content-na.drive.amazonaws.com/cdproxy/templink/_y4Zj0oljkNSvPa5MRNwjQ7m4EqVyzjul7wEWgSFFSQLAYspN?viewBox=1440)

## Making it Reusable

We have a working implementation, but we're not quite finished yet.

We have a large chunk of logic just sitting in our link function, which violates the best practice of keeping link functions/controllers lean.  In addition, it's quite likely that other components would benefit from multiple entry points. So it makes sense to abstract that custom transclusion logic out into a service that can be more widely used.

** MultiTransclude Service**
```javascript
angular.module("ot-components")

.factory("MultiTransclude", function() {
  return {
    transclude: function(elem, transcludeFn) {
      transcludeFn(function(clone) {
        angular.forEach(clone, function(cloneEl) {
          var destinationId = cloneEl.attributes["transclude-to"].value;
          var destination = elem.find('[transclude-id="'+ destinationId +'"]');
          if (destination.length) {
            destination.append(cloneEl);
          } else { 
            cloneEl.remove();
          }
        });
      });
    }
  };
});
```

Now `ot-site` can simply pull in the service, to give us our final implementation:


```javascript
angular.module("ot-components")

.directive("otSite", function(MultiTransclude) {
  return {
    scope: {},
    transclude: true,
    template: ...,
    link: function(scope, elem, attr, ctrl, transcludeFn) {
      MultiTransclude.transclude(elem, transcludeFn);
    }
  };
});
```




That's it for container components in Angular 1.3!  

Stay tuned for Part 3, which will cover Angular 2.
