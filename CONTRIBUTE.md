# Guidelines for Contributing

## General Guidelines

* Pull requests are welcome.
* Please follow these style guides: https://github.com/styleguide

## HTML Guidelines

* Use input placeholder and a matching label with class ui-hidden-accessible: http://view.jquerymobile.com/master/demos/forms-label-hidden-accessible/

## JavaScript Guidelines

There is some sharing of JavaScript between the rails and phonegap apps. The
phonegap app loads JQuery, JQueryMobile, a phonegap index.js that does
initialization and myplaceonline.js which is shared between the two apps
(these are equivalent to application.js.coffee). After phonegap loads the
homepage, it will dynamically load the rest of the javascript files
(these are equivalent to application_extra.js.coffee). The JavaScript files
myplaceonline.js should have all code that is required offline and everything
else should go into myplaceonline_final.js or page-specific JS files.
When updating myplaceonline.js, update the version at the top of the file and
update the file in both apps and do rebuilds.

## Team members

* Kevin Grigorenko: kevin@myplaceonline.com
