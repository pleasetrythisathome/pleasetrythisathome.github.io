---
layout: post
title:  "Choropleth Mapping with React and D3"
date:   2014-03-20 13:10:40
categories: [react d3 visualization choropleth]
---

I've been spending a lot of time with React (and it's even better clojurescript wrapper, [om](https://github.com/swannodette/om)) lately, and it's pretty great. This isn't a React tutorial persay, so if you're not familiar with it, I suggest you [check it out](http://facebook.github.io/react/) first. Same goes for [d3.js](http://d3js).

D3 selections are a nice idea, but the chained approach to DOM creation get increasingly messy and confusing as the complexity of the rendered material increases. I'll save expounding on the benefit of React to another post, but by substituting React for DOM management, we can continue to use d3's powerful visualization creation functions (projections, paths, scales, interpolators, colors, etc.) within a flexible component context.

First, I'd like to port the [d3 choropleth example](http://bl.ocks.org/mbostock/4060606) to React. I'm going to try to stay as close to the d3 code as possible.

If you're the type that just wants to see something that works, the finished port is [here](http://bl.ocks.org/pleasetrythisathome/9713092)

Let's start by creating an empty React component and rendering it into the DOM.

glossing over include statements, pick your poison

in our index.html

{% highlight html %}
<div class="react"></div>
{% endhighlight %}

in our js file

{% highlight javascript %}
var Choropleth = {};

Choropleth.Map = React.createClass({
  render: function() {
    var div = React.DOM.div;

    return div({
      className: "choropleth"
    }, "Hello Choropleth");
  }
});

React.renderComponent(Choropleth.Map(), this.document.getElementById("main"));
{% endhighlight %}

instead of a div, the component should output an svg of the correct size

{% highlight javascript %}
Choropleth.Map = React.createClass({
  getDefaultProps: function() {
    return {
      width: 960,
      height: 500
    };
  },
  render: function() {
    var svg = React.DOM.svg;

    return svg({
      className: "choropleth",
      width: this.props.width,
      height: this.props.height
    },
  }
});
{% endhighlight %}

next we'll load the map features and save the features as component stateIsoCodes

{% highlight javascript %}
Choropleth.Map = React.createClass({
  // initialize state to prevent null pointers
  getInitialState: function() {
    return {
      counties: [],
      states: {}
    };
  },
  componentWillMount: function() {
    var cmp = this;

    queue()
      // load us.json
      .defer(d3.json, "/assets/data/us.json")
      .await(function(error, us) {
        // set component state
        cmp.setState({
          // convert counties to individual features
          counties: topojson.feature(us, us.objects.counties).features,
          // states don't need to be shaded, so can just be a mesh
          states: topojson.mesh(us, us.objects.states, function(a, b) { return a !== b; })
        });
      });
  }
});
{% endhighlight %}

let's draw our map

{% highlight javascript %}
Choropleth.Map = React.createClass({
  // initialize state to prevent null pointers
  getInitialState: function() {
    return {
      counties: [],
      states: {}
    };
  },
  componentWillMount: function() {
    var cmp = this;

    queue()
      // load us.json
      .defer(d3.json, "/assets/data/us.json")
      .await(function(error, us) {
        // set component state
        cmp.setState({
          // convert counties to individual features
          counties: topojson.feature(us, us.objects.counties).features,
          // states don't need to be shaded, so can just be a mesh
          states: topojson.mesh(us, us.objects.states, function(a, b) { return a !== b; })
        });
      });
  },
  render: function() {
    var cmp = this;

    // I prefer aliasing DOM elements over jsx
    var svg = React.DOM.svg;
    var g = React.DOM.g;
    var path = React.DOM.path;

    // create the path generator
    var pathGenerator = d3.geo.path();

    return svg({
      className: "choropleth,
      width: this.props.width,
      height: this.props.height
    },
               g({
                 className: "counties"
               },
                 _.map(this.state.counties, function(county) {
                   return path({
                     className: cmp.quantize(cmp.rateById.get(county.id)),
                     d: pathGenerator(county)
                   });
                 })),
               path({
                 className: "states",
                 d: this.path(this.state.states)
               }));
});
{% endhighlight %}

and css

{% highlight css %}
.counties {
  fill: none;
}

.states {
  fill: none;
  stroke: #fff;
  stroke-linejoin: round;
}
{% endhighlight %}

to create the choropleth colors, we'll first load the county unemployment data from tsv, and use it to create a hashmap that we can later access by countyId

{% highlight javascript %}
Choropleth.Map = React.createClass({
  getInitialState: function() {
    return {
      counties: [],
      states: {},
      rateById: d3.map()
    };
  },
  componentWillMount: function() {
    var cmp = this;

    queue()
      .defer(d3.json, "/assets/data/us.json")
      .defer(d3.tsv, "/assets/data/unemployment.tsv", function(d) {
        // rateById values
        // there are more idiomatic ways to do this that would avoid direct state mutation,
        // but let's do it this way for the sake of keeping close to the original code
        cmp.state.rateById.set(d.id, +d.rate);
      })
      .await(function(error, us) {
        cmp.setState({
          counties: topojson.feature(us, us.objects.counties).features,
          states: topojson.mesh(us, us.objects.states, function(a, b) { return a !== b; })
        });
      });
  }
});
{% endhighlight %}

next, we need to add a quantize function to add county color classes

{% highlight javascript %}
Choropleth.Map = React.createClass({
  getInitialState: function() {
    return {
      counties: [],
      states: {},
      rateById: d3.map()
    };
  },
  componentWillMount: function() {
    var cmp = this;

    queue()
      .defer(d3.json, "/assets/data/us.json")
      .defer(d3.tsv, "/assets/data/unemployment.tsv", function(d) {
        cmp.state.rateById.set(d.id, +d.rate);
      })
      .await(function(error, us, data) {
        cmp.setState({
          counties: topojson.feature(us, us.objects.counties).features,
          states: topojson.mesh(us, us.objects.states, function(a, b) { return a !== b; })
        });
      });
  },
  quantize: d3.scale.quantize()
    .domain([0, 0.15])
    .range(d3.range(9).map(function(i) {
      return "q" + i + "-9";
    })),

  render: function() {
    var cmp = this;

    var svg = React.DOM.svg;
    var g = React.DOM.g;
    var path = React.DOM.path;

    var pathGenerator = d3.geo.path();

    return svg({
      className: "choropleth Blues",
      width: this.props.width,
      height: this.props.height
    },
               g({
                 className: "counties"
               },
                 _.map(this.state.counties, function(county) {
                   return path({
                     // retrieve the county data from our hash and quantize
                     className: cmp.quantize(cmp.state.rateById.get(county.id)),
                     d: pathGenerator(county)
                   });
                 })),
               path({
                 className: "states",
                 d: pathGenerator(this.state.states)
               }));
  }
});
{% endhighlight %}

That's it!

In the next post we'll make a zooming and panning map using state transitions
