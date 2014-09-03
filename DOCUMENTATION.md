## Idiogrammatik

### Contents

The following two sections should get you up and running quickly, with a good
handle on how idiogrammatik works. The section that follows is a simple
description of the API exposed by idiogrammatik. The final section includes a
few "recipes" for quickly and easily adding useful functionaly to Idiogrammtik
using its extensible API. The code itself is small and rather self-contained at
around 600 commented source lines of code.

The API is inspired by Mike Bostock's
["Towards Resuable Charts"](http://bost.ocks.org/mike/chart/), with some
liberties taken with the functionaly purity (as much as rendering data-bound
SVGs allows) in order to make Idiogrammtik more pleasant to work with as an
embeddable visualization.

The project is in use at [Hammerlab](https://github.com/hammerlab), and thus
PRs, issues, comments, etc. are much appreciated.

### Introductions

#### Short Introduction

After loading idiogrammatik.js and [d3.js](http://d3js.org/), you'll want to
load the cytoband data asynchronously, then call a configured karyogram chart on
the selection where you'd like to render the chart. From there you can further
extend and customize the karyogram. A minimal example follows.

```javascript
// Initialize and configure a karyogram.
var kgram = idiogrammatik()
    .on('click', function(position, kgram) {
      console.log(position);
    })
    .on('mouseover', function(position, kgram) {
      console.log(position);
    })
    .redraw(function(svg, scale) {
      var data = svg.selectAll('.chromosome').data();

      svg.selectAll('.cname')
          .data(data, function(d) { return d.key; })
        .enter().append('text')
          .attr('class', 'cname')
          .attr('y', -9)
          .text(function(d) { return d.key; });

      svg.selectAll('.cname')
          .attr('x', function(d) { return scale(d.start); });
    });

idiogrammatik.load(function(err, data) {
  if (err)  return console.log('error: ', err);

  // Render the karyogram in <body>.
  d3.select('body')
      .datum(data)
      .call(kgram);

  // Add a highlight to the rendered karyogram (all of chromosome four).
  var highlightChr4 = kgram.highlight('chr4', 0, 'chr5', 0, {color: 'blue'});

  // Pan & zoom to a particular position in the karyogram.
  kgram.zoom(1450000000, 1550000000);
});
```

#### Complete Introduction

Idiogrammtik works in four stages:

1. Load the cytoband data asynchronously.
2. Initialize and customize a karyogram object.
3. Bind the data to a [d3.js](http://d3js.org/) selection & call the karyogram
   on the selection.
4. Manipulate the drawn chart with the resultant object.

Loading the data can be done like so:

```javascript
idiogrammatik.load(function(err, data) {
  // Now what?
});
```

We'll also want to configure our karyogram.

```javascript
var kgram = idiogramamtik();
```

We could customize it, as in the example above, but this'll do for now.

Next, we'll want to bind the data (once loaded) to the element we'll want the
chart to occupy, in the style of d3. We can then call the kgram we initialized
to render it with that data in that element.

```javascript
idiogrammatik.load(function(err, data) {
  d3.select('#kgram')
      .datum(data)
      .call(kgram);
});
```

At this point, a nice and simple karyogram will be rendered. Now we can further
customize it and interact with it. For example, we can highlight chromosome one,
or zoom in on it.

```javascript
var h1 = kgram.highlight('chr1', 0, 'chr2', 0);
// And then remove the highlight:
h1.remove();
// And zoom in on chromosome one:
var start = kgram.positionFromRelative('chr1', 0).absoluteBp,
    end = kgram.positionFromRelative('chr2', 0).absoluteBp;
kgram.zoom(start, end);
```

### API

* idiogrammatik.**load**(*callback*)

Asynchronously loads the cytoband data [cytoband.tsv](cytoband.tsv) and calls `callback(error, data`. The data is cached (at `idiogrammatik.__data__`) so that subsequent calls to load are immediate.

* **idiogrammatik**()

Constructs a new idiogrammatik object. The below functions work on the resultant object, called `kgram`.

##### Configuration

Configuration must occur before the karyogram is called and drawn for the first
time.

* kgram.**width**([*width*])

If a width is provided, sets the width of the SVG to be rendered. Otherwise,
returns the width of the SVG. *Default is 800.*

* kgram.**height**([*height*])

If a height is provided, sets the height of the SVG to be rendered. Otherwise,
returns the height of the SVG. *Default is 100.*

* kgram.**margin**([*margin*])

If a margin is provided, sets the margin of the SVG to be rendered. Otherwise,
returns the margin of the SVG. *Default is {top: 50, bottom: 20, left: 20,
right: 20}.* Must have keys `top`, `bottom`, `right`, `left`.

* kgram.**stainer**([*stainer*])

Function which determines the colors of the
[Giemsa stained](http://en.wikipedia.org/wiki/Giemsa_stain) karyogram. Must
return valid SVG color strings for string values `gneg`, `gpos`, `acen`, `gvar`,
`stalk`.

* kgram.**idiogramHeight**([*height*])

If a height is provided, sets the height of the actual idiogram to be
rendered. Otherwise, returns the height. *Default is 7.*

* kgram.**highlightHeight**([*height*])

If a height is provided, sets the height of the highlights to be
rendered. Otherwise, returns the height of the highlights. *Default is 21.*

* kgram.**centromereRadius**([*radius*])

If a radius is provided, sets the radius of the centromere dots to be
rendered. Otherwise, returns the radius. *Default is 1.5.*

* kgram.**armClipRadius**([*radius*])

If a radius is provided, sets the radius of the pinched chromosome arms to be
rendered. Otherwise, returns the radius. *Default is 10.*

##### Hooks

* kgram.**svg**()

Return the d3 SVG selection the karyogram is rendered in.

* kgram.**scale**()

Return the d3
[linear scale](https://github.com/mbostock/d3/wiki/Quantitative-Scales#linear-scales)
mapping absolute base pair positions to X coordinates on the SVG plane.

* kgram.**zoom**([domain])

Pans and zooms the karyogram to display the domain (absolute base pairs).

* kgram.**highlight**(start, end, *options*)

Adds a highlight to the karyogram. `start` & `end` are either absolute base pair positions or position objects with keys `bp` and `chromosome`, with `bp` being the relative base position within the chromosome specified by `chromosome` (e.g. "chr1", "chr2", ..., "chrY").

`start` and `end` can also be a `position` object passed to event listener callbacks in `on` (below).

An alternative calling syntax is provided for convenience:

`kgram.highlight(chrNameStart, relativeBpStart, chrNameEnd, relativeBpEnd)`

`options` is an optional object with possible keys `color` and `opacity`, a SVG color string and a float 0-1 respectively. *Default is `{color: 'yellow', opacity: 0.2}`.*

This returns a highlight object which has a method `remove()` which removes the highlight from the karyogram.

* kgram.**on**(type, callback)

Registers event listeners on the karyogram.

Possible `type`s are "zoom", "zoomstart", "zoomend", "mousemove", "mousedown", "mousedown", "click".

`callback` is passed `position`, an object like the following:

```javascript
{ chromosome: {key: 'chr8', basePairs: 145138636,
               center: 1436550276, key: "chr8",
               end: 1536488912, start: 1391350276,
               pArm: {...}, qArm: {...}, bands: [...]},
  fmtAbsoluteBp: "1,500,000,000", fmtRelativeBp: "108,649,724",
  relativeBp: 108649724, absoluteBp: 1500000000 }
```

And `kgram`, a reference to the current kgram.

* kgram.**redraw**(*redrawFunction*)

If `redrawFunction` is passed, sets it to be called every time the karyogram is
redrawn. `redrawFunction` is called after all other redrawing is done, and is
passed the d3 `svg` selection and the current `xscale`. If not, forces a
redrawing of the karyogram.

##### Utility

* kgram.**positionFromAbsolute**(bp)

Returns a position object (as described above) from a given absolute base
position.

* kgram.**positionFromRelative**(chrName, bp)

Returns a position object (as described above) from a given relative base
position within a given chromosome (described by name, e.g. "chr22" or "chrX").

##### Highlights (Management)

* kgram.**highlights**()

Return an array of all highlight objects. They can be accessed individually, and
removed by calling `h.remove()` for a given highlight, `h`.

* kgram.highlights().**remove**()

Removes all highlights from the karyogram (by calling `.remove()` on all of
them).


### Examples & Recipes

##### Selecting ranges

It's easy enough to select and highlight ranges of the genome by extending the
kgram itself. The below code shows an example using the highlight API and
events. In this manner, ranges can be selected by shift-clicking the region
start and end-points.

```javascript
idiogrammatik.load(function(err, data) {
  var lastPos = null, selection = null, shifted = false;

  window.onkeydown = function(e) { if (e.shiftKey) shifted = true; };
  window.onkeyup = function(e) {
    if (shifted && !e.shiftKey) {
      shifted = false;
      lastPos = null;
    }
  };

  var kgram = idiogrammatik()
    .on('click', function(position, kgram) {
      if (!position.chromosome) return;

      if (shifted) {
        if (selection) {
          selection.remove();
          selection = null;
        }

        if (lastPos) {
          selection = kgram.highlight(lastPos, position);
          lastPos = null;
        } else {
          lastPos = position;
        }
      }
  });
});
```

In a similar manner, tooltips can be drawn on the karyogram (using the
'mouseover' event instead of the 'click' event, and appending SVG elements to
the `kgram.svg()` object).


##### Custom redraw functionality

If you wanted to add elements to the graph and have them update when the graph
redraws, you might do something like the below (which adds labels above each
chromosome).

```javascript
// svg is the d3 selection of the svg this karyogram belongs to.
// scale is the linear d3 scale mapping absolute base pairs to x
//     coordinates in the SVG.
kgram.redraw(function(svg, scale) {
  // Extract the actual chromosome data.
  var data = svg.selectAll('.chromosome').data();

  // Appends the elements once.
  svg.selectAll('.cname')
      .data(data, function(d) { return d.key; })
    .enter().append('text')
      .attr('class', 'cname')
      .attr('y', -9)
      .text(function(d) { return d.key; });

  // Places the text with the scale.
  svg.selectAll('.cname')
      .attr('x', function(d) { return scale(d.start); });
});
```

##### Extensive customization and event-driven hooks

The below code demonstrates some of the current functionality:

```javascript
idiogrammatik.load(function(err, data) {
  if (err) return console.log('error: ', err);

  var kgram = idiogrammatik()
      .width(1000)
      .height(85)
      .on('click', function(position) {
        console.log('clicked at ' + position.absoluteBp);
        if (position.chromosome) console.log(position.chromosome);
      })
      .on('mouseover', function(position) {
        console.log(position);
      })
      .on('drag', function(position) {
        console.log(position);
      })
      .on('zoom', function(position) {
        console.log(position);
      })
      .highlight('chrX', 0, 'chrY', 0)
      .highlightHeight(25)
      .centromereRadius(0) // removes the centromere dots.
      .idiogramHeight(11);

  d3.select('body')
      .datum(data)
      .call(kgram);


  // We can also add highlights after the idiogram has been displayed:
  kgram.highlight({chromosome: 'chr15', bp: 0},
                 {chromosome: 'chr17', bp: 1000000});

  var h = kgram.highlight(0,
                         2000000,
                         {color: 'red', opacity: 0.5}); // Absolute basepairs.

  // We can remove highlights wirh the return value of highlight() calls after the
  // graph is drawn, or can remove whichever highlights you want with
  // kgram.highlights()[n].remove()
  h.remove();
  kgram.highlights().remove(); // remove all highlights;

  // We can get the position information of a certain base pair like so:
  kgram.positionFromAbsoluteBp(1500000000);
  // { absoluteBp: 1500000000, chromosome: {key: 'chr8', start: ..., ...},
  //   fmtAbsoluteBp: "1,500,000,000",fmtRelativeBp: "108,649,724", relativeBp: 108649724 }

  // Or, similarly, from a relative position:
  kgram.positionFromRelativeBp('chr8', 108649724) // -> the same result as above

  // We can zoom to a particular position with kgram.zoom(abs1, abs2);
  // e.g.
  kgram.zoom(1400000000, 1650000000);
});
```