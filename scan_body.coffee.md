# SCAN page body

## Imports

### Core imports

	url = require 'url'


### Library imports

	metah = require 'metah'

	{curry, pipe} = require 'ramda'


### Relative imports

	body_script = require './scan_script'

	make_page = require './base_page'


## Get body DOM

h -> Object state -> Text

Create the HTML body section.
This contains a search form and a list of Redis keys from the scan.

	get_body = curry (h, state)->
		meta = metah h

		h 'body', [
			h 'form', [
				h 'input',
					name: 'count'
					type: 'hidden'
					value: state.query.count

				h 'input',
					name: 'cursor'
					type: 'hidden'
					value: 0

				h 'label#search_input_wrap.text', [
					h 'span', 'Pattern'

					h 'input#search_input',
						name: 'pattern'
						type: 'text'
						value: state.query.pattern
				]

				# h 'label.checkbox', [
				# 	h 'input',
				# 		checked: 'checked'
				# 		name: 'wrap_in_stars'
				# 		type: 'checkbox'
				# 		value: 'true'
				#
				# 	h 'span', 'Wrap in *'
				# ]

				h 'button#search_button', 'Search',
					type: 'submit'
			]

NOTE: Setting attribute to `0` (non-string) will not add the attribute.

			h 'ul#search_results',
				'data-last-cursor': String(state.next_cursor) or '0'
			, [
				for key in state.keys
					h 'li', [
						h 'a', href: "/get/#{key}", key
					]
			]

			h 'nav#search_nav', [
				h 'a',
					href: url.format
						query:
							count: state.query.count
							cursor: state.next_cursor
							pattern: state.query.pattern
				, 'Next'
			]

			meta.script body_script
		]


## Exports

h -> Object state -> Text

h
: require('hyperscript')

state
: Properties are content used to render the page.

Returns the entire HTML text for the Redis SCAN page.
Suitable for sending to a client or writing to a file.

	# module.exports = pipe(ap([get_body, make_page]), apply(pipe))
	module.exports = curry (h)->
		pipe(get_body(h), make_page(h))
