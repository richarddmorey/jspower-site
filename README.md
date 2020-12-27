
<h1 align="center">jspower-site</h1>

* [jekyll-vue-template](https://github.com/Splode/jekyll-vue-template)
* [jspower](https://github.com/richarddmorey/jspower), which uses
* [libRmath.js](https://github.com/R-js/libRmath.js)

This project is in a very preliminary state. In particular, I have no idea what I'm doing when it comes to [Vue.js](https://vuejs.org/). This is an experiment!

See ["Power and Precision"](https://medium.com/@richarddmorey/power-and-precision-47f644ddea5e?sk=20a7fd66048b5ead68050b791ebbf79b) for a statistical explanation of the ideas behind this power analysis interface.

For now the only test/design supported is the independent samples t test, as a proof of concept.

If you want to compile this project, grab the repository and follow the directions for [jekyll-vue-template](https://github.com/Splode/jekyll-vue-template). 

## Components

### Power curve (chart)


The power chart is the main interface to the power analysis. It is designed with specific statistical affordances in mind. In particular, it is designed to highlight that:

* power is a curve, not a single value
* the "null" (0 difference) isn't the only thing we can test (ie, equivalence testing needn't be thought of separately)
* there are several interesting points along the curve, not just one
* confidence/"precision" is tightly linked with the concept of power

#### How the interface works

Handles: 

* Bottom (white) handle: Moves up and down to manipulate the Type I error rate of the test (α, or "size")
* Middle (red) handle: Moves in any direction. Use to manipulate the effect size and power simultaneously (see also "Lock curve" under settings)
* Top (white) handle: Moves left and right to manipulate the effect size that attains 1-α power, or&mdash;equivalently&mdash;the width of the CI for a just-significant result ("precision")
* Gray/shaded region: "region of unreliability", or the true effect sizes that do not produce consistently significant or nonsignificant results (at level α).

### Text

A text explanation and alternative interface to specify values. Any grey rectangles indicate values that can be modified.

### R code

R code to run if you'd like to reproduce/verify the calculations.

### Distributions (chart)

To be added; will be typical (but responsive) representations of the null and various alternative distributions, along with the criteria.

### Power contour (chart)

To be added. See here for an example: [tweet with image](https://twitter.com/richarddmorey/status/1292019113481601024)


## Options

### Global settings

* **fix es**: By default, as N is changed the effect size is changed to keep the power constant. Selecting this option will keep the effect size constant and change the power instead.
* **fix n2**: By default, the ratio of `n1` and `n2` is fixed as `n1` is increased. If this option is selected, `n2` is fixed as `n1` is increased. A reference power curve is shown to indicate the limit as n1 approaches infinity.
* **Add new analysis**: You can have two analyses saved at one time, and flip back and forth (useful for comparison). Eventually both power curves will be shown at the same time. For now, the analyses have random names.


### Chart settings

* **digits**: How many digits to round to in the chart overlay
* **Type S, two-tailed**: If selected, add additional curves showing the Type S error rate (assuming an α-level criterion on the other tail) and the sum of the power and Type S error rate, which is the two-tailed power for the 2α test.
* **Lock curve w/ red handle**: If this is set, moving the red handle will not change the design, but instead moves the red handle *along* the curve
* **Reset handle**: If you lose your red handle, click this to reset if to the effect size with 50% power
* **Reset axes**: If you need to reset the axes (e.g. when you move from one analysis to another) click this

### Text settings

* **Show text?**: Show/hide the text explanation of the analysis? 
* **digits**: How many digits to round to in the chart overlay

### R code settings

* **Show R code?**: Show/hide the R code to verify the power analysis?

