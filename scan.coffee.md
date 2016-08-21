# Scan page

## Core imports

	url = require 'url'


## Library imports

	R = require 'ramda'

	{always, ap, append, apply, applySpec, curry, equals, flatten, flip, ifElse, join, juxt, nth, pipe, prepend, prop, T, tap} = R


## Relative imports

	get_url_from_request = require './get_url_from_request'

	make_page = require './base_page'

	send_error = require './send_error'

	send_ok = require './send_ok'


## Make body

h -> Object state -> Text

Create the HTML body section.
This contains a search form and a list of Redis keys from the scan.

	get_body = curry (h, state)->
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

			h 'ul#search_results', [
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
					reject err
				else
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
		.then send_ok.json
