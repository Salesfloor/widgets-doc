widgets
=======

[![Build Status](https://magnum.travis-ci.com/Salesfloor/widgets.svg?token=eKsV996yzpvPdygcHk87&branch=dev)](https://magnum.travis-ci.com/Salesfloor/widgets)[![Code Climate](https://codeclimate.com/repos/54872a9c69568051d604d94f/badges/0eb3101dcc28ec6f4bb9/gpa.svg)](https://codeclimate.com/repos/54872a9c69568051d604d94f/feed)[![Test Coverage](https://codeclimate.com/repos/54872a9c69568051d604d94f/badges/0eb3101dcc28ec6f4bb9/coverage.svg)](https://codeclimate.com/repos/54872a9c69568051d604d94f/feed)


## Documentation
####  [Retailer Integration](http://salesfloor.github.io/widgets/)
####  Tests utilities - grunt jsdoc:tests (will be in tests/docs/index.html)

## Getting started 



### Install
npm install
grunt build
grunt build-configs
grunt replace --force --envr=dev


### Platform basis
The widget system is comprised of 2 sections, on one side you have widgets included on the retailer side, those are built without witout any framework, they are builts to be small & not interfer with the retailer website javascript code.

Widgets can also call iframes, iframes & widgets can speak to each other via a messaging system. 

Iframes code use backbone & are more sophisticated. There is 2 type of iframes, front iframes (sidebar, footer) & call to action iframes. While similar, front facing have a smaller foot print to help speed up loading time.

*Recap*
Widgets: no frameworks, plain javascript
Front facing iframes: Small stack, backbone.
Call to action iframes: Full stack, backbone, marionette, etc.


### Understanding widgets initialization

Widgets has an intricate instantiation system, here an overview of the execution order.

* Widgets.js is added on retailer website using document.write.
* Verify is ie8 & less, stop execution
* Setup messaging system with messages on window.
* Start Widget initialization.
* Wait for DOM ready. On cross domain we wait for the cookie iframe to be loaded & sent back cookies
* Load all widgets
* Each widgets check if it should be enabled
* Enable widgets


## Widgets

### Adding a new widget

Widgets use a name convention system, your widget name must stay consistant troughout the project.


Under fe/widgets/ create a new folder with your widget name. Then in this folder create a new file using the same name, **widget.[name].js**.

The **load** method will be called automatically.

#### Basic Example
```javascript
(function() {
    sf_widget.widgets.sidebar ={
      /**
       * Load  widget
       */
      load : function(){
        this.append();
      },
      append : function(options){
        var setting = options || {};
        sf_widget.utils.appendWidget({
          sf_store_widget : sf_widget.sf_store_widget,
          container : "body",
          content :  this.content.widget(setting),
          css : sf_widget.css.widget(),
          iframeId  : 'sf-widget-companion'
        });
      },
      content : {
        widget : function (options) {
              popup = '<div id="sf-widget-companion-wrapper" style="left:'+leftpos+';bottom:'+bottomPos+'px;position:fixed;z-index:99999999;display:block;vertical-align:middle;zoom:1;">' +
                    '<iframe id="sf-widget-companion" src="' + sf_widget.options.salesfloor_site + '/stores/'+sf_widget.sf_store_widget+'/widgets/sidebar?animate=&type='+media+animateInView+widgetDefautState+uniqueId+sfIp+'" width="'+sf_widget.options.widgetWidth+'" height="'+widgetDefaultHeight+'" style="border:none;overflow: hidden;display:block;visibility:hidden;"  scrolling="no" seamless allowTransparency="true" frameBorder="0"><!--iframecontent--></iframe></div>';
          return popup;
        }
      }
    };

    sf_widget.loadedWidgets = sf_widget.loadedWidgets + 1;
})();
```

### Adding loading exemption rules

Loading rules are handled by retailer under fe/configs/[retailer]/rules. Rules help you stop the executions fo your widgets based on different criterias. File name convention must again be followed, creating rules for a new widgets, you will need to create a file in each retailer folder following this pattern: **rules.[widgetname].js**

#### Example
```javascript
(function() {
  sf_widget_configs.saks.rules.footer =[
    {
      'type':'regex',
      'regex' : /checkout/,
      'testOn' : function(){return window.location.href;},
      'enableOn' : false
    },
    {
      'type':'regex',
      'regex' : /popup/,
      'testOn' : function(){return window.location.href;},
      'enableOn' : false
    }
  ];
})();
```

### Regex

Regex can be executed on any elements in the page or event the window ur.

#### Example
```javascript
{
  'type':'regex', 
  'regex' : /checkout/, // regex executed
  'testOn' : function(){return window.location.href;}, // element regex tested on
  'enableOn' : false // Should you enable widget on true or false?
}
```

### Function

A Function can be executed to determine if a widget should be loaded. Returning true will render the widget, returning false will disable it.

#### Example
```javascript
{
'type':'func',
'func' : function(){
	var lsCookie = sf_widget.utils.cookies.get("ls_intntl_siteid");
	var render = true;
	var SF_ID = "ukmIz9EpP84";
	
	if(lsCookie){
	  // example ukmIz9EpP84-q18gjPXUhQvZZmWKLAcz8g
	  // Seems like our id is ukmIz9EpP84
	  var lsCookieSections = lsCookie.split("-");
	  var linkshareId = lsCookieSections[0];

	  if(linkshareId === SF_ID){
	    render = true;
	  }else{
	    render = false;
	  }
	}
	return render;
}
```


### Preloading funtions

Preload functions are executed before dom.ready. It is particularly useful if you need to remove elements from the page & you don't want to have a glitchy behavior. Preload functions are unique per widgets, you cannot have one per retailer.

Preloading functions can be added in *fe/preload* following this file pattern : [widgetname].preload.js.

The **init** method will be called automatically.

#### Example
```javascript
sf_widget.preload.menu ={
  	/**
   	* Load before dom ready
   	* Append css to hide current main menu so you do not have a weird flicker
   	* Init is called automatically
   	*/
 	init : function(){
    	if(!sf_widget.rules.shouldEnable('menu', sf_widget.options.retailer)){ return; }

    	var css = sf_widget.options.menuContainer+"{visibility:hidden;}",
	        head = document.head || document.getElementsByTagName('head')[0],
	        style = document.createElement('style');

    	style.type = 'text/css';

    	if (style.styleSheet) {   
      		style.styleSheet.cssText = css;
    	}else{
      		style.appendChild(document.createTextNode(css)); 
    	}
    	head.appendChild(style);
 	}
};
```

### Calling Iframes with postMessage

After the iframe is loaded, you can call iframes backbone view methods like this:
```javascript
document.getElementById("iframeID").contentWindow.postMessage(JSON.stringify({
    "action" : "myBackboneViewMethos",
    "url" : $(e.currentTarget).attr("href")
}, '*');
```

## Iframes

Chances are you will want to add some UI to your widgets. No UI elements can be added directly in our retailers website. To add UI you will need to add an iframe.


First you must add your html & javascript for that iframes. 

### Javascript in Iframe

Under fe/templates you can add your javascript file, template.[widgetname].js. You will need to link your backbone view to your html document, this is done manually like this *var myFooterView = new FooterView({ el: $("#companionFooter") });*.

Iframe backbone views inherit form a Base view that takes care of connecting the iframe to the retailer website.

#### Example
```javascript
var FooterView = BaseView.extend({
    events:{
        "click .js-link-main" : "changeLocation"
    },
    name:"footer",
    // Executed after retailer parent script & iframe script is connected
    afterHandshake : function(){
        this.sendCookie();
    },
    initialize : function(){
        var self = this;
        // Call BaseVIew init, required
        this.constructor.__super__.initialize.apply(this);
    },
    changeLocation : function(e){
        e.preventDefault();
        this.parent && this.parent.postMessage(JSON.stringify({
            "widget" : "footer",
            "action" : "changeLocation",
            "url" : $(e.currentTarget).attr("href")
        }), this.origin);
    },
    setPresenceUI: function(enabled){
        this.$(".sf-icon-chat").toggleClass("sf-is-disabled", !enabled);
        this.$("span.availability").toggleClass("is-available", enabled);
    }
});
var myFooterView = new FooterView({ el: $("#companionFooter") });
```

### Adding your HTML

HTML can be added in the templates/widgets folder, base.[widgetname].html. We use the twig templating engine. You must add your own javascript & css yourself.

#### Example
```html
{% extends "base.html" %}

	{% block css %}
		<link rel="stylesheet" href="/css/menu.{{cacheBusterKey}}.css">
	{% endblock %}
	{% block content %}
		{% block contentSection %}
		This is some content
		{% endblock %}
	    <script src="/js/menuTemplate.min.{{cacheBusterKey}}.js"></script>
	{% endblock %}
```
### Calling parent script

You can call your parent widget easily using post messages:
```javascript
this.parent && this.parent.postMessage(JSON.stringify({
    "widget" : "footer",
    "action" : "changeLocation", // method you want to call
    "url" : $(e.currentTarget).attr("href")
}), this.origin);
```

### afterHandshake Method

If you need to post a message at your iframe rendering, you must use the method afterHandshake that will take care of waiting until you can properly send messages to the parent script.

```javascript
afterHandshake : function(){
    this.sendCookie();
},
```

### Messages in-between iframes

You can speak in-between iframes the same way you can call the parent script.

```javascript
this.parent && this.parent.postMessage(JSON.stringify({
    "widgetId" : "myIframeID",
    "action" : "changeLocation", // method you want to call in the backbone view
    "url" : $(e.currentTarget).attr("href")
}), this.origin);
```


## Cookies

Cookies are a complex subjet, the system use cookies extensively, throughout the app life cycle. When the core widget load itself, it actually add all the cookies into sf_widget.cookies.

In cross domain mode (like saks or babiesrus) when the core is loaded it will load the plugin cookie manager that in turm will send back cross domain cookies.

###  sf_widget.utils.cookies.get(cookieName)

Cookies are never fetch from document.cookies, there state is managed in sf_widget.cookies. 

###  sf_widget.utils.cookies.set(cookie, type)

When Setting a cookie you must pass a cookie object & a cookie type.

#### Cookie object example
```javascript
sf_widget.utils.cookies.set({
	name: 'sf_animate_in_view', 
	cvalue: "no", 
	exdays:900000
});
```

### Duration

The duration can be set to a number or with "session". Session cookies are a bit more clever than they appear. Following a set of rules they will disapear when you change tabs or close your browser.


### Posting cookies in iframe widget.

Never post your cookies in the iframe.
```javascript
var data = JSON.stringify({
    "action" : "set",
    "utils" : "cookies",
	name:"sf_wdt_sidebar_store", 
	cvalue:"test", 
	exdays:"session"
});
this.parent && this.parent.postMessage(data, this.origin);
```

## Tracking sales

The widget can be configured to track sales, we currently do that by scrapping the page, so when onboarding a new client will have to configure the scrapper. Be careful! Sometimes retailers have multiple confirmation pages. This is done by adding a tracking configuration object in **fe/config/[retailer]/tracking.[retailer].js**. Array name convention is: **sf_widget_configs.sw.tracking**

Following this  convention, once the file is included with the widget, the tracking config file will automatically be picked up. The tracking configuration is based on a series of objects that will dictate what and when we should track data.

### Options

### pageRegex (regex)

PageRegex tells on what page we should track this data, this is a regex applied on window.location.


### loadChecker (Function)

Function return true or false, used in combination of pageRegex to further qualify the page.

### trackEventsFlag (boolean)

If enabled, the tracker will only track that event once in a page load.

### trackEvents (Function)
Function where you can define events to track elements happening the the page life cycle. Executed on page load.


### type (customer | transaction | transaction-item)

Type of data tracked, will change the img url tracking pixel corresponding to a controller in the back-end.


### data (elements | map)

Data to track can be used 2 ways, a static object element, or a map that will loop over. The map is generally used to track a list of items in a transaction.


### Example 
```javascript
    sf_widget_configs.sw.tracking  =[
      {
        'pageRegex':/\/checkout/,
        'loadChecker' : function(){
            if(sf_widget.utils.cookies.get('sf_wdt_tracking') === "true" && document.getElementById("CT_Main_0_txtBillingFirstName")){
              return true;
            }else{
              return false;
            }
        },
        // only enable tracking once
        trackEventsFlag:true,
        // when loading the tracking it will execute this function
        trackEvents : function(tracker, trackItem){
          // With SW te only way to track customer info is to listen to changes in the checkout form
          document.getElementById("CT_Main_0_txtBillingFirstName").addEventListener("change", function(){
            tracker.pushEvent(trackItem);
          });
        },
        // Used to map data to api controller
        type:"customer",
        'data' : {
            elements : {
              customer_name : function(){
                return document.querySelector("#CT_Main_0_txtBillingFirstName").value + " " + document.querySelector("#CT_Main_0_txtBillingLastName").value;
              },
              customer_email : function(){
                return document.querySelector("#CT_Main_0_txtEmail").value;
              }
            }
        }
      },
      {
        'pageRegex':/\/checkout\/confirm/,
        // Look if we should gather data on this specidic entry
        'loadChecker' : function(){
            return sf_widget.utils.cookies.get('sf_wdt_tracking') === "true" ? true : false;
        },
        // Used to map data to api controller
        type:"transaction",
        'data' : {
            elements : {
              trx_id : ".XpadS15 > .floatLeft > .formRow + .formRow .colWrap .floatLeft + .floatLeft",
              customer_id : function(){ 
                return sf_widget.utils.cookies.get('sf_wdt_customer_id') || "" ; 
              }
            }
        }
      },
      {
          'pageRegex':/\/checkout\/confirm/,
          // Look if we should gather data on this specidic entry
          'loadChecker' : function(){
              return sf_widget.utils.cookies.get('sf_wdt_tracking') === "true" ? true : false;
          },
          // Used to map data to api controller
          type:"transaction-item",
          'data' : {
            map : {
              on: ".cart tr",
              item : "td",  
              itemLength : 9,           
              data : {
                trx_detail_total : ".Total p",
                product_id : function(row){
                  var brand = sf_widget.utils.trim(row.querySelector('.item .name').innerText);
                  var color = row.querySelector('.color').innerHTML;
                  return "MD5("+brand.toUpperCase()+" - "+color.toUpperCase()+")";
                }
              }
            }
          }
        }
    ];
```
