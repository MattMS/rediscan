# Scan page

## Core imports

	url = require 'url'


## Library imports

	metah = require 'metah'

	R = require 'ramda'

	{always, ap, append, apply, applySpec, curry, equals, flatten, flip, ifElse, join, juxt, nth, pipe, prepend, prop, T, tap} = R


## Relative imports

	get_url_from_request = require './get_url_from_request'

	make_page = require './base_page'

	send_error = require './send_error'

	send_ok = require './send_ok'


## Body script

	body_script = ->
		check_for_call = (cursor, ul)->
			if 0 != cursor and ul.lastChild.getBoundingClientRect().bottom < window.innerHeight
				call_scan_json cursor, handle_scan_json


### Perform JSON request

		call_scan_json = (cursor, callback)->
			request = new XMLHttpRequest()

			request.onreadystatechange = (event)->
				req = event.target

				if XMLHttpRequest.DONE == req.readyState
					if 200 != req.status
						console.error(req)

					else
						handle_scan_json JSON.parse req.responseText

The `true` parameter indicates that this request is asynchronous.

- https://developer.mozilla.org/en-US/docs/AJAX/Getting_Started

			request.open('GET', "/scan.json?count=12&cursor=#{cursor}", true)
			request.send()


### Handle JSON response

		handle_scan_json = (json_response)->
			[cursor, values] = json_response

			ul = document.getElementById 'search_results'

			for key in values
				li = document.createElement 'li'

				a = document.createElement 'a'
				a.setAttribute 'href', "/get/#{key}"
				a.innerHTML = key

				li.appendChild a
				ul.appendChild li

			ul.setAttribute 'data-last-cursor', cursor

			check_for_call cursor, ul


### Add event listeners

		check_for_call_from_event = ->
			ul = document.getElementById 'search_results'

			cursor = ul.getAttribute 'data-last-cursor'

			check_for_call cursor, ul

		document.addEventListener 'DOMContentLoaded', (event)->
			search_nav = document.getElementById 'search_nav'

- https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/style

			search_nav.style.display = 'none'

			check_for_call_from_event()

- https://developer.mozilla.org/en-US/docs/Web/Events/scroll

		window.addEventListener 'scroll', (event)->
			check_for_call_from_event()


## Make body

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

				h 'label#search_input_wrap', [
					h 'span', 'Pattern'

					h 'input#search_input',
						name: 'pattern'
						type: 'text'
						value: state.query.pattern
				]

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


## Parse URL to Redis command

	get_count_from_url_query = pipe(prop('count'), flip(R.or)(12))

	get_cursor_from_url_query = pipe(prop('cursor'), flip(R.or)(0), parseInt)

	# add_pattern = ifElse(equals(''), always([]), flip(append)(['MATCH']))
	get_pattern_from_url_query = pipe(prop('pattern'), flip(R.or)('*'))

	get_command_from_request = pipe(get_url_from_request.query, juxt([
		get_cursor_from_url_query,
		pipe(get_pattern_from_url_query, flip(append)(['MATCH'])),
		pipe(get_count_from_url_query, flip(append)(['COUNT']))
	]), flatten)


## Redis SCAN command

	redis_scan_from_request = (log, client, request)->
		new Promise (resolve, reject)->
			command = get_command_from_request(request)

			log.trace("Redis command is `SCAN #{join(' ', command)}`.")

			client.scan command, (err, result)->
				if err
					log.error(err: err, 'Redis SCAN failed.')

					reject err

				else
					log.trace
						redis_scan_cursor: result[0]
						redis_scan_values: result[1]
					, 'Redis SCAN completed.'

					resolve result


## Make scan page

h -> Object state -> Text

h
: require('hyperscript')

state
: Properties are content used to render the page.

Returns the entire HTML text for the Redis SCAN page.
Suitable for sending to a client or writing to a file.

	# scan_html_maker = pipe(ap([get_body, make_page]), apply(pipe))
	scan_html_maker = (h)->
		pipe(get_body(h), make_page(h))


## Exports

### HTML

	module.exports.html = curry (h, log, redis_client, request, response)->
		log.trace req: request, 'Started scan.html handler.'

		fix_scan_result = applySpec
			keys: nth 1
			next_cursor: nth 0
			query: always get_url_from_request.query request

		make_html = scan_html_maker h

		# scan_is_done = tap log_trace 'Redis SCAN done; building page.'

		make_page_from_scan_result = pipe(fix_scan_result, make_html, send_ok.html(response))

		redis_scan_from_request log, redis_client, request
		.catch send_error.html_500('Redis SCAN failed.', log, request, response)
		.then make_page_from_scan_result


### JSON

	module.exports.json = curry (log, redis_client, request, response)->
		log.trace req: request, 'Started scan.json handler.'

		redis_scan_from_request log, redis_client, request
		.catch send_error.json_500('Redis SCAN failed.', log, request, response)
		.then send_ok.json(response)
