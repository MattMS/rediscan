# Redis browser HTTP server

## Core imports

	url = require 'url'


## Library imports

	h = require 'hyperscript'

	{always, append, apply, bind, curry, drop, equals, flatten, flip, ifElse, join, juxt, pipe, prepend, prop, slice, T, tap} = require 'ramda'


## Relative imports

	base_page_maker = require('./base_page')(h)

	get_url_from_request = require './get_url_from_request'

	handle_scan = require('./scan')

	send_error = require('./send_error')

	send_ok = require('./send_ok')


## Redis GET handler

	wrap_code = pipe(flip(append)(['pre']), apply(h))


	handle_get_html = curry (log, redis_client, request, response)->
		log.trace req: request, 'Started /get/ handler.'

		key = pipe(get_url_from_request.path, drop(5))(request)

		send_result = pipe(
			# NOTE: This fails because it receives `result` as a parameter.
			# tap(log_trace('Redis GET done; building page.')),
			wrap_code,
			base_page_maker,
			send_ok.html(response)
		)

		redis_get redis_client, key
		.catch send_error.html_500('Redis GET failed.', log, request, response)
		.then send_result


	redis_get = (client, key)->
		new Promise (resolve, reject)->
			client.get key, (err, result)->
				if err
					reject err
				else
					resolve result


## Exports

	handle_scan_html = handle_scan.html(h)

	handle_scan_json = handle_scan.json

	module.exports = curry (log, redis_client, request, response)->
		log.trace req: request, 'Received request.'

		# pipe(get_url_from_request.path, cond([
		# 	[propEq('pathname', '/'), handle_scan_html(log, redis_client)],
		# 	[pipe(prop('pathname'), slice(0, 5), equals('/get/')), handle_get_html(log, redis_client)],
		# 	[propEq('pathname', '/scan.json'), handle_scan_json(log, redis_client)],
		# 	[T, handle_missing_page]
		# ]))(request, response)

		url_path = get_url_from_request.path(request)

		if '/' == url_path
			handle_scan_html log, redis_client, request, response

		# else if '/index.js' == pathname
		# 	handle_ok_js request, response

		else if '/get/' == url_path.slice 0, 5
			handle_get_html log, redis_client, request, response

		else if '/scan.json' == url_path
			handle_scan_json log, redis_client, request, response

		else
			send_error.text_404 request, response
