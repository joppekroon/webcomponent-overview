# WebComponent Overview

<!-- vscode-markdown-toc -->
* [Custom Elements](#CustomElements)
	* [Constructor](#Constructor)
	* [connectedCallback](#connectedCallback)
	* [disconnectedCallback](#disconnectedCallback)
	* [adoptedCallback](#adoptedCallback)
	* [attributeChangedCallback](#attributeChangedCallback)
	* [observedAttributes](#observedAttributes)
	* [:defined pseudo-class](#definedpseudo-class)
* [Custom built-in elements (currently unsupported)](#Custombuilt-inelementscurrentlyunsupported)
* [Shadow DOM](#ShadowDOM)
	* [Slotting](#Slotting)
	* [Styling](#Styling)
		* [From Light to Shadow](#FromLighttoShadow)
		* [From  Shadow to Light](#FromShadowtoLight)
* [HTML Import](#HTMLImport)
	* [Example](#Example)
	* [Problems with HTML imports](#ProblemswithHTMLimports)
* [HTML Template](#HTMLTemplate)
* [Polyfills](#Polyfills)
* [Browser support](#Browsersupport)
* [Resources](#Resources)

<!-- vscode-markdown-toc-config
	numbering=false
	autoSave=false
	/vscode-markdown-toc-config -->
<!-- /vscode-markdown-toc -->



## <a name='CustomElements'></a>Custom Elements
* A custom element, a.k.a an *autonomous custom element* , should be defined with an ES6 class extending `HTMLElement`.
* The name of a custom element *must* contain a dash (-) character, this is to prevent clashes with future built-in elements.
* A custom element must be registered with the `CustomElementRegistry`.

	```javascript
	class XElement extends HTMLElement {
		static get observedAttributes() {/* use only when necessary */}
		
		constructor() {
			super();
		}
		
		connectedCallback() {/* use only when necessary */}
		disconnectedCallback() {/* use only when necessary */}
		adoptedCallback() {/* use only when necessary */}
		attributeChangedCallback() {/* use only when necessary */}
	}
	// an autonomous custom element
	window.customElements.define("x-element", XElement);
	```
* An autonomous custom element applies the defined behaviour to a custom tag.

	```html
	<x-element></x-element>
	```

### <a name='Constructor'></a>Constructor
* The constructor is run when the element is 'upgraded', i.e. it is both registered with the `CustomElementRegistry` and added to the DOM.
* A parameter-less call to `super()` must be the first statement.
* The constructor should be used only for setting up initial state and default values, event listeners and possibly a shadow root.
* Work should be deferred to the `connectedCallback` as much as possible. 

### <a name='connectedCallback'></a>connectedCallback
Called when the element is registered and inserted in the DOM. This is the appropriate place to do actual (time-consuming) work like fetching resources or rendering. 

Note that this callback can be called more than once.

### <a name='disconnectedCallback'></a>disconnectedCallback
Called when the element is detached from the DOM. Intended for cleanup purposes.

### <a name='adoptedCallback'></a>adoptedCallback
Called when the element is adopted into a new document through the `document.adoptNode` method.

### <a name='attributeChangedCallback'></a>attributeChangedCallback
Called whenever any of the attributes from the `observedAttributes` changes. 

```javascript
class FlagIcon extends HTMLElement {
	...
	attributeChangedCallback(name, oldValue, newValue) { 
		if ("country".equals(name)) {
			// 'this' refers to the element, so is fairly safe to use 
			this._countryCode = newValue;
		}
	}
	...
}
```

### <a name='observedAttributes'></a>observedAttributes
An array, defined on the class, of the attribute names for which the `attributeChangedCallback` should be called when they change.

```javascript
class FlagIcon extends HTMLElement {
	...
	static get observedAttributes() { return ["country"]; }
	...
}
```

### <a name='definedpseudo-class'></a>:defined pseudo-class
The :defined pseudo-class is applicable to each element that is known to the user agent, i.e. either it is a built-in element or the constructor of the custom element was successfully run.

This for example allows for hiding the non-functional custom elements in CSS if the element definitions for the non-essential elements are being downloaded asynchronously to improve the initial load.

Note that the :defined pseudo class can't be polyfilled, so it will only work for user agents that implement the custom element v1 specifications natively.

## <a name='Custombuilt-inelementscurrentlyunsupported'></a>Custom built-in elements (currently unsupported)
* An element extending a built in element is also known as a *customized built-in element*.
* It is intended to work exactly like an autonomous custom element, however inheriting all behavior that the element that is extended would have.
* In stead of extending `HTMLElement`, to create a customized built-in element it must extend a specific subclass like `HTMLButtonElement`, and supply an object specifying what element it extends as multiple elements use the same interface classes.

```javascript
	class XButton extends HTMLButtonElement {
		...
	}
	
	window.customElements.define("x-button", XButton, { extends: "button" })
```
* In stead of a specific tag, the base element tag is used and enhanced with the `is` property set to the name of the customized built-in element.
```html
	<button is="x-button">Click me</button>	
```

## <a name='ShadowDOM'></a>Shadow DOM
* The shadow DOM defines a scope within which styles are applicable, together with the custom element spec, this allows for webcomponents with self contained HTML, CSS, and JS.
*  A component's DOM is self-contained (e.g. document.querySelector() won't return nodes in the component's shadow DOM), Thus, simple id's and classes are enough for identification because they won't clash with the containing document.
* A shadow tree can be attached to all elements that do not already host their own shadow DOM (`<texarea>`, `input`),  or where it does not make sense (`<img>`).

```javascript
const header = document.createElement('header');
const shadowRoot = header.attachShadow({mode: 'open'});
const caption = document.createElement('h1');

caption.appendChild(document.createTextNode('Hello Shadow DOM'));
shadowRoot.appendChild(caption)
```

* A shadow tree can have a style element and even a stylesheet. These styles are local to the shadow DOM and will not bleed into the document.

### <a name='Slotting'></a>Slotting
* The markup within an element is called 'Light DOM'.
* By default, when a shadow tree is attached to an element, it's 'Light DOM' is ignored.
* If the shadow tree defines slots, the 'Light DOM' is distributed into those slots, to create a 'Flattened DOM tree'.
* Slots can have a name and default content. The name is used to identify a specific slot for light DOM content, if there is no light DOM for a slot, the default content  is used. 
* The slot with no name is the default slot, and all light DOM that doesn't specify a slot is moved there. Note: Technically you can have multiple unnamed slots, but only the first will then be populated.

```html
	<div>
		<!-- this is the light DOM -->
		<span>foo</span> 
		<span slot="bar">bar</span>
		<span slot="nonexistent">qux</span>
		<span>quux</span>
	</div>
```
 
```html
	 <!-- Visual representation of the shadow DOM -->
	<h1>
		<slot>Header here</slot>
	</h1>
	<slot name="bar"></slot>
	<p>
		<slot name="unused">No light dom for me</slot>
	</p>
```

```html
	<!-- Result, a.k.a. the flattened DOM tree -->
	<!-- 
	 * The 'foo' and 'quux' span are moved to the default slot, replacing the 
	   default content
	 * The 'bar' span is moved to the 'bar' slot
	 * The 'qux' span is ignored as it's specified slot does not exist
	 * The 'unused' slot shows the default content as no Light DOM was moved there
	--> 
	<div>
		<h1>
			<slot>
				<span>foo</span>
				<span>quux</span>
			</slot>
		</h1>
		<slot name="bar">
			<span slot="bar">bar</span>
		</slot>
		<p>
			<slot name="unused">No light dom for me</slot>
		</p>
	</div>
```

### <a name='Styling'></a>Styling
* CSS selectors from the outer page don't apply inside the component, and vice versa. This allows for the use of simpler class names and id's as they won't conflict, which also has a positive effect on performance.
* Styles can be defined within the shadow DOM for the elements from the light DOM, however the styles from the surrounding document take precedence.

#### <a name='FromLighttoShadow'></a>From Light to Shadow
* The shadow DOM can exert limited influence on the host element (the element to which the shadow tree is attached) and the light DOM that is distributed into it's slots. However, in all cases, styling applied in the surrounding document will take precedence.
* The `:host` selector selects the host element
* The `:host(<selector>)` selector selects the host element only if the given `<selector>` matches the host.
* The `:host-context(<compound-selector>)` selects the host element only if the given `<compound-selector>` matches the host or any of it's ancestors. This, for example, allows for theming across the page by toggling a class on the body element. (Although Custom CSS properties may be preferable here)
* The `::slotted(<compound-selector>)` pseudo selector selects any _top level_ element from the light DOM that matches the `<compound-selector>` and is distributed into any of the shadow DOM's slots. 

#### <a name='FromShadowtoLight'></a>From  Shadow to Light
* The surrounding document can not influence the styling of the shadow DOM, unless the shadow DOM supplies styling hooks through CSS custom properties.
* CSS custom properties are required to have two dashes before the name (--*), and can have any value;
* They can then be picked up using `var(--foo)`, or `var(--foo, <fallback-value>)`

```html
	<style>
		div {
			/* fallback if custom properties are not supported (IE11) */
			color: green
			--header-color: green;
		}
	</style>
	<div id="div-host"></div> <!-- green -->
	<span id="span-host"></span> <!-- red -->

	<!-- Visual representation of the Shadow DOM -->
	<style>
		h1 {
			color: var(--header-color, red);
		}
	</style>
	<h1>Hello World</h1>
```

## <a name='HTMLImport'></a>HTML Import
* HTML imports allow fragments of HTML to be moved into another document. It does not need to be a full HTML page (with a `<head>`, `<body>`, etc) but may contain anything and all that is allowed within HTML, like markup content, script, styling and of course HTML imports of their own.
* The mimetype of the import will be `text/html`
* You can define an HTML import declaring a `<link rel="import">`
 
```html
	<head>
		<!-- external resources need to be CORS enabled -->
		<link rel="import" href="/path/to/imports/stuff.html">
	</head>
```

* Or you can create one with javascript.

```javascript
	var link = document.createElement('link');
	link.rel = 'import';
	link.href = 'file.html';
	document.head.appendChild(link);
```

* An HTML import will not just add its content to the document, it will make it available for use later.

```javascript
	var content = document.querySelector('link[rel="import"]').import;
```

* Styling will be applied, and script will be executed when it is encountered.
* Imports are de-duplicated automatically, meaning that multiple declarations of an import from the same url will only trigger retrieval and parsing once, thus the scripts will only execute the first time. 
* By default, the retrieval of an import is blocking when downloading, just like stylesheets, (although the parsing of the document may continue) preserving execution order of the scripts and to prevent a FOUC (Flash of unstyled content) because an import may contain styling.
* The retrieval can be done asynchronously by setting the `async` property, in this case (although it can also be done on sync retrievals) it may be useful to also provide an `onload` and  possibly an `onerror` callback.

```html
	<script>
		/* The callbacks need to be known before they are used */	
		function handleLoad(e) {
			console.log('Loaded import: ' + e.target.href);
		}
		function handleError(e) {
			console.log('Error loading import: ' + e.target.href);
		}
	</script>

	<link rel="import" href="file.html" async 
		onload="handleLoad(event)" onerror="handleError(event)">
```

* Script in the import is executed from the context of the window that contains the importing document, thus functions defined in the import are added to `window` for everyone to use and `document` refers to the main document.
* The 'document' of the import itself can be retrieved through `document.currentScript.ownerDocument`. However, `currentScript` is only available when the script is initially being processed. So if you need the `ownerDocument` for code executing within a callback or event handler you'll have to capture it for later use.
* The [HTML import polyfill][polyfill_repo] exposes the 'document' through `document._currentScript.ownerDocument` (note the underscore). 
It will add a `HTMLImports.useNative` property to be able to detect which one should be used.  

### <a name='Example'></a>Example

main.html
```html 
	<!DOCTYPE html>
	<html>
		<head>
			<title> Import test </title>
			<link rel="import" href="/theimport.html">
		</head>
		<body>
			<h1>Hello world</h1>
			<script>
				var content = document.querySelector('link[rel="import"]').import;
				var imported = content.querySelector('h1');
				
				/* This will copy the header from the import */
				document.body.appendChild(imported.cloneNode(true));
				console.log(!!content.querySelector('h1')); /* true */
				
				/* This will yank the header out of the import */
				document.body.appendChild(imported);
				console.log(!!content.querySelector('h1')); /* false */
			</script>
		</body>
	</html>
```

theimport.html
```html
	<link rel="stylesheet" href="importsheet.css" type="text/css"> 
	<style>
		h1 {
			/* this is applied to both headers */
			color: blue;
		}
	</style>
	<h1>Hello imported world</h1>
	<script>
		/* document refers to the main document */
		var headers = document.querySelectorAll('h1');
		
		/* 
		 * This will only change the header in the main document, 
		 * as the header defined here will not be imported yet 
		 * due to the execution order
		 */
		headers.forEach((h) => {
			h.style.fontFamily = "fantasy";
		});
		
		/*
		 * This will specifically select the header in this document
		 * and apply the styling before it is imported
		 */
		var thisHeader = document.currentScript.ownerDocument.querySelector('h1');
		thisHeader.style.border = "1px dashed";
		thisHeader.style.display = "inline";
	</script>
```

importsheet.css
```css
	h1 {
		/* this is applied to both headers */
		font-style: italic; 
	}
```



### <a name='ProblemswithHTMLimports'></a>Problems with HTML imports
* De-duplication allows for a form of dependency management, i.e. including jquery multiple times will only result in jquery being downloaded once. However, this [only works when the url is the exact same][html_imports_dep_man].
* Due to the upcoming ES6 modules implemtation across browsers which is expected to influence the direction the HTML import specification will take to adress the concerns with dependency management, [Mozilla has decided **not** to implement the current draft specification][html_imports_mozilla]. Their reasons are explained in an email discussion [by Anne van Kesteren][html_imports_AvK] [and Boris Zbarsky][html_imports_BZ]. However, due to the existence of a polyfill, this is not seen as an impediment for webcomponents.


## <a name='HTMLTemplate'></a>HTML Template
* HTML template is the first of the WebComponent specs to be implemented by all browser vendors, only IE11 requires a polyfill.
* A HTML template is indicated with the `<template>` tag, and can be placed virtually anywhere.
* The content of the `<template>` tag is put into a different html document fragment by the parser and as such is completely inert. This means script won't run, markup won't render, and styles won't apply. It does not even have to be valid HTML.
* The `content` property of a template contains the template document fragment and can be imported or cloned. Adding the clone to the DOM will activate the contents of that template, and at that time it has to be valid HTML.

```javascript
	const clone = template.content.cloneNode(true);
	const clone2 = document.importNode(template.content, true); 
```

* As the content of the template lives in a different document fragment, it will not show up when selecting nodes. You need to specifically go through the template's content property to select it.

```html
	<h1>I'm in the main document</h1>
	<template>
		<h1>I'm in a template</h1>
	</template>

	<script>
		const template = document.querySelector('template');
		
		console.log(document.querySelectorAll('h1').length); // 1
		console.log(template.content.querySelectorAll('h1').length); // 1
		
		document.body.appendChild(template.content.cloneNode(true));
		
		console.log(document.querySelectorAll('h1').length); // 2
	</script>
```

## <a name='Polyfills'></a>Polyfills
* All modern browsers have at least indicated to be working towards implementing the 4 specifications that comprise webcomponents, in the mean time there are polyfills available to patch the gaps at [https://github.com/WebComponents/webcomponentsjs][polyfill_repo]. 
* The polyfills are applied asynchronously, so to be sure everything is ready you should move any logic that requires the polyfills to be in place into the custom WebComponentsReady event. If you are working with lower level polyfills you may have to wait for the DOMContentLoaded event.

```javasript
	window.addEventListener('WebComponentsReady', function(e) {
	  // imports are loaded and elements have been registered
	  console.log('Components are ready');
	});
```
* There are several methods available to make sure (only) the necessary polyfills are included. 
	* `webcomponentsjs/webcomponents-loader.js` will feature detect which polyfills are necessary and download only those for the cost of one extra request.
	* `webcomponentsjs/webcomponents-lite.js` contains all the polyfills, using only those that are actually necessary, and as such will work in all cases but may be a heavier download than necessary.
	* Have the server serve the exact set necessary for the browser making the request. This is the option with the minimal download and requests but will require the server to be capable of serving different assets based on user agent, which may be more hassle than it is worth.
*  The specification of Custom Elements requires ES6 support, but not all browsers support this (IE11). WebComponents can be transpiled to ES5 and provide the same functionality to those browsers, however if, for simplicity, the transpiled sources are used for all browsers it is necessary to include `/webcomponentsjs/custom-elements-es5-adapter.js`. This is a shim for browsers that support Custom Elements natively to make sure things won't break for them. 
* Note that the polyfills and the shim don't need to be, and should not be transpiled.

## <a name='Browsersupport'></a>Browser support

* :white_check_mark:: natively supported
* :large_orange_diamond:: supported with polyfill
* :x:: not supported

<table>
	<tr>
		<td></td>
		<td colspan="6">Desktop</td>
		<td colspan="3">Mobile</td>
	<tr>
	<tr>
		<td></td>
		<td>Chrome</td>
		<td>FireFox</td>
		<td>Edge</td>
		<td>IE11</td>
		<td>Safari</td>
		<td>Opera</td>
		<td>IOS Safari</td>
		<td>Android Browser</td>
		<td>Chrome for Android</td>
	</tr>
	<tr>
		<td>Custom Elements (v1)</td>
		<td>:white_check_mark:</td>
		<td>:large_orange_diamond:</td>
		<td>:large_orange_diamond:</td>
		<td>:large_orange_diamond:</td>
		<td>:white_check_mark:</td>
		<td>:white_check_mark:</td>
		<td>:white_check_mark:</td>
		<td>:white_check_mark:</td>
		<td>:white_check_mark:</td>
	</tr>
	<tr>
		<td>Shadow DOM</td>
		<td>:white_check_mark:</td>
		<td>:large_orange_diamond:</td>
		<td>:large_orange_diamond:</td>
		<td>:large_orange_diamond:</td>
		<td>:white_check_mark:</td>
		<td>:white_check_mark:</td>
		<td>:white_check_mark:</td>
		<td>:white_check_mark:</td>
		<td>:white_check_mark:</td>
	</tr>
	<tr>
		<td>HTML Import</td>
		<td>:white_check_mark:</td>
		<td>:large_orange_diamond:</td>
		<td>:large_orange_diamond:</td>
		<td>:large_orange_diamond:</td>
		<td>:large_orange_diamond:</td>
		<td>:white_check_mark:</td>
		<td>:large_orange_diamond:</td>
		<td>:white_check_mark:</td>
		<td>:white_check_mark:</td>
	</tr>
	<tr>
		<td>HTML Template</td>
		<td>:white_check_mark:</td>
		<td>:white_check_mark:</td>
		<td>:white_check_mark:</td>
		<td>:large_orange_diamond:</td>
		<td>:white_check_mark:</td>
		<td>:white_check_mark:</td>
		<td>:white_check_mark:</td>
		<td>:white_check_mark:</td>
		<td>:white_check_mark:</td>
	</tr>
	<tr>
		<td>CSS custom properties</td>
		<td>:white_check_mark:</td>
		<td>:white_check_mark:</td>
		<td>:white_check_mark:</td>
		<td>:x:</td>
		<td>:white_check_mark:</td>
		<td>:white_check_mark:</td>
		<td>:white_check_mark:</td>
		<td>:white_check_mark:</td>
		<td>:white_check_mark:</td>
	</tr>
</table>

## <a name='Resources'></a>Resources
1. [Custom Element Specification][custom_element_specification]
1. [Shadow DOM v1: Self-Contained Web Components][shadow_dom]
1. [Open vs Closed Shadow DOM][open_vs_closed]
1. [WebComponents polyfill repository][polyfill_repo]
1. [HTML Imports #include for the web][html_imports]
1. [The Problem With Using HTML Imports For Dependency Management][html_imports_dep_man] 
1. [Mozilla and Web Components: Update][html_imports_mozilla]
1. [Re: HTML imports in Firefox (Anne van Kesteren)][html_imports_AvK]
1. [Re: HTML imports in Firefox (Boris Zbarsky)][html_imports_BZ] 
1. [Polymer: Billions Served; Lessons Learned (Google I/O '17)][polymer_lessons_learned]
1. [Browser support overview][browser_support]

[custom_element_specification]:
	https://w3c.github.io/webcomponents/spec/custom/#custom-element-conformance
	"Custom Element Specification"
[shadow_dom]:
	https://developers.google.com/web/fundamentals/getting-started/primers/shadowdom
	"Shadow DOM v1: Self-Contained Web Components"
[open_vs_closed]:
	https://blog.revillweb.com/open-vs-closed-shadow-dom-9f3d7427d1af
	"Open vs Closed Shadow DOM"
[polyfill_repo]:
	https://github.com/webcomponents/webcomponentsjs
	"WebComponents polyfill repository"
[html_imports]:
	https://www.html5rocks.com/en/tutorials/webcomponents/imports/
	"HTML Imports #include for the web"
[html_imports_dep_man]:
	https://www.tjvantoll.com/2014/08/12/the-problem-with-using-html-imports-for-dependency-management/
	"The Problem With Using HTML Imports For Dependency Management"
[html_imports_mozilla]:
	https://hacks.mozilla.org/2014/12/mozilla-and-web-components/
	"Mozilla and Web Components: Update"
[html_imports_AvK]:
	http://lists.w3.org/Archives/Public/public-webapps/2014OctDec/0616.html
	"Re: HTML imports in Firefox (Anne van Kesteren)"
[html_imports_BZ]:
	http://lists.w3.org/Archives/Public/public-webapps/2014OctDec/0614.html
	"Re: HTML imports in Firefox (Boris Zbarsky)"
[polymer_lessons_learned]:
	https://youtu.be/assSM3rlvZ8
	"Polymer: Billions Served; Lessons Learned (Google I/O '17)"
[browser_support]:
	https://www.polymer-project.org/2.0/docs/browsers
	"Browser support overview"
