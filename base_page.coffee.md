# Page base

This module contains the common elements for all pages.


## Library imports

	metah = require 'metah'

	R = require 'ramda'

	{append, apply, concat, curryN, flip, pipe, prepend, prop} = R


## Relative imports

	package_json = require './package.json'


## HTML head section

h -> DOM

	get_head = (h)->
		meta = metah h

		h 'head', [
			meta.charset()

			meta.author package_json.author
			meta.description package_json.description
			meta.http_equiv()
			meta.viewport()
		]


## Top-level functions

These functions are used to create the highest-level parts of the HTML file.


### Add doctype

Text -> Text

	add_doctype = concat '<!doctype html>'


### Convert DOM to text

DOM -> Text

	get_html_text = prop 'outerHTML'


### Make html tag

[DOM] -> Array

Prepare the arguments used to create the HTML tag.

	wrap_in_html = flip(append)(['html', {lang: 'en'}])


## Exports

h -> DOM -> Text

h
: require('hyperscript')

DOM
: DOM content of the page.

	module.exports = curryN 2, (h, content)->
		add_head = prepend(get_head(h))

		make_page_html = pipe(R.of, add_head, wrap_in_html, apply(h))

		pipe(make_page_html, get_html_text, add_doctype)(content)
